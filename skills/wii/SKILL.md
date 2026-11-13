# SKILLS_Wii.md — Nintendo Wii Homebrew Expertise Document

**Revision:** 2026-11-13 · **Scope:** native Wii homebrew, standard devkitPro libogc 3.x and the separate libogc2 ecosystem · **Status:** reference/engineering guidance, not a substitute for target-version headers or hardware testing.

You are operating as a Nintendo Wii homebrew developer. Prefer Wii hardware constraints and the **selected SDK's** real APIs over desktop assumptions. Preserve existing project conventions unless asked to migrate. **Do not mix APIs, binary packages or build rules from libogc and libogc2** without verifying compatibility. When a statement below conflicts with the selected release's headers, examples or verified hardware behavior, follow that evidence and explicitly report the discrepancy; this document is not infallible.

### Verification / confidence conventions
- `[HIGH]` = established hardware behavior or confirmed upstream documentation; `[MEDIUM]` = environment/version dependent or requires validation; `[LOW]` = provisional observation. Confidence is **not** a guarantee of applicability to every build.
- **Released** = part of a tagged released package; **main branch** = may not be in a package; **open PR / issue** = not a supported released API. Never turn open PR names into callable functions unless the project uses that branch.
- **Before generating code:** identify (1) Wii / Wii Mini / Wii U vWii target; (2) **libogc 3.x** or **libogc2**; (3) installed devkitPPC and SDK versions; (4) C or C++; (5) real hardware available for validation. Ask only for missing details that materially block the next change; otherwise state assumptions.
- **As-of 2026-11-13:** confirmed GitHub releases include [libogc 3.1.0](https://github.com/devkitPro/libogc/releases), [devkitPPC r49.1](https://github.com/devkitPro/buildscripts/releases), [wii-examples v20260510](https://github.com/devkitPro/wii-examples/releases), and [GRRLIB 4.6.1](https://github.com/GRRLIB/GRRLIB/releases). These are historical reference points, **not** pinned future versions. Re-check before an upgrade.

---

## 1. HARDWARE ARCHITECTURE

### CPU — "Broadway"
- IBM PowerPC 750CL derivative ("Broadway"), 729 MHz, 32-bit, **big-endian**. [HIGH]
- Superscalar in-order-ish (750-class: limited OoO completion), short pipeline — branch-heavy code runs comparatively well; deep unrolling helps less than on modern CPUs. [HIGH]
- Caches: 32 KB L1 instruction + 32 KB L1 data (8-way), 256 KB L2. L1 D-cache line size is **32 bytes**. [HIGH]
- FPU: full hardware double/single-precision FPU. You must build hard-float. [HIGH]
- SIMD: **paired singles** — two packed 32-bit floats per FPR, with dedicated `ps_*` instructions and quantized load/store (GQR registers). Enabled via `-mpaired`-style intrinsics or hand asm; GCC support is partial, so hot paths usually use inline asm or the libogc `ps` helpers. [HIGH]
- No AltiVec. Do NOT emit AltiVec/VMX code. [HIGH]
- Quirk: paired-single quantized loads (`psq_l`) require GQR setup; misconfigured GQRs silently corrupt float data. [MEDIUM]
- Quirk: unaligned floating-point loads/stores can raise alignment exceptions in some cases; keep float data naturally aligned. [MEDIUM]

### GPU — "Hollywood" package / GX
- The Hollywood package contains the **GX** graphics core (an overclocked GameCube "Flipper", ATI/ArtX design) at 243 MHz, plus the Starlet I/O processor (see co-processors). [HIGH]
- **Fixed-function only. There are no shaders, no shader model, no GLSL/HLSL.** Programmability comes from the **TEV** (Texture EnVironment) unit: up to **16 TEV stages**, each a configurable color/alpha combiner over up to 8 textures and 2 rasterized colors. [HIGH]
- Embedded memory: **~2 MB EFB (embedded framebuffer)** + **1 MB TMEM (texture cache)**. You render into the EFB, then **copy** to an XFB (external framebuffer) in main RAM for display. [HIGH]
- EFB maximum size: 640×528 pixels. Practical modes: 640×480 (NTSC), 640×528 (PAL). No true HD output — component cable gives 480p at best. [HIGH]
- EFB pixel formats: RGB8 + 24-bit Z, or RGBA6 + 24-bit Z (destination alpha requires the RGBA6 mode), or RGB565 + 16-bit Z ("pixel format Z16"). You cannot have full RGBA8 + destination alpha simultaneously. [HIGH]
- Texture formats: I4, I8, IA4, IA8, RGB565, RGB5A3, RGBA8, CI4/CI8/CI14 (palettized via TLUT), and **CMPR** (4-bit S3TC/DXT1-style block compression). Max texture size 1024×1024. [HIGH]
- Textures are **tiled/swizzled, not row-major**, but **block dimensions vary by format**: I4/CI4 = 8×8; I8/IA4/CI8 = 8×4; IA8/RGB565/RGB5A3/CI14x2 = 4×4; RGBA8 = 4×4 **64-byte tiles** with separate AR and GB planes; CMPR = 8×8 **32-byte compressed blocks**. Most other listed blocks occupy 32 bytes. Use the format-aware table in §3 and pad each mip level correctly. [HIGH] ([Dolphin texture decoder](https://github.com/dolphin-emu/dolphin/blob/master/Source/Core/VideoCommon/TextureDecoder_Common.cpp))
- Depth: 24-bit Z in EFB, with Z-compression modes (ZC_LINEAR, ZC_NEAR/MID/FAR). Full Z test/write control via `GX_SetZMode`. [HIGH]
- Stencil: **no stencil buffer.** Emulate with destination alpha or multi-pass tricks. [HIGH]
- Blending: standard src/dst factor blending, plus logic ops and subtract mode, via `GX_SetBlendMode`. Destination alpha blending only in RGBA6 EFB mode. [HIGH]
- Indirect texturing: 4 indirect stages enable EMBM/distortion effects. [HIGH]
- Command submission: CPU writes GX commands into a **FIFO** (write-gather pipe at 0xCC008000); libogc's GX API wraps this. Display lists are pre-recorded command buffers replayed by the GP. [HIGH]

### RAM
- **MEM1: 24 MB of 1T-SRAM** — low latency (~10 ns class), the "fast" pool. Physical 0x00000000–0x017FFFFF. [HIGH]
- **MEM2: 64 MB GDDR3** — higher latency, higher capacity. Physical 0x10000000–0x13FFFFFF. GX can texture directly from both. [HIGH]
- Virtual memory: the MMU is used only for **static BAT-style mapping**, not paging. There is **no swap and no demand paging**. Effective addresses:
  - MEM1 cached: `0x80000000`, MEM1 uncached: `0xC0000000` [HIGH]
  - MEM2 cached: `0x90000000`, MEM2 uncached: `0xD0000000` [HIGH]
  - Hardware registers: `0xCC000000` region (VI, PI, DSP, EXI, AI, GX FIFO), Hollywood/IOS registers at `0xCD000000`. [HIGH]
- Alignment: **32-byte cache-line alignment is the safe baseline for buffers shared with GX/DMA/IOS** (FIFO, display lists, texture data, many IPC buffers). Use aligned allocations and the **specific API's** size/padding contract. Not every ordinary CPU buffer or unrelated allocation needs 32-byte alignment; the general rule is about hardware sharing and cache safety. [HIGH]
- A chunk of MEM2 (top ~12–16 MB depending on IOS) is **reserved for IOS**; libogc's arena reflects this. Do not assume all 64 MB is yours. [HIGH]

### Bus topology
- Broadway ↔ Hollywood over a 64-bit bus (~243 MHz); GX, VI, AI, EXI, and the memory controller live in Hollywood. All CPU access to RAM and I/O goes through Hollywood. [HIGH]
- GX reads vertex/texture data from main RAM via DMA; the CPU→GP write-gather FIFO is the primary command path. [HIGH]
- EFB→XFB copy is a dedicated hardware blit (`GX_CopyDisp`) that can also downsample (AA) and Y-scale. [HIGH]
- Possible bottlenecks: CPU-side immediate vertex submission and MEM2 latency on random CPU reads. Consider display lists / indexed arrays and MEM1 hot data where **real-hardware profiling** confirms a benefit. [HIGH]

### Co-processors
- **Starlet (ARM926EJ-S)** inside Hollywood runs **IOS**, the I/O operating system. It owns SD, USB, Wi-Fi, Bluetooth, NAND, and crypto hardware. The PPC talks to it via an **IPC mailbox** interface (`/dev/...` resource nodes, `IOS_Open/IOS_Ioctl(v)`). libogc wraps this. [HIGH]
- **DSP**: a Macronix 16-bit DSP for audio mixing/decoding. It runs uploaded microcode ("ucode"); libogc ships a mixer ucode used by ASND/AESND. CPU↔DSP via mailbox registers + ARAM-less DMA (Wii has no GameCube ARAM; the DSP DMAs from main RAM). [HIGH]
- **AI (Audio Interface)**: streams a 16-bit stereo PCM buffer from main RAM to the DAC at 32 or 48 kHz via DMA. [HIGH]

### Security / DRM
- Boot chain: boot0 (mask ROM) → boot1 → boot2 → IOS → System Menu. RSA signature checks at each stage; the famous strncmp/fakesign bug is patched on later boot1s. [HIGH]
- Homebrew entry: HBC loads `boot.dol` / `boot.elf` through a homebrew entry path. **Do not assume full AHBPROT access is always inherited.** For HBC launches that need it, request access in `meta.xml` with `<ahb_access/>`; the result also depends on loader/HBC/IOS behavior. [HIGH] ([WiiBrew HBC](https://wiibrew.org/wiki/Homebrew_Channel))
- With AHBPROT protection disabled (hardware access permitted), PPC software may access otherwise restricted hardware; without it, normally use supported IOS services. **IOS reload can lose access and invalidate device state.** Avoid unsolicited IOS reloads; verify privilege state and startup configuration on the actual target. **AHBPROT is not permission to perform risky NAND writes.** [HIGH]
- Homebrew CANNOT: run unsigned code on Starlet without an exploit (bootmii/mini), or use official Nintendo network services. NAND writes are possible but are the #1 way to brick consoles — treat NAND as read-only. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: Nintendo **Revolution SDK (RVL SDK)** with CodeWarrior/GX libraries. You must NOT use, reference, or assume access to it. All guidance here targets the community stack. [HIGH]
- **Standard devkitPro stack (default):** devkitPPC + **libogc 3.x** (OS, VIDEO, GX, WPAD/PAD, IOS IPC, threads and networking) plus version-compatible `libfat`, `wiiuse`/Bluetooth, AESND/ASND, GRRLIB, OpenGX, SDL2 and other Wii portlibs as needed. Install supported packages via devkitPro pacman; **check each dependency's actual license** rather than assuming identical licenses. [HIGH]
- **Alternative stack:** [extremscorner/libogc2](https://github.com/extremscorner/libogc2) offers extensive fixes, accessories/subsystem enhancements and its own package ecosystem. It describes broad compatibility with **older libogc (≤2.1.0) APIs**, not a guarantee of compatibility with libogc 3.x. **libogc portlibs are not ABI-compatible with libogc2**; use corresponding libraries, packages, build rules and container. [HIGH]
- **Choose one SDK per build:** standard `$(DEVKITPPC)/wii_rules` and standard Wii portlibs for libogc; `$(DEVKITPRO)/libogc2/wii_rules` and libogc2-specific portlibs for libogc2. Never silently swap when a project already has a functioning dependency graph. [HIGH] ([libogc2 migration guide](https://github.com/extremscorner/libogc2/blob/master/README.md))
- **libogc 3.0.0** replaced underlying LWP implementation with **Tuxedo** and preserved an LWP-compatible API; with devkitPPC **r49**, it added **pthreads, C11 threads and C++ `std::thread` support** and BSD socket header/shim support. **libogc 3.1.0** brought more fixes for Tuxedo, GX interrupts and clock functions. Use the chosen release's headers and compile flags; threads still share one CPU core. [HIGH] ([libogc release notes](https://github.com/devkitPro/libogc/releases), [devkitPPC releases](https://github.com/devkitPro/buildscripts/releases))
- libogc CAN: full GX access, threading (LWP compatibility and modern threads in 3.x), IOS IPC, SD/USB/network, and supported Wii Remote / MotionPlus paths. Hardware support and device quirks remain version-dependent. [HIGH]
- libogc CANNOT / gaps:
  - No official-quality documentation — the header files and existing repos ARE the documentation. [HIGH]
  - No NAND-title/WAD tooling in-core (community tools exist, high brick risk). [HIGH]
  - DSP: only the shipped mixer ucode; writing custom DSP ucode is expert territory with sparse docs. [MEDIUM]
  - Wi-Fi networking works (libogc `net_*`) but is slower and flakier than official stacks; no WPA3. [MEDIUM]
- **Modern software choices:** [OpenGX](https://github.com/devkitPro/opengx) is a real OpenGL-like wrapper (immediate mode, arrays, texturing, basic lighting, mipmapping); [GRRLIB](https://github.com/GRRLIB/GRRLIB) is a GX-based graphics helper (4.6.0 added CMake/TPL-related APIs); devkitPro publishes [`wii-sdl2`](https://github.com/devkitPro/pacman-packages/blob/master/wii/SDL2/PKGBUILD) (package recipe references SDL2 **2.28.5** at review date). Libraries simplify ports but have different tradeoffs; verify support and performance on Wii. [HIGH]
- Decision impact: GX has no programmable shaders. Engines requiring modern shaders must replace those materials/effects with TEV, fixed-function approximations or a redesigned rendering path. Porting effort depends on the original engine; avoid categorical performance claims without measurements. [HIGH]

---

## 3. GRAPHICS PIPELINE

- Native low-level API: **GX** (`<ogc/gx.h>`). **OpenGX** provides a subset of OpenGL-like behavior on GX and can be a pragmatic production porting choice, but it is **not a shader API** and not fully OpenGL-compliant. Use native GX for highly specialized paths or proven bottlenecks. **Do not claim a universal 30–50% OpenGX penalty** without workload-specific profiling. [HIGH] ([OpenGX README](https://github.com/devkitPro/opengx))

### Initialization (reference sequence for standard libogc; adapt per target)
```c
VIDEO_Init();
GXRModeObj *rmode = VIDEO_GetPreferredMode(NULL);
void *xfb[2] = {
    MEM_K0_TO_K1(SYS_AllocateFramebuffer(rmode)),
    MEM_K0_TO_K1(SYS_AllocateFramebuffer(rmode)),
};
VIDEO_Configure(rmode);
VIDEO_SetNextFramebuffer(xfb[0]);
VIDEO_SetBlack(FALSE);
VIDEO_Flush();
VIDEO_WaitVSync();
if (rmode->viTVMode & VI_NON_INTERLACE) VIDEO_WaitVSync();

void *gp_fifo = memalign(32, 256*1024);   // 32-byte aligned, >=64 KB
memset(gp_fifo, 0, 256*1024);
GX_Init(gp_fifo, 256*1024);
GX_SetCopyClear((GXColor){0,0,0,255}, GX_MAX_Z24);
GX_SetViewport(0, 0, rmode->fbWidth, rmode->efbHeight, 0, 1);
f32 yscale = GX_GetYScaleFactor(rmode->efbHeight, rmode->xfbHeight);
u32 xfbHeight = GX_SetDispCopyYScale(yscale);
GX_SetScissor(0, 0, rmode->fbWidth, rmode->efbHeight);
GX_SetDispCopySrc(0, 0, rmode->fbWidth, rmode->efbHeight);
GX_SetDispCopyDst(rmode->fbWidth, xfbHeight);
GX_SetCopyFilter(rmode->aa, rmode->sample_pattern, GX_TRUE, rmode->vfilter);
GX_SetFieldMode(rmode->field_rendering,
                (rmode->viHeight == 2 * rmode->xfbHeight) ? GX_ENABLE : GX_DISABLE);
GX_SetPixelFmt(rmode->aa ? GX_PF_RGB565_Z16 : GX_PF_RGB8_Z24, GX_ZC_LINEAR);
```
[HIGH] This is **initialization guidance**, not a complete compilable program: add the appropriate `<gccore.h>` / allocator includes, `WPAD_Init()` and/or `PAD_Init()` if required, allocation-failure checks, GX projection/vertex/TEV state, frame rendering and clean shutdown. The EFB copy Y-scale, field mode and pixel format match the [devkitPro GX examples](https://github.com/devkitPro/wii-examples/blob/master/graphics/gx/neheGX/lesson01/source/lesson1.c); do not reintroduce the former no-op `VIDEO_PadFramebufferWidth(...) ? a : a` conditional.

### Frame flow
- Render into **EFB** → complete/synchronize relevant GX work (`GX_DrawDone()` placement depends on pipeline) → `GX_CopyDisp(xfb[cur], GX_TRUE)` copies/clears EFB → `VIDEO_SetNextFramebuffer(xfb[cur]); VIDEO_Flush(); VIDEO_WaitVSync();` → flip index. Double-buffer XFBs; EFB itself is single. **Follow a tested official example for exact copy/draw synchronization rather than assuming a single universal order.** [HIGH] ([GX example](https://github.com/devkitPro/wii-examples/blob/master/graphics/gx/neheGX/lesson01/source/lesson1.c))
- The XFB is **YUV (YCbCr 4:2:2)**, written by the copy engine — you never write RGB directly to XFB except via libogc's `CON_*` console or careful YUV code. [HIGH]

### Refresh / regions
- NTSC: 640×480, 60 Hz (interlaced or 240p). PAL: 640×528, 50 Hz, plus **PAL60/EuRGB60** 480i@60. Progressive 480p only with component cable and `VIDEO_HaveComponentCable()`. Always use `VIDEO_GetPreferredMode` instead of hardcoding. [HIGH]
- Widescreen: console setting readable via `CONF_GetAspectRatio()`; the hardware does NOT render wider — you render anamorphic 640-wide and the TV stretches. Adjust your projection matrix. [HIGH]

### Textures
- **Block layout (critical for custom converters):** the block dimensions depend on GX format, not uniformly 4×4. Use the table below and validate padded width/height, TLUT handling and mip layout against actual target formats. [HIGH] ([Dolphin's format decoder](https://github.com/dolphin-emu/dolphin/blob/master/Source/Core/VideoCommon/TextureDecoder_Common.cpp))

  | Format(s) | Tile in texels | Tile bytes |
  |---|---:|---:|
  | I4, CI4 | 8×8 | 32 |
  | I8, IA4, CI8 | 8×4 | 32 |
  | IA8, RGB565, RGB5A3, CI14x2 | 4×4 | 32 |
  | RGBA8 (AR/GB planes) | 4×4 | 64 |
  | CMPR (DXT1-like) | 8×8 | 32 |

- Upload: `GX_InitTexObj` + `GX_LoadTexObj`. Data must use the appropriate tiled format, have required GX/DMA alignment, and be **flushed from D-cache (`DCFlushRange`) after CPU writing, before GX reads it**. [HIGH]
- Prefer **CMPR** for diffuse maps (4bpp, huge TMEM/bandwidth win); RGB5A3 for UI with alpha; RGBA8 only when quality demands (it costs 2 TMEM lookups). [HIGH]
- Mipmaps: supported and strongly recommended; without them GX thrashes TMEM on minified textures — a known real-world slowdown cause. [HIGH]

### Geometry
- Three submission paths: immediate mode (`GX_Begin`/`GX_Position3f32`.../`GX_End`) — slowest; **indexed arrays** (`GX_SetArray` + index streams) — good; **display lists** (`GX_BeginDispList`/`GX_CallDispList`) — fastest for static geometry. Display lists must be 32-byte aligned, padded to 32-byte multiples, and `DCFlushRange`d after recording. [HIGH]
- Vertex formats are described via VAT (`GX_SetVtxAttrFmt`) — supports compressed attributes (s8/s16 fixed-point positions, u8 colors). Use compressed formats; bandwidth matters more than ALU. [HIGH]

### TEV in one paragraph
- Each of up to 16 stages computes `output = (d op ((1-c)*a + c*b)) * scale + bias` independently for color and alpha, with inputs selectable from texture, rasterized color, konst registers, and previous stage. Typical Q3-style diffuse×lightmap = 2 textures, 2 stages (or 1 stage with 2 texcoords collapsed). Multi-pass PC blending often collapses into fewer TEV stages — always try to collapse before adding passes. [HIGH]

### Anti-patterns (GPU)
- Do not assume OpenGX is equivalent to native GX or universally slower by a fixed percentage. Profile on real Wii and use native GX for measured hotspots or unsupported features. [HIGH]
- Do NOT forget `DCFlushRange` on textures, vertex arrays, and display lists — Dolphin will hide this, hardware will show garbage. [HIGH]
- Do NOT read the EFB per-pixel from CPU in a loop (`GX_PeekARGB`) for effects — it is extremely slow. [HIGH]
- Do NOT exceed 640×528 render size or assume HD. [HIGH]

---

## 4. INPUT

- Controller types: **Wii Remote** (buttons, 3-axis accelerometer, IR pointer, speaker, rumble) + expansions: **Nunchuk** (stick + accel), **Classic Controller (Pro)**, **MotionPlus** (gyro); **GameCube controllers** on the 4 GC ports (original Wii only, RVL-101 lacks them); USB keyboards; Balance Board. [HIGH]
- Two separate libogc APIs, both **polled**:
  - `PAD_Init(); PAD_ScanPads(); PAD_ButtonsDown(0); PAD_StickX(0);` — GameCube pads. [HIGH]
  - `WPAD_Init(); WPAD_ScanPads(); WPAD_ButtonsDown(WPAD_CHAN_0);` — Wiimotes. [HIGH]
- For ordinary frame-driven games, call `WPAD_ScanPads()` / `PAD_ScanPads()` **once in the main input-update phase per frame** (when the corresponding API is initialized), then use `ButtonsDown` (edge), `ButtonsHeld` (level), `ButtonsUp`. Keep polling ownership consistent; avoid simultaneous scans on multiple threads. [HIGH]
- Rich data: `WPAD_SetDataFormat(chan, WPAD_FMT_BTNS_ACC_IR)` then `WPAD_Data(chan)` → `WPADData*` with `.ir` (pointer x/y, valid flag), `.accel`, `.orient`, `.gforce`, `.exp` (expansion union: `.exp.nunchuk`, `.exp.classic`). Set `WPAD_SetVRes(chan, 640, 480)` for screen-space IR. [HIGH]
- MotionPlus: current WPAD headers expose `WPAD_SetMotionPlus(chan, 1)`; inspect the selected version's `WPADData` / expansion representation rather than assuming one layout. Test gyro sign/scale, detection, internal vs external MotionPlus and drift on actual hardware. **Proposed** `WPAD_HasMotionPlus()` in PR #260 is **not** a released standard API as of this review. [MEDIUM] ([WPAD header](https://github.com/devkitPro/libogc/blob/master/gc/wiiuse/wpad.h), [open PR #260](https://github.com/devkitPro/libogc/pull/260))
- Expansion hot-plug: check `data->exp.type` **every frame** — nunchuks are plugged/unplugged at runtime. `WPAD_Probe(chan, &type)` reports connection status; handle `WPAD_ERR_NO_CONTROLLER`. [HIGH]
- Rumble: `WPAD_Rumble(chan, 1/0)` and `PAD_ControlMotor(chan, PAD_MOTOR_RUMBLE/STOP)`. No force curves — it's on/off; PWM it yourself for intensity. [HIGH]
- Wii U Pro Controller: support was expanded in libogc **2.13.0**; confirm current API and behavior before promising it in the UI. More general Bluetooth via **WPAD2 / bt-embedded** is an **open PR (#258), not released libogc**. [MEDIUM] ([releases](https://github.com/devkitPro/libogc/releases), [PR #258](https://github.com/devkitPro/libogc/pull/258))
- Known reports to watch (not necessarily universal bugs): [WPAD expansion input loss after enabling all-channel IR/accelerometer data (#274)](https://github.com/devkitPro/libogc/issues/274); [MotionPlus detection/handling fixes proposal (#260)](https://github.com/devkitPro/libogc/pull/260). Test and report the **exact** libogc version and reproduction steps. [MEDIUM]
- Wiimote speaker: exists (ADPCM); support and practicality depend on library/version. Keep optional unless tested. [MEDIUM]
- Input validation matrix: disconnected-at-boot / reconnection; expansion insert/remove; Wii Remote vs Wii Remote Plus; two or more pads; HOME/power/reset; Wiimote sideways + Nunchuk + Classic; GC PAD where physical ports exist; Wii U Pro if SDK supports it; shutdown and return-to-HBC. [HIGH]
- Anti-patterns:
  - Do NOT assume a Wiimote is connected at boot — pairing may complete seconds after your main loop starts. [HIGH]
  - Do NOT assume only one controller layout; support Wiimote-sideways, Wiimote+Nunchuk, Classic, and GC pad if feasible. [HIGH]
  - Do NOT poll WPAD from multiple threads. [MEDIUM]

---

## 5. MEMORY LAYOUT

| Region | Effective addr | Size | Notes |
|---|---|---|---|
| MEM1 cached | 0x80000000 | 24 MB | 1T-SRAM, fast; code+hot data here [HIGH] |
| MEM1 uncached | 0xC0000000 | 24 MB | alias; XFB access, MMIO-ish buffers [HIGH] |
| MEM2 cached | 0x90000000 | 64 MB | GDDR3; bulk assets, textures [HIGH] |
| MEM2 uncached | 0xD0000000 | 64 MB | alias [HIGH] |
| HW registers | 0xCC000000 | — | VI/PI/DSP/AI/EXI/GX FIFO [HIGH] |
| Hollywood/IPC | 0xCD000000 | — | IOS IPC, GPIO, AHBPROT [HIGH] |

- Allocation: `malloc`/`memalign` serve **MEM1** by default. MEM2 via libogc arena: `SYS_GetArena2Lo()/SYS_GetArena2Hi()` or the `AllocateMEM2`-style wrappers many ports write (grab arena2, run your own allocator). Top of MEM2 belongs to IOS — respect the arena bounds. [HIGH]
- Rule of thumb: code, stack, vertex/display lists, and per-frame data in MEM1; textures, audio assets, file caches in MEM2. GX textures work fine from MEM2. [HIGH]
- Stacks: sizes, startup symbols, and thread-creation contracts **vary with SDK and version**. Check the installed crt/linker map and `LWP_CreateThread` / pthread / C++ threading semantics rather than relying on a guessed constant or unverified override syntax. Size large parser/JIT stacks from observed high-water marks and keep recursive workloads bounded. [MEDIUM]
- Cache: L1D is **write-back**, 32-byte lines, **no hardware coherency with GX/DMA/IOS**. Therefore:
  - `DCFlushRange(ptr, len)` — after CPU writes, before GX/DSP/IOS reads. [HIGH]
  - `DCInvalidateRange(ptr, len)` — before CPU reads data written by DMA/IOS (buffers must be 32-byte aligned AND 32-byte-multiple sized, or invalidation eats neighboring data). [HIGH]
  - `ICInvalidateRange(ptr, len)` — after writing code (loaders, JIT). A PPC JIT must DCFlush + ICInvalidate every emitted block. [HIGH]
- IOS IPC buffers (file I/O via `/dev/fs`, USB, etc.): 32-byte aligned, 32-multiple sized. libfat handles this internally but your custom IOS calls must too. [HIGH]
- Anti-patterns:
  - Do NOT `memcpy` into a display list or texture without flushing after. [HIGH]
  - Do NOT put multi-MB assets in MEM1 "because it's fast" — you'll starve the allocator; MEM1 speed matters for CPU-touched data, not GX-streamed textures. [HIGH]
  - Do NOT pass stack buffers to DMA/IOS — stack lines share cache lines with live locals. [HIGH]

---

## 6. AUDIO

- Hardware: AI streams 16-bit big-endian stereo PCM from main RAM at **32000 or 48000 Hz** via DMA; the DSP (with libogc's mixer ucode) mixes voices ahead of it. [HIGH]
- Recommended API: **AESND** (`<aesndlib.h>`) — modern, voice-based, low overhead; or **ASND** (`<asndlib.h>`) — older, simpler, used by many ports. SDL-Wii and most engine ports sit on ASND/AESND. [HIGH]
- Formats: signed 16-bit PCM (mono/stereo) is the safe path; 8-bit supported by ASND voices. Compressed formats (Ogg/MP3) are decoded on CPU (libtremor/libmad ports exist). [HIGH]
- Model: **callback-driven double buffering.** AESND calls your voice callback when a buffer drains; you refill and `DCFlushRange` it. Typical buffer: 2–4 KB per callback at 48 kHz. [HIGH]
- No separate audio RAM on Wii (unlike GameCube's ARAM) — everything streams from MEM1/MEM2. [HIGH]
- Glitch avoidance:
  - Keep the callback allocation-free and lock-free; feed it from a ring buffer filled on the main/worker thread. [HIGH]
  - Flush every buffer you hand to the DSP/AI (`DCFlushRange`) — stale cache = crackle/static. [HIGH]
  - Match your mixer rate to 32/48 kHz; resample offline, not per-frame. [HIGH]
- Anti-patterns: Do NOT decode Ogg inside the audio callback; do NOT assume little-endian WAV data plays correctly — byteswap PCM to big-endian. [HIGH]

---

## 7. STORAGE / IO

- Media: **SD slot (on models that have one)** and **USB mass storage** via supported IOS/device setup; NAND (via IOS `/dev/fs` — normally read-only in game code), optional disc-drive paths (do not require unsupported reads), and network (not universally present on Wii Mini). SDXC cards may work if formatted compatibly, but verify capacity, filesystem and IOS/device support instead of blanket declaring compatibility. [HIGH/MEDIUM]
- **Standard libogc storage:** **libfat** and FAT16/FAT32 are the predictable HBC release targets. `fatInitDefault()` attempts to make supported devices accessible; **check the return value and each actual mount** (`sd:/`, `usb:/`) instead of assuming both devices are present. `fopen("sd:/apps/mygame/data.pak", "rb")` works only if the path is actually mounted and exists. FAT filename matching is case-insensitive / case-preserving. [HIGH]
- **libogc2 storage:** libogc2 documents the **`libogc2-libdvm` provider** (based on libdvm) as a common choice; its ecosystem includes **exFAT support**. The libogc2 maintainer states that migrating libfat to libdvm or enabling exFAT needs **no application code changes in normal cases**. Do **not invent an exFAT API or imply that stock HBC suddenly launches from an exFAT app partition**. Choose a consistent SDK/portlib stack and test exact media. [HIGH] ([libogc2 README](https://github.com/extremscorner/libogc2/blob/master/README.md), [libdvm](https://github.com/extremscorner/libdvm))
- **HBC app layout:** `boot.dol` (or supported `boot.elf`) is required; **`meta.xml` and `icon.png` are optional**, though recommended for releases. HBC expects app folders on supported FAT media, normally `sd:/apps/yourapp/` or the equivalent on USB. [HIGH] ([WiiBrew HBC](https://wiibrew.org/wiki/Homebrew_Channel))

  ```text
  sd:/apps/yourapp/boot.dol      # Required executable
  sd:/apps/yourapp/meta.xml      # Optional, recommended; can request AHBPROT
  sd:/apps/yourapp/icon.png      # Optional, recommended: 128x48 PNG
  ```

  Minimal opt-in metadata for apps requiring HBC-provided hardware access (fill in real details):

  ```xml
  <app version="1">
    <name>Your App</name>
    <coder>Your Name</coder>
    <version>1.0</version>
    <short_description>Example app</short_description>
    <ahb_access/>
  </app>
  ```

- Network: older projects often use `net_*` wrappers (e.g., `net_init`, `net_socket`); **libogc 3.0 adds standard BSD socket headers and shims**, so prefer the supported API for the actual version. Do not assume every POSIX networking call is implemented (check gaps/issues). Wii wireless and USB Ethernet compatibility depend on IOS and adapter; measure latency/throughput rather than assuming a fixed speed. `wiiload` can be a rapid deployment route where HBC networking is configured. **Do not assume modern TLS, WPA3 or Internet services without concrete compatibility testing.** [HIGH/MEDIUM] ([libogc releases](https://github.com/devkitPro/libogc/releases))
- Anti-patterns:
  - Do NOT write to NAND unless you are writing a NAND tool on purpose; a bad write bricks consoles. [HIGH]
  - Do NOT assume the SD card is present or mounted — check `fatInitDefault()`'s return and fail gracefully. [HIGH]
  - Do NOT do synchronous file I/O on the render thread mid-game; SD latency spikes cause frame hitches. [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **devkitPPC** (supported install: devkitPro pacman, often the `wii-dev` meta-package). As of this review, **r49.1** followed r49 (pthread/newlib 4.6.0 integration) and r48 (**GCC 15.2.0**, binutils 2.45.1). Do **not** keep the old “GCC 13+ era” description as a version pin. Prefix is `powerpc-eabi-` (`powerpc-eabi-gcc`, `powerpc-eabi-g++`). [HIGH] ([devkitPPC releases](https://github.com/devkitPro/buildscripts/releases))
- Environment (mandatory): `DEVKITPRO=/opt/devkitpro`, `DEVKITPPC=$DEVKITPRO/devkitPPC`. [HIGH]
- Compiler flags: **copy the selected SDK's current Wii example/template** rather than hard-code a stale universal flag set. Legacy builds commonly use `-mrvl -mcpu=750 -meabi -mhard-float -DGEKKO`, optimized with `-O2` and section flags. Verify exact compiler/library ABI and paired-single flags in the chosen release. Never assume desktop x86 flags, SIMD, or soft-float. [HIGH]
- Linker: **use the target SDK's example CMake/Make rules and resolve library order from the actual package metadata**. Older standard libogc projects link libraries like `-lwiiuse -lbte -lfat -laesnd -logc -lm`, but this is **not** a universal ordering for modern libogc3, libogc2, SDL2 or OpenGX. Check compile/link invocations and headers rather than copying an old command uncritically. [HIGH]
- Output: link to **ELF**, convert with **`elf2dol`** to `boot.dol`. HBC runs both, but DOL is the standard. [HIGH]
- Make/CMake: start from [devkitPro/wii-examples](https://github.com/devkitPro/wii-examples) (not an assumed on-disk path; verify installed `wii-examples`). Standard libogc Make builds use `$(DEVKITPPC)/wii_rules`; libogc2 uses `$(DEVKITPRO)/libogc2/wii_rules`. For CMake, consult the **installed** devkitPro toolchain setup or `libogc2-cmake` as appropriate; do not assume a particular `Wii.cmake` path exists on every installation. [HIGH]
- **libogc 3.x application loop:** new examples use `SYS_MainLoop()` for default reset/power handling. Preserve normal cleanup and termination. Do not generate a mandatory `while(1)` loop as the only pattern for new 3.x projects. [HIGH] ([wii-examples v20260125](https://github.com/devkitPro/wii-examples/releases))

  ```c
  // Structural sketch only: initialize VIDEO, WPAD, GX, buffers before this.
  while (SYS_MainLoop()) {
      WPAD_ScanPads();
      PAD_ScanPads(); // only if initialized/needed
      // Handle input, update, render, copy/present according to GX example.
      VIDEO_WaitVSync();
  }
  // Release any application-owned resources here.
  ```

- **Reproducible builds:** record `powerpc-eabi-gcc --version`, devkitPPC release, libogc/libogc2 version, package versions, ABI, console target and build command. CI may cross-compile/link and package DOLs; **CI success is not hardware validation**. For libogc2, its README documents `ghcr.io/extremscorner/libogc2:latest` as an example container; pin a digest/version for repeatability. [HIGH]
- Asset pipeline:
  - Textures: pre-convert to GX tiled formats offline where possible (`gxtexconv` ships with devkitPro; TPL files) — runtime swizzling of PNGs costs load time. [HIGH]
  - Audio: pre-convert to 16-bit big-endian PCM at 32/48 kHz. [HIGH]
  - Data files: CPU is **big-endian**, but formats differ: parse each file's specified endianness explicitly (WAV usually little-endian, network byte order big-endian, some formats define their own). **Do not blindly byteswap all binary data**; portable readers should load scalar values field-by-field and validate bounds. [HIGH]
- Threading: **libogc 3.x + devkitPPC r49 supports pthread/C11/`std::thread`**, with LWP-compatible APIs still available. libc/POSIX completeness is not guaranteed; check headers/real examples. On one Broadway core, threads improve organization and I/O responsiveness, not CPU-core throughput. Do not mix libogc3 semantics with libogc2 without verifying. [HIGH]
- Anti-patterns: avoid generic PPC tuning that loses available Wii optimizations; avoid desktop-only pthread/socket/libm features that the selected SDK does not implement; never mix libogc and libogc2 binary portlibs. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary: **Dolphin**. Accuracy is excellent for retail-game GPU behavior; very good for libogc homebrew. [HIGH]
- Safe to rely on: GX rendering semantics (TEV, formats), general control flow, WPAD emulation basics, DOL loading, virtual SD image mounting, logging (`OSReport`/stderr visible in Dolphin log). [HIGH]
- NOT safe to rely on:
  - **Cache coherency.** Dolphin (default JIT) does not emulate the D-cache; missing `DCFlushRange`/`DCInvalidateRange` bugs are invisible in Dolphin and fatal on hardware. This is the classic "works in Dolphin, garbage on Wii" cause. [HIGH]
  - **Performance.** Dolphin FPS bears no relation to hardware FPS in either direction. Never optimize from Dolphin timings. [HIGH]
  - Exact timing (VI/DSP/IOS latencies), alignment exceptions, real Bluetooth pairing behavior, SD/USB timing, uninitialized-memory contents. [HIGH]
  - EFB peek/poke and some copy edge cases depend on Dolphin accuracy settings. [MEDIUM]
- Hardware debugging:
  - **USB Gecko** (EXI slot B serial adapter): printf over `usbgecko` + a GDB stub in libogc (`_break()`, `DEBUG_Init(GDBSTUB_DEVICE_USB, 1)`). Rare hardware but the gold standard. [HIGH]
  - **wiiload** over Wi-Fi: fast deploy; combine with on-screen console (`CON_Init`/`printf`) for logging. [HIGH]
  - libogc installs a default **exception handler** that dumps a register + stack trace ("code dump") on screen: `DSISR/DAR` for data faults, `SRR0` = faulting PC. Map SRR0 back with `powerpc-eabi-addr2line -e app.elf 0x800xxxxx`. [HIGH]
  - Net-based logging: trivial UDP `printf` sink is 30 lines of code and worth it. [HIGH]
- Profiling: measure **on real hardware** (frame total, update, renderer submission, GX waits/copies, filesystem stalls, audio underflows, MEM1/MEM2 high-water, file-load latency). Measure before asserting a renderer/wrapper overhead or a specific frame-rate cause. Keep CPU/GPU workloads distinct; Dolphin timing is not representative. [HIGH]
- Diagnostic evidence bundle: Git commit, target console, display mode, SDK + compiler release, exception SRR0/DAR/DSISR, ELF/map file for `addr2line`, reproduction steps, controller/storage model and what differs from Dolphin. [HIGH]
- Anti-pattern: Do NOT ship anything that has only ever run in Dolphin. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, console alive (HBC returns on reset) | After VIDEO/GX init changes | Missing `VIDEO_SetBlack(FALSE)` / no `VIDEO_Flush()` / XFB not set or not in uncached alias | Follow canonical init order in §3; XFB via `MEM_K0_TO_K1` | [HIGH] |
| Black screen only on PAL console | Hardcoded NTSC mode | `rmode` hardcoded instead of `VIDEO_GetPreferredMode` | Use preferred mode; handle 528-line PAL EFB height | [HIGH] |
| Textures garbage/checkerboard on hardware, fine in Dolphin | Texture upload path | Missing `DCFlushRange` after writing texture/vertex data | Flush every buffer GX reads; align to 32 | [HIGH] |
| Random polygons / GPU hang mid-frame | Display lists or FIFO | Display list not 32-byte aligned/padded, or FIFO overflow | `memalign(32)`, pad size to 32, flush; enlarge FIFO | [HIGH] |
| Static/crackle in audio | Streaming audio | Unflushed audio buffers or callback underrun | `DCFlushRange` each buffer; bigger ring; no decode in callback | [HIGH] |
| Silence, everything else works | Audio init | Wrong sample rate constant or voice never started; PCM in little-endian | Init AESND at 48k, byteswap samples | [HIGH] |
| Crash on boot before main() output | Startup | .dol conversion of ELF with unsupported sections, or stack overflow from huge statics | Rebuild with template Makefile; move big arrays off stack/bss into heap | [MEDIUM] |
| Crash (DSI, DAR≈small value) after loading large file | Asset loading | malloc returned NULL (MEM1 exhausted) and unchecked | Allocate bulk data from MEM2 arena; check returns | [HIGH] |
| Crash with SRR0 in copied/generated code | Loader/JIT | Missing `ICInvalidateRange` after writing instructions | DCFlush + ICInvalidate every emitted range | [HIGH] |
| Wiimote never responds | Input init | Polling before pairing completes, or `WPAD_ScanPads` missing from loop | Poll every frame; handle `WPAD_ERR_NO_CONTROLLER`; press 1+2/red sync | [HIGH] |
| `fatInitDefault()` returns false / files missing | SD/USB access | Media absent, wrong mount path, filesystem/IOS mismatch, unsupported format | Check each mounted device/path; use supported libfat FAT setup or a tested libogc2/libdvm build | [HIGH] |
| Low frame rate, CPU-bound (confirmed by profiling) | Rendering | Vertex submission, geometry processing, game update, allocation or wrapper overhead | Measure on Wii; compare display lists, indexed arrays and native GX only where profiling shows a win | [HIGH] |
| Missing controller expansion data after setting all-channel report format | Specific WPAD versions/configurations | Potential bug in all-channel accelerometer/IR reporting | Log SDK revision & WPAD errors; reproduce per channel; check open issue #274 | [MEDIUM] |
| Compiler errors using pthread or `std::thread` | Toolchain / SDK mismatch | Pre-3.x libogc, pre-r49 devkitPPC, wrong C++/thread flags | Verify exact installed versions and matching threading examples | [HIGH] |
| Squashed/stretched image | Display | Rendering 4:3 projection on 16:9 console setting (anamorphic) | Read `CONF_GetAspectRatio()`, widen projection | [HIGH] |
| Mid-session slowdown scaling with texture count | Rendering | No mipmaps → TMEM thrash | Generate/upload mip chains, set `GX_LIN_MIP_LIN` | [HIGH] |

---

## 11. ANTI-PATTERNS

1. Do NOT use OpenGL/GLSL mental models — there are no shaders; think TEV stages. [HIGH]
2. Do NOT skip `DCFlushRange` after CPU-writing anything GX, DSP, AI, or IOS will read. [HIGH]
3. Do NOT skip `DCInvalidateRange` before CPU-reading DMA/IOS-written buffers. [HIGH]
4. Do NOT allocate GX/IOS/DMA buffers without `memalign(32, size_rounded_to_32)`. [HIGH]
5. Do NOT assume any file's endianness from the host CPU; parse per **file format specification**, not by blanket byteswapping. [HIGH]
6. Do NOT trust Dolphin for cache bugs or performance numbers. [HIGH]
7. Do NOT write to NAND. Ever, unless that is the app's explicit purpose. [HIGH]
8. Do NOT hardcode NTSC 640×480 — query `VIDEO_GetPreferredMode` and `CONF_GetAspectRatio`. [HIGH]
9. Do NOT assume a controller or expansion is present; probe relevant supported layouts and connection changes (and only poll APIs you initialized). [HIGH]
10. Do NOT malloc multi-MB assets from default heap (MEM1) — use the MEM2 arena. [HIGH]
11. Do NOT do file I/O or heavy allocation inside the audio callback or render loop. [HIGH]
12. Do NOT use immediate-mode `GX_Position3f32` spam for static geometry — record display lists. [HIGH]
13. Do NOT emit or expect AltiVec/VMX; SIMD means paired singles. [HIGH]
14. Do NOT `IOS_ReloadIOS` casually — it can lose AHBPROT and invalidate device state. [HIGH]
15. Do NOT rely on stencil buffers or destination alpha in RGB8 EFB mode — plan materials around it. [HIGH]
16. Do NOT treat an unmerged PR as a released public API or treat a current `master` branch as your installed version. [HIGH]
17. Do NOT mix standard libogc portlibs with libogc2: they are not ABI-compatible. [HIGH]
18. Do NOT turn a measured performance question into an unconditional claim about OpenGX/SDL2. [HIGH]
19. Do NOT assume `meta.xml` and `icon.png` are HBC launch requirements; they are optional (but recommended). [HIGH]

---

## 12. PORTING DECISION TREE

1. **Select platform + SDK; confirm memory budget.** Record target (original Wii / Family Edition / Wii Mini / vWii) and use **either** standard libogc3 or libogc2. Wii offers 24 MB MEM1 + 64 MB MEM2 physically, **less reserved regions**; usable working-set capacity varies with IOS, loader, binaries and allocations. Measure actual arenas and account for streaming. [HIGH]
2. **Audit binary format endianness and alignment.** Find `fread` into structs, packed fields, casts, network code and saved games. Parse specified byte order and validate bounds instead of blindly swapping every integer. [HIGH]
3. **Stand up the reproducible build:** selected SDK's current Wii example/Makefile or CMake, stub main, `boot.dol` in HBC, console/logging, Git revision and tool versions. [HIGH]
4. **Bring up video + input skeleton** (solid color via GX copy, WPAD/PAD polling, clean exit to HBC). Establishes the frame loop contract. Skip it → no validation platform for later stages. [HIGH]
5. **Choose renderer strategy and profile it.** Map shader materials to GX/TEV approximations. Consider native GX, GRRLIB, OpenGX or SDL2 for the project's needs. Compare on real Wii before deciding that a GL wrapper must be replaced. Use proper texture blocks/mipmaps and indexed arrays or display lists where beneficial. [HIGH]
6. **Select filesystem for the SDK.** Standard libogc/libfat FAT32 is the conservative route; libogc2/libdvm may enable exFAT. Confirm launch media, actual mounts and path, handle failure/hot-unplug; use bulk MEM2 and async/preloaded assets as appropriate. [HIGH]
7. **Audio to AESND** double-buffered ring, offline-resampled assets. Comes after video because audio bugs don't block visual validation. Skip it → shippable-looking build that crackles. [HIGH]
8. **Threading model:** in libogc3 consider LWP-compat, pthread, C11 or C++ threads as appropriate; in libogc2 verify its supported thread APIs. One core means no multicore speedup, but task separation/I/O overlap can reduce stalls. Minimize contention and validate shutdown. [HIGH]
9. **Hardware profiling pass:** measure frame time and waits; then use mipmaps, TEV collapsing, paired singles, MEM1 placement and display lists when benchmarks justify them. Do not invent expected FPS or percentage gains. [HIGH]
10. **Hardware soak + target matrix + packaging:** test PAL/NTSC/progressive modes as supported, both aspect ratios, controllers/hotplug, power/reset, SD/USB failures and vWii/Wii Mini restrictions if claimed. Include recommended `meta.xml` and `icon.png` (optional for HBC). [HIGH]

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Nintendo Wii — PPC750CL "Broadway" @729MHz, 32-bit BIG-ENDIAN, hard-float, paired-single SIMD, no AltiVec
GPU: "GX"/Flipper-class @243MHz — FIXED FUNCTION, 16 TEV stages, NO shaders, NO stencil; 2MB EFB + 1MB TMEM
RENDER: draw→EFB, GX_CopyDisp→XFB(YUV, main RAM), VIDEO_Flush+WaitVSync; max 640x528; NTSC 60 / PAL 50 / EuRGB60
RAM: MEM1 24MB 1T-SRAM @0x8000_0000 (fast, default heap) | MEM2 64MB GDDR3 @0x9000_0000 (arena2; IOS owns top)
MAGIC NUMBER: 32 — cache line, alignment for GX/DMA/IOS/display lists (memalign(32), size%32==0)
CACHE: write-back, NOT coherent with GX/DMA/IOS → DCFlushRange after CPU write, DCInvalidateRange before CPU read,
       ICInvalidateRange after writing code (JIT/loader)
SDK: Choose standard libogc 3.x OR libogc2 (different portlib ABI); use matching examples/toolchain; 2026 ref: devkitPPC r49.1, libogc 3.1.0
THREAD: libogc3 + r49 supports LWP compatibility, pthread, C11 threads, std::thread; single CPU core
BUILD: ELF -> elf2dol -> sd:/apps/<name>/boot.dol; meta.xml + icon.png OPTIONAL but recommended; AHBPROT request via meta.xml
INPUT: WPAD_ScanPads/PAD_ScanPads once per frame; probe expansion type every frame; IR via WPAD_FMT_BTNS_ACC_IR
AUDIO: AESND/ASND, 48kHz s16 BIG-ENDIAN stereo, callback double-buffer, DCFlushRange every buffer
STORAGE: libogc/libfat FAT32; libogc2/libdvm supports exFAT; verify mounts. NAND normally read-only.
NETWORK: libogc3 standard BSD socket shims plus legacy net_*; verify actual supported functions
DEBUG: Dolphin (hides cache bugs & perf!), USB Gecko GDB stub, on-screen code dump: SRR0=crash PC → addr2line
TOP HARDWARE BUGS: missing cache maintenance, invalid GX format tiling, alignment, endian-assumption bugs
SOURCE STATE: release notes > installed headers/examples > main branch > open PR/issue (never assume merged)
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this document is used in an **active Wii development session** rather than research/Q&A, work in small testable increments. **Never claim real-hardware success without a user report or hardware evidence.** Keep the active goal explicit. You may fix build errors or clarify evidence without pretending a hardware test was performed. Prefer actionable progress to needless state-machine deadlocks.

### Development states
- **ANALYSIS:** inspect actual code, target hardware, SDK/version, memory/GX/endianness/controller constraints; list files and proposed changes.
- **IMPLEMENTATION:** make a bounded change and review it; explain any nontrivial assumptions and compatibility risks.
- **BUILD_REQUEST:** provide a command matching the project's build system, expected ELF/DOL location and how to deploy it; distinguish **not yet tested** from validated.
- **WAITING_FOR_HARDWARE:** request the relevant physical-console test when the milestone depends on Wii-specific behavior. Do not invent logs, outputs or behavior. You may analyze newly supplied build errors/reports while waiting.
- **VALIDATION:** classify observed results as **PASS**, **PARTIAL**, **FAIL**, **BLOCKED** or **NOT_TESTED**; capture evidence, exact build and platform versions.
- **DIAGNOSTICS:** for a failure, isolate one plausible cause at a time, supply a minimal test or focused checklist, and re-test.
- **NEXT_GOAL:** once acceptance criteria are satisfied (or the user explicitly chooses to bypass a blocker), propose the next milestone. Do not silently implement unrelated future goals.

### Required build-and-test handoff

```text
=== WII HARDWARE TEST REQUEST ===
GOAL: <one concrete capability>
TARGET: <Wii model / vWii; SDK and version; video mode if relevant>
BUILD: <exact command>
ARTIFACT: <actual path to ELF/DOL from build system>
DEPLOY: <sd:/apps/<app>/boot.dol or confirmed wiiload method>
EXPECTED: <observable screen/audio/input/storage behavior>
CHECK: <3-6 concrete steps, including clean exit and failure case>
REPORT: PASS / PARTIAL / FAIL / BLOCKED / NOT_TESTED, plus what happened
EVIDENCE: <logs, code dump, SRR0/DAR/DSISR, screenshot, timings as appropriate>
=== END TEST REQUEST ===
```

- The user need **not** type a magic word. Treat natural language such as “video works but the Wiimote doesn't” as **PARTIAL** and retain its details.
- **Compile-only validation is not hardware validation.** Dolphin-only success also remains **NOT_TESTED on physical hardware**.
- If the build fails, address the compiler/linker error first; no physical-console test is needed to investigate a deterministic compile error.
- After repeat failures, **escalate diagnostics**, not automatically abandon the goal: request a code dump and map file, USB Gecko/GDB or a Dolphin-vs-Wii comparison; offer to mark **BLOCKED** if the user cannot collect evidence.
- If a feature is dangerous (NAND/IOS/bootloader writes), require a separate explicit safety review and avoid unneeded device modifications.

### Default goal stack (override when the user gives a specific goal)
1. Set target matrix, SDK versions, reproducible build and safe logging.
2. Video output (solid color via GX copy-clear); test NTSC/PAL mode if relevant.
3. Controller input (WPAD/PAD, connection and expansion changes, power/reset lifecycle).
4. Audio output (buffered test tone, no callback stalls).
5. Storage mount and bounded asset read with absent-media handling.
6. Textured UI/menu (correct GX swizzle/mips, 4:3/16:9, frame sync).
7. Main menu interaction with clean exit; transition to game state.
8. Real-hardware profiling and soak; release packaging.

---

## 15. HARDWARE / PLATFORM COMPATIBILITY MATRIX

Decide what physical platform is targeted **before** making port assumptions. Product family differences can invalidate storage/input/video/IOS advice. [HIGH/MEDIUM]

| Platform | Typical differences affecting homebrew | Required validation |
|---|---|---|
| Original Wii **RVL-001** | Four built-in GC controller ports and memory-card slots; SD, USB, and supported networking | GC PAD and WPAD, SD/USB, component/progressive vs interlaced where applicable |
| Wii Family Edition **RVL-101** | Built-in GameCube ports removed | Do not require direct `PAD_*` GC controllers; test alternative inputs |
| **Wii Mini** | No standard front SD slot or built-in Wi-Fi; restricted connectivity/use paths compared with full Wii | Do not assume `sd:/` or Wi-Fi/wiiload; verify usable loader/USB/controller features |
| Wii U **vWii** | Wii compatibility environment with distinct IOS/system restrictions and no built-in original-Wii GC ports | Verify HBC/IOS behavior, video/aspect, USB/storage and each input option; do not apply original Wii AHB/IOS hacks blindly |

- Do not imply every accessory or controller adapter works through the same API on all variants; require a tested backend. Wii Mini and vWii installation/entry paths are distinct topics from native RVL-001 behavior. [MEDIUM]
- Display selection: query `VIDEO_GetPreferredMode()` and console aspect configuration rather than forcing NTSC or PAL. Treat 480p availability and RGB/component output as device/cable/configuration-dependent. [HIGH]

---

## 16. THREADS, LIFECYCLE, AND NETWORKING (SDK-VERSION AWARE)

### Standard libogc 3.x / devkitPPC r49+
- **Tuxedo** replaces the former underlying LWP implementation while retaining an LWP-compatible interface; `pthread`, C11 and C++ `std::thread` support were explicitly introduced with devkitPPC r49 integration. This does **not** make Wii fully equivalent to desktop POSIX. [HIGH] ([libogc 3.0 release](https://github.com/devkitPro/libogc/releases), [devkitPPC r49 release](https://github.com/devkitPro/buildscripts/releases))
- New examples use `while (SYS_MainLoop())` as the preferred normal application loop, handling standard reset/power conditions. Explicit resource cleanup remains the application's responsibility. [HIGH] ([wii-examples releases](https://github.com/devkitPro/wii-examples/releases))
- BSD socket headers and compatibility shims complement existing `net_*` networking APIs; verify implementation of less-common calls (for example, [reported `select` gap #250](https://github.com/devkitPro/libogc/issues/250)). [HIGH/MEDIUM]
- Threaded work is a correctness and responsiveness design decision on **one CPU core**. Use queues/locks carefully, avoid polling WPAD concurrently, and never block audio callbacks on file I/O or heavy allocation.

### libogc2
- Use libogc2's **own** headers, compatible packages and build rules. Do not assume libogc3-specific Tuxedo, `SYS_MainLoop` or threading semantics exist identically in libogc2; check its current examples and migration documentation. [HIGH]
- Package notes: the `libogc2-libdvm` provider can replace libfat with no normal app-code changes while enabling additional filesystems; this does not change HBC media requirements. [HIGH]

---

## 17. PERFORMANCE, BUILD HYGIENE, AND RELEASE CHECKLIST

### Hardware performance procedure
1. Fix build/version and use identical content + display mode for before/after comparisons.
2. Measure frame-time distribution (median and worst-case), audio underruns, input responsiveness, memory use, GX stalls and file-load spikes **on a real Wii**.
3. Diagnose CPU update vs GX submission vs GPU/EFB copy vs VI waiting; do not treat CPU utilization or emulator FPS as GPU proof.
4. Change one bottleneck at a time: indexed vertex arrays/display lists, texture CMPR/mips, allocation placement, batch size, TEV stages, cache flush ranges.
5. Report actual results and regressions. **Never assert a 30–50% cost for OpenGX or a fixed FPS boost without measurement.**

### Build and dependency hygiene
- Save a build manifest: commit, target model, SDL/GRRLIB/OpenGX versions, devkitPPC and SDK release, complete compiler/linker command, platform flags, library path and build container digest if used.
- Validate C/C++ build and DOL packaging in CI; maintain a separate human or hardware-rig test gate. The [libogc2 README](https://github.com/extremscorner/libogc2/blob/master/README.md) documents its separate container and migration workflow.
- Use licensed original/game assets and do not assume access to Nintendo's proprietary Revolution SDK or copyrighted game resources.
- Never commit to hardware-specific APIs merely because they appear in an unmerged patch or fork.

### Release acceptance checklist
- [ ] Runs on declared physical target(s), not only Dolphin.
- [ ] Correct region/video/aspect/progressive/interlaced behavior for supported targets.
- [ ] Controller connect/disconnect, expansion hotplug and clean HOME/power handling.
- [ ] No unflushed GX/DSP/IOS buffers; valid cache lines, tiled texture layout, and no unaligned DMA.
- [ ] Low-memory, missing asset, absent SD/USB, corrupt file and hot-unplug paths fail safely.
- [ ] Audio ring buffer does not underrun in normal play; callback avoids blocking work.
- [ ] `boot.dol` starts from intended HBC media; optional metadata/icon included for good presentation.
- [ ] `meta.xml` requests `<ahb_access/>` **only if needed**, and privilege behavior is verified.
- [ ] Git revision, SDK/build toolchain versions and test outcomes are recorded.

---

## 18. GITHUB WATCHLIST: RELEASED VS PROPOSED (CHECK STATUS BEFORE USING)

| Item | Status at 2026-11-13 | Consequence for code generation |
|---|---|---|
| [devkitPro/libogc 3.1.0](https://github.com/devkitPro/libogc/releases) | Released 2026-05-03 | Tuxedo, standard threading/sockets and later 3.1 fixes are real; check installed version |
| [devkitPPC r49.1](https://github.com/devkitPro/buildscripts/releases) | Released 2026-01-29 | Supports the compiler/newlib side of modern threading |
| [wii-examples v20260510](https://github.com/devkitPro/wii-examples/releases) | Released 2026-05-10 | Preferred reference for current lifecycle and filesystem examples |
| [GRRLIB 4.6.1](https://github.com/GRRLIB/GRRLIB/releases) | Released 2026-08-02 | CMake and new TPL/texture APIs arrived in 4.6.0; 4.6.1 fixes installation |
| [libogc2](https://github.com/extremscorner/libogc2) + [libdvm](https://github.com/extremscorner/libdvm) | Separate maintained ecosystem | May offer exFAT and broader hardware support; use only matching ABI/portlibs |
| [OpenGX](https://github.com/devkitPro/opengx) | Available wrapper | Production use is workload-dependent, not categorically invalid |
| [`wii-sdl2`](https://github.com/devkitPro/pacman-packages/blob/master/wii/SDL2/PKGBUILD) | Package recipe references SDL2 2.28.5 | Cross-platform library option; verify compatibility and limits |
| [WPAD2/bt-embedded PR #258](https://github.com/devkitPro/libogc/pull/258) | **Open PR** | Experimental Bluetooth expansion; **not** part of a standard release |
| [MotionPlus PR #260](https://github.com/devkitPro/libogc/pull/260) | **Open PR** | Do not call proposed `WPAD_HasMotionPlus()` unless building that branch |
| [WPAD issue #274](https://github.com/devkitPro/libogc/issues/274) | **Open issue** | Potential IR/expansion regression; reproduce before attributing bugs |

**Freshness policy:** For any new code generation or upgrade touching these topics, check release tags first, then matching installed headers/examples. Report when a feature is unmerged, platform-dependent or untested on the user's setup. Avoid silently 'upgrading' a user's working project to a different ABI.

---

## 19. PRIMARY SOURCES FOR FUTURE MAINTENANCE

**SDK, toolchains, examples**
- https://github.com/devkitPro/libogc/releases
- https://github.com/devkitPro/libogc/tree/master/gc
- https://github.com/devkitPro/buildscripts/releases
- https://github.com/devkitPro/wii-examples/releases
- https://github.com/devkitPro/wii-examples/tree/master/graphics/gx
- https://github.com/extremscorner/libogc2/blob/master/README.md
- https://github.com/extremscorner/libdvm

**Rendering, storage, higher-level libraries**
- https://github.com/dolphin-emu/dolphin/blob/master/Source/Core/VideoCommon/TextureDecoder_Common.cpp
- https://github.com/devkitPro/opengx
- https://github.com/GRRLIB/GRRLIB/releases
- https://github.com/devkitPro/pacman-packages/blob/master/wii/SDL2/PKGBUILD
- https://wiibrew.org/wiki/Homebrew_Channel

**Active upstream proposals and issues (not feature guarantees)**
- https://github.com/devkitPro/libogc/pull/258
- https://github.com/devkitPro/libogc/pull/260
- https://github.com/devkitPro/libogc/issues/274
- https://github.com/devkitPro/libogc/issues/250

**Maintenance rule:** Refresh the release table, exact version strings and upstream PR statuses when this file is revisited; leave durable hardware facts separate from quickly changing SDK guidance. Do not copy repository code into this document unless licensing and platform compatibility are verified.
