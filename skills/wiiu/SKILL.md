---
name: wiiu
description: Expert Wii U homebrew development knowledge base and workflow enforcer. Use this skill WHENEVER the user works on Wii U homebrew in any form — porting engines (ioquake3, SDL games), writing GX2 renderer code, wut/devkitPPC build issues, Aroma/WUHB packaging, MEM1/MEM2/foreground-bucket memory questions, VPAD/KPAD input, AX audio, Cemu-vs-hardware discrepancies, cache-coherency bugs, or R700/Latte shader work. Also trigger on mentions of "Wii U", "GX2", "wut", "Aroma", "Cafe OS", "Espresso", "Latte", ".rpx", ".wuhb", or CafeGLSL, even if the user does not explicitly ask for the skill. When used in an active dev session, the goal-oriented hardware-validation state machine in Section 14 is MANDATORY.
---

# SKILLS: Nintendo Wii U Homebrew Development

You are operating as a Wii U homebrew systems expert. Follow every rule in this document. Confidence tags: [HIGH] = verified by working public code/docs, [MEDIUM] = likely correct but verify, [LOW] = uncertain, run the proposed test before relying on it.

---

## 1. HARDWARE ARCHITECTURE

### CPU — "Espresso"
- Tri-core IBM PowerPC 750-derived design (evolution of the Wii's Broadway), clocked at 1.243875 GHz. 32-bit, **big-endian**. [HIGH]
- Per-core L1: 32 KB instruction + 32 KB data. Asymmetric L2: core 0 = 512 KB, core 1 = 2 MB, core 2 = 512 KB. Schedule your heaviest thread on core 1. [HIGH]
- FPU: hard-float with **paired singles** (2× float32 SIMD, same as GameCube/Wii). There is NO AltiVec/VMX. Do NOT emit AltiVec intrinsics. [HIGH]
- Cache lines are 32 bytes, write-back. Caches are NOT coherent with GPU DMA — you must flush manually (Section 5). [HIGH]
- No out-of-order execution wizardry: it is a short-pipeline in-order-ish 750 core. Branch-heavy interpreter loops run acceptably; memory latency to MEM2 is the usual bottleneck. [MEDIUM]

### GPU — "Latte"
- AMD R700-family design (internally "GPU7", roughly RV730-class), ~550 MHz, 320 unified stream processors. Fully programmable: vertex/pixel/geometry shaders in **R700 ISA**. [HIGH]
- Latte is also the northbridge: it contains the memory controller, the ARM926 "Starbuck" security/IO processor (runs IOSU), and the legacy Wii "Hollywood" GX block used only in vWii mode. [HIGH]
- Output: TV up to 1920×1080, plus a second independent render target streamed to the GamePad (DRC) at 854×480. You render BOTH every frame if you want both displays live. [HIGH]
- 32 MB of fast on-package MEM1 acts as the preferred location for color/depth render targets. [HIGH]
- Texture support: 2D/3D/cube/array, NPOT, mipmaps, BC1–BC5 (DXT1/3/5 + ATI1/2) compression, sRGB. Max 2D texture size 8192×8192. [MEDIUM — R700 spec; verify 8192 on hardware with a large-texture test]

### RAM
- 2 GB DDR3-1600 total ("MEM2"). Cafe OS reserves ~1 GB; a foreground app gets roughly 1 GB of MEM2. [HIGH]
- **MEM1**: 32 MB fast graphics memory. Allocate render targets here. [HIGH]
- **Foreground bucket**: 40 MB region available ONLY while your app is in the foreground. TV/DRC scan buffers must live here. You MUST free it when losing foreground (HOME menu overlay) via ProcUI callbacks. [HIGH]
- MEM0 (~3 MB) is kernel-only; not usable. [HIGH]
- Cafe OS runs the PPC MMU with fixed mappings; app code/data lives in effective addresses around 0x00800000 (code) and 0x10000000+ (data). You do not manage page tables. There is no swap. [MEDIUM]
- Alignment: GPU-visible buffers commonly need 256-byte alignment (uniform blocks), surfaces need the alignment returned by `GX2CalcSurfaceSizeAndAlignment()` — often 0x800–0x1000 for tiled surfaces. Never guess surface alignment. [HIGH]

### Bus / DMA
- The CPU has no direct path to peripherals; everything routes through Latte. GX2 command buffers are written by the CPU into MEM2 and fetched by the GPU via DMA — hence the strict cache-flush discipline. [HIGH]
- `GX2CopySurface` / DMAE provide GPU-side blits; prefer them over CPU memcpy for surface data. [MEDIUM]

### Co-processors
- **Starbuck (ARM926)**: runs IOSU, the security/IO OS. Handles FS, network, USB, and crypto. The PPC talks to it via IPC (`/dev/...` IOS handles). Homebrew normally goes through Cafe OS libs (coreinit, nn) rather than raw IPC; extended FS rights come from Mocha/Aroma's IOSU exploit. [HIGH]
- **DSP**: an audio DSP exists but homebrew audio goes through the AX voice mixer API (sndcore2) — you do not program the DSP directly. [HIGH]

### Security / DRM
- Boot chain: boot0 → boot1 → IOSU → Cafe OS, all signature-checked. Homebrew entry today is via the **Aroma** environment (typically PayloadFromRPX / UDPIH / ISFShax installation paths). Under Aroma, your homebrew runs as a normal userspace Cafe OS title (.rpx/.wuhb) with Mocha providing patched IOSU FS access. [HIGH]
- You CAN: access SD, USB storage, network, all input devices, full GX2. You CANNOT (from plain userspace): touch IOSU internals, raw NAND, or other titles' memory without additional exploits. Do not design around kernel access. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: Nintendo **Cafe SDK** (leaked). You must NOT use it, reference its headers, paths, or build flags. All code must compile with the community toolchain. [HIGH]
- Homebrew: **wut** (Wii U Toolchain, devkitPro) + **devkitPPC** cross-compiler. Zlib-licensed, actively maintained: https://github.com/devkitPro/wut. Companion ecosystem: **libwhb** (bundled helpers), **libmocha** (IOSU FS access), **WUPS** (plugin system), **wums** (module system), **SDL2 Wii U port**, **CafeGLSL** (runtime GLSL→R700 shader compiler). [HIGH]
- wut CAN: everything needed for games — GX2, AX, VPAD/KPAD, FS, sockets, threads, ProcUI. Its headers are reverse-engineered but broadly complete for game development. [HIGH]
- wut CANNOT (vs official): 
  - No offline shader compiler. The official SDK compiled shaders from source; homebrew must either (a) use **CafeGLSL** at runtime, (b) hand-assemble R700 with latte-assembler-style tools, or (c) ship prebuilt `.gsh` binaries. This is THE defining feature gap and drives renderer architecture. [HIGH]
  - Some nn:: subsystems (accounts, e-shop, Miiverse-era) are partially or un-stubbed — irrelevant for ports. [MEDIUM]
- Decision rule: for a Q3-class renderer, prefer a small fixed set of hand-written/precompiled R700 shaders over runtime GLSL translation. Translation layers (ANGLE, GL-on-GX2) have repeatedly failed to reach shippable quality on this platform. [HIGH]

---

## 3. GRAPHICS PIPELINE

- API: **GX2** (wut `<gx2/*.h>`). There is no OpenGL, no Vulkan, no ANGLE that works acceptably. Do NOT attempt GL translation layers for performance-critical code. [HIGH]

### Initialization (canonical order)
1. `GX2Init(attributes)` — pass a command-buffer pool attribute list (or NULL for defaults). [HIGH]
2. Query TV mode: `GX2GetSystemTVScanMode()`; compute scan buffer sizes with `GX2CalcTVSize()` / `GX2CalcDRCSize()`. [HIGH]
3. Allocate TV and DRC **scan buffers from the foreground bucket** (`MEMGetBaseHeapHandle(MEM_BASE_HEAP_FG)` + `MEMAllocFromFrmHeapEx`), then `GX2SetTVBuffer()` / `GX2SetDRCBuffer()`. [HIGH]
4. Allocate color + depth `GX2ColorBuffer`/`GX2DepthBuffer` — prefer **MEM1** frame heap; fall back to MEM2 if it doesn't fit. Initialize via `GX2InitColorBuffer` etc., alignment from the surface struct. [HIGH]
5. `GX2SetupContextStateEx(state, TRUE)` and `GX2SetContextState(state)` — one context state per render thread. [HIGH]
6. `GX2SetTVEnable(TRUE)` / `GX2SetDRCEnable(TRUE)`. [HIGH]

### Frame loop
- Render to your color buffer → `GX2CopyColorBufferToScanBuffer(cb, GX2_SCAN_TARGET_TV)` (and `_DRC`) → `GX2SwapScanBuffers()` → `GX2Flush()` → `GX2WaitForFlip()` (or `GX2WaitForVsync()`). Set swap interval with `GX2SetSwapInterval(1)` for 60 Hz lock. [HIGH]
- Refresh: 59.94 Hz on all Wii U HDMI output regardless of region; there is no PAL 50 Hz concern over HDMI. [MEDIUM — verify only if targeting component/PAL analog out]
- Double buffering is the norm (2 scan buffers per target); GX2 handles flip queueing. [HIGH]
- You MUST pump **ProcUI** every frame (`ProcUIProcessMessages`) and handle `PROCUI_STATUS_RELEASE_FOREGROUND` by freeing the foreground bucket, else HOME menu hard-locks the console. [HIGH]

### Shaders
- R700 ISA. wut structs: `GX2VertexShader`, `GX2PixelShader`, `GX2FetchShader` (fetch shaders are generated at runtime from `GX2AttribStream[]` via `GX2InitFetchShaderEx` — this is how vertex layouts bind). [HIGH]
- Uniforms: uniform registers (`GX2SetVertexUniformReg`) or uniform blocks (256-byte aligned buffers, `GX2SetVertexUniformBlock`). Uniform block data must be **big-endian swapped per 32-bit word** when written by the CPU. [HIGH]
- CafeGLSL (`CompileVertexShader`/`CompilePixelShader` from the runtime lib) compiles GLSL on-console; good for bring-up, adds startup cost and a runtime dependency. For release, snapshot compiled shaders to `.gsh` and load those. [HIGH]

### Textures / buffers
- `GX2Surface` + `GX2CalcSurfaceSizeAndAlignment()` is mandatory before allocating. Default is **tiled** layout; CPU-written textures are easiest as `GX2_TILE_MODE_LINEAR_ALIGNED`, but tiled is significantly faster for the GPU — for static assets, tile at load time (`GX2CopySurface` from a linear staging surface does the swizzle on-GPU). [HIGH]
- After ANY CPU write to GPU-read memory: `DCFlushRange(ptr,size)` then `GX2Invalidate(GX2_INVALIDATE_MODE_CPU_TEXTURE | ...)` with the matching mode flags. Missing this works on Cemu and breaks on hardware. [HIGH]
- Depth/stencil: D24S8, D32F supported; full blending, separate alpha blend, and color masks per target as per R700. [HIGH]

### Anti-patterns
- Do NOT render at 1080p by default; 720p color+depth fits MEM1, 1080p does not (a 1080p RGBA8+D24S8 pair ≈ 16 MB+ each with alignment). [MEDIUM]
- Do NOT interleave CPU writes into a command buffer region the GPU is consuming; use per-frame fenced ring buffers (`GX2SetSemaphore`/`OSTime` fencing or triple-buffered dynamic data). [HIGH]
- Do NOT forget the DRC: shipping a TV-only image leaves the GamePad black and looks broken to users. Minimum: copy the TV color buffer to the DRC scan target. [HIGH]

---

## 4. INPUT

- Controller types: Wii U GamePad (DRC), Wii U Pro Controller, Wii Remote (+Nunchuk/Classic/MotionPlus), USB HID devices, Wii Balance Board. [HIGH]
- Model: **polled**, once per frame.

### GamePad — VPAD
- `#include <vpad/input.h>`; `VPADInit()`; then per frame: `VPADRead(VPAD_CHAN_0, &status, 1, &error)`. [HIGH]
- `VPADStatus` gives `hold`/`trigger`/`release` button masks, two analog sticks (`leftStick`, `rightStick`, floats −1..1), touch (`tpNormal` — run through `VPADGetTPCalibratedPoint()` before use), accelerometer, gyro, magnetometer. [HIGH]
- Check `error == VPAD_READ_SUCCESS`; `VPAD_READ_NO_SAMPLES` is normal (reuse last state), `VPAD_READ_INVALID_CONTROLLER` means GamePad off. [HIGH]
- Rumble: `VPADControlMotor(VPAD_CHAN_0, pattern, length)` with an on/off byte pattern. [MEDIUM]

### Wii Remotes / Pro Controller — KPAD/WPAD
- `KPADInit()`; enable Pro Controller support with `WPADEnableURCC(TRUE)`. Poll with `KPADReadEx(chan, &data, 1, &err)` for channels 0–3. [HIGH]
- Check `data.extensionType` every read — it distinguishes Core/Nunchuk/Classic/Pro and controllers can hot-swap extensions at any time. [HIGH]

### Anti-patterns
- Do NOT assume the GamePad is the only controller; a Q3-class port should merge VPAD chan 0 + KPAD chans 0–3 into player slots. [HIGH]
- Do NOT treat `VPAD_READ_NO_SAMPLES` as disconnect. [HIGH]
- Do NOT read input from multiple threads without locking; poll once per frame on the main loop. [MEDIUM]

---

## 5. MEMORY LAYOUT

| Region | Size | Use | How to allocate |
|---|---|---|---|
| MEM2 (DDR3) | ~1 GB usable | Everything: code, heap, assets, command buffers | `malloc`/`memalign` (wut default heap) [HIGH] |
| MEM1 | 32 MB | Color/depth render targets, hottest GPU buffers | `MEMGetBaseHeapHandle(MEM_BASE_HEAP_MEM1)` → `MEMAllocFromFrmHeapEx` [HIGH] |
| Foreground bucket | 40 MB | TV/DRC scan buffers ONLY (lost on background) | `MEMGetBaseHeapHandle(MEM_BASE_HEAP_FG)` → `MEMAllocFromFrmHeapEx` [HIGH] |
| MEM0 | ~3 MB | Kernel — untouchable | n/a [HIGH] |

- MEM1 and FG are **frame heaps**: allocation is stack-like; free with `MEMFreeToFrmHeap(heap, MEM_FRM_HEAP_FREE_ALL)` or saved states. Plan allocation order accordingly. [HIGH]
- Threads: `OSCreateThread` with an explicit stack you allocate; default main-thread stack is set by the RPX (wut default ~128 KB–2 MB depending on spec — if you see stack smash in deep recursion, allocate your own thread with a bigger stack; do not assume PC-sized 8 MB stacks). [MEDIUM — confirm your build's stack via `OSCheckActiveThreads`/linker map]
- Cache: PPC write-back, 32-byte lines. Functions: `DCFlushRange` (CPU→memory, before GPU reads), `DCInvalidateRange` (memory→CPU, after GPU/DMA writes), `ICInvalidateRange` (after writing code). GPU side: `GX2Invalidate(mode, ptr, size)`. The rule: **every CPU-written, GPU-read buffer gets DCFlushRange + GX2Invalidate; every GPU-written, CPU-read buffer gets GX2 drain + DCInvalidateRange.** [HIGH]
- DMA/GPU buffer alignment: round sizes to 32-byte multiples minimum; use API-returned alignments for surfaces. Flushing a range not 32-byte aligned flushes neighbors — never flush a range that shares a cache line with data another thread is writing. [HIGH]

### Anti-patterns
- Do NOT put large textures in MEM1; reserve it for render targets. [HIGH]
- Do NOT malloc scan buffers from MEM2 default heap — display will fail or ProcUI foreground transitions will crash. [HIGH]
- Do NOT skip freeing FG-bucket memory on `PROCUI_STATUS_RELEASE_FOREGROUND`. [HIGH]

---

## 6. AUDIO

- API: **AX** voice mixer via sndcore2 (`<sndcore2/*.h>`). `AXInitWithParams(&(AXInitParams){ .renderer = AX_INIT_RENDERER_48KHZ, ... })`. [HIGH]
- Output: 48 kHz final mix, TV + DRC devices independently mixable. Up to 96 hardware-mixed voices. [HIGH]
- Voice formats: PCM8, PCM16 (big-endian!), and Nintendo 4-bit ADPCM. Set per-voice sample rate via src ratio (`AXSetVoiceSrcRatio`), per-device volume matrices (`AXSetVoiceDeviceMix`). [HIGH]
- Model: the AX renderer runs on 3 ms frames; register `AXRegisterAppFrameCallback` to refill streaming voices, or (simpler for ports) use the **SDL2 audio backend**, which wraps AX with a conventional callback — recommended for ioquake3-class ports. [HIGH]
- Voice buffers live in MEM2; `DCFlushRange` after the CPU writes sample data before the voice reads it. [HIGH]

### Anti-patterns
- Do NOT do file I/O, allocation, or heavy math inside the AX frame callback — 3 ms budget shared with the mixer; starve it and you get crackle. Double-buffer PCM and just swap pointers. [HIGH]
- Do NOT feed little-endian PCM16 — swap on load. [HIGH]
- Do NOT assume 44.1 kHz output; resample to 48 kHz or set the voice src ratio. [HIGH]

---

## 7. STORAGE / IO

- Media: SD card (primary), USB mass storage, network. Optical disc is not relevant to homebrew. [HIGH]
- Under **Aroma**, mount the SD with `WHBMountSdCard()` (libwhb) → path root `fs:/vol/external01/`, or use **libmocha** (`Mocha_MountFS`) for extended access (USB, NAND — be careful). Standard C stdio (`fopen`) works on mounted paths through wut's devoptab. [HIGH]
- Case sensitivity: FAT32 SD is case-insensitive/case-preserving. Do NOT rely on case-only filename distinctions. [HIGH]
- WUHB bundles embed a read-only romfs mounted at `fs:/vol/content/` — ship default assets there; use SD for user data/mods. [HIGH]
- App layout for Aroma: `sd:/wiiu/apps/<yourapp>/<yourapp>.wuhb`. The `.wuhb` carries metadata + icon + TV/DRC splash inside (supplied to `wuhbtool` at build time: `--name`, `--short-name`, `--author`, `--icon=128x128.png`, `--tv-image`, `--drc-image`). No separate meta.xml is needed for .wuhb. [HIGH]
- Network: BSD-style sockets from `<sys/socket.h>` (nsysnet). Bring the connection up first with `ACInitialize()`/`ACConnect()`. Wi-Fi only unless the user has the USB LAN adapter. UDP+TCP both fine — Q3 netplay works. [HIGH]
- FS calls are serviced by IOSU over IPC: they are SLOW relative to PC (milliseconds per operation). Batch reads, load big files in large chunks, never fopen/fclose per small file in a loop. [HIGH]

### Anti-patterns
- Do NOT write to NAND / system titles. SD and USB only. [HIGH]
- Do NOT do synchronous FS from the render loop; preload or use a loader thread on core 0/2. [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **devkitPPC** (via devkitPro pacman: `dkp-pacman -S wiiu-dev`) + **wut**. Keep both updated together; wut headers move. [HIGH]
- Cross prefix / triplet: `powerpc-eabi-` (e.g., `powerpc-eabi-gcc`). [HIGH]
- Env vars: `DEVKITPRO=/opt/devkitpro`, `DEVKITPPC=$DEVKITPRO/devkitPPC`. [HIGH]
- Compiler flags (set by wut rules; know them anyway): `-mcpu=750 -meabi -mhard-float -D__WIIU__ -D__WUT__`. Big-endian is the default for this target. [HIGH]
- Link: `-specs=$DEVKITPRO/wut/share/wut.specs -lwut` (plus `-lwhb`, `-lmocha`, `-lSDL2` as needed). [HIGH]
- Output chain: `.elf` → **elf2rpl** → `.rpx` → **wuhbtool** → `.wuhb`. wut's Makefile rules (`$DEVKITPRO/wut/share/wut_rules`) and CMake toolchain (`$DEVKITPRO/wut/share/wut.toolchain.cmake`, then `wut_create_rpx()` / `wut_create_wuhb()`) automate this. [HIGH]
- Minimal Makefile: copy from `wut/samples/make/helloworld`; for CMake: `cmake -DCMAKE_TOOLCHAIN_FILE=$DEVKITPRO/wut/share/wut.toolchain.cmake`. [HIGH]
- Asset pipeline:
  - Textures: ship PNG/TGA and convert to GX2 surfaces at load, or pre-swizzle offline for fast loads. [MEDIUM]
  - Shaders: develop with CafeGLSL at runtime; for release, dump compiled programs to `.gsh` and load those to cut startup time. [HIGH]
  - Audio: pre-convert to 48 kHz PCM16 big-endian or ADPCM. [HIGH]
- Deploy fast with **wiiload** (network push to Aroma's homebrew menu) during iteration; SD copy for clean tests. [HIGH]

### Anti-patterns
- Do NOT compile with `-msoft-float`; hardware is hard-float and wut libs are built hard-float — ABI mismatch links but crashes. [HIGH]
- Do NOT hand-roll ELF loading conversions; use the wut rules. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary emulator: **Cemu** (open-source, actively developed). Runs .rpx/.wuhb directly. [HIGH]
- Cemu gets RIGHT: GX2 API semantics broadly, shader behavior (it translates R700→host), FS, VPAD basics, AX at functional level. Good for logic bring-up and renderer correctness sanity checks. [HIGH]
- Cemu gets WRONG or HIDES (must test on hardware):
  - **Cache coherency**: Cemu does not model PPC/GPU cache incoherency. Missing `DCFlushRange`/`GX2Invalidate` bugs are invisible on Cemu and produce garbage/hangs on hardware. This is the #1 "works on Cemu, dies on console" cause. [HIGH]
  - **Performance**: host-GPU accelerated; Cemu FPS says nothing about console FPS. Never optimize from Cemu numbers. [HIGH]
  - MEM1/FG heap pressure, exact alignment faults, ProcUI foreground transitions, real controller quirks, timing races on the tri-core CPU. [MEDIUM]
- Hardware debugging:
  - `OSReport`/`WHBLogPrintf` + Aroma **LoggingModule**: logs over network (UDP) or to SD — attach `WHBLogUdpInit()` and listen with a UDP log client on your PC. [HIGH]
  - Aroma's crash handler shows exception type, SRR0 (crash PC), GPRs, and a stack trace on screen; map SRR0 to source with `powerpc-eabi-addr2line -e app.elf 0x...`. [HIGH]
  - `OSFatal("msg")` = printf-of-last-resort visible on screen. [HIGH]
  - No practical GDB stub for Cafe OS userspace in the standard Aroma setup; rely on logging + addr2line. [MEDIUM]

### Anti-patterns
- Do NOT ship anything validated only on Cemu. [HIGH]
- Do NOT chase a hardware-only graphics bug in the renderer logic before auditing every CPU→GPU buffer for flush/invalidate pairs. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, console alive (logs flow) | After GX2 init | Scan buffers not in FG bucket, or `GX2SetTVEnable` missing, or no `GX2CopyColorBufferToScanBuffer` before swap | Allocate scan buffers from `MEM_BASE_HEAP_FG`; verify full init order in §3 | HIGH |
| Black screen, console hard-locked | First frame or HOME press | ProcUI not pumped / FG memory not released | Add `ProcUIProcessMessages` to loop; handle RELEASE_FOREGROUND | HIGH |
| Textures garbage/flickering on hardware, fine on Cemu | Any CPU-uploaded texture | Missing `DCFlushRange` + `GX2Invalidate` after CPU write | Flush+invalidate every upload path | HIGH |
| Geometry exploded / vertices scrambled | Dynamic vertex buffers | GPU reading a ring region the CPU is rewriting (no fencing), or endian mismatch in vertex data | Fence per-frame ring segments; verify attrib formats/endianness | HIGH |
| Uniforms wrong / colors swapped | Uniform blocks | Forgot per-32-bit-word byteswap of uniform block memory | Swap words when filling uniform block buffers | HIGH |
| Audio crackle/stutter | Streaming music | Refill work too slow inside 3 ms AX frame callback | Double-buffer PCM outside callback; callback only swaps pointers | HIGH |
| Silence, no error | AX voices set up | Device mix matrix left zeroed, or PCM16 little-endian | `AXSetVoiceDeviceMix` to nonzero; byteswap samples | HIGH |
| Crash on boot, DSI exception, SRR0 in memcpy | Loading assets | Unaligned/oversized write past an FG/MEM1 frame-heap allocation | Check allocation sizes vs `GX2CalcSurfaceSizeAndAlignment`; addr2line SRR0 | HIGH |
| Crash after loading large file | Big pk3/pak load | MEM2 heap exhaustion (~1 GB incl. fragmentation) or 32-bit size assumptions | Log heap headroom (`MEMGetAllocatableSizeForExpHeapEx`); stream instead of slurp | MEDIUM |
| Input dead, GamePad works in HOME | In-game only | `VPADInit` missing, wrong channel, or treating NO_SAMPLES as disconnect | Init once; poll chan 0; reuse last sample on NO_SAMPLES | HIGH |
| SD "file not found" | fopen fails | SD not mounted (`WHBMountSdCard` not called) or wrong root (`fs:/vol/external01/` vs `sd:/`) | Mount first; print resolved path; mind devoptab prefix | HIGH |
| 15–20 FPS where 60 expected | First hardware perf test | Linear-tiled textures, per-draw uniform reg spam, or FS calls in frame loop | Tile static textures; batch state; move I/O off render thread | MEDIUM |
| Wrong aspect / image offset on TV | 480p/1080i TVs | Scan buffer sized for a different `GX2TVRenderMode` than the system mode | Size buffers from `GX2GetSystemTVScanMode` result, not a constant | HIGH |
| Works from wiiload, fails from SD icon | Aroma launch | Assets referenced by relative path; CWD differs / romfs not used | Use absolute `fs:/vol/content/` or SD paths everywhere | MEDIUM |

---

## 11. ANTI-PATTERNS (habits to unlearn)

1. Do NOT assume little-endian anywhere: file formats, network structs, vertex data, PCM — this is a big-endian machine.
2. Do NOT skip `DCFlushRange`/`GX2Invalidate` because "it renders fine" — on Cemu it always will.
3. Do NOT use OpenGL/ANGLE translation layers for the shipping renderer; write GX2 natively.
4. Do NOT allocate scan buffers or render targets with `malloc`.
5. Do NOT ignore ProcUI; a homebrew that hard-locks on the HOME button is considered broken.
6. Do NOT leave the DRC (GamePad) screen black.
7. Do NOT do blocking file I/O or shader compilation on the render thread.
8. Do NOT put streaming/large assets in MEM1.
9. Do NOT assume one controller type or a single player on VPAD.
10. Do NOT rely on multi-megabyte stacks; deep recursion needs an explicit big-stack thread.
11. Do NOT compile soft-float or with AltiVec enabled.
12. Do NOT trust Cemu framerate, timing, or absence of crashes as hardware truth.
13. Do NOT fopen hundreds of small files; IOSU IPC round-trips are slow — pack assets.
14. Do NOT write uniform blocks without the per-word byteswap.
15. Do NOT reference or depend on the leaked Cafe SDK — wut only.

---

## 12. PORTING DECISION TREE (engine → Wii U)

1. **Build a null-renderer, null-audio port that boots and logs.** Why first: proves toolchain, endianness fixes in file loading, and heap behavior before graphics complicate everything. Skip it and every later bug is ambiguous between logic and renderer.
2. **Stand up video: clear-color frame loop with ProcUI.** Why: the §3 init sequence and FG-bucket discipline are the platform's trickiest boilerplate; validate them in isolation. Skip → black screens you'll misattribute to the engine.
3. **Input next (VPAD + KPAD merged).** Why: cheap, and you need it to navigate menus for all later testing. Skip → you can't exercise the game.
4. **Filesystem/asset loading from romfs + SD with endian-safe readers.** Why: Q3-class engines assume LE file layouts; audit every `fread` into structs. Skip → corrupted assets misread as renderer bugs.
5. **Renderer: fixed small shader set first (world+lightmap, model, UI), fetch shaders per vertex format, MEM1 targets.** Why: shader infrastructure is the biggest feature gap (no offline compiler); keep the set tiny. Skip/over-scope → weeks in CafeGLSL debugging.
6. **Dynamic data ring buffers with fencing + full flush audit.** Why: correctness on hardware. Skip → "random" corruption late in the project.
7. **Audio via SDL2-AX or direct AX voices.** Why late-ish: silent games are testable; broken renderers aren't. Skip → nothing breaks, it's just unfinished.
8. **Performance pass ON HARDWARE: tiling, batching, core-1 placement, GX2 perf queries.** Why last: you need a correct baseline. Skip → shipping 20 FPS.
9. **Release hardening: .wuhb packaging, `.gsh` shader snapshots, HOME/foreground soak test, SD-launch path audit.** Skip → works for you, fails for users.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Nintendo Wii U | Aroma homebrew | wut + devkitPPC (powerpc-eabi-)
CPU: Espresso 3×PPC750 @1.24GHz, 32-bit BIG-ENDIAN, paired-singles FPU (no AltiVec),
     L2: 512K/2M/512K (put hot thread on core 1), 32-byte cache lines, write-back
GPU: Latte (AMD R700 "GPU7") @550MHz, GX2 API only (no GL), R700 ISA shaders,
     shaders via CafeGLSL (runtime) or prebuilt .gsh — NO offline compiler in wut
RAM: MEM2 ~1GB usable (malloc) | MEM1 32MB (render targets, frm heap)
     | FG bucket 40MB (scan buffers ONLY, freed on background)
COHERENCY: CPU write → GPU read: DCFlushRange + GX2Invalidate. ALWAYS. Cemu hides this.
DISPLAY: GX2Init → CalcTVSize → FG-bucket scan bufs → SetTVBuffer → ContextState →
         render → CopyColorBufferToScanBuffer(TV+DRC) → SwapScanBuffers → WaitForFlip
         Pump ProcUIProcessMessages every frame or HOME button hard-locks.
INPUT: VPADRead(chan0) GamePad; KPADReadEx(0-3)+WPADEnableURCC for Pro/Wiimote
AUDIO: AX/sndcore2, 48kHz, 96 voices, PCM16 BIG-ENDIAN or ADPCM, 3ms frames (SDL2 wraps it)
FS: WHBMountSdCard → fs:/vol/external01/ ; WUHB romfs → fs:/vol/content/ ; IOSU FS = slow, batch it
BUILD: make w/ wut_rules or cmake wut.toolchain.cmake → .elf → elf2rpl → .rpx → wuhbtool → .wuhb
DEPLOY: wiiload (net) for iteration; sd:/wiiu/apps/<app>/<app>.wuhb for release
DEBUG: WHBLogUdpInit + Aroma LoggingModule; crash screen SRR0 → powerpc-eabi-addr2line
EMULATOR: Cemu = logic/API checks only. Perf + cache bugs = REAL HARDWARE ONLY.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this skill is active in a development session (not pure reference Q&A), you MUST follow this state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **ANALYSIS** — Examine the codebase, identify Wii U-specific blockers, plan the change.
- **IMPLEMENTATION** — Write/modify code using this document. No hardware testing occurs here.
- **BUILD_REQUEST** — Output exact build commands; ask the user to compile and run on real hardware.
- **WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. No code, no speculative fixes, no debugging loops.
- **VALIDATION** — User reports results; classify SUCCESS / PARTIAL / FAILURE.
- **NEXT_GOAL** — On SUCCESS, propose the next milestone and wait for confirmation.

### TRANSITION RULES
1. **ANALYSIS → IMPLEMENTATION**: only after you list the specific files to modify.
2. **IMPLEMENTATION → BUILD_REQUEST**: only after a complete, compilable change, including: exact `make`/`cmake` command; expected output artifact (`.rpx`/`.wuhb` path); transfer method (wiiload or SD copy to `sd:/wiiu/apps/...`); and exactly what the user should observe (screen/audio/controller).
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: you MUST output this exact header, then STOP generating:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current goal, e.g., "Render clear-color frame on TV+DRC"]
   BUILD: [Command]
   DEPLOY: [wiiload <ip> app.wuhb | copy to sd:/wiiu/apps/...]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [what you saw]
     - FAILURE: [what you saw — error codes, black screen, crash screen SRR0, logs]
   === STOP ===
   ```
4. **WAITING_FOR_HARDWARE → VALIDATION**: triggered ONLY by a user message containing "SUCCESS" or "FAILURE". Anything else ("kind of works", "almost") → ask for the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: output a DEBUG BUILD PROTOCOL — either a minimal C test case isolating the failure, OR a checklist of exactly 3 diagnostics (e.g., "verify scan buffer came from MEM_BASE_HEAP_FG", "confirm DCFlushRange covers the full upload size", "log GX2GetSystemTVScanMode result"). Ask the user to run it and report back. You must NOT rewrite the whole implementation and must NOT guess-and-patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: after providing the protocol, return to waiting.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: propose the next milestone from the goal stack; do NOT implement until confirmed.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [UDP log capture via Aroma LoggingModule / crash-screen SRR0 + addr2line / Cemu-vs-hardware differential / GBAtemp-ForTheUsers community].`
- Ask whether to (a) mark the goal BLOCKED and skip, or (b) supply logs/register dumps for deeper analysis.

### DEFAULT GOAL STACK (unless the user supplies one)
1. Initialize video output (solid-color framebuffer, TV + DRC, ProcUI-clean)
2. Initialize controller input (VPAD button log over UDP)
3. Initialize audio output (48 kHz sine wave via AX)
4. Load assets from SD/romfs
5. Render main-menu framebuffer
6. Main-menu input loop
7. Transition to game state

Work ONLY on the active goal (top of stack). Never implement future goals speculatively.
