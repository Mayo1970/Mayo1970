---
name: dc
description: Sega Dreamcast homebrew development expertise (KallistiOS/KOS, SH-4, PowerVR2, AICA, Maple bus, GD-ROM). Use this skill whenever the user mentions Dreamcast, DC, KallistiOS, KOS, sh-elf, PVR, AICA, Maple, VMU, GDEMU, dcload, dc-tool, mkdcdisc, CDI images, 1ST_READ.BIN, or porting any engine/game (ioQuake3, SDL apps, emulators) to the Sega Dreamcast — even if they don't say "homebrew" explicitly. Also invoke for SH-4 assembly, PowerVR2 tile rendering, VQ texture compression, or Dreamcast-specific debugging. When active in a development session, obey the Section 14 state machine with === HARDWARE TEST REQUIRED === gates.
---

# SKILLS_DC.md — Sega Dreamcast Homebrew Expertise Document

You are operating as a Sega Dreamcast homebrew development expert. Follow every rule in this document. Every technical claim carries a confidence tag: [HIGH], [MEDIUM], or [LOW]. When you see [LOW], propose a hardware test before relying on the claim.

---

## 1. HARDWARE ARCHITECTURE

### CPU
- Hitachi/Renesas SH-4 (SH7091, a custom SH7750 variant) @ 200 MHz. [HIGH]
- 32-bit RISC, SuperH ISA, running in **little-endian** mode on Dreamcast. [HIGH]
- Caches: 8 KB instruction cache, 16 KB operand (data) cache, 32-byte cache lines. Operand cache is write-back by default, configurable per-page write-through. [HIGH]
- Half of the operand cache (8 KB) can be reconfigured as on-chip scratchpad RAM ("OCRAM" at 0x7C000000); KOS does not do this by default. [MEDIUM]
- FPU: single-precision optimized. Special 4D SIMD-like instructions: `FIPR` (4-component dot product) and `FTRV` (4x4 matrix × vector transform) using the back FP register bank. ~1.4 GFLOPS theoretical. [HIGH]
- Double-precision exists but is drastically slower and the standard KOS ABI is compiled `-m4-single-only`, which makes `double` an alias-level hazard: mixing true-double code with KOS libraries breaks the ABI. Treat `double` as forbidden. [HIGH]
- Store Queues (SQ): two 32-byte write-combining queues mapped at 0xE0000000–0xE3FFFFFF, the fastest way to burst data to the Tile Accelerator or VRAM. KOS exposes `sq_cpy()`, `sq_set()`. [HIGH]
- `pref` instruction prefetches a cache line; also triggers SQ flush when targeting the SQ area. [HIGH]
- No MMU use in practice: KOS runs flat physical-mapped, no memory protection, no virtual memory. An MMU exists but nothing uses it. [HIGH]
- Quirk: unaligned loads/stores raise an exception — no silent fixup like x86. All 32-bit accesses must be 4-byte aligned. [HIGH]

### GPU
- NEC/VideoLogic PowerVR2 (CLX2) inside the "Holly" system ASIC @ 100 MHz. [HIGH]
- **Tile-Based Deferred Renderer (TBDR)**: the Tile Accelerator (TA) bins submitted polygons into 32×32 pixel tiles; the CORE rasterizes tile-by-tile with an on-chip tile buffer, so depth/color bandwidth to VRAM is nearly free. [HIGH]
- Fixed-function only. **No shaders of any kind.** Per-vertex Gouraud color, one texture per polygon, offset (specular) color, table/vertex fog, punch-through alpha test. [HIGH]
- Hardware order-independent translucency: translucent polygons can be per-pixel autosorted by the hardware (autosort mode) at a performance cost. [HIGH]
- Five display lists per frame: Opaque, Opaque Modifier, Translucent, Translucent Modifier, Punch-Through. A polygon goes in exactly one. [HIGH]
- Modifier volumes provide stencil-like shadow/light volumes without a user-visible stencil buffer. [HIGH]
- 8 MB VRAM (dedicated), accessible via a 64-bit path (0xA4000000 area, texture layout) and a 32-bit linear path (0xA5000000 area). [HIGH]
- Max texture 1024×1024, dimensions must be powers of two. Formats: RGB565, ARGB1555, ARGB4444, YUV422, 4/8-bit paletted, bump map; storage as twiddled (swizzled), twiddled+VQ compressed (~8:1), or non-twiddled ("strided") with restrictions. [HIGH]
- Internal rendering is 32-bit per tile with per-pixel translucency sorting; framebuffer output is typically RGB565 (can be higher for still capture). [MEDIUM]
- Max practical resolution 640×480 (VGA/interlaced); 768×480-class modes exist but are edge cases. [MEDIUM]

### RAM
- 16 MB main SDRAM, 64-bit bus @ 100 MHz (~800 MB/s peak). Physical 0x0C000000, seen cached at 0x8C000000 (P1) and uncached at 0xAC000000 (P2). [HIGH]
- 8 MB VRAM (see GPU). Texture memory is whatever remains after framebuffers, TA object/OPB buffers — budget ~5–6 MB usable for textures in a typical double-buffered 640×480 setup. [MEDIUM]
- 2 MB Audio RAM (ARAM) attached to the AICA, accessed by SH-4 over the slow G2 bus at 0xA0800000 (uncached). G2 writes must respect FIFO status; KOS helpers handle this. [HIGH]
- No virtual memory. Alignment: 4-byte for word access, **32-byte for anything DMA'd or SQ-burst**. [HIGH]

### Bus topology
- SH-4 ↔ Holly over a 64-bit bus. Holly bridges: TA FIFO (polygon submission at 0x10000000), PVR DMA, G1 bus (GD-ROM drive), G2 bus (AICA, expansion port, modem/BBA), and the Maple bus (peripherals). [HIGH]
- DMA channels: PVR DMA (main RAM → TA/VRAM), SPU/G2 DMA (main RAM → ARAM), GD-ROM DMA (disc → main RAM). All require 32-byte aligned, cache-managed source/destination. [HIGH]
- Bottlenecks: G2 bus to ARAM is slow (~short bursts, FIFO-limited); GD-ROM sustained read ~1–1.8 MB/s outer track (12x CAV). Plan streaming around these. [MEDIUM]

### Co-processors
- **AICA sound chip (Yamaha)**: embedded ARM7DI core @ ~22.6 MHz (runs the sound driver program from ARAM), 64 hardware voices (16-bit PCM, 8-bit PCM, 4-bit Yamaha ADPCM), 128-step DSP for effects. SH-4 communicates by writing the ARM program + data into ARAM over G2 and poking AICA registers. KOS ships a precompiled ARM sound driver — you do not need the arm-eabi toolchain unless replacing it. [HIGH]
- **GD-ROM drive controller**: its own microcontroller; SH-4 talks to it through an ATA-like packet interface on G1 (`0xA05F7000` region). KOS wraps this in `fs_iso9660` + cdrom syscalls. [HIGH]
- Maple bus controller in Holly handles peripheral serial protocol via DMA descriptors. [HIGH]

### Security / DRM
- Boot chain: Boot ROM → checks disc bootstrap (IP.BIN) → loads 1ST_READ.BIN. **No cryptographic signatures, no hypervisor.** [HIGH]
- The MIL-CD compatibility path lets unmodified consoles boot burned CD-Rs — this is the entire basis of DC homebrew. Consoles manufactured from roughly late 2000 (some VA2/"non-MIL-CD" units) removed this; a GDEMU/MODE ODE or serial/BBA loading sidesteps it. [HIGH]
- 1ST_READ.BIN on CD boots "scrambled"; the scramble is a known deterministic algorithm handled by `mkdcdisc`/`scramble`. Binaries loaded by dcload or an ODE are plain unscrambled. [HIGH]
- Homebrew has full unrestricted hardware access — every register listed here is writable. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: **Sega Katana SDK** (SH-4 native, "Set" releases) and Windows CE for Dreamcast (Microsoft). Both are proprietary/leaked. **Do NOT use, reference, or assume access to Katana or WinCE SDKs.** [HIGH]
- Homebrew: **KallistiOS (KOS)** — the canonical, actively maintained SDK. BSD-like license (attribution required), commercial use allowed; nearly all modern indie DC releases use it. [HIGH]
- Current era: KOS 2.x (2.1.0 and later). GCC 9–15 supported via the bundled `dc-chain` toolchain builder; C17/C++20 fully supported, preliminary C23/C++23. Newlib libc. [HIGH]
- KOS provides: kernel threads (preemptive), Maple device framework, PVR API, sound (sfx + streaming), VFS (`/cd`, `/rd` romdisk, `/vmu`, `/pc` via dcload, `/ram`), TCP/IP network stack (BBA + modem PPP), and `kos-ports` package manager (SDL 1.2/2, GLdc OpenGL 1.x subset, zlib, libpng, libjpeg, Lua, mp3/ogg, etc.). [HIGH]
- KOS 2.2.0 changed the VMU save API (`vmu_pkg`) in a build-breaking way — check which KOS version the project targets before writing save code. [HIGH]
- What KOS CANNOT do vs official: no Ninja/Kamui high-level 3D model pipeline (bring your own formats), no official performance analyzer, GD-ROM area reading only matters for licensed GD discs (irrelevant for homebrew), no WinCE compatibility layer. GLdc is OpenGL 1.1-ish immediate/vertex-array subset — **not** GL2+, no shaders, incomplete blending corner cases. [MEDIUM]
- Alternative minimal libs exist (libronin, DreamHAL for SH-4 math headers), but default to KOS unless the user says otherwise. [MEDIUM]

---

## 3. GRAPHICS PIPELINE

- Use the **KOS PVR API** (`<dc/pvr.h>`) for 3D/2D accelerated rendering. Use `vid_*` (`<dc/video.h>`) for raw framebuffer-only programs. Use GLdc only when porting GL code, and expect to rewrite hot paths to raw PVR. [HIGH]

### Initialization
```c
#include <kos.h>
pvr_init_params_t params = {
    /* bin sizes per list: opaque, op-modifier, translucent, tr-modifier, punch-thru */
    { PVR_BINSIZE_16, PVR_BINSIZE_0, PVR_BINSIZE_16, PVR_BINSIZE_0, PVR_BINSIZE_16 },
    512 * 1024,        /* vertex buffer size */
    0, 0, 0            /* dma, fsaa, autosort disabled/defaults */
};
pvr_init(&params);
```
Set a bin size of 0 for any list you never submit — it reclaims VRAM. [HIGH]

- Video mode is auto-detected from the cable at KOS init (`vid_check_cable()`): VGA → 640×480@60 progressive; NTSC composite → 640×480 interlaced or 320×240; PAL → 50 Hz variants. You can force modes with `vid_set_mode(DM_640x480, PM_RGB565)`. [HIGH]

### Frame flow
- Per frame: `pvr_wait_ready()` → `pvr_scene_begin()` → for each list: `pvr_list_begin(PVR_LIST_OP_POLY)` … submit headers/vertices … `pvr_list_finish()` → `pvr_scene_finish()`. [HIGH]
- Submission methods: **Direct Render** (`pvr_dr_target`/`pvr_dr_commit`, writes vertices through Store Queues — fastest CPU-side path) or **DMA vertex buffers** (`pvr_vertex_dma` mode, build lists in main RAM, PVR DMA streams to TA while CPU works). ioQuake3-class ports want DMA mode. [HIGH]
- Rendering is internally triple-buffered in the sense that TA binning of frame N+1 overlaps CORE rasterization of frame N; KOS manages the framebuffer flip on vsync. [MEDIUM]
- A vertex is submitted as a 32-byte `pvr_vertex_t`; the last vertex of each strip must set `PVR_CMD_VERTEX_EOL`. Geometry is **triangle strips only** at the hardware level (single triangles = strips of 3). [HIGH]

### VSync / refresh
- NTSC/VGA: 60 Hz. PAL: 50 Hz (PAL-M/PAL-N oddities exist in BIOS region data but treat PAL=50). `pvr_wait_ready()` blocks until the TA is ready for a new scene, effectively frame-pacing you. [HIGH]

### Textures
- Allocate VRAM with `pvr_mem_malloc()` / free with `pvr_mem_free()`. Never `malloc()` for textures. [HIGH]
- Upload with `pvr_txr_load()` / `pvr_txr_load_ex()` (handles twiddling) or DMA. Formats: `PVR_TXRFMT_RGB565`, `ARGB1555`, `ARGB4444`, `YUV422`, `PAL4BPP/PAL8BPP`, modifiers `_TWIDDLED`, `_VQ_ENABLE`, `_NONTWIDDLED` (stride). [HIGH]
- Power-of-two only, 8×8 up to 1024×1024. Non-square is fine. Mipmaps supported for twiddled square textures. [HIGH]
- **VQ compression** (~8:1, 2 KB codebook + indices) is the platform superpower: a 512×512 RGB565 texture drops from 512 KB to ~66 KB. Precompress offline (`vqenc` from KOS utils, or `pvrtex`). [HIGH]
- Paletted textures use global palette RAM configured via `pvr_set_pal_format()`. [HIGH]

### Depth / blend
- Depth: 32-bit floating-point 1/W based, per-tile on-chip — configure compare with `PVR_DEPTHCMP_*` in the polygon header. Because it is 1/W, **use W-friendly projection; depth near/far tricks from PC do not transfer.** [HIGH]
- Blending: src/dst factor selection per polygon header (`PVR_BLEND_*`) — only in the translucent/punch-through lists. Opaque list ignores alpha entirely. [HIGH]
- No user stencil buffer; use modifier volumes for volume effects. [HIGH]

### Anti-patterns
- Do NOT submit translucent geometry to the opaque list (renders, but alpha is dropped) or opaque to the translucent list (kills fill-rate and sorting). [HIGH]
- Do NOT change texture/blend state per triangle expecting free state changes — each state change is a new 32-byte polygon header; batch by material. [HIGH]
- Do NOT use desktop OpenGL habits via GLdc for the final renderer of a serious port; GLdc is a bootstrap. [MEDIUM]
- Do NOT assume the framebuffer is CPU-addressable during PVR rendering — mixing `vid_*` direct writes with an active PVR scene corrupts output. [MEDIUM]

---

## 4. INPUT

- All peripherals live on the **Maple bus**: 4 ports (A–D), each with a main device + up to 2 sub-peripheral slots (VMU, rumble pack, microphone). [HIGH]
- Device types: standard controller, arcade stick, VMU (screen+buttons+128 KB flash), Puru Puru (Jump Pack rumble), keyboard, mouse, lightgun, fishing rod, twin stick, maracas, racing wheel, Dreameye camera. [HIGH]
- KOS model: Maple is scanned automatically by a periodic background poll (once per frame via vblank). You read the latest cached state — effectively **polled**, with optional hot-plug callbacks. [HIGH]

### Reading a controller
```c
maple_device_t *dev = maple_enum_type(0, MAPLE_FUNC_CONTROLLER);
if(dev) {
    cont_state_t *st = (cont_state_t *)maple_dev_status(dev);
    if(st->buttons & CONT_START) { /* ... */ }
    int x = st->joyx;   /* -128..127 */
    int lt = st->ltrig; /* 0..255 analog triggers */
}
```
Or iterate all pads with `MAPLE_FOREACH_BEGIN(MAPLE_FUNC_CONTROLLER, cont_state_t, st) ... MAPLE_FOREACH_END()`. [HIGH]

- Standard pad: digital D-pad, A/B/X/Y/Start, one analog stick, two analog triggers. **There is no second analog stick and no L3/R3/Select** — plan control schemes accordingly (huge issue for FPS ports; support mouse+keyboard, which the hardware genuinely has). [HIGH]
- Rumble: `<dc/maple/purupuru.h>`, `purupuru_rumble()` with effect parameters; first-party rumble pack support was fixed in recent KOS — verify KOS version if rumble misbehaves. [MEDIUM]
- VMU: `<dc/maple/vmu.h>` + `vmu_pkg` for saves (API changed in KOS 2.2.0), `vmu_draw_lcd()` for the 48×32 mono LCD. [HIGH]
- Keyboard (`MAPLE_FUNC_KEYBOARD`) and mouse (`MAPLE_FUNC_MOUSE`) have full KOS drivers — a legitimate primary input for FPS/RTS ports. [HIGH]
- Disconnect/reconnect: `maple_enum_type()` returns NULL when absent; re-enumerate every frame rather than caching `maple_device_t*` across frames. Attach/detach callbacks exist (`maple_attach_callback`). [MEDIUM]

### Anti-patterns
- Do NOT assume port A always has a controller — arcade sticks, keyboards, or nothing may be there. Enumerate by function, not by port. [HIGH]
- Do NOT poll Maple manually in a tight loop; the background scan already runs and manual raw frames will fight it. [MEDIUM]
- Do NOT hardcode analog deadzones at zero — real sticks rest off-center; use a ±~16 deadzone. [MEDIUM]

---

## 5. MEMORY LAYOUT

### SH-4 address windows (apply to all physical addresses)
| Window | Base | Property |
|---|---|---|
| P1 | 0x80000000 | Cached, untranslated — normal code/data |
| P2 | 0xA0000000 | **Uncached**, untranslated — MMIO, DMA-visible views |
| P3 | 0xC0000000 | Cached, translated (unused, no MMU) |
| P4 | 0xE0000000 | Control space: Store Queues 0xE0000000–0xE3FFFFFF, CPU regs |
[HIGH]

### Physical map (key regions)
| Physical | Size | What |
|---|---|---|
| 0x00000000 | 2 MB | Boot ROM |
| 0x00200000 | 256 KB | Flash (settings — treat as read-only) |
| 0x005F6800+ | — | Holly/system registers (PVR regs at 0x005F8000) |
| 0x00700000 | — | AICA registers (via G2) |
| 0x00800000 | 2 MB | Sound RAM (ARAM, uncached access 0xA0800000) |
| 0x04000000 | 8 MB | VRAM, 64-bit access path |
| 0x05000000 | 8 MB | VRAM, 32-bit linear path |
| 0x0C000000 | 16 MB | Main SDRAM |
| 0x10000000 | — | TA polygon FIFO (write-only) |
[HIGH]

- Program load address: **0x8C010000** (cached view of 0x0C010000); the first 64 KB of RAM holds BIOS/syscall vectors — never overwrite 0x8C000000–0x8C00FFFF. [HIGH]
- Allocation: `malloc()` serves main RAM (newlib heap grows toward the stack); `pvr_mem_malloc()` serves VRAM; ARAM is managed by the sound driver (`snd_mem_malloc()` for raw use). [HIGH]
- Default main stack in KOS is 64 KB for the main thread [MEDIUM — verify in current `arch/dreamcast/kernel/startup.s` / linker script; test: recursive function with depth counter until crash]. Thread stacks are set at `thd_create` time. Deep-recursion engines (Quake-lineage `R_RecursiveWorldNode`) should be checked against stack growth.
- Cache: write-back operand cache, 32-byte lines, **no hardware coherence with DMA**. Before DMA *from* a buffer: `dcache_flush_range(addr, len)`. After DMA *into* a buffer: `dcache_inval_range(addr, len)` (or use an uncached P2 pointer for small MMIO-ish buffers). After writing code (JIT/self-modifying): flush dcache then `icache_flush_range()`. [HIGH]
- DMA buffers: 32-byte aligned start AND length (`__attribute__((aligned(32)))` or `memalign(32, n)`). [HIGH]
- Store Queue writes bypass cache but still require the destination flush semantics of the SQ mechanism (`sq_cpy` handles it). [HIGH]

### Anti-patterns
- Do NOT DMA from a stack buffer or any buffer you haven't flushed — you'll ship stale cache lines and get "works in emulator, garbage on hardware." [HIGH]
- Do NOT read ARAM with wide/burst CPU accesses ignoring G2 FIFO rules; use KOS `spu_memload/spu_memread` helpers. [HIGH]
- Do NOT put frequently CPU-touched data in P2 uncached space "to be safe" — it's a 10x+ slowdown; manage the cache instead. [HIGH]

---

## 6. AUDIO

- Hardware: Yamaha AICA — 64 voices, per-voice pitch/volume/pan, ADPCM (4-bit Yamaha), 8/16-bit PCM, plus a 128-step effects DSP. 2 MB ARAM holds the ARM driver program + all sample data. [HIGH]
- Recommended API: KOS `snd_sfx_*` for one-shot effects and `snd_stream_*` for streamed music. kos-ports supply Ogg/MP3/ADX decoders that plug into snd_stream. [HIGH]
- Sample rates up to 44.1 kHz (native mixer rate 44100 Hz); mono or stereo (stereo = paired voices); 16-bit PCM, 8-bit PCM, Yamaha ADPCM. [HIGH]
- `snd_sfx_load("/rd/shot.wav")` loads a WAV (PCM or Yamaha ADPCM) fully into ARAM — budget the 2 MB. [HIGH]
- Streaming model: **polled callback**. You register a `snd_stream_alloc(cb, bufsize)` callback that supplies PCM when asked, and you must call `snd_stream_poll(hnd)` regularly (every frame) from a normal thread — the callback fills a ring buffer that G2-DMAs into ARAM. [HIGH]
- Typical stream buffer: 0x10000 (64 KB) works well; too small ⇒ starvation crackle when the frame hitches, too large ⇒ latency + ARAM waste. [MEDIUM]
- To avoid glitches: never let the render loop stall longer than the buffered audio duration; do decode work (Ogg/MP3) in a dedicated KOS thread at slightly elevated priority; keep the stream callback itself trivial (memcpy from a decode ring). [MEDIUM]
- Custom ARM driver work (replacing KOS's) requires the arm-eabi toolchain — only do this if the user explicitly wants sequenced/tracker playback on the ARM side. [HIGH]

### Anti-patterns
- Do NOT decode Ogg/MP3 inside the stream callback on the main thread's frame budget. [HIGH]
- Do NOT assume ARAM is memcpy-fast; bulk loads go through G2 helpers and are slow — load audio during load screens, not mid-frame. [HIGH]
- Do NOT exceed ~2 MB total resident samples; evict per-level. [HIGH]

---

## 7. STORAGE / IO

- Media reachable from homebrew:
  - **GD-ROM drive** reading CD-R/MIL-CD homebrew discs → `/cd` (ISO9660, KOS auto-mounts). [HIGH]
  - **Optical Drive Emulators** (GDEMU, MODE, USB-GDROM) present images as the GD drive — same `/cd` path, no code changes. [HIGH]
  - **Romdisk**: a ROMFS image linked into the binary, mounted at `/rd` — the zero-hassle way to ship assets during development. [HIGH]
  - **VMU flash**: `/vmu/a1` … (port letter + slot), 128 KB per VMU, block-based; use `vmu_pkg` framing so files show in the BIOS manager. [HIGH]
  - **dcload host filesystem**: `/pc/...` maps to your PC's directory when running via dc-tool (serial or IP) — the primary dev-loop filesystem. [HIGH]
  - **SD card**: via serial-port SD adapter (`fs_dcsd`-class drivers) or G1-ATA/IDE mods; support varies by KOS version and driver — [MEDIUM], verify the exact adapter before promising SD support.
  - **Network**: Broadband Adapter (BBA) or LAN adapter → KOS TCP/IP stack (sockets API); modem via PPP. dc-tool-ip also rides the BBA. [HIGH]
- ISO9660 notes: KOS `/cd` handles Joliet-ish long names via mkdcdisc conventions; treat paths as **case-insensitive on /cd but case-sensitive on /pc and /rd** [MEDIUM — test: open the same file with wrong case on each mount]. Always use forward slashes.
- There is no "loader directory standard" like Wii's /apps: a homebrew release is a bootable disc image (CDI for burning/GDEMU, or plain files for ODE menus). Ship: `IP.BIN` (bootstrap with metadata/region/VGA flags), scrambled `1ST_READ.BIN`, your assets. `mkdcdisc` generates all of it from an ELF. [HIGH]
- Internal flash (settings) is writable by hardware but: **do NOT write to internal flash** — a corrupt flash bricks settings and there is almost never a reason. [HIGH]

### Anti-patterns
- Do NOT fopen dozens of small files per frame from /cd — GD-ROM seeks are ~hundreds of ms; pack files (pak/zip) and stream sequentially. [HIGH]
- Do NOT assume /pc exists in release builds — gate dcload paths behind a dev flag. [HIGH]
- Do NOT store saves anywhere but VMU (players expect it, and everything else may be absent). [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **sh-elf** GCC built by the KOS `dc-chain` builder (`$KOS_BASE/utils/dc-chain`). Use the **stable profile** (currently a modern GCC; KOS 2.x supports GCC 9–15). Optional second toolchain: `arm-eabi` for custom AICA drivers only. [HIGH]
- Cross prefix / triplet: `sh-elf-` (e.g. `sh-elf-gcc`, `sh-elf-g++`, `sh-elf-objcopy`). [HIGH]
- Environment: source `environ.sh` before anything. Key vars: `KOS_BASE`, `KOS_PORTS`, `KOS_CC_BASE`, `PATH` including `/opt/toolchains/dc/sh-elf/bin`. Every build shell needs `source /opt/toolchains/dc/kos/environ.sh`. [HIGH]
- Compile through the wrappers `kos-cc`, `kos-c++`, `kos-ld`, `kos-ar` — they inject the required flags and link `libkallisti`. If invoking sh-elf-gcc raw, the critical flags are:
  - `-ml` (little-endian) [HIGH]
  - `-m4-single-only` (SH-4, single-precision-only FPU model — **ABI-defining**, everything must match) [HIGH]
  - `-ffunction-sections -fdata-sections`, `-O2`, `-fomit-frame-pointer` (typical KOS defaults) [MEDIUM]
  - Startup/linker script come from `$KOS_BASE/kernel/arch/dreamcast` via kos-ld; base address 0x8C010000. [HIGH]
- Output: ELF. From there:
  - Dev loop: run the **ELF directly** with `dc-tool-ip -t <dc-ip> -x prog.elf` or `dc-tool-ser -t /dev/ttyUSB0 -x prog.elf` (console runs dcload). [HIGH]
  - Release: `mkdcdisc -e prog.elf -o game.cdi` → burnable/GDEMU-ready CDI. mkdcdisc handles objcopy→BIN, scrambling, IP.BIN, ISO layout; flags add name/icon/region/VGA-enable and data dirs (`-d assets/`). [HIGH]
  - Manual path if needed: `sh-elf-objcopy -R .stack -O binary prog.elf prog.bin` → `scramble prog.bin 1ST_READ.BIN`. [HIGH]

### Minimal Makefile template
```makefile
TARGET = game.elf
OBJS = main.o renderer.o romdisk.o
KOS_ROMDISK_DIR = romdisk

include $(KOS_BASE)/Makefile.rules

all: $(TARGET)
$(TARGET): $(OBJS)
	kos-cc -o $(TARGET) $(OBJS) -lm
clean:
	-rm -f $(TARGET) $(OBJS) romdisk.img
```
`KOS_ROMDISK_DIR` + the stock rules auto-generate `romdisk.o` from a directory of assets. [HIGH]

- Asset pipeline (offline, before build):
  - Textures → power-of-two, then `vqenc`/`pvrtex` to twiddled or VQ `.dt/.kmg/.pvr`-style blobs. [HIGH]
  - Audio SFX → 44.1/22.05 kHz WAV, Yamaha ADPCM (`wav2adpcm` in KOS utils) to quarter the ARAM cost. [HIGH]
  - Music → Ogg Vorbis ~96–128 kbps for streaming. [MEDIUM]
  - Models → your own binary format, pre-stripped into triangle strips, pre-scaled floats. No standard format. [HIGH]

### Anti-patterns
- Do NOT mix objects built with different FPU flags (`-m4-single-only` vs default) — silent FP corruption at link/runtime. [HIGH]
- Do NOT link desktop-assuming libraries that spawn processes/mmap — KOS has threads, not processes; no mmap. [HIGH]
- Do NOT forget `source environ.sh` — the classic "sh-elf-gcc: command not found" / wrong-libc failure. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary: **Flycast** (open source, most accurate mainstream option, actively developed; also libretro core). Use it for day-to-day iteration. [HIGH]
- Alternatives: lxdream/lxdream-nitro (dev-oriented, aging), Demul (Windows, closed, accurate PVR in many cases), redream (closed). [MEDIUM]
- Flycast gets RIGHT (safe to rely on): PVR display lists and blending for common cases, TA submission semantics, Maple controllers/VMU basics, GD-ROM ISO reading, AICA ARM driver execution well enough for KOS audio. [MEDIUM]
- Emulators get WRONG or hide (MUST test on hardware):
  - **SH-4 cache behavior**: most emus don't emulate the operand cache → missing `dcache_flush_range` before DMA works in emu, corrupts on hardware. This is the #1 emu-vs-hw divergence. [HIGH]
  - Exact DMA/G2 FIFO timing and TA buffer overflow behavior. [MEDIUM]
  - Real GD-ROM seek/read latency (emu ISO reads are instant → streaming code that "works" starves on hardware). [HIGH]
  - Unaligned access exceptions are sometimes tolerated by emus. [MEDIUM]
  - VRAM bandwidth limits, fill-rate, exact autosort cost → never trust emulator FPS. [HIGH]
  - PAL/NTSC cable-detect edge cases and 240p signaling on real TVs. [MEDIUM]
- Hardware debugging:
  - **dcload-serial** (Coder's Cable / USB serial mod) or **dcload-ip** (BBA/LAN) gives: program upload, stdout redirected to your PC console, host filesystem `/pc`, exception dumps with full register + stack trace on crash, and a **GDB stub** (`dc-tool -g` + `sh-elf-gdb`). This is the canonical debug rig. [HIGH]
  - KOS prints panics/assertions over the dcload console; `dbglog()`/`printf` go there too. [HIGH]
  - No dcload? Fall back to on-screen `bfont` text and color-coded `vid_clear()` states. [HIGH]

### Anti-patterns
- Do NOT ship after emulator-only testing; cache + GD latency bugs are invisible there. [HIGH]
- Do NOT profile on Flycast and report FPS as truth; measure with `timer_us_gettime64()` on hardware. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen at boot, drive spins | CD-R boot | 1ST_READ.BIN not scrambled, or IP.BIN missing/wrong | Rebuild disc with `mkdcdisc`; don't hand-roll the ISO | [HIGH] |
| Black screen only on VGA box | VGA cable | IP.BIN VGA flag off, or forced TV-only video mode | Enable VGA in mkdcdisc flags; use cable auto-detect, don't force DM modes | [HIGH] |
| Boots in emulator, instant crash on hardware | Any DMA use | Missing `dcache_flush_range`/`dcache_inval_range` around DMA buffers | Flush before DMA-out, invalidate after DMA-in; 32-byte align buffers | [HIGH] |
| Pink/green garbage textures | PVR rendering | Texture not twiddled while header says twiddled, or wrong format bits/stride | Match `PVR_TXRFMT_*` flags to actual data; use `pvr_txr_load_ex` or pre-twiddle offline | [HIGH] |
| Polygons flicker/disappear randomly | Heavy scenes | TA vertex buffer or OPB overflow (too much geometry for configured bin sizes) | Increase `pvr_init` vertex buffer / bin sizes; cut geometry; check `pvr_get_stats` | [MEDIUM] |
| Audio silent, everything else fine | KOS sound | `snd_stream_init`/driver not started, or WAV in unsupported float/24-bit format | Init sound subsystem; convert assets to 16-bit PCM or Yamaha ADPCM | [HIGH] |
| Audio crackles when loading/heavy frames | Streaming music | `snd_stream_poll` starved (called too rarely) or decode on main thread | Poll every frame; move decode to its own thread; enlarge stream buffer | [HIGH] |
| Crash with "unaligned memory access" exception dump | Any | 32-bit load/store to non-4-byte-aligned address (packed structs, cast char* to int*) | memcpy through aligned temporaries; `__attribute__((packed))` audits | [HIGH] |
| Crash after loading large file | Asset loading | Heap exhaustion in 16 MB main RAM, or overwriting 0x8C000000 low 64 KB | Check malloc returns; budget memory; keep load address ≥ 0x8C010000 | [HIGH] |
| Controller ignored / NULL deref on input | Maple | Cached `maple_device_t*` after disconnect, or enumerating by port not function | Re-enumerate each frame via `maple_enum_type`; NULL-check | [HIGH] |
| "File not found" on /cd but file is on disc | ISO layout | Path case mismatch or file outside the data track mkdcdisc built | Verify with `fs_open` of exact path; add dir via `mkdcdisc -d`; test case variants | [MEDIUM] |
| Everything runs at half/two-thirds speed on PAL console | PAL region | 50 Hz mode + frame-locked game logic | Decouple logic from vsync or offer 60 Hz mode (most PAL DC TVs accept it via cable detect) | [MEDIUM] |
| Slow performance, CPU idle | PVR-bound | Autosort on huge translucent scenes, oversized textures uncompressed, per-tri state changes | VQ-compress, batch by material, minimize translucent list, consider sorted-list mode | [MEDIUM] |
| Wrong aspect / image offset on real TV | NTSC/PAL TV | Borders/overscan differ from emulator window; 640×480i assumptions | Keep HUD inside ~90% safe area; test composite and VGA both | [HIGH] |
| Random FP garbage in math after integrating a lib | Mixed builds | Object files compiled without `-m4-single-only` linked into KOS build | Rebuild all third-party code with kos-cc; never link prebuilt desktop-SH objects | [HIGH] |

---

## 11. ANTI-PATTERNS (PLATFORM RULES)

1. Do NOT use `double` anywhere in hot code or across ABI boundaries — the whole platform is built `-m4-single-only`. [HIGH]
2. Do NOT skip `dcache_flush_range`/`dcache_inval_range` around DMA; the emulator will hide this bug. [HIGH]
3. Do NOT allocate textures with `malloc()`; VRAM is a separate 8 MB pool via `pvr_mem_malloc()`. [HIGH]
4. Do NOT ship non-power-of-two or >1024px textures; the PVR cannot sample them. [HIGH]
5. Do NOT put alpha-blended geometry in the opaque list or vice versa. [HIGH]
6. Do NOT assume a second analog stick exists; design controls for one stick + triggers, and support Maple keyboard/mouse. [HIGH]
7. Do NOT perform per-frame file access on /cd; GD-ROM seeks murder frame time — pack and preload. [HIGH]
8. Do NOT write to the internal 256 KB flash. Ever. [HIGH]
9. Do NOT cast misaligned byte buffers to `int*`/`float*`; SH-4 raises exceptions on unaligned access. [HIGH]
10. Do NOT decode audio inside the stream callback or main render loop; use a thread. [HIGH]
11. Do NOT trust emulator FPS, cache behavior, or disc latency; validate every milestone on hardware. [HIGH]
12. Do NOT hand-write IP.BIN/scramble steps when `mkdcdisc` exists; a mis-scrambled binary is an invisible black-screen. [HIGH]
13. Do NOT use OpenGL (GLdc) as the final renderer for performance-critical ports; drop to the PVR API for hot paths. [MEDIUM]
14. Do NOT busy-wait on vblank with interrupts disabled; KOS threading and the Maple/audio subsystems depend on interrupts running. [MEDIUM]
15. Do NOT reference, link, or reproduce Katana/WinCE SDK headers or code; all examples must build on stock KOS. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Build a null-platform ELF with kos-cc.** Stub renderer/audio/input; get the engine's core compiling with `-m4-single-only` and newlib. *Why first*: flushes out `double` usage, POSIX-isms (fork, mmap, dlopen), and endianness assumptions before any DC code exists. *Skip it and*: you'll debug platform code and libc gaps simultaneously.
2. **Memory budget audit.** Engine heap + assets must fit ~15 MB main / ~5–6 MB VRAM / 2 MB ARAM. Add allocator instrumentation now. *Why*: DC ports die on memory, not CPU, first. *Skip it and*: mysterious late-stage OOM crashes after loading large levels.
3. **Video bring-up.** `pvr_init`, clear-color frame loop at 60 Hz, on-screen bfont debug text. *Why*: gives a visible heartbeat + debug channel for everything after. *Skip it and*: you're blind on hardware.
4. **Input bring-up.** Maple controller enumeration + mapping layer (pad, keyboard, mouse). *Why*: needed to drive menus during all later testing; forces the one-stick control design early. *Skip it and*: control-scheme rework late in the port.
5. **File I/O layer.** Route the engine VFS to `/rd` (dev) then `/cd` (release), with a pak/large-file strategy and `/pc` for the dcload dev loop. *Why*: asset loading precedes any real rendering. *Skip it and*: GD-ROM seek storms and 20-second level loads.
6. **Renderer port, phase 1 (correctness).** Translate draw calls to PVR lists: opaque vs translucent vs punch-through classification, texture conversion to twiddled/VQ at asset-build time, triangle-strip generation. Use DMA vertex mode for anything Quake-class. *Why*: this is 60% of the port. *Skip proper list classification and*: broken blending and unusable fill-rate.
7. **Audio port.** SFX → ADPCM in ARAM; music → threaded Ogg streaming. *Why now*: independent of renderer, cheap win once threading works. *Skip it and*: nothing breaks, but ship-blocking later under memory pressure if ARAM budget was never planned.
8. **Hardware validation pass.** Full playthrough on a real console via dcload; watch for cache/DMA bugs and disc latency. *Why*: emulator lies (Section 9). *Skip it and*: release-day black screens on real hardware.
9. **Renderer/CPU optimization phase.** FIPR/FTRV matrix math (DreamHAL-style headers), store-queue submission, VQ everything, material batching, LOD/culling tuned to TA limits. *Why last*: optimize only validated-correct code. *Skip it and*: 12 fps and a dead project — but at least a correct one to optimize.
10. **Release packaging.** `mkdcdisc` CDI, VMU save + icon, region/VGA flags, PAL 50/60 handling. *Why last*: pure packaging. *Skip it and*: it boots for you and nobody else.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Sega Dreamcast | CPU: SH-4 (SH7091) 200MHz little-endian, 8K I$/16K D$ (write-back, 32B lines),
  FPU single-precision (-m4-single-only, NO doubles), FIPR/FTRV 4D SIMD, Store Queues @0xE0000000
GPU: PowerVR2 CLX2 @100MHz, tile-based deferred, FIXED FUNCTION (no shaders), 8MB VRAM,
  lists: OP/TR/PT (+modifier vols), tex: POT ≤1024², RGB565/1555/4444/YUV/PAL, twiddled + VQ (~8:1)
RAM: 16MB main @0x8C000000 (P1 cached)/0xAC.. (P2 uncached), load addr 0x8C010000, 2MB ARAM (AICA, G2 bus)
AUDIO: Yamaha AICA, ARM7 driver, 64ch PCM/ADPCM, 44.1kHz; KOS snd_sfx + snd_stream (poll every frame)
INPUT: Maple bus, enumerate by MAPLE_FUNC_* every frame; 1 analog stick + 2 analog triggers; kbd/mouse exist
SDK: KallistiOS (KOS 2.x), sh-elf GCC via dc-chain, kos-cc/kos-c++, source environ.sh first
BUILD: Makefile + $(KOS_BASE)/Makefile.rules → ELF; dev: dc-tool-ip/-ser -x prog.elf (dcload = console+GDB+/pc)
RELEASE: mkdcdisc -e prog.elf -o game.cdi (handles scramble+IP.BIN); saves → VMU only (/vmu/a1, vmu_pkg)
FS: /cd (ISO9660, SLOW seeks — pack files), /rd romdisk, /pc dcload, /vmu; do NOT touch internal flash
CACHE/DMA: 32-byte align; dcache_flush_range before DMA-out, dcache_inval_range after DMA-in — EMULATORS HIDE THIS
EMU: Flycast for iteration; hardware-validate every milestone (cache, GD latency, fill-rate all differ)
KILLERS: doubles, unaligned casts, per-frame /cd reads, opaque/translucent list mixups, malloc'd textures
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this SKILL.md is used in an active development session (not just reference), you must follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify platform-specific blockers, and plan the implementation.
- **STATE: IMPLEMENTATION** — Write/modify code based on this SKILL.md and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output the exact build commands and ask the user to compile and flash/run on real hardware.
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back results. Classify the result as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES

1. **ANALYSIS → IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change. You must include:
   - Exact `make` / build command (with `source environ.sh` reminder)
   - Expected output file name and location (e.g., `game.elf`, `game.cdi`)
   - How to transfer to the target hardware (dc-tool-ip, dc-tool-ser, burned CD-R, GDEMU SD card)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color framebuffer at 60 Hz"]
   BUILD: [Command]
   DEPLOY: [Method: dc-tool-ip -t <ip> -x game.elf | dc-tool-ser | mkdcdisc → GDEMU/CD-R]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including any error codes, black screen, crashes, dcload exception dumps]
   === STOP ===
   ```
   After this header, you STOP generating. You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE → VALIDATION**: Triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - If the user says "SUCCESS": Move to NEXT_GOAL.
   - If the user says "FAILURE": Move to DEBUG_PROTOCOL.
   - If the user says anything else (e.g., "it kind of works", "almost"): Ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: You must output a **DEBUG BUILD PROTOCOL**:
   - A minimal C/assembly test case that isolates the failure (e.g., a 30-line pvr_init + clear-color loop, a bare `dcache_flush_range` DMA round-trip test),
   - OR a checklist of 3 specific diagnostic steps (e.g., "Confirm the dcload exception dump PC value and map it with sh-elf-addr2line", "Verify the texture pointer came from pvr_mem_malloc", "Check vid_check_cable() return value on this console").
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the debug protocol, you return to WAITING_FOR_HARDWARE state.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: You propose the next milestone from the goal stack below. You do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [sh-elf-gdb over dc-tool -g / dcload serial console dump / Flycast-vs-hardware differential test / Simulant Discord or DCEmulation forums].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide dcload exception dumps / emulator logs / register dumps for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this default:
1. Initialize video output (solid color framebuffer via pvr_init + clear color)
2. Initialize controller input (read button presses via Maple, on-screen bfont feedback)
3. Initialize audio output (play sine wave / test WAV via snd_sfx)
4. Load assets from storage (/rd romdisk first, then /cd or /pc)
5. Render main menu framebuffer (textured quads in the opaque list)
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
