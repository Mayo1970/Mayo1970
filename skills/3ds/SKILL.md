---
name: 3ds
description: Expert operational brief for Nintendo 3DS homebrew development (devkitARM, libctru, citro3d, PICA200, NDSP, 3dsx/CIA). Use this skill whenever the user mentions 3DS, New 3DS, Old 3DS, libctru, citro3d, citro2d, PICA200, devkitARM, 3dsx, CIA, Luma3DS, Rosalina, Azahar/Citra, Homebrew Launcher, or wants to port, build, debug, or optimize any game or engine (ioquake3, SDL, emulators, decomps) for the Nintendo 3DS. Also use it for 3DS hardware questions (screens, VRAM, DSP, ARM11), build failures, black screens, or performance work on 3DS. When used in an active development session, you MUST follow the Goal-Oriented Workflow with Hardware Validation Gates in Section 14.
---

# SKILLS_3DS.md — Nintendo 3DS Homebrew Expertise Document

You are developing homebrew for the Nintendo 3DS family (Old 3DS/3DS XL/2DS and New 3DS/New 3DS XL/New 2DS XL). Follow these instructions exactly. Prefer open-source community SDKs (devkitARM + libctru) in all cases. Do NOT reference, link to, or assume access to Nintendo's proprietary CTR SDK.

---

## 1. HARDWARE ARCHITECTURE

### CPU
- Main CPU: ARM11 MPCore cluster. Old 3DS: 2 cores @ 268 MHz. New 3DS: 4 cores @ 804 MHz (can also run at 268 MHz for compatibility). [HIGH]
- ISA: ARMv6K, 32-bit, little-endian. Thumb-2 is NOT available (ARMv6K has Thumb-1 only); compile ARM-mode code. [HIGH]
- FPU: VFPv2 per core, hardware single/double precision. Use hard-float ABI. No NEON — ARMv6 SIMD is limited to 32-bit packed integer ops. [HIGH]
- Caches: 16 KB L1 I-cache + 16 KB L1 D-cache per core. New 3DS adds a shared 2 MB L2 cache — a major reason N3DS is far faster than the clock ratio suggests. [HIGH]
- Core usage: core0 = application core (yours). core1 = system core; on Old 3DS you may only borrow a slice of it via `APT_SetAppCpuTimeLimit` (typically 30%, max ~80% depending on mode). On New 3DS, cores 2–3 are available to applications. [HIGH]
- Security co-CPU: ARM9 (ARM946E-S @ 134 MHz, 2× on N3DS) runs Process9, handling NAND, SD raw access, AES/SHA/RSA crypto. You talk to it only indirectly through FS/PS services. [HIGH]
- Quirk: unaligned loads/stores mostly work on ARMv6K but are slow and can trap depending on CP15 config — keep data naturally aligned. [MEDIUM]

### GPU
- DMP PICA200 @ 268 MHz. Hybrid pipeline: **programmable vertex and geometry shaders** (custom SIMD ISA, assembly only) + **fixed-function fragment stage** with 6 configurable texture-combiner (TEV-like) stages, hardware per-fragment lighting via lookup tables, fog, procedural textures, and a shadow/gas unit. There are NO fragment shaders. [HIGH]
- Screens: top 400×240 (800×240 in stereoscopic/wide modes), bottom 320×240. Framebuffers are physically rotated 90°: a "240-wide × 400-tall" buffer, filled column-major relative to what the user sees. citro3d/citro2d hide this; raw framebuffer code must not ignore it. [HIGH]
- Max texture 1024×1024, minimum 8×8, power-of-two dimensions only. Textures are stored in 8×8 tiles with Morton (Z-order) swizzling — use `tex3ds` at build time or `C3D_TexUpload`-compatible conversion at runtime. [HIGH]
- Texture formats: RGBA8, RGB8, RGBA5551, RGB565, RGBA4, LA8, HILO8, L8, A8, LA4, L4, A4, ETC1, ETC1A4. ETC1(A4) is the compressed format of choice. [HIGH]
- Color buffers: RGBA8/RGB8/RGBA5551/RGB565/RGBA4. Depth: 16-bit, 24-bit, or D24S8 (24-bit depth + 8-bit stencil). Full blending equations and stencil ops supported. [HIGH]
- Rendering is command-list driven (GX command lists submitted via gsp). citro3d builds and submits these for you. [HIGH]

### RAM
- FCRAM: 128 MB (Old 3DS) / 256 MB (New 3DS). Application-usable heap: ~64 MB in standard Old3DS mode, up to ~124 MB in New3DS application mode. Under the Homebrew Launcher you inherit the host title's memory mode. [HIGH]
- VRAM: 6 MB total, two 3 MB banks (A/B), physical base 0x18000000. Fastest memory for framebuffers and hot textures. Not CPU-cached; CPU access is slow — write via GX DMA or allocate with `vramAlloc` and fill with `GX_TextureCopy`. [HIGH]
- DSP RAM: 512 KB, private to the audio DSP. ARM9 internal RAM: 1 MB (1.5 MB N3DS), not accessible to your code. [HIGH]
- Virtual memory: yes, the ARM11 has an MMU and processes run in virtual address space. Key virtual regions: heap at 0x08000000, linear heap (GPU-visible, physically contiguous) at 0x30000000 (0x31000000 on older firmware ranges — use libctru's `__ctru_linear_heap` machinery, never hardcode), VRAM mapped at 0x1F000000. [HIGH]
- Alignment: GPU buffers (vertex data, textures, display transfers) must be in linear memory or VRAM and aligned; use `linearAlloc`/`linearMemAlign` (0x80 alignment covers all GX cases). [HIGH]

### Bus / DMA
- CPU ↔ FCRAM ↔ GPU share bandwidth; VRAM has a dedicated fast path to the GPU. Framebuffers in VRAM, render targets in VRAM, bulk assets in FCRAM linear heap. [HIGH]
- GX DMA engine handles DisplayTransfer (tiled→linear + downscale/format conversion), TextureCopy, and memory fill. Use it instead of memcpy for anything GPU-visible. [HIGH]
- Old 3DS practical bottlenecks: CPU clock (268 MHz) and FCRAM bandwidth. Fill rate is rarely the limit at 400×240. [MEDIUM]

### Co-processors
- Audio DSP: CEVA TeakLite-class DSP running Nintendo's firmware ("DSP component"). libctru's NDSP driver gives you 24 hardware-mixed voices. Homebrew must load a dumped DSP firmware blob (see §6). [HIGH]
- New 3DS extras: QTM (head-tracking) module and hardware MVD (H.264 decode) service; niche, partially documented. [MEDIUM]

### Security / boot
- Boot chain: boot9/boot11 ROMs → ARM9 firmware → ARM11 kernel; everything signed (RSA). The boot9strap exploit (sighax) gives persistent boot-time code execution; Luma3DS is the standard CFW. [HIGH]
- Homebrew runs as a normal userland process. You cannot touch ARM9 memory, raw NAND, or other processes unless you explicitly use Luma/Rosalina extensions. With Luma, kernel access is possible but you must NOT require it for a normal game/port. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: Nintendo CTR SDK ("CTR_SDK", used with IS-CTR debug hardware). Do NOT use, reference, or assume it. [HIGH]
- Homebrew: **devkitARM** toolchain + **libctru** (OS/services/input/audio) + **citro3d** (GPU) + **citro2d** (2D on top of citro3d), all from devkitPro, permissively licensed (libctru: zlib). Install via devkitPro pacman, package group `3ds-dev`. [HIGH]
- What homebrew CAN do: full GPU (vertex/geometry shaders, all combiner features), 24-voice DSP audio, all input including gyro/accel/touch/C-stick, SD and RomFS filesystems, sockets/HTTP/SSL, camera, mic, stereoscopic 3D, multi-threading with core affinity, New 3DS 804 MHz mode. [HIGH]
- What homebrew CANNOT (or should not) do: eShop/account services, local wireless UDS is partially supported (works, but less battle-tested [MEDIUM]), StreetPass/SpotPass, official game-card access beyond what FS exposes, Miiverse-era services (dead anyway). No official-style C fragment shaders exist on ANY SDK — that's hardware. [HIGH]
- Critical gaps affecting design: vertex shaders are written in **assembly** (picasso); there is no GLSL/HLSL path. Any engine expecting programmable fragment shading must be redesigned around the 6-stage combiner. [HIGH]

---

## 3. GRAPHICS PIPELINE

- Use **citro3d** for 3D and **citro2d** for 2D. Do NOT write raw GPU register pokes unless citro3d demonstrably cannot express what you need. Do NOT attempt to port OpenGL directly; there is no GL driver. (A GL-on-PICA shim, `picaGL`, exists for porting GL1.x code — acceptable bootstrap, expect ~50–70% of native performance. [MEDIUM]) [HIGH]

### Initialization (canonical)
```c
gfxInitDefault();                       // raw framebuffer mode (2D CPU rendering)
// -- or, for GPU rendering --
gfxInitDefault();
C3D_Init(C3D_DEFAULT_CMDBUF_SIZE);
C3D_RenderTarget* top = C3D_RenderTargetCreate(240, 400, GPU_RB_RGBA8, GPU_RB_DEPTH24_STENCIL8);
C3D_RenderTargetSetOutput(top, GFX_TOP, GFX_LEFT,
    GX_TRANSFER_FLIP_VERT(0) | GX_TRANSFER_OUT_FORMAT(GX_TRANSFER_FMT_RGB8));
```
Note the target is created 240×400 (rotated), and your projection matrix must bake in the rotation — use `Mtx_OrthoTilt` / `Mtx_PerspTilt` from citro3d, which exist exactly for this. [HIGH]

### Frame flow
- GPU renders to a VRAM render target → `C3D_FrameEnd(0)` queues a DisplayTransfer to the LCD framebuffer → gsp flips on VBlank. Double buffering is handled by gfx/citro3d. [HIGH]
- Frame loop skeleton: `while (aptMainLoop()) { hidScanInput(); C3D_FrameBegin(C3D_FRAME_SYNCDRAW); ...draw...; C3D_FrameEnd(0); }`. `C3D_FRAME_SYNCDRAW` gives you VSync. [HIGH]
- Refresh rate: ~59.83 Hz on all consoles and both screens. There is no NTSC/PAL distinction — it's a handheld with a fixed LCD. [HIGH]
- Stereoscopic 3D: render the scene twice (GFX_LEFT/GFX_RIGHT targets) with an interaxial offset; enable with `gfxSet3D(true)`; read the slider via `osGet3DSliderState()`. Costs ~2× vertex/fill work. [HIGH]

### Shaders
- Vertex/geometry shaders: PICA assembly, assembled with **picasso** (`.v.pica` → `.shbin`, embedded by the Makefile). ~4 KB instruction memory, 96 float uniform vectors, no dynamic loops beyond hardware loop counters. [HIGH]
- Fragment stage: configure up to 6 `C3D_TexEnv` combiner stages (per-stage sources: textures 0–3, vertex color, fragment lighting, constant; ops: modulate, add, interpolate, dot3, etc.). Think "Wii TEV, slightly friendlier." [HIGH]

### Anti-patterns (GPU)
- Do NOT upload textures every frame from cached heap memory without `GSPGPU_FlushDataCache` — you will get stale/garbage texels. [HIGH]
- Do NOT use non-power-of-two textures; pad to POT and adjust UVs. [HIGH]
- Do NOT render at 800×240 wide mode expecting free AA — it doubles fill/vertex cost. [MEDIUM]
- Do NOT forget the 90° screen rotation when writing raw framebuffer code: pixel (x,y) on screen is `fb[(x*240 + (239-y)) * bpp]` for the top screen in the default orientation. [HIGH]

---

## 4. INPUT

- Built-in: D-pad, A/B/X/Y, L/R, Start/Select, Circle Pad, touch screen (resistive, single-touch), accelerometer, gyroscope, 3D slider, volume slider, shell state. New 3DS adds C-stick and ZL/ZR. Circle Pad Pro (IR peripheral) adds C-stick/ZL/ZR to Old 3DS. [HIGH]
- Model: **polled**. Call `hidScanInput()` exactly once per frame, then read:
  - Buttons: `hidKeysDown()` (pressed this frame), `hidKeysHeld()`, `hidKeysUp()` — bitmasks (`KEY_A`, `KEY_START`, …). [HIGH]
  - Circle Pad: `hidCircleRead(&pos)` → s16 x/y roughly in −156..156. Apply your own deadzone (~15–20). [HIGH]
  - Touch: `hidTouchRead(&t)` → 320×240 coordinates; only valid while `KEY_TOUCH` is held. [HIGH]
  - C-stick / ZL / ZR: `KEY_CSTICK_*`, `KEY_ZL/ZR` — delivered through the same key API when `irrstInit()`/N3DS HID extensions are active; on N3DS libctru handles it transparently. [MEDIUM]
  - Accelerometer/gyro: `HIDUSER_EnableAccelerometer()` / `HIDUSER_EnableGyroscope()`, then `hidAccelRead` / `hidGyroRead`. Disable when unused (battery). [HIGH]
- No rumble in the console itself. [HIGH]
- Disconnects don't exist for built-in controls; handle only Circle Pad Pro pairing loss (treat as centered stick, not an error). [MEDIUM]
- Anti-pattern: do NOT call `hidScanInput()` from multiple threads or multiple times per frame — edge detection (`KeysDown`) breaks. [HIGH]
- Anti-pattern: do NOT map anything essential to ZL/ZR/C-stick without an Old 3DS fallback. [HIGH]

---

## 5. MEMORY LAYOUT

| Region | Virtual base | Size | Use | Notes |
|---|---|---|---|---|
| Application heap | 0x08000000 | up to ~64 MB (O3DS) / ~124 MB (N3DS mode) | malloc/new | cached, fast for CPU [HIGH] |
| Linear heap | 0x30000000 | shares app allocation | `linearAlloc` — GPU-visible buffers | cached for CPU; MUST flush before GPU reads [HIGH] |
| VRAM | 0x1F000000 | 6 MB | `vramAlloc` — render targets, hot textures | uncached, slow CPU access, fastest for GPU [HIGH] |
| Shared/IPC, TLS, stacks | kernel-managed | — | — | do not touch manually [HIGH] |

- Allocation APIs: `malloc` (heap), `linearAlloc/linearMemAlign/linearFree`, `vramAlloc/vramFree`, `mappableAlloc` (rare). All vertex buffers, textures uploaded via GX, and audio buffers for NDSP must come from **linear** memory (or VRAM for textures/targets). [HIGH]
- Split heap: libctru divides the app memory between normal heap and linear heap. Override with `u32 __ctru_heap_size;` / `u32 __ctru_linear_heap_size;` globals before main if a port needs a bigger linear pool (e.g., lots of textures). Default linear heap is 32 MB. [MEDIUM] — verify with `linearSpaceFree()` at boot.
- Stack: main-thread stack defaults to 32 KB. Ports of PC code (Quake-family especially) overflow this instantly. Set `u32 __stacksize__ = 0x100000;` (or more) at global scope. Threads get their stack size from `threadCreate`. [HIGH]
- Cache: L1 D-cache is write-back. The GPU does NOT snoop it. After the CPU writes any buffer the GPU will read (vertices, textures, command data outside citro3d's management), call `GSPGPU_FlushDataCache(addr, size)`. citro3d flushes its own internal buffers, not yours. NDSP similarly requires flushed wave buffers — `DSP_FlushDataCache` (ndsp examples do this). [HIGH]
- DMA/GX alignment: allocate GPU buffers with at least 0x80 alignment (`linearMemAlign(size, 0x80)`); `linearAlloc` already guarantees this. [MEDIUM]
- Anti-pattern: do NOT pass stack or plain-heap pointers to GX/citro3d/NDSP. Silent corruption or hangs. [HIGH]
- Anti-pattern: do NOT put large streaming textures in VRAM if you update them from the CPU every frame — CPU writes to VRAM are painfully slow; stage in linear RAM and `GX_TextureCopy`. [HIGH]

---

## 6. AUDIO

- Hardware: dedicated audio DSP (TeakLite-class) mixing up to **24 voices** in hardware; output 32728 Hz stereo to speakers/headphones. [HIGH]
- API: **NDSP** in libctru (`ndspInit`, `ndspChnSetFormat`, `ndspChnWaveBufAdd`, …). Do NOT use the legacy CSND path for applications. [HIGH]
- CRITICAL: NDSP requires the DSP firmware blob. The user must dump it once on their console with the **DSP1** homebrew, producing `sdmc:/3ds/dspfirm.cdc`. Without it, `ndspInit()` fails and you get silence. Always check the return code and surface a clear on-screen message. [HIGH]
- Formats: PCM8, PCM16, and DSP-ADPCM (Nintendo 4-bit ADPCM, ~3.5:1; encode with `dspadpcm`-compatible tools). Per-voice sample-rate conversion, volume, panning, and simple IIR filters are done on the DSP for free. [HIGH]
- Buffering model: queue-of-wave-buffers per channel. You submit `ndspWaveBuf` structs (data in **linear memory**, flushed); the DSP consumes them; you poll `status == NDSP_WBUF_DONE` or use `ndspSetCallback` (called on the audio thread every ~4.6 ms frame). Keep ≥2 buffers of 4096+ samples queued per streaming channel to survive frame spikes. [HIGH]
- For music, prefer decoding (stb_vorbis/dr_mp3) on a worker thread into a ring of wave buffers; on Old 3DS put the decoder thread on core1 after `APT_SetAppCpuTimeLimit(30)`. [MEDIUM]
- Anti-patterns: do NOT decode/decompress inside the NDSP callback (it must return in well under 4.6 ms); do NOT reuse a wave buffer before it reports DONE; do NOT allocate audio data with plain `malloc`. [HIGH]

---

## 7. STORAGE / IO

- Accessible media: SD card (`sdmc:/`), embedded read-only RomFS (`romfs:/`, packed into the 3dsx/CIA), save data archives, and the network. No USB mass storage. Game cards are not usable for homebrew data. [HIGH]
- Init: SD is mounted automatically by libctru's runtime; RomFS needs `romfsInit()` (and `romfsExit()`). Standard C stdio (`fopen("romfs:/textures/wall.t3x", "rb")`) works on both. [HIGH]
- SD is FAT32/exFAT — treat paths as **case-insensitive but case-preserving**; RomFS lookups are case-sensitive as authored. Ports from case-sensitive filesystems: normalize your asset names. [MEDIUM]
- SD throughput is modest (~2–10 MB/s depending on card and access pattern); prefer few large reads over many small ones. [MEDIUM]
- Homebrew Launcher layout: `sdmc:/3ds/<AppName>/<AppName>.3dsx` (+ optional `<AppName>.smdh` if not embedded — the standard Makefile embeds it). Icon/title/author come from the SMDH built by `smdhtool`. CIA installs need `makerom` + `bannertool` and a Luma/FBI-equipped console. Ship 3dsx as the primary format. [HIGH]
- Network: 802.11b/g Wi-Fi. BSD sockets via `soc:U`:
```c
static u32* socbuf;
socbuf = memalign(0x1000, 0x100000);      // 4KB-aligned, multiple of 0x1000
socInit(socbuf, 0x100000);
```
TCP/UDP both work; also `httpc` (HTTP) and `sslc` (TLS) services. No Ethernet. [HIGH]
- Anti-patterns: do NOT write to NAND or system save archives; do NOT assume the SD is present (HBL requires it, but CIA installs can run without — check `fopen` results); do NOT hold files open across `aptMainLoop` sleep/home transitions without handling suspend. [MEDIUM]

---

## 8. BUILD SYSTEM

- Toolchain: **devkitARM** (from devkitPro pacman; install group `3ds-dev`, which pulls libctru, citro3d, citro2d, picasso, tex3ds, 3dsxtool, smdhtool). Keep the toolchain current; do not pin ancient releases. [HIGH]
- Cross prefix: `arm-none-eabi-` (devkitARM build). Target: ARM/ARMv6K EABI, little-endian, hard-float. [HIGH]
- Required flags (from the canonical 3ds Makefile):
  - CFLAGS: `-march=armv6k -mtune=mpcore -mfloat-abi=hard -mtp=soft -mword-relocations -ffunction-sections` plus `-D__3DS__` (older code checks `_3DS`). `-mtp=soft` is mandatory — the kernel reserves the hardware TLS register. [HIGH]
  - LDFLAGS: `-specs=3dsx.specs -march=armv6k -mtune=mpcore -mfloat-abi=hard -mtp=soft`
  - LIBS: `-lcitro2d -lcitro3d -lctru -lm` (order matters), with `-L$(DEVKITPRO)/libctru/lib`. [HIGH]
- Output chain: ELF → **3dsxtool** → `.3dsx` (+SMDH). Optional CIA: ELF → `makerom` with an RSF file. The devkitPro example Makefiles (`$(DEVKITPRO)/examples/3ds`) do all of this — start from one; do not hand-roll. [HIGH]
- Environment: `DEVKITPRO=/opt/devkitpro`, `DEVKITARM=$DEVKITPRO/devkitARM`, and `$DEVKITPRO/tools/bin` on PATH. [HIGH]
- Asset pipeline:
  - Textures: `tex3ds` at build time → `.t3x` (handles POT padding, tiling/swizzle, ETC1 compression, mipmaps). Load with `Tex3DS_TextureImport*`. [HIGH]
  - Shaders: `.v.pica` → picasso → `.shbin`, auto-embedded as `<name>_shbin` symbols by the Makefile. [HIGH]
  - Audio: pre-convert to 16-bit PCM or DSP-ADPCM at ≤32728 Hz offline. [MEDIUM]
  - Big asset sets: put them in RomFS (`ROMFS := romfs` in the Makefile) rather than requiring loose SD files. [HIGH]
- Anti-patterns: do NOT use `-mfloat-abi=soft` (throws away the VFP); do NOT omit `-mtp=soft` (random crashes in threaded code); do NOT link desktop libs expecting POSIX completeness — newlib on 3DS lacks fork, mmap, signals. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary emulator: **Azahar** (the 2025 merger of the Lime3DS and PabloMK7 forks that continued Citra after its March 2024 discontinuation). Use it for iteration speed; homebrew 3dsx loads directly. [HIGH]
- Emulator gets RIGHT (safe to rely on): general libctru/citro3d behavior, screen layout/rotation, RomFS/SD emulation, basic NDSP audio, most GPU features, sockets (via host network). [MEDIUM]
- Emulator gets WRONG or hides (must test on hardware):
  - Performance. The emulator's FPS says nothing about a 268 MHz ARM11. NEVER optimize from emulator framerate. [HIGH]
  - Cache coherency. Missing `GSPGPU_FlushDataCache` often works in the emulator and corrupts on hardware. [HIGH]
  - Memory limits/heap split, stack overflows, exact linear-heap exhaustion behavior. [MEDIUM]
  - DSP timing edge cases, mic, camera, IR, StreetPass, precise stereoscopic output, battery/lid events. [MEDIUM]
- Hardware debugging: **Luma3DS + Rosalina** provides a Wi-Fi **GDB stub** (Rosalina menu → Debugger → enable; connect `arm-none-eabi-gdb` to `<console-ip>:4003`). This is the single most valuable debugging tool — use it before adding printf archaeology. [HIGH]
- Crash dumps: Luma writes exception dumps to `sdmc:/luma/dumps/arm11/*.dmp`; parse with `luma3ds_exception_dump_parser` to get PC/LR/registers/stack, then map addresses via `arm-none-eabi-addr2line -e app.elf`. Always ask the user for the dump on any hardware crash. [HIGH]
- `consoleInit(GFX_BOTTOM, NULL)` + printf on the bottom screen is the quickest on-device tracing; gdb `monitor`-less builds can also log to `sdmc:/` files. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, HBL returns or hangs | Right after launch | Crash before first frame: missing `gfxInitDefault()`, stack overflow (default 32 KB), or uncaught service init failure | Set `__stacksize__`, init gfx first, check every `Result` with `R_FAILED` | [HIGH] |
| Black/frozen top screen but bottom console works | GPU path | `C3D_FrameEnd` never reached, render target/output not bound, or waiting on wrong screen | Verify `C3D_RenderTargetSetOutput` flags and frame loop order | [HIGH] |
| Garbage/flickering textures or vertices | First frames or after asset load | Missing `GSPGPU_FlushDataCache` after CPU writes to linear buffers | Flush every CPU-written GPU buffer before draw | [HIGH] |
| Texture renders as noise/diagonal stripes | Any textured draw | Raw (linear) pixel data uploaded without 8×8 Morton tiling, or NPOT size | Use tex3ds `.t3x` assets; pad to power-of-two | [HIGH] |
| Silence, `ndspInit` returns error | Audio init | `dspfirm.cdc` missing on SD | Have user run DSP1 dump; check init result and display message | [HIGH] |
| Audio crackles/stutters | During gameplay spikes | Wave buffer underrun (too few/too small buffers) or decode in callback | ≥2×4096-sample buffers queued; move decode to worker thread | [HIGH] |
| Crash when loading a large file | Ports with big assets | Heap/linear heap exhaustion (64 MB O3DS) or reading into undersized stack buffer | Check `linearSpaceFree()`; stream instead of whole-file loads | [HIGH] |
| Input dead / keys stuck | Frame loop | `hidScanInput()` missing, called multiple times, or called off-thread | Exactly one call per frame on the main thread | [HIGH] |
| `fopen` fails on assets that exist | RomFS or SD | `romfsInit()` not called, or case mismatch (RomFS is case-sensitive) | Call romfsInit; normalize asset name casing | [HIGH] |
| Runs on N3DS, slideshow on O3DS | Performance | Code budgeted at 804 MHz + L2; O3DS is 268 MHz, no L2 | Profile on O3DS; enable N3DS clock via `osSetSpeedupEnable(true)` only as bonus | [HIGH] |
| Prefetch/data abort in `arm11/*.dmp` at wild address | Any | Dangling pointer, or GPU buffer freed while in flight | Parse dump, `addr2line` the PC/LR; keep buffers alive until `C3D_FrameEnd` fences | [HIGH] |
| Image squashed/rotated 90° | Raw framebuffer code | Ignoring the physically rotated 240×400 framebuffer | Use `Mtx_*Tilt` projections or index fb as column-major | [HIGH] |
| Threads on core1 never run (O3DS) | Multithreaded port | No `APT_SetAppCpuTimeLimit` before creating core1 threads | Call it (e.g., 30) once at boot; check `threadCreate` affinity result | [HIGH] |

---

## 11. ANTI-PATTERNS

1. Do NOT assume programmable fragment shaders exist — design all material work around the 6-stage combiner. [HIGH]
2. Do NOT skip `GSPGPU_FlushDataCache` after CPU-writing any GPU-read buffer; the emulator will forgive you and hardware will not. [HIGH]
3. Do NOT allocate GPU or NDSP buffers with `malloc` — use `linearAlloc`/`vramAlloc`. [HIGH]
4. Do NOT keep the default 32 KB main stack when porting PC code — set `__stacksize__`. [HIGH]
5. Do NOT use non-power-of-two or >1024px textures. [HIGH]
6. Do NOT budget performance on New 3DS or the emulator — the Old 3DS at 268 MHz is your floor. [HIGH]
7. Do NOT call `hidScanInput()` more than once per frame or from worker threads. [HIGH]
8. Do NOT run your frame loop without `aptMainLoop()` — HOME, sleep, and power events will misbehave. [HIGH]
9. Do NOT build with soft-float or without `-mtp=soft`. [HIGH]
10. Do NOT write raw framebuffer code as if the screen were 400×240 row-major — it is 240-wide and rotated. [HIGH]
11. Do NOT put per-frame CPU-updated buffers in VRAM; stage in linear RAM and GX-copy. [HIGH]
12. Do NOT decode audio or touch the filesystem inside the NDSP callback. [HIGH]
13. Do NOT require ZL/ZR/C-stick or the N3DS clock — degrade gracefully on Old 3DS. [HIGH]
14. Do NOT rely on POSIX features newlib lacks (fork, mmap, signals, pthreads-by-default — use libctru threads or enable devkitPro's pthread shims deliberately). [MEDIUM]
15. Do NOT ship SD-loose assets when RomFS embedding works — it eliminates an entire class of "file not found" reports. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Boot a skeleton (gfx + console + aptMainLoop) before touching engine code.** Priority: proves toolchain, HBL deployment, and your test loop. Skip it and every later failure is unattributable. [HIGH]
2. **Raise the stack and audit memory budget.** Sum the engine's static + heap needs against 64 MB (O3DS). Skip it → mysterious boot crashes and OOM at first level load. [HIGH]
3. **Stub the renderer; get the game loop running with a solid-color screen and console logging.** Confirms endianness (non-issue: LE), 32-bit assumptions, and file I/O before graphics complexity. Skip it → you debug logic and GPU simultaneously. [HIGH]
4. **Filesystem layer → RomFS/sdmc paths, case normalization, big-read patterns.** Skip it → asset loading "works on my PC" failures. [HIGH]
5. **Renderer decision:** GL1.x-style engine → picaGL bootstrap, then migrate hot paths to citro3d; anything else → citro3d directly, materials mapped onto TexEnv stages, vertex work into picasso shaders. Skip the mapping analysis → you discover mid-port that a required fragment effect is inexpressible. [HIGH]
6. **Input mapping with Old 3DS-complete controls** (Circle Pad + touch as extra buttons/mouse). Skip it → unplayable on 90% of consoles. [HIGH]
7. **Audio via NDSP, DSP-ADPCM assets, worker-thread streaming.** Comes after video because silence is shippable, stutter is not. Skip buffer discipline → crackle reports forever. [HIGH]
8. **Old 3DS performance pass:** profile with `svcGetSystemTick`/on-screen timers, move work to core1 (`APT_SetAppCpuTimeLimit`), cut overdraw, ETC1 everything. Skip it → N3DS-only port. [HIGH]
9. **Hardening:** HOME/sleep handling via apt hooks, error surfaces for missing dspfirm/SD, exception-dump-friendly release builds (keep the ELF!). Skip it → undebuggable field reports. [HIGH]

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM  Nintendo 3DS — ARM11 MPCore ARMv6K LE, 2c@268MHz (O3DS) / 4c@804MHz+2MB L2 (N3DS), VFPv2, no NEON, no Thumb-2
GPU       DMP PICA200 @268MHz: ASM vertex/geometry shaders (picasso), FIXED fragment w/ 6 TexEnv stages, no frag shaders
SCREENS   top 400x240 (+stereo), bottom 320x240 touch; framebuffers ROTATED 90° (240xH) — use Mtx_*Tilt; ~59.83Hz
RAM       128MB FCRAM (app ~64MB) O3DS / 256MB N3DS; 6MB VRAM; linearAlloc=GPU-visible, vramAlloc=targets/hot tex
CACHE     GPU doesn't snoop L1: GSPGPU_FlushDataCache(ptr,size) after every CPU write to GPU/DSP buffers
TEXTURES  POT only, max 1024², 8x8 Morton tiled — build with tex3ds → .t3x; ETC1/ETC1A4 for compression
AUDIO     NDSP: 24 DSP voices, PCM8/16 + DSP-ADPCM, out 32728Hz; REQUIRES sdmc:/3ds/dspfirm.cdc (DSP1 dump)
INPUT     polled: hidScanInput() once/frame; hidKeysDown/Held/Up, hidCircleRead, hidTouchRead; C-stick/ZL/ZR = N3DS/CPP only
FS        romfs:/ (romfsInit, case-sensitive, embed assets) + sdmc:/ (FAT, case-insensitive); HBL: /3ds/App/App.3dsx
NET       socInit(memalign(0x1000, 0x100000), 0x100000); BSD sockets + httpc/sslc; Wi-Fi b/g only
BUILD     devkitARM + libctru + citro3d/2d (pacman: 3ds-dev); arm-none-eabi-; -march=armv6k -mtune=mpcore
          -mfloat-abi=hard -mtp=soft; -specs=3dsx.specs; ELF→3dsxtool→.3dsx (+SMDH); CIA via makerom (optional)
STACK     default 32KB — set: u32 __stacksize__ = 0x100000;   CORE1 (O3DS): APT_SetAppCpuTimeLimit(30) first
DEBUG     Azahar emulator (never trust its FPS); Luma3DS Rosalina GDB stub :4003; crashes → sdmc:/luma/dumps/arm11
SPEED     osSetSpeedupEnable(true) = N3DS 804MHz bonus, never a requirement; budget for O3DS 268MHz
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this document is used in an active development session (not pure reference/Q&A), you MUST follow this state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify 3DS-specific blockers (memory, shaders, stack, cache flushes), and plan.
- **STATE: IMPLEMENTATION** — Write/modify code based on this document. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output exact build commands; ask the user to compile and run on real hardware.
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back. Classify as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next milestone and ask for confirmation.

### STATE TRANSITION RULES
1. **ANALYSIS → IMPLEMENTATION**: only after you have listed the specific files to modify to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: only after providing a complete, compilable change, including:
   - Exact build command (typically `make` in the project root with devkitPro env set)
   - Expected output file (e.g., `output/AppName.3dsx`)
   - Transfer method (copy to `sdmc:/3ds/AppName/`, or `3dslink AppName.3dsx` with HBL netloader active)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: you MUST output this exact header, then STOP generating — no fixes, no guesses:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render top-screen solid color framebuffer"]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including error codes, black screen, crashes, /luma/dumps files]
   === STOP ===
   ```
4. **WAITING_FOR_HARDWARE → VALIDATION**: triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - "SUCCESS" → NEXT_GOAL. "FAILURE" → DEBUG_PROTOCOL.
   - Anything else ("kind of works", "almost") → ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: output a **DEBUG BUILD PROTOCOL** — either a minimal C test case isolating the failure, OR exactly 3 specific diagnostic steps (e.g., "check `ndspInit()` return on the bottom-screen console", "confirm `GSPGPU_FlushDataCache` covers the vertex buffer", "pull `sdmc:/luma/dumps/arm11/` and paste the parsed dump"). Ask the user to run it and report back. You must NOT rewrite the implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: return to WAITING_FOR_HARDWARE after providing the protocol.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: propose the next milestone from the goal stack. Do NOT implement until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [Rosalina GDB stub / luma exception dump parse / Azahar-vs-hardware comparison / GBAtemp-devkitPro forum].`
- Ask the user whether to (a) mark the goal BLOCKED and skip, or (b) provide GDB session output / register dumps for further analysis.

### GOAL STACK (default, if the user does not provide one)
1. Initialize video output (solid color on top screen via gfx framebuffer)
2. Initialize controller input (print pressed keys on bottom-screen console)
3. Initialize audio output (NDSP sine wave; verifies dspfirm.cdc)
4. Load assets from RomFS/SD (read a file, print size + checksum)
5. Render main menu framebuffer (citro2d text + sprite)
6. Main menu input loop (navigate with D-pad/Circle Pad + touch)
7. Transition to game state

Work ONLY on the active goal (top of stack). Do NOT implement future goals speculatively.
