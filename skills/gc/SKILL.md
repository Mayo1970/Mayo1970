---
name: gc
description: Nintendo GameCube homebrew development expertise — Gekko CPU, Flipper/GX graphics, devkitPPC + libogc2 toolchain, DOL builds, memory/cache rules, AI/DSP audio, SD Gecko/Swiss/PicoBoot deployment, and Dolphin-vs-real-hardware pitfalls. Use this skill whenever the user mentions GameCube, GC, NGC, libogc, libogc2, devkitPPC, GX, TEV, .dol files, Swiss, SD Gecko, SD2SP2, PicoBoot, GC Loader, ARAM, Gekko/Flipper, or porting any game or engine to GameCube — even if they never say "skill" or "homebrew". In active development sessions, this skill also enforces a strict goal-oriented workflow with hardware validation gates (Section 14).
---

# Nintendo GameCube Homebrew — SKILLS.md

You are assisting with **Nintendo GameCube** homebrew development. Follow every rule in this document. It overrides your generic PC/mobile development instincts.

**Confidence tags** — every technical claim carries one:
- `[HIGH]` = multiple verified sources / working public code / official docs
- `[MEDIUM]` = likely correct, limited recent verification, or inferred from similar hardware
- `[LOW]` = sparse or conflicting sources — verify before relying on it (a verification test is suggested)

## OPERATING RULES (read first)

- You must PREFER the open-source community toolchain (devkitPPC + libogc2). You must NOT reference, link, install, or emit build flags for the leaked official Nintendo Dolphin SDK, CodeWarrior, or MusyX. All code examples must compile with the community toolchain. If a hardware feature is only documented in leaked material, mark it `[LOW]` and propose a hardware test instead of citing the leak.
- Do NOT give generic PC/mobile advice unless explicitly marked cross-platform.
- Do NOT hallucinate APIs or hardware features. The GameCube has **no OpenGL, no shaders, no virtual memory paging, no little-endian mode**. Say so and give the platform alternative.
- Do NOT omit "obvious" details that are platform-specific here: cache flushing, big-endianness, and 32-byte alignment are correctness requirements, not optimizations.
- If you circle on a topic or hit contradictory information, STOP and output a **DEBUG BUILD PROTOCOL**: a minimal C/asm test case that definitively resolves the uncertainty on real hardware or Dolphin.
- In an active development session, you MUST obey the state machine in **Section 14**. Never bypass the WAITING_FOR_HARDWARE state.

---

## 1. HARDWARE ARCHITECTURE

### CPU — IBM "Gekko"
- PowerPC 750CXe derivative, **485 MHz**, 32-bit, **big-endian**, PowerPC EABI. [HIGH]
- Caches: **32 KB L1 instruction + 32 KB L1 data** (8-way, **32-byte lines**), **256 KB on-die L2**. [HIGH]
- Data cache is **write-back** — CPU writes are NOT visible to the GPU/DMA engines until you flush (see Section 5). [HIGH]
- FPU: full 64-bit IEEE, plus Gekko's unique **paired singles** SIMD: each FPR holds 2×float32; quantized load/store via 8 GQR registers converts packed u8/s8/u16/s16 ↔ float in one instruction. Used heavily for vertex math. [HIGH] Compiler support for paired singles in devkitPPC GCC is partial/version-dependent — libogc2's `gu`/`ps` math functions use hand-written asm; prefer those or inline asm rather than assuming autovectorization. [MEDIUM]
- 16 KB of the D-cache can be **locked** as scratchpad with its own DMA engine (locked-cache DMA). libogc exposes `LCEnable()`/`LCLoadBlocks()`/`LCStoreBlocks()`. [MEDIUM]
- **Write-gather pipe** at `0xCC008000`: non-cached burst-write port the GX FIFO uses; `GX_Position3f32()` etc. write through it. [HIGH]
- Front-side bus: 64-bit @ 162 MHz to Flipper, ~1.3 GB/s. [HIGH]
- MMU exists (BAT registers) but libogc maps memory flat; **no demand paging, no swap**. [HIGH]

### GPU — ArtX/ATI "Flipper"
- 162 MHz, **fixed-function** rasterizer — there are **no programmable shaders**. Per-pixel programmability comes from the **TEV (Texture EnVironment)**: up to **16 combiner stages**, **8 simultaneous textures**, 8 hardware lights, indirect texturing (EMBM-style). [HIGH]
- **Embedded framebuffer (eFB)**: ~2 MB of on-die 1T-SRAM. Max size **640×528** pixels. Pixel formats: `GX_PF_RGB8_Z24`, `GX_PF_RGBA6_Z24` (required for destination alpha and AA), `GX_PF_RGB565_Z16`. [HIGH]
- You never scan out the eFB directly. You **copy** it (`GX_CopyDisp`) to an **XFB (external framebuffer)** in main RAM; the copy converts to **YUV 4:2:2** and can apply vertical scaling/deflicker. The VI (Video Interface) scans out the XFB. [HIGH]
- **1 MB TMEM** texture cache; textures live in main RAM and are cached into TMEM by the hardware. [HIGH]
- Textures: max **1024×1024**; formats `I4, I8, IA4, IA8, RGB565, RGB5A3, RGBA8, CMPR (4bpp S3TC-like), CI4, CI8, CI14X2`; all textures are **tiled/swizzled in blocks** (not linear) and must be **32-byte aligned**. `RGBA8` is stored as two interleaved AR/GB cache-line groups — never treat it as linear RGBA. [HIGH]
- Depth: 24-bit Z with compression and early-Z; alpha compare; standard blend equations + logic ops. Destination alpha only exists in `RGBA6_Z24`. [HIGH]
- Anti-aliasing: eFB-based multisampling exists but halves usable eFB height (max ~640×264 with AA) — rarely worth it. [MEDIUM]

### RAM
- **24 MB "Splash" 1T-SRAM** main memory — low latency (~10 ns class), ~2.6 GB/s peak. [HIGH size / MEDIUM exact bandwidth]
- **16 MB ARAM** (auxiliary DRAM on the DSP side, ~81 MHz). The CPU **cannot address ARAM directly** — access is DMA-only (`AR_StartDMA`, or queued via ARQ). Use it for audio samples and asset staging. [HIGH]
- No memory bank speed traps beyond that: all of MEM1 is uniform; ARAM is simply slow + DMA-only. [HIGH]
- Alignment: **32 bytes** for anything touched by DMA, GX, DVD, or SD drivers. [HIGH]

### Address map (physical 0x00000000–0x017FFFFF mirrored)
| Region | Address | Meaning |
|---|---|---|
| Cached MEM1 | `0x80000000–0x817FFFFF` | Normal code/data (write-back cached) [HIGH] |
| Uncached MEM1 | `0xC0000000–0xC17FFFFF` | Same RAM, uncached — XFBs, MMIO-adjacent buffers [HIGH] |
| MMIO base | `0xCC000000` | CP `+0x0000`, PE `+0x1000`, VI `+0x2000`, PI `+0x3000`, MI `+0x4000`, DSP/AR-DMA `+0x5000`, DI `+0x6000`, SI `+0x6400`, EXI `+0x6800`, AI `+0x6C00`, GX FIFO write-gather `+0x8000` [HIGH — YAGCD] |
| Low-mem OS globals | `0x80000000–0x800000FF` | e.g. memory size @`0x80000028`, console type @`0x8000002C`, arena lo/hi @`0x80000030/34`, bus/CPU clock @`0x800000F8/FC` [MEDIUM] |

### Bus topology
Flipper is the hub: it contains the memory controller, GPU, DSP, and all I/O. CPU↔Flipper over the FSB; peripherals hang off **EXI** (SPI-like, 3 channels: memory cards, SD Gecko, BBA, IPL ROM/RTC, USB Gecko) and **SI** (4 controller ports, JoyBus protocol). Dedicated DMA engines: GX command FIFO, ARAM DMA, DSP mailbox+DMA, DI (disc) DMA, EXI DMA, AI DMA, VI scanout. The CPU should orchestrate DMA, not memcpy bulk data. [HIGH]

### Co-processor — Audio DSP
Custom 16-bit DSP @ 81 MHz executing **ucode** (microcode) uploaded at runtime; communicates via mailbox registers + DMA; mixes voices and decodes GameCube 4-bit ADPCM. Official AX/MusyX ucodes are off-limits; homebrew uses the free mixer ucodes shipped with libogc/libogc2 (ASND/AESND, and the newer `libansnd` for libogc2). [HIGH concept / MEDIUM library specifics]

### Security / DRM
- Boot: masked IPL ROM (BS1/BS2, on an EXI-attached MX chip that also holds the font and SRAM). Disc "authentication" is the proprietary miniDVD format + drive firmware handshake — **no per-title cryptographic signatures, no hypervisor**. Once your code runs, homebrew has **full hardware access**. [HIGH overall / MEDIUM on IPL internals]
- Homebrew entry points: **PicoBoot** (RP2040 modchip that injects an IPL payload and boots `/ipl.dol` from SD) [HIGH], **GC Loader** (optical drive emulator), XenoGC drive chip, Datel SD Media Launcher / Action Replay (`autoexec.dol`), Swiss as the universal launcher, and per-game savegame exploits [MEDIUM on the exploit list].
- What homebrew cannot practically touch: reflashing the IPL ROM (it's mask ROM; PicoBoot bypasses rather than rewrites) and the DVD drive's internal firmware without drive-specific tools. [MEDIUM]

---

## 2. OFFICIAL VS HOMEBREW SDK

- **Official**: "Nintendo Dolphin SDK" + CodeWarrior compiler, MetroTRK debugger, MusyX audio middleware. Leaked, proprietary — **you must not use or reference it**. [HIGH]
- **Homebrew**: **devkitPPC** (GCC-based `powerpc-eabi` cross toolchain from devkitPro) + **libogc2** (Extrems' actively maintained fork, the library Swiss and GCMM build against). [HIGH]
- **Toolchain history you must know**: in May 2025 the original **libogc** was publicly accused of containing code derived from RTEMS and Nintendo's SDK; the Homebrew Channel repo was archived and the ecosystem fractured. As of 2026, **libogc2** (`github.com/extremscorner/libogc2`) is the recommended, actively maintained option; it is API-compatible with libogc ≤2.1.0 and installable via devkitPro pacman (`pacman -S libogc2 libogc2-examples`). Makefiles switch from `$(DEVKITPPC)/gamecube_rules` to `$(DEVKITPRO)/libogc2/gamecube_rules`. [HIGH — verify current package state before advising installs, this situation is still evolving]
- Companion packages: `gamecube-tools` (elf2dol, gxtexconv), `libogc2-libdvm` (preferred FAT+exFAT filesystem layer, drop-in for libfat), `opengx` (partial OpenGL-1.x-style shim over GX for ports), SDL port, GRRLIB (2D/3D convenience), `libansnd` (newer audio mixer). [HIGH — from libogc2 README]

**What the homebrew SDK CAN do**: full GX graphics, VI video modes incl. 480p, PAD/SI input, EXI devices, memory cards (CARD API), ARAM, DSP audio via free ucodes, LWP threads, timers/IRQs, BBA Ethernet networking (BSD-ish `net_*` sockets), FAT/exFAT on SD Gecko / SD2SP2 / GC Loader / IDE-EXI, DVD reads on unlocked drives. [HIGH/MEDIUM mix]

**What it CANNOT do / critical gaps**:
- No official AX/MusyX feature set — the free mixer ucodes are simpler (fewer voice effects, no official 3D audio pipeline). [MEDIUM]
- No CodeWarrior-specific pragmas/intrinsics; paired-single autovectorization is limited (use libogc2 `gu*` math or asm). [MEDIUM]
- Reading **retail discs** from homebrew requires an unlocked drive (modchip/GC Loader); a stock console's drive won't serve arbitrary reads to homebrew. [MEDIUM]
- No official profiler/debugger — use USB Gecko + the libogc gdb stub (Section 9). [HIGH]

---

## 3. GRAPHICS PIPELINE

**API: GX** (libogc2's `gx.h`). There is no OpenGL/Vulkan/DirectX. `opengx` exists only as a porting shim — write native GX for anything performance-sensitive. [HIGH]

### Initialization (canonical bring-up — memorize this order)
```c
#include <gccore.h>
#include <malloc.h>
#include <string.h>

static void *xfb[2]; static u32 fb = 0;
static GXRModeObj *rmode;

int main(void) {
    VIDEO_Init();
    rmode = VIDEO_GetPreferredMode(NULL);              // picks NTSC/PAL/480p correctly
    xfb[0] = MEM_K0_TO_K1(SYS_AllocateFramebuffer(rmode));  // XFB must be used UNCACHED
    xfb[1] = MEM_K0_TO_K1(SYS_AllocateFramebuffer(rmode));
    VIDEO_Configure(rmode);
    VIDEO_SetNextFramebuffer(xfb[0]);
    VIDEO_SetBlack(FALSE);
    VIDEO_Flush();
    VIDEO_WaitVSync();
    if (rmode->viTVMode & VI_NON_INTERLACE) VIDEO_WaitVSync();

    void *fifo = memalign(32, 256*1024);               // GX FIFO: 32B aligned, >=256 KB
    memset(fifo, 0, 256*1024);
    GX_Init(fifo, 256*1024);
    GX_SetCopyClear((GXColor){0,0,0,0xFF}, GX_MAX_Z24);
    GX_SetViewport(0, 0, rmode->fbWidth, rmode->efbHeight, 0, 1);
    GX_SetDispCopyYScale((f32)rmode->xfbHeight / (f32)rmode->efbHeight);
    GX_SetScissor(0, 0, rmode->fbWidth, rmode->efbHeight);
    GX_SetDispCopySrc(0, 0, rmode->fbWidth, rmode->efbHeight);
    GX_SetDispCopyDst(rmode->fbWidth, rmode->xfbHeight);
    GX_SetCopyFilter(rmode->aa, rmode->sample_pattern, GX_TRUE, rmode->vfilter);
    GX_SetPixelFmt(rmode->aa ? GX_PF_RGB565_Z16 : GX_PF_RGB8_Z24, GX_ZC_LINEAR);
    // ... vertex formats, TEV, matrices ...
    while (1) {
        // draw with GX_Begin/GX_End or display lists
        GX_DrawDone();                                 // wait for GPU
        GX_CopyDisp(xfb[fb], GX_TRUE);                 // eFB -> XFB, clear eFB
        VIDEO_SetNextFramebuffer(xfb[fb]);
        VIDEO_Flush();
        VIDEO_WaitVSync();
        fb ^= 1;
    }
}
```
[HIGH — this is the standard libogc/libogc2 example pattern]

### Framebuffer flow
Draw into the **eFB** → `GX_CopyDisp()` to one of **two XFBs** in main RAM (double buffering) → flip via `VIDEO_SetNextFramebuffer` + `VIDEO_Flush` on VSync. The eFB itself is single — it is your only render target (render-to-texture = `GX_CopyTex` from eFB). [HIGH]

### Video modes / refresh
- **NTSC**: 480i @ 59.94 Hz, 240p supported. **PAL**: 576i @ 50 Hz (taller XFB!), 288p; **PAL60/EURGB60** = 480i @ 60 Hz on PAL consoles. **480p progressive** requires the component/digital cable — detect with `VIDEO_HaveComponentCable()`. Always start from `VIDEO_GetPreferredMode(NULL)`; never hardcode. [HIGH]

### Textures
- Convert offline with **gxtexconv** (gamecube-tools) to TPL, or generate tiled data at build time. Load with `TPL_OpenTPLFromMemory`/`GX_InitTexObj` → `GX_LoadTexObj(&obj, GX_TEXMAP0)`. [HIGH]
- After ANY CPU write to texture memory: `DCFlushRange(data, size);` then `GX_InvalidateTexAll();`. Skipping this works in Dolphin and shows garbage on hardware. [HIGH]
- Max 1024×1024, 32-byte aligned, tiled formats only (Section 1). Use `CMPR` (4bpp) for most color textures, `RGB5A3` when you need cheap alpha, `RGBA8` sparingly (it's 2 cache lines per 4×4 block). Mipmaps supported and cheap — use them. [HIGH]

### TEV in one paragraph
Set stage count `GX_SetNumTevStages(n)`; per stage route inputs `GX_SetTevOrder(stage, texcoord, texmap, color)` and configure the combiner `GX_SetTevOp(stage, GX_MODULATE / GX_REPLACE / ...)` or the raw `GX_SetTevColorIn/ColorOp` forms. Think of TEV as 16 chained `(a,b,c,d) → a*(1-c)+b*c+d` register combiners — you can build multitexture, specular, cel-shading, and EMBM this way, but there is **no arbitrary shader code**. [HIGH]

### Anti-patterns (GPU)
- Do NOT call GL/EGL anything. Do NOT expect shaders. [HIGH]
- Do NOT read the eFB per-pixel from the CPU in a loop (PE access is slow); use `GX_CopyTex` and read the copy. [MEDIUM]
- Do NOT write vertex/texture data and draw without `DCFlushRange`. [HIGH]
- Do NOT exceed 640×528 eFB or forget PAL's 576-line XFB. [HIGH]
- Do NOT assume destination alpha exists unless you set `GX_PF_RGBA6_Z24`. [HIGH]

---

## 4. INPUT

### Hardware
- 4 **SI** ports (JoyBus). Devices: standard GameCube controller (analog stick, C-stick, analog L/R triggers **with** digital click, A/B/X/Y/Z/Start, D-pad, rumble), **WaveBird** wireless (identical but **no rumble motor**), DK Bongos, dance mats, ASCII keyboard controller, GBA via link cable (GBA-as-controller / JoyBoot), Logitech Speed Force wheel. [HIGH list / MEDIUM exotic details]
- Memory-card-slot peripherals (microphone, modem, BBA) are **EXI**, not SI. [HIGH]

### Model: polled, once per frame
```c
PAD_Init();                       // once
// per frame:
u32 connected = PAD_ScanPads();   // scans all 4 channels
u32 down = PAD_ButtonsDown(PAD_CHAN0);   // newly pressed this frame
u32 held = PAD_ButtonsHeld(PAD_CHAN0);
s8  sx = PAD_StickX(PAD_CHAN0),  sy = PAD_StickY(PAD_CHAN0);      // main stick
s8  cx = PAD_SubStickX(PAD_CHAN0), cy = PAD_SubStickY(PAD_CHAN0); // C-stick
u8  lt = PAD_TriggerL(PAD_CHAN0), rt = PAD_TriggerR(PAD_CHAN0);   // analog triggers
if (down & PAD_BUTTON_START) { /* ... */ }
PAD_ControlMotor(PAD_CHAN0, PAD_MOTOR_RUMBLE);  // or PAD_MOTOR_STOP / PAD_MOTOR_STOP_HARD
```
[HIGH API]

- Stick values are signed 8-bit relative to a **power-on origin**; real-world usable range is roughly ±72…±100 and asymmetric per unit. Apply a dead zone (~±15) and clamp; never assume ±127. Triggers are u8 with a resting offset (~20–30) and max ~230. [MEDIUM ranges — calibrate]
- Connect/disconnect: check the bitmask from `PAD_ScanPads()` / per-channel error state each frame; iterate **all 4 channels** — players plug into any port. Holding X+Y+Start ~3 s re-origins a controller (hardware feature). [MEDIUM]
- Rumble is a simple on/off motor; there is no force curve. Never gate gameplay logic on rumble (WaveBird users get nothing). [HIGH]

### Anti-patterns (input)
- Do NOT read sticks before calling `PAD_ScanPads()` that frame. [HIGH]
- Do NOT assume port 1 only, or that a "controller" is a standard pad (bongos report as a pad with weird mappings). [MEDIUM]
- Do NOT treat trigger analog and trigger click (`PAD_TRIGGER_L/R` button bits) as the same signal. [HIGH]

---

## 5. MEMORY LAYOUT

### The map you actually use
| What | Where | Notes |
|---|---|---|
| Code + data + heap | `0x80003100` upward | crt0 loads DOL sections here; newlib `malloc` draws from the **arena** [HIGH] |
| Arena | `SYS_GetArenaLo()` … `SYS_GetArenaHi()` | Everything between loaded image and top-of-RAM reservations [HIGH] |
| XFBs | allocated from arena, accessed via `MEM_K0_TO_K1()` uncached | ~600 KB each at 480i [HIGH] |
| GX FIFO | arena, `memalign(32, …)` | ≥256 KB [HIGH] |
| Stack | set by crt0 near the top of MEM1; default size is small (order of 128 KB) | [LOW — verify: print `SYS_GetArenaHi()` and `&stack_var`; to be safe, give worker code its own `LWP_CreateThread` stack] |
| ARAM (16 MB) | DMA only | `AR_Init()` + `AR_Alloc`/`AR_StartDMA`, or queued `ARQ_PostRequest` [MEDIUM exact API shape] |

- Allocate DMA-visible buffers with `memalign(32, size)` and round sizes up to 32. Static buffers: `static u8 buf[SZ] ATTRIBUTE_ALIGN(32);`. [HIGH]
- Uncached mirror (`0xC…` / `MEM_K0_TO_K1`) is for XFBs and occasional DMA result peeking — never run hot loops over uncached memory. [HIGH]

### Cache discipline (this is where hardware bugs live)
- D-cache: **write-back**, 32-byte lines. I-cache separate. No bus snooping for GX/DMA reads of CPU-written data. [HIGH]
- `DCFlushRange(p, n)` — write back **and** invalidate. Use after the CPU writes any buffer that GX/DSP/DVD/EXI/ARAM-DMA will read. [HIGH]
- `DCStoreRange(p, n)` — write back only (keeps lines valid) — fine for one-way CPU→GPU data. [HIGH]
- `DCInvalidateRange(p, n)` — discard lines. Use **before** the CPU reads a buffer that DMA just wrote (disc reads, ARAM→MEM1, EXI). Beware: invalidating a partially-dirty line loses data — keep DMA buffers exclusively DMA-owned and 32-byte aligned so lines are never shared. [HIGH]
- `ICInvalidateRange(p, n)` after writing code (loaders, self-modifying) and after `DCFlushRange` of the same range. [HIGH]

### Anti-patterns (memory)
- Do NOT `malloc` (unaligned) for anything DMA touches. [HIGH]
- Do NOT put the GX FIFO, XFB, or DMA buffers in ARAM — it is not CPU/GPU addressable. [HIGH]
- Do NOT assume you have 24 MB free: subtract ~1.2 MB XFBs + 256 KB FIFO + code + stack before budgeting assets. [HIGH]
- Do NOT share a 32-byte cache line between CPU-owned and DMA-owned data. [HIGH]

---

## 6. AUDIO

### Hardware
- **AI (Audio Interface)**: DMA-streams **16-bit big-endian, interleaved stereo PCM** from main RAM to the DAC at **48 kHz or 32 kHz** (`AI_SAMPLERATE_48KHZ` / `AI_SAMPLERATE_32KHZ`). This is the only path to the speakers. [HIGH]
- **DSP** (Section 1) mixes voices and decodes 4-bit GameCube ADPCM under a ucode; homebrew ucodes ship with the library. [HIGH]

### Recommended homebrew APIs
1. **Raw AI double-buffering** (maximum control, what emulators/ports use):
```c
static u8 abuf[2][4096] ATTRIBUTE_ALIGN(32);   // 32B aligned, len % 32 == 0
static int cur = 0;
static void dma_cb(void) {                     // IRQ context: swap only!
    AUDIO_StopDMA();
    AUDIO_InitDMA((u32)abuf[cur], sizeof(abuf[0]));
    AUDIO_StartDMA();
    cur ^= 1;                                  // signal mixer thread to refill abuf[cur]
}
AUDIO_Init(NULL);
AUDIO_SetDSPSampleRate(AI_SAMPLERATE_48KHZ);
AUDIO_RegisterDMACallback(dma_cb);
/* pre-fill abuf[0], DCFlushRange it */ dma_cb();
```
2. **ASND / AESND** (voice mixing, sample-rate conversion) — easiest for sound effects + music. On libogc2, also consider **libansnd**. [MEDIUM — API names stable, feature details vary by fork]
- Formats in practice: s16 BE PCM (native), 8-bit via mixers, ADPCM via DSP, MP3 via `ppc-libmad`, OGG via tremor ports. [MEDIUM]
- ARAM is the traditional home for sample banks (official AX streamed voices from ARAM); homebrew mixers usually mix straight from MEM1 — put big/idle sample data in ARAM and DMA it in as needed. [MEDIUM]

### Avoiding glitches
- Buffers: ≥2 × 4096 bytes (~10.7 ms each @48 kHz stereo). Smaller invites underruns during disc/SD reads. [MEDIUM]
- `DCFlushRange` every buffer **after filling, before the DMA reads it**. [HIGH]
- The DMA callback runs at interrupt level: no `malloc`, no decoding, no file I/O — flip pointers/flags only; decode in a dedicated LWP thread. [HIGH]
- Keep audio thread priority above rendering; a skipped frame is invisible, a skipped audio buffer is a pop. [MEDIUM]

### Anti-patterns (audio)
- Do NOT feed little-endian PCM (your WAV files are LE — byteswap in the asset pipeline). [HIGH]
- Do NOT do heavy math or I/O in the audio callback. [HIGH]
- Do NOT assume 44.1 kHz exists natively — resample assets to 48 k or 32 k offline. [HIGH]

---

## 7. STORAGE / IO

### Media homebrew can use
| Media | Bus | Notes |
|---|---|---|
| SD Gecko (memcard slot A/B) | EXI | SPI-mode SD; the workhorse. `__io_gcsda` / `__io_gcsdb`. EXI clocks at 16 MHz here. [HIGH] |
| SD2SP2 (serial port 2, under console) | EXI | `__io_gcsd2`. EXI2 clocks at 32 MHz — roughly 2× a card-slot SD Gecko, so it is the adapter to prefer for anything streaming. **`fatInitDefault()` cannot mount it — see below.** [HIGH] |
| GC Loader | DI replacement | ODE; also exposes SD to homebrew via libogc2 drivers. [MEDIUM] |
| IDE-EXI / M.2 adapters | EXI | HDD/SSD. [MEDIUM] |
| Memory cards | EXI | `CARD_*` API, 8 KB sectors, proprietary FS. [HIGH] |
| Retail miniDVD | DI | Only with unlocked drive (modchip/ODE); `DVD_Read` needs 32B-aligned buffer, length %32, offset %4. [MEDIUM] |
| BBA (Broadband Adapter) | EXI | 10/100 Ethernet; rare hardware. [HIGH] |

### Filesystem

**`fatInitDefault()` does NOT mean "mount whatever is plugged in". On GameCube it mounts exactly two devices.** [HIGH — verified 2026-08-19 against devkitPro `libogc/lib/cube/libfat.a`: the only device names in the archive are `carda` and `cardb`]

| Device root | Interface | Mounted by |
|---|---|---|
| `carda:/` | `__io_gcsda` (SD Gecko, memory card slot A) | `fatInitDefault()` |
| `cardb:/` | `__io_gcsdb` (SD Gecko, memory card slot B) | `fatInitDefault()` |
| *(nothing)* | `__io_gcsd2` (**SD2SP2, serial port 2**) | **nobody — you must do it yourself** |

libfat's GameCube device table has **no entry for `__io_gcsd2`**. A port that only calls `fatInitDefault()` will never see an SD2SP2, which is the fastest and most commonly recommended adapter. Mount it by hand:

```c
#include <fat.h>
#include <sdcard/gcsd.h>          // __io_gcsda / __io_gcsdb / __io_gcsd2

// Both, not either.  fatInitDefault() brings up carda:/cardb: and is also what
// makes a Swiss argv[0] of "carda:/..." resolvable for path rebasing; the
// explicit mount is the only way to reach an SD2SP2.  A console with just one
// of the two adapters is the normal case, so only fail when nothing mounted.
bool haveCards  = fatInitDefault();
bool haveSd2sp2 = fatMountSimple("sd2sp2", &__io_gcsd2);
if(!haveCards && !haveSd2sp2)
    fail("no storage");
```
The name passed to `fatMountSimple` is yours to choose and becomes the devoptab root (`sd2sp2:/…`). **Pick one name and keep it across sibling ports on the same machine** so one SD card layout serves all of them. [HIGH]

- IDE-EXI / GC Loader SD likewise need their own driver + explicit mount; do not assume a default root exists. [MEDIUM]
- **Robust pattern**: take the launch path from `argv[0]` (Swiss passes a full `device:/path/app.dol`) and rebase all asset paths onto that device instead of hardcoding `carda:/`. Note this only works if that device is actually mounted — which is a second reason to call `fatInitDefault()` even when you also mount by hand. [MEDIUM]
- Prefer **libdvm** (`libogc2-libdvm`) over classic libfat: drop-in, adds exFAT. FAT is case-insensitive/preserving — never rely on case. Format cards FAT32, MBR. [HIGH]
- Standard C `fopen/fread` work once mounted (devoptab). Reads into `memalign(32,…)` buffers are fastest. [HIGH]

### Loader conventions (no Wii-style `/apps` standard exists on GC)
| Loader | Boots |
|---|---|
| PicoBoot | `/ipl.dol` at SD root [HIGH] |
| Swiss | any `.dol`/`.elf` you browse to; supports per-game `.cli`/`.ini` argument files [HIGH] |
| Datel SD Media Launcher / SDLOAD | `/autoexec.dol` [MEDIUM] |
Ship: `yourapp.dol` (+ optional icon/banner only if targeting Swiss's file browser niceties). [MEDIUM]

### Memory cards (saves)
`CARD_Init("GAME","00")` → `CARD_Mount(slot, workarea, detach_cb)` (workarea = `CARD_WORKAREA` bytes, 32B aligned) → `CARD_Open/Create/Read/Write` in sector multiples → `CARD_Unmount`. Always unmount; always handle `CARD_ERROR_NOCARD`. Writes wear flash — save on demand, never per-frame. [MEDIUM exact constants / HIGH discipline]

### Network (BBA only)
libogc's GC net stack: `if_config(localip, netmask, gateway, TRUE, 20)` then BSD-style `net_socket/net_bind/net_connect/net_send/net_recv`. TCP+UDP. No Wi-Fi on GameCube, ever. [MEDIUM]

### Anti-patterns (storage)
- Do NOT treat `fatInitDefault()` as "mounts everything" — it is carda:/cardb: and nothing else. Recommending SD2SP2 for bandwidth while shipping only `fatInitDefault()` gives the user an adapter the code cannot read. [HIGH]
- Do NOT hardcode `carda:/` — honor `argv[0]`. [MEDIUM]
- Do NOT write the memory card in a loop or leave it mounted across long gameplay. [HIGH]
- Do NOT DVD_Read into unaligned/odd-sized buffers. [MEDIUM]
- Do NOT assume the SD is present or FAT-formatted — check the mount return and fail with an on-screen message. [HIGH]

---

## 8. BUILD SYSTEM

### Toolchain
- **devkitPPC** via devkitPro pacman. Fresh setup:
```bash
sudo (dkp-)pacman -S gamecube-dev            # devkitPPC + gamecube-tools + classic deps
sudo (dkp-)pacman -S libogc2 libogc2-examples libogc2-libdvm   # current recommended library
```
[HIGH — package names from libogc2 README; re-verify with a search if installs fail, the post-2025 packaging is still settling]
- Environment (must be set): `DEVKITPRO=/opt/devkitpro`, `DEVKITPPC=$DEVKITPRO/devkitPPC`. [HIGH]
- Cross prefix / triplet: `powerpc-eabi-` (`powerpc-eabi-gcc`, `powerpc-eabi-g++`). [HIGH]

### Flags (what the rules files set — do not fight them)
- `MACHDEP = -DGEKKO -mogc -mcpu=750 -meabi -mhard-float` — big-endian PPC750, EABI, **hard float**. [HIGH]
- Link: `-logc -lm` via `-L$(DEVKITPRO)/libogc2/lib/cube` (the libogc2 rules resolve this) plus `-lfat`/`-ldvm`, `-lasnd`, `-lmad`, etc. as used. [MEDIUM exact lib spellings — copy a shipping example Makefile]
- Output: link to **ELF**, convert with **elf2dol** to **`.dol`** (the native executable format). The rules do this automatically (`%.dol: %.elf`). [HIGH]

### Minimal Makefile (libogc2 style)
```make
include $(DEVKITPRO)/libogc2/gamecube_rules   # was $(DEVKITPPC)/gamecube_rules with old libogc
TARGET   := myapp
BUILD    := build
SOURCES  := source
DATA     := data          # bin2s embeds files here as symbols: file_bin/file_bin_end
LIBS     := -logc -lm
# standard devkitPro boilerplate follows — start from libogc2-examples, do not write from scratch
```
[HIGH pattern / MEDIUM that your exact example ships with these names]

### Asset pipeline (run before compiling)
- **Textures** → `gxtexconv -i tex.png -o tex.tpl colfmt=<CMPR|RGB5A3|RGBA8>` (gamecube-tools), or `.scf` script files for batches. Embed TPLs via the `DATA` dir. [HIGH tool / MEDIUM flag spellings]
- **Audio** → resample to 48 kHz, convert to **big-endian** s16 raw (`sox in.wav -r 48000 -c 2 -b 16 -e signed -B out.raw`) or keep OGG/MP3 for streaming decode. [MEDIUM]
- **Models/levels** → your own big-endian binary format; write the byteswap in the exporter, not on console. [HIGH principle]

### Run it
- **Dolphin** opens `.dol`/`.elf` directly (drag-drop). [HIGH]
- **Hardware**: copy `.dol` to SD → launch from Swiss; or name it `/ipl.dol` for PicoBoot autoboot. [HIGH]

### Anti-patterns (build)
- Do NOT build `-msoft-float` (ABI mismatch with the libraries — instant weirdness). [HIGH]
- Do NOT strip `-mogc`/`-meabi` or "upgrade" `-mcpu` to a generic PPC. [HIGH]
- Do NOT hand-roll crt0/linker scripts — libogc2's know the arena, stack, and DOL layout. [HIGH]
- Do NOT ship an ELF to end users; ship the DOL. [HIGH]

---

## 9. EMULATOR VS HARDWARE

### Dolphin (the only emulator that matters here)
Use a current **dev build**, not the years-old "stable". Accuracy is excellent but *deliberately* not cycle- or cache-accurate by default. [HIGH]

**Safe to rely on (Dolphin gets RIGHT)**:
- GX rendering semantics (TEV, formats, copies) to near pixel accuracy. [HIGH]
- CPU instruction correctness, DSP behavior for the common homebrew ucodes, PAD/memcard emulation, `.dol/.elf` loading. [HIGH]

**Must test on real hardware (Dolphin hides or gets WRONG)**:
- **Data-cache coherency — the #1 trap**: Dolphin does not emulate the D-cache by default, so a missing `DCFlushRange` runs perfectly in Dolphin and produces garbage/hangs on hardware. Treat "works in Dolphin, broken on console" as *cache bug until proven otherwise*. [HIGH]
- Timing: real DVD/SD latency, ARAM DMA speed, FIFO backpressure, VI timing edge cases, actual frame budget. Dolphin FPS ≠ hardware FPS. [HIGH]
- Unaligned DMA and marginal eFB/XFB configs that Dolphin tolerates. [MEDIUM]
- Exact analog stick ranges/origins of physical controllers. [HIGH]

### Hardware debugging
- **USB Gecko** (EXI, memcard slot B): serial console (`CON_EnableGecko(CARD_SLOTB, TRUE)` routes `printf`; raw `usb_sendbuffer`) and a **GDB stub**: `DEBUG_Init(GDBSTUB_DEVICE_USB, CARD_SLOTB); _break();` then `powerpc-eabi-gdb app.elf` + `target remote /dev/ttyUSB0`. [MEDIUM exact signatures — check libogc2 `debug.h`]
- **Crash screens**: libogc installs an exception handler that dumps registers + stack to screen (and Gecko if enabled). `SRR0` = faulting PC, `DAR` = bad data address; resolve against your build's `.map` file / `powerpc-eabi-addr2line -e app.elf 0x800xxxxx`. Photograph it; it is your core dump. [HIGH]
- No debugger hardware? Fall back to on-screen `printf` console (`CON_Init` / `console_init`) and binary-search instrumentation. [HIGH]

### Anti-patterns (emu)
- Do NOT optimize based on Dolphin FPS. [HIGH]
- Do NOT declare cache handling correct until a real console ran it. [HIGH]
- Do NOT debug hardware-only faults by guessing — get a Gecko or add on-screen state dumps. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen at boot, no video ever | First VIDEO bring-up | XFB pointer used cached (missing `MEM_K0_TO_K1`), missing `VIDEO_SetBlack(FALSE)`/`VIDEO_Flush`/`WaitVSync`, or hardcoded wrong TV mode for region | Follow Section 3 init order exactly; use `VIDEO_GetPreferredMode(NULL)` | HIGH |
| Works in Dolphin, garbage/hang on console | Any CPU-written buffer GX/DMA reads | Missing `DCFlushRange` (Dolphin skips D-cache) | Flush every written buffer before GPU/DMA use; `GX_InvalidateTexAll` after texture uploads | HIGH |
| Rainbow/blocky corrupted textures | Texture upload | Linear (untiled) data, wrong `GX_TF_*`, or non-32B-aligned texture | Convert with gxtexconv; `memalign(32)`; match format enums | HIGH |
| Renders a few frames then freezes | GX per-frame loop | FIFO overflow (too small) or missing `GX_DrawDone` sync before copy | ≥256 KB FIFO; `GX_DrawDone()` each frame | MEDIUM |
| Silence | Audio init | Buffer not 32B-aligned / length not %32, forgot `AUDIO_StartDMA`, or little-endian samples | `memalign(32)`, byteswap to BE s16, start DMA after pre-fill | HIGH |
| Audio pops/stutter under load | Streaming audio | Decoding inside the DMA callback or buffers too small | Decode in an LWP thread; ≥2×4 KB buffers; callback swaps pointers only | MEDIUM |
| Crash screen, DSI, DAR = small/odd address | Anywhere | NULL/uninitialized pointer deref (DAR shows the bad address) | `addr2line` SRR0 against the ELF; fix the pointer | HIGH |
| Crash screen, ISI at garbage PC | After loading/patching code or bad vtable | Jump through invalid function pointer, or loaded code without `DCFlushRange`+`ICInvalidateRange` | Flush D + invalidate I cache over loaded code; audit callbacks | MEDIUM |
| `fatInitDefault()` returns false / SD "not detected" | SD Gecko | Card exFAT with classic libfat, no MBR, wrong slot, flaky adapter | Format FAT32+MBR or install libdvm; try both slots; use `argv[0]` device | MEDIUM |
| SD Gecko works, **SD2SP2 is invisible** — mount succeeds but the device root never exists | Storage bring-up | Only `fatInitDefault()` was called. libfat's GC table has no `__io_gcsd2` entry, so an SD2SP2 is never mounted no matter how healthy the card is | `fatMountSimple("sd2sp2", &__io_gcsd2)` alongside `fatInitDefault()`; `#include <sdcard/gcsd.h>` (Section 7) | HIGH |
| Crash or corruption after loading a large file | Asset loading | Heap exhausted (24 MB total!) or read into undersized/unaligned buffer | Check `malloc` for NULL; stream in chunks; `memalign(32)`; stage in ARAM | MEDIUM |
| Controller ignored | Input loop | `PAD_Init` missing, no `PAD_ScanPads()` this frame, or only channel 0 checked | Scan every frame; iterate all 4 channels | HIGH |
| Runs at half speed on console, fine in Dolphin | Performance | Hot loops over uncached memory, immediate-mode GX spam, per-small-buffer flushes | Batch into display lists/arrays; flush once per buffer; keep hot data cached | MEDIUM |
| Picture squashed / cut off / rolling on PAL | Video mode | NTSC timings forced on PAL console, eFB→XFB YScale mismatch, viWidth/viXOrigin abuse | Derive everything from `rmode` fields; test PAL, PAL60, NTSC, 480p | MEDIUM |
| Saves vanish / card "corrupted" message in games | Memory card writes | Unmount skipped, sector-size violations, or writing while removing | `CARD_Unmount` always; write sector multiples; debounce saves | MEDIUM |

---

## 11. ANTI-PATTERNS (habits to unlearn on GameCube)

1. Do NOT assume little-endian: the CPU, GX, DSP, and every file the console reads natively are **big-endian** — byteswap at asset-build time.
2. Do NOT skip cache flushes because Dolphin ran it fine.
3. Do NOT pass unaligned pointers or odd sizes to anything with "DMA", "GX", "DVD", "AR", or "CARD" in the name — 32 bytes, always.
4. Do NOT reach for OpenGL/Vulkan/shaders; think in GX + TEV stages.
5. Do NOT malloc, decode, log, or touch files inside interrupt callbacks (audio DMA, VBlank, card detach).
6. Do NOT assume a 60 Hz 640×480 world — PAL is 50 Hz/576i and 480p needs a component cable.
7. Do NOT treat ARAM like RAM — it is DMA-only staging space.
8. Do NOT budget as if 24 MB were free; XFBs, FIFO, code, and stack eat ~2 MB+ before your first asset.
9. Do NOT busy-poll `VIDEO_WaitVSync()` as your only timing source for audio or physics.
10. Do NOT rely on rumble or assume all pads have it (WaveBird).
11. Do NOT hardcode `carda:/` paths — rebase on `argv[0]` from the loader. And do NOT assume `fatInitDefault()` reached every adapter: it mounts carda:/cardb: and nothing else, so an SD2SP2 needs an explicit `fatMountSimple`.
12. Do NOT write saves frequently or leave the memory card mounted "for convenience".
13. Do NOT use threads as if you had multiple cores — one CPU; LWP threads are cooperative-ish latency tools, not parallelism.
14. Do NOT ship debug ELFs or rely on filenames with case sensitivity.
15. Do NOT reference official-SDK symbol names or leaked headers in code or advice — libogc2 equivalents only.

---

## 12. PORTING DECISION TREE (existing engine/game → GameCube)

Work strictly top-down. Each step lists **what / why now / what breaks if skipped**.

1. **Prove the toolchain: build & run a solid-color framebuffer** — isolates environment problems from engine problems. *Skip it and* every later failure is ambiguous (toolchain vs code).
2. **Endianness & serialization audit** — grep every `fread` into structs, every binary asset loader; move byteswapping into the exporter. *Skip it and* all loaded data is silently garbage.
3. **Memory budget on paper** — engine allocations vs 24 MB (minus ~2 MB system). Decide now what streams, what lives in ARAM, what gets cut. *Skip it and* you'll OOM mid-port and refactor allocators under pressure.
4. **Stub the renderer, then GX backend** — first `#ifdef` all draw calls to nothing (get the game *logic* running with on-screen printf), then implement a GX backend (optionally bootstrap via `opengx`, then go native TEV). *Skip it and* you debug logic and rendering simultaneously.
5. **32-byte alignment pass on every I/O and GPU buffer** — vertex buffers, textures, audio, file reads. *Skip it and* you inherit intermittent hardware-only corruption.
6. **File I/O layer → libfat/libdvm with `argv[0]` rebasing** — one mount call, one path-join helper. *Skip it and* nothing loads outside your dev SD layout.
7. **Input mapping → PAD** — GC has A/B/X/Y/Z/Start/D-pad, two sticks, two analog triggers; design the mapping (menus especially) rather than emulating a missing button. *Skip it and* the game is unplayable in review.
8. **Audio backend → double-buffered AI (or ASND/AESND voices)** — resample assets offline to 48 kHz BE. *Skip it and* you ship silence or crackle.
9. **First real-hardware pass NOW, not at the end** — run steps 1–8 on a console; fix every "Dolphin-only" success. *Skip it and* cache bugs compound unfindably.
10. **Performance pass on hardware** — display lists / vertex arrays over immediate mode, paired-single math via `gu*`, texture format diet (CMPR), consider locked-cache for hot data. *Skip it and* you optimize the wrong things off Dolphin numbers.
11. **Region & video-mode matrix** — NTSC 480i, PAL 576i, PAL60, 480p; derive from `rmode`. *Skip it and* PAL users get rolling/cropped video.
12. **Saves (CARD) + polish** — memory card save/load with proper unmount and error UI, Swiss-friendly `.dol` packaging. *Skip it and* progress loss = the only review anyone writes.

### 12b. Special case: retargeting an existing **Wii** port to GameCube

The two consoles share the CPU family, the ABI, GX, ASND, libfat, LWP and devkitPPC, so most of a Wii port moves unchanged. What does not move is small, specific, and fails at compile or link time rather than silently — which is good news, provided you know to look for it.

**libogc APIs that are `HW_RVL`-only.** These exist in the headers on both targets but are `#if defined(HW_RVL)`-gated, so a GameCube build fails to compile: [HIGH — verified against devkitPro libogc headers, 2026-08-19]

| Wii-only | Why | GameCube answer |
|---|---|---|
| `SYS_SetPowerCallback`, `SYS_DoPowerCB` | The Wii's power button is a soft button handled by IOS. A GameCube's switch cuts the rail. | No callback exists; `SYS_SetResetCallback` still works and is the only button you get. |
| `SYS_GetArena2Lo/Hi/Size`, `SYS_SetArena2*` | Arena2 *is* MEM2. | One arena. `SYS_GetArena1Size()` is the whole answer to "how much is left". |
| `SYS_GetHollywoodRevision` | Different GPU. | Nothing to query. |
| `<ogc/usbstorage.h>` / `__io_usbstorage` | No USB host on GameCube at all. | EXI storage only (Section 7). |
| `WPAD_*` / `-lwiiuse -lbte` | Bluetooth. | `PAD_*` over SI; drop both libs from the link line. |

**`MALLOC_MEM2 = 1` must be deleted, not adapted.** Wii ports very often override this weak libogc symbol to move the whole sbrk arena into MEM2. On GameCube there is no second arena to point it at, and the default of 0 is already correct. Deleting it means the heap shrinks by however much MEM2 was carrying — plan the replacement (smaller budgets, ARAM staging) *before* the first build, not after it fails. [HIGH]

**`-G 0 -msdata=none` is usually required.** The EABI reaches `.sdata`/`.sbss` through r2/r13 with a signed 16-bit displacement, so the small-data area caps at 64 KB total and every static under the `-G` threshold lands there. Game-sized ports blow past it. The symptom is a link error — `relocation truncated to fit: R_PPC_EMB_SDA21` — not a runtime fault. Pass it on both the compile and link side. [HIGH]

**Measure BSS before believing any memory plan.** BSS is the line that gets estimated wrong, and it is not close: one ioquake3 GameCube port measured **11 MB** of BSS against a 1–2 MB estimate, a 5–10× miss, on a game far smaller than a GTA. Run `powerpc-eabi-size <target>.elf` on the very first link that succeeds and re-derive every budget from the real number. Add `-Wl,-Map,<target>.elf.map` so the big symbols are attributable (`nm --size-sort` on the ELF). [HIGH]

**The GX backend, the ASND backend, endianness work and libfat all port for free.** Do not rewrite them. `-mogc` already selects `ogc.ld` (verified with `gcc -v`), so the arena layout is correct without an explicit `-T`. The realistic budget is: memory plan, the API table above, the storage bring-up, and then performance — Gekko is 485 MHz against Broadway's 729 and Flipper is 162 MHz against Hollywood's 243, a uniform ~0.67× on both. [HIGH]

---

## 13. QUICK REFERENCE CHEAT SHEET

```
CONSOLE : Nintendo GameCube — "Gekko" PPC750CXe @485MHz, 32-bit BIG-ENDIAN, hard-float + paired singles
MEMORY  : 24MB 1T-SRAM  cached 0x80000000 / uncached 0xC0000000  |  16MB ARAM = DMA-ONLY  |  ALIGN 32B for all DMA/GX
GPU     : "Flipper" 162MHz FIXED-FUNCTION — API = GX, no GL/shaders. TEV 16 stages, 8 tex, 8 lights.
FB FLOW : draw -> eFB (2MB, max 640x528) -> GX_CopyDisp -> XFB (YUV, main RAM, use MEM_K0_TO_K1) -> VI scanout
VIDEO   : NTSC 480i/240p 59.94Hz | PAL 576i 50Hz | PAL60 | 480p via component (VIDEO_HaveComponentCable) — use VIDEO_GetPreferredMode
TOOLS   : devkitPPC + libogc2  ->  pacman -S gamecube-dev libogc2 libogc2-examples libogc2-libdvm
MAKE    : include $(DEVKITPRO)/libogc2/gamecube_rules ; flags -DGEKKO -mogc -mcpu=750 -meabi -mhard-float
OUTPUT  : ELF -> elf2dol -> app.dol ; run in Dolphin, or SD card + Swiss ; PicoBoot autoboots /ipl.dol
CACHE   : DCFlushRange(after CPU writes, before GX/DMA reads) | DCInvalidateRange(before CPU reads DMA output)
          ICInvalidateRange(after loading code) | Dolphin DOES NOT emulate D-cache — hardware will expose you
GX FRAME: draw ... GX_DrawDone(); GX_CopyDisp(xfb[i],GX_TRUE); VIDEO_SetNextFramebuffer(xfb[i]); VIDEO_Flush(); VIDEO_WaitVSync(); i^=1
TEXTURES: gxtexconv -> TPL, tiled formats (CMPR/RGB5A3/RGBA8...), max 1024x1024, 32B aligned, then GX_InvalidateTexAll
AUDIO   : AI DMA, s16 BIG-ENDIAN stereo @48k/32k only, 2+ buffers of 4KB (32B aligned), never decode in the IRQ callback
INPUT   : PAD_Init once; PAD_ScanPads() every frame; sticks s8 w/ origin+deadzone; triggers analog u8; WaveBird = no rumble
STORAGE : fatInitDefault() = carda:/ cardb:/ ONLY (SD Gecko slots A/B). SD2SP2 needs its own
          fatMountSimple("sd2sp2",&__io_gcsd2) from <sdcard/gcsd.h> -- libfat has no entry for it.
          Rebase paths from argv[0]; saves via CARD_* (8KB sectors)
DEBUG   : USB Gecko serial + GDB stub; crash screen SRR0=PC DAR=bad addr -> powerpc-eabi-addr2line -e app.elf
GOLDEN  : big-endian | 32-byte align | flush caches | GX not GL | 24MB total | test on REAL HARDWARE early
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this skill is used in an **active development session** (writing/porting code, not just answering reference questions), you MUST follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify platform-specific blockers, plan the implementation.
- **STATE: IMPLEMENTATION** — Write/modify code based on this SKILLS.md and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output the exact build commands and ask the user to compile and flash/run on real hardware.
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back results. Classify as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES
1. **ANALYSIS → IMPLEMENTATION**: only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: only after you have provided a complete, compilable code change, including:
   - Exact `make` / build command
   - Expected output file name and location (e.g. `build/myapp.dol`)
   - How to transfer to hardware (SD + Swiss, `/ipl.dol` for PicoBoot, etc.)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: you MUST output this exact header, then STOP generating — no fixes, no guesses:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render main menu framebuffer"]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including any error codes, black screen, crashes]
   === STOP ===
   ```
4. **WAITING_FOR_HARDWARE → VALIDATION**: triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - "SUCCESS" → NEXT_GOAL.
   - "FAILURE" → DEBUG_PROTOCOL.
   - Anything else ("kind of works", "almost") → ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: output a **DEBUG BUILD PROTOCOL**:
   - A minimal C/assembly test case that isolates the failure, OR
   - A checklist of 3 specific diagnostic steps (e.g., "Confirm the XFB pointer went through MEM_K0_TO_K1", "Verify the storage device actually mounted — fatInitDefault() covers only carda:/cardb:, an SD2SP2 needs its own fatMountSimple", "Add DCFlushRange over the vertex buffer and retest").
   - Ask the user to run the diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: after providing the debug protocol, return to WAITING_FOR_HARDWARE.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: propose the next milestone from the goal stack. Do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [USB Gecko GDB stub / serial printf / Dolphin-vs-hardware comparison / gc-forever community].`
- Ask the user whether to:
  a) Skip this goal and mark it BLOCKED, or
  b) Provide Dolphin logs / crash-screen photos / register dumps for further analysis.

### GOAL STACK (user-defined or default)
If the user provides a GOAL_STACK at session start, use it. Otherwise use this default:
1. Initialize video output (solid color framebuffer)
2. Initialize controller input (read button presses)
3. Initialize audio output (play sine wave)
4. Load assets from SD/storage
5. Render main menu framebuffer
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
