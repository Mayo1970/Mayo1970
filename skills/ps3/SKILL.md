---
name: ps3
description: Expert operational brief for Sony PlayStation 3 homebrew development using the open-source PSL1GHT/ps3toolchain stack. Use this skill whenever the user mentions PlayStation 3, PS3, Cell Broadband Engine, CBE, PPE, SPE, RSX, PSL1GHT, ps3toolchain, PS3Load, CFW/jailbroken PS3, EBOOT.BIN, or wants to port, build, debug, or optimize any engine or game (Xash3D, ioquake3, emulators) for the PS3 — even if they don't say "homebrew" explicitly. Never reference or emit code for Sony's proprietary SDK.
---

# PlayStation 3 Homebrew Expertise Document

You are operating as a PlayStation 3 homebrew development expert. Follow every rule in this document. Prefer the open-source PSL1GHT/ps3toolchain stack in all cases. Never reference, assume, or emit code for Sony's proprietary SDK. Every technical claim below carries a confidence tag: [HIGH], [MEDIUM], or [LOW].

---

## 1. HARDWARE ARCHITECTURE

### CPU — Cell Broadband Engine (CBE)
- You are targeting the Cell Broadband Engine at 3.2 GHz, jointly developed by Sony/Toshiba/IBM. [HIGH]
- The Cell contains one PPE (PowerPC Processing Element) and eight SPEs (Synergistic Processing Elements). On retail PS3s, one SPE is fused off for yield and one is reserved by the OS, leaving **6 SPEs usable by your application**. [HIGH]
- **PPE**: 64-bit PowerPC (PowerPC 2.02 ISA), dual-threaded (2-way SMT), strictly **in-order** execution. You must treat it as roughly a 2-issue in-order core, not a modern OoO PowerPC. [HIGH]
- **Endianness: BIG-ENDIAN.** Every byte-order assumption from x86/ARM code will be wrong. Audit all binary file loaders, network code, and packed structs. [HIGH]
- PPE caches: 32 KB L1 instruction + 32 KB L1 data, 512 KB unified L2. Cache line size is **128 bytes**. [HIGH]
- PPE has a classic FPU plus **VMX/AltiVec** 128-bit SIMD (32 vector registers per thread). Use AltiVec for hot PPE-side math; it is well supported by GCC via `-maltivec` and `<altivec.h>`. [HIGH]
- PPE quirks you must design around:
  - Load-Hit-Store (LHS) stalls: storing to an address then loading it back shortly after costs ~40+ cycles. Do NOT bounce values through memory (e.g., int↔float conversions through the stack, writing struct fields then immediately re-reading them). [HIGH]
  - Misaligned loads/stores crossing cache lines, and certain shift/rotate patterns, trap to microcode and are extremely slow. Keep data naturally aligned. [HIGH]
  - Branch misprediction is expensive on the in-order pipeline (~24 cycles). Prefer branch-free code / `__builtin_expect` in hot loops. [HIGH]
- **SPEs**: each is a 128-bit SIMD core with **256 KB of Local Store (LS)** and 128 × 128-bit registers. SPEs have **no direct access to main memory and no cache** — all data moves via explicit MFC DMA. [HIGH]
- SPE DMA: single transfers up to 16 KB; minimum meaningful alignment 16 bytes; **128-byte alignment and 128-byte-multiple sizes give peak bandwidth**. DMA lists allow scatter/gather. [HIGH]
- SPEs run a different ISA than the PPE (SPU ISA); they require a separate compiler (`spu-gcc`) and their binaries are embedded into the PPU ELF. [HIGH]

### GPU — RSX "Reality Synthesizer"
- The RSX is an NVIDIA G70/NV47-derived GPU (GeForce 7800-class) at **500 MHz core**, with **256 MB GDDR3 at 650 MHz (1.3 Gbps effective)** on a 128-bit local bus (~22.4 GB/s local bandwidth). [HIGH]
- Fully programmable pipeline: **Shader Model 3.0-class**, 24 fragment pipelines, 8 vertex pipelines. There is no fixed-function-only mode you should target; all rendering goes through vertex + fragment programs. [HIGH]
- Shading language: NVIDIA **Cg** dialect. In the homebrew stack you compile shaders **offline** with PSL1GHT's `cgcomp`. There is **no runtime shader compiler** available to homebrew — this is a hard architectural constraint. Plan all shader permutations at build time. [HIGH]
- Max render resolution 1920×1080; typical homebrew targets 720p (1280×720) for fill-rate headroom. [HIGH]
- Framebuffer: linear or tiled color surfaces in RSX local memory (GDDR3), typically X8R8G8B8; depth Z16 or Z24S8. Surface pitch must be 64-byte aligned. [HIGH]
- Texturing: 2D/3D/cube textures, linear or swizzled layouts, DXT1/DXT3/DXT5 compression, up to 4096×4096. Swizzled + mipmapped textures are dramatically friendlier to the RSX texture cache than linear no-mip textures. [HIGH]
- The RSX can fetch vertex/texture data from **main XDR memory over FlexIO**, but at lower bandwidth and higher latency than GDDR3. Use main-memory placement deliberately, not accidentally. [HIGH]

### RAM
- **Split memory architecture — you must internalize this:**
  - 256 MB **XDR** main RAM (Rambus) at 3.2 GHz effective, ~25.6 GB/s, attached to the Cell. [HIGH]
  - 256 MB **GDDR3** video RAM attached to the RSX. [HIGH]
- Asymmetric access speeds (the single most important perf table on this platform):
  - Cell → XDR: fast (~25 GB/s). [HIGH]
  - RSX → GDDR3: fast (~22 GB/s). [HIGH]
  - RSX reading XDR over FlexIO: usable (~15–20 GB/s theoretical, less in practice). [HIGH]
  - **Cell reading GDDR3: catastrophically slow (~16 MB/s).** Never read back from video memory on the CPU. Writes from Cell to GDDR3 are acceptable (~4 GB/s, write-combined). [HIGH]
- No user-visible virtual memory tricks for homebrew: you get a flat process address space managed by the LV2 kernel; no swapping. [HIGH]
- Under GameOS with CFW/HEN, your application typically has on the order of **~190–213 MB of usable main RAM** after OS reservations; do not assume the full 256 MB. Exact figure varies by firmware — treat >180 MB as unsafe without measurement. [MEDIUM — verify with a malloc-until-fail probe at boot on your target firmware]
- Alignment: 16-byte alignment is the safe general minimum for vector data; 128-byte for anything touched by SPE DMA or cache-line-sensitive code; 64-byte pitch alignment for RSX surfaces; 1 MB alignment/granularity for mapping main memory into RSX IO space. [HIGH]

### Bus topology
- The **EIB (Element Interconnect Bus)** is a 4-ring bus connecting PPE, 8 SPEs, the memory controller (MIC → XDR), and the FlexIO interface. Aggregate bandwidth ~200 GB/s; it is effectively never your bottleneck. [HIGH]
- **FlexIO** links the Cell to the RSX: ~20 GB/s Cell→RSX, ~15 GB/s RSX→Cell. All "RSX reads vertex data from main RAM" traffic goes over this link. [HIGH]
- DMA engines: each SPE has its own MFC DMA engine (16 queued transfers each); the RSX has DMA/transfer engines for surface blits and data movement (exposed via gcm transfer commands in homebrew). [HIGH]
- Practical bottlenecks in order of likelihood: (1) PPE single-thread throughput, (2) RSX fragment/texture-cache behavior, (3) FlexIO for main-memory vertex streams. XDR bandwidth itself is rarely the limit. [MEDIUM]

### Co-processors
- The 6 usable **SPEs** are the platform's headroom. Offload skinning, culling, vertex transform/interleave, audio mixing, decompression (zlib on SPU exists) to SPEs. The PPE alone is roughly comparable to a weak dual-thread PowerPC; the SPEs are where the Cell's power lives. [HIGH]
- SPE programming model in homebrew: compile SPU ELF with `spu-gcc`, embed it, load and run via LV2 SPU thread syscalls (`sysSpuImageImport`, SPU thread groups) or raw SPU support in PSL1GHT. Communicate via DMA to/from main memory, mailboxes (32-bit), and signal notification registers. [HIGH]
- Audio has **no dedicated audio DSP** exposed to homebrew — mixing is done in software on PPE/SPE and submitted to the audio output system (see §6). [HIGH]
- A separate south-bridge I/O controller handles USB/Ethernet/BT/storage, accessed only through LV2 syscalls, never directly. [HIGH]

### Security / DRM
- Boot chain: bootldr → lv0 → lv1 (**hypervisor**) → lv2 (GameOS kernel). Everything is signed; the hypervisor mediates hardware access. [HIGH]
- Homebrew runs on **CFW** (e.g., Evilnat, Rebug) or **PS3HEN** on hybrid firmware (HFW) for non-CFW-compatible models (most Slims/all Super Slims). Both patch LV2 to accept unsigned/fself executables. [HIGH]
- What homebrew CAN access: full RSX via gcm, all 6 app SPEs, USB, Bluetooth pads, HDD (`/dev_hdd0`), USB storage, Blu-ray drive reads, Ethernet/Wi-Fi networking, and (on CFW/HEN) LV2 syscalls including peek/poke. [HIGH]
- What homebrew CANNOT (or must not) touch: lv1/lv0 internals without console-bricking risk, the isolated SPU (SPE #7, security), syscon, and `/dev_flash` writes (bricks consoles when done wrong). [HIGH]
- HEN vs CFW nuance: HEN is a per-boot exploit (must re-enable after power cycle) and has slightly less capability than full CFW (e.g., some peek/poke and plugin scenarios); for typical PSL1GHT homebrew the difference is irrelevant. [MEDIUM]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official SDK: Sony's proprietary PS3 SDK ("CellSDK" / PSGL / GCM / SN Systems ProDG tooling). You must NOT reference, use, or assume access to it. It is mentioned here only so you can recognize its API names in forum posts and translate them to homebrew equivalents. [HIGH]
- **Homebrew SDK: PSL1GHT** (github.com/ps3dev/PSL1GHT) on top of **ps3toolchain** (github.com/ps3dev/ps3toolchain). Open source (largely MIT/BSD-style). This is your ONLY assumed toolchain. [HIGH]
- PSL1GHT provides:
  - `rsx` library — a gcm-workalike command-buffer API for the RSX (init, surfaces, textures, shaders, draw calls, flips). [HIGH]
  - `io` — gamepad (`ioPad*`), keyboard, mouse. [HIGH]
  - `audio` — audio port output (`audio*`). [HIGH]
  - `sysutil`, `sysmodule`, `lv2` wrappers — filesystem, threads, events, module loading, on-screen keyboard, msg dialogs. [HIGH]
  - `net` — sockets, plus `netctl` for connection state. [HIGH]
  - `cgcomp` — offline vertex/fragment shader compiler producing RSX microcode from Cg-style source. [HIGH]
  - Tools to produce runnable output: `fself`/make-self equivalents, `sfo` and `pkg` (native C tools from PSL1GHT `tools/sfo_pkg/`, replacing the legacy Python `sfo.py`/`pkg.py` from `ps3py`, which is no longer built), `sprxlinker` (shipped in ps3dev tools; Makefile rules in `ppu_rules`). CLI syntax is unchanged from the Python versions. [HIGH]
  - Useful community layers on top: **Tiny3D** (simple 3D/2D helper by Hermes), **NoRSX** (2D framebuffer graphics), SDL 1.2/2 ports of varying completeness. [MEDIUM — SDL2 port completeness varies by fork; verify the specific fork before depending on it]
- What PSL1GHT CANNOT do vs official:
  - **No runtime shader compilation** (official had cgc at runtime via PSGL; homebrew compiles offline with cgcomp only). This blocks engines that generate GLSL/Cg at runtime — e.g., Quake3e-style runtime GLSL is structurally impossible without a shader cache redesign. [HIGH]
  - `cgcomp` supports a subset of Cg: expect missing intrinsics, weaker optimization, and occasional miscompiles vs NVIDIA cgc. Keep shaders simple; validate output visually. [MEDIUM]
  - No PSGL/OpenGL ES implementation of production quality. Treat the platform as "gcm-style command buffer or nothing." [HIGH]
  - No official-grade debugger (no ProDG). Debugging is printf-over-network/file plus RPCS3 (§9). [HIGH]
  - No documented access to some OS services (trophies, official save-data dialogs work partially at best). [MEDIUM]
- Decision rule: if a feature requires the official SDK, redesign around it or mark the feature BLOCKED. Do NOT emit code that only compiles against leaked headers. [HIGH]

---

## 3. GRAPHICS PIPELINE

- API: PSL1GHT's `rsx` library (gcm-workalike). You build command buffers that the RSX consumes. There is no OpenGL. Do NOT write GL code and hope for a wrapper. [HIGH]

### Initialization sequence (canonical PSL1GHT flow)
1. Allocate a host command-buffer region in main memory (1 MB is typical) with `memalign(1024*1024, size)` — the mapping granularity into RSX IO space is 1 MB. [HIGH]
2. `context = rsxInit(CB_SIZE, host_size, host_addr)` — creates the gcm context and maps host memory for the RSX. A 64 KB command buffer (`CB_SIZE = 0x10000`) is the conventional default. [HIGH]
3. Query and set video state: `videoGetState(0, 0, &state)` → `videoGetResolution(state.displayMode.resolution, &res)` → fill a `videoConfiguration` (resolution id, `VIDEO_BUFFER_FORMAT_XRGB`, pitch = `rsxGetFixedUint16((res.width*4))` or simply `width*4` rounded to 64) → `videoConfigure(0, &vconfig, NULL, 0)`. [HIGH]
4. Allocate color buffers in RSX memory: `rsxMemalign(64, pitch*height)` per buffer; get RSX offsets with `rsxAddressToOffset(ptr, &offset)`; register each as a display buffer with `gcmSetDisplayBuffer(id, offset, pitch, width, height)`. [HIGH]
5. Allocate a depth buffer the same way (Z24S8, 4 bytes/pixel). [HIGH]
6. Per frame: set the render surface (`rsxSetSurface`) to the back buffer, draw, then flip.

### Framebuffer flow / flipping
- Use **double buffering** minimum; triple buffering if you can spare GDDR3 and want to absorb frame-time spikes. [HIGH]
- Flip protocol per frame: wait until previous flip completed (`gcmGetFlipStatus()` loop + `gcmResetFlipStatus()`), then `gcmSetFlip(context, buffer_id)`, `rsxFlushBuffer(context)`, and `gcmSetWaitFlip(context)` to fence subsequent commands until the flip occurs. [HIGH]
- VSync: `gcmSetFlipMode(GCM_FLIP_VSYNC)` for vsynced flips; `GCM_FLIP_HSYNC` for uncapped (tearing) — useful for raw perf measurement only. [HIGH]
- Refresh rates: HDMI output is 59.94/60 Hz for 480p/720p/1080p modes on all regions; PAL consoles additionally support 576i/50 Hz over analog. Design frame pacing for 60 Hz and treat 50 Hz as a legacy analog case. [HIGH]

### Textures
- Formats: A8R8G8B8, R5G6B5, A1R5G5B5, A4R4G4B4, L8, A8L8, DXT1/DXT23/DXT45, plus float formats (W16Z16Y16X16F etc.) for render targets. [HIGH]
- Layouts: **linear** (pitch-based) or **swizzled** (Morton order; power-of-two dimensions required). Swizzled is markedly faster to sample. [HIGH]
- Max texture size 4096×4096. [HIGH]
- **Always upload full mipmap chains for 3D-scene textures.** Missing mips force minified sampling of the top level and thrash the RSX texture cache — a proven multi-ms/frame regression in real ports. [HIGH]
- Texture pitch alignment: 64 bytes for linear textures used as render targets; plain sampled linear textures need pitch ≥ width*bpp and should stay 64-aligned by convention. [MEDIUM]
- Textures may live in RSX local memory (fast, preferred) or mapped main memory (RSX samples over FlexIO — slower; acceptable for streaming/low-frequency assets). [HIGH]

### Depth / stencil / blending
- Depth: Z16 or Z24S8 (use Z24S8; stencil comes with it). Depth-cull (ZCull) units accelerate early rejection when the depth surface is configured correctly. [HIGH]
- Full SM3-class blending: standard src/dst factors, min/max, separate alpha blend equations, alpha test. [HIGH]
- Two-sided stencil supported. [HIGH]

### Shaders
- Write vertex programs (`.vcg`) and fragment programs (`.fcg`) in Cg-subset syntax; compile at **build time** with `cgcomp` into binary blobs linked into the ELF (PSL1GHT Makefile rules do this automatically for `*.vcg`/`*.fcg`). [HIGH]
- Load at runtime with `rsxLoadVertexProgram` / fragment program setup via `rsxLoadFragmentProgramLocation` (fragment program microcode must reside in RSX-addressable memory; local memory is the safe choice). [HIGH]
- **Fragment program constants are patched into the microcode**, not stored in registers. Updating a fragment shader uniform means rewriting the program memory (`rsxSetFragmentProgramParameter` handles this) and is expensive — batch/minimize per-draw fragment constant changes; prefer vertex constants or texture-based parameters. [HIGH]
- Vertex program limits: 512 instruction slots, 468 constant slots (NV47-class). Fragment programs support real branching but it is slow; prefer compile-time permutations. [MEDIUM]

### Anti-patterns (GPU)
- Do NOT attempt runtime shader compilation — there is no compiler on-target. [HIGH]
- Do NOT read the framebuffer or any GDDR3 back on the PPE (16 MB/s). Use RSX-side transfers to main memory if you need readback. [HIGH]
- Do NOT ship linear, mipless textures for 3D scenes. [HIGH]
- Do NOT update fragment-program constants per draw call in hot paths. [HIGH]
- Do NOT let the command ring buffer wrap without a fence/jump strategy — reserve space or segment the buffer, or the RSX will consume garbage. [MEDIUM — symptom is intermittent hangs/corruption under load; verify with a stress test that submits oversized frames]

---

## 4. INPUT

- Primary controllers: **Sixaxis / DualShock 3** over Bluetooth or USB. Up to **7 controllers** simultaneously. DS3 adds rumble to Sixaxis; both have accelerometer + gyro (gyro is Z-axis only on Sixaxis-era hardware) and pressure-sensitive buttons. [HIGH]
- Also present on the system: USB HID keyboards/mice (via `ioKb*`/`ioMouse*`), Bluetooth remotes, and (hardware-dependent) PS Move — Move support in homebrew is sparse; treat as unavailable unless you verify a working library. [MEDIUM]
- Arbitrary non-DS3 USB pads are NOT handled by the LV2 pad service; supporting them requires raw USB work — out of scope for the standard `ioPad` path. [HIGH]

### Model: polled
- Initialize once: `ioPadInit(7)`. [HIGH]
- Per frame: `ioPadGetInfo(&padinfo)`; for each port `i` where `padinfo.status[i]` is nonzero, `ioPadGetData(i, &paddata)`. This is a **polling** model — there are no input events/callbacks. [HIGH]
- Buttons: bitfields in `paddata.BTN_*` (e.g., `BTN_CROSS`, `BTN_START`); PSL1GHT exposes them as bit-struct members (`paddata.BTN_CROSS`). [HIGH]
- Analog sticks: `paddata.ANA_L_H/V`, `ANA_R_H/V`, 0–255 with center ≈ 128. Apply your own deadzone (~±16 minimum); hardware centers drift. [HIGH]
- Pressure-sensitive button values and sixaxis sensor data (accel X/Y/Z + gyro) are in the extended fields of the pad data struct; sensor data may require enabling the pad "sensor mode" via `ioPadSetPortSetting(port, PAD_SETTING_SENSOR_ON)`. [MEDIUM — field naming varies between PSL1GHT versions; verify against your installed `io/pad.h`]
- Rumble: `ioPadSetActDirect(port, &actparam)` — small motor is on/off, large motor takes 0–255 intensity. Only meaningful on DS3, not Sixaxis. [HIGH]

### Disconnect/reconnect
- Controllers appear/disappear asynchronously (BT sleep, USB unplug). You must re-check `ioPadGetInfo` every frame and treat a port going to status 0 as a pause-worthy event; do NOT cache "controller 0 exists" at boot. [HIGH]
- `ioPadGetData` on a just-disconnected port returns stale/zero data, not an error you can rely on — use `padinfo.status`. [MEDIUM]

### Anti-patterns (input)
- Do NOT assume port 0 is the active player — after a BT reconnect the pad can enumerate on a different port. Scan all 7. [HIGH]
- Do NOT read pads from multiple threads without serialization; the LV2 pad service is not documented as thread-safe for homebrew. [LOW — confirm by hammering ioPadGetData from two PPU threads and checking for corrupt reads]
- Do NOT block the render thread waiting for input state changes; poll and move on. [HIGH]

---

## 5. MEMORY LAYOUT

- You run in an LV2-managed virtual address space; you do not deal with raw physical addresses for normal work. The two pools you manage are **main (XDR) heap** and **RSX local (GDDR3) heap**. [HIGH]
- Main heap: standard `malloc`/`memalign` from newlib operates on your ~190–213 MB main allocation (§1). [MEDIUM — probe on target firmware]
- RSX local memory: allocate with `rsxMemalign(align, size)`; convert to RSX offsets with `rsxAddressToOffset(ptr, &off)`. All surfaces, fragment programs, and (preferably) textures live here. ~249 MB is allocatable after reserved regions; framebuffers eat into it (720p double-buffer + Z ≈ 11 MB). [MEDIUM]
- Mapping main memory for RSX access: memory passed to `rsxInit` as the host region is IO-mapped for the RSX; additional mappings use `rsxMapMemory`/gcm map calls with **1 MB granularity and alignment**. Vertex/index buffers in mapped main memory work but stream over FlexIO. [HIGH]
- I/O registers: never touched directly by homebrew; all device access is via LV2 syscalls/hypercalls. Do NOT hunt for MMIO addresses. [HIGH]

### Cache architecture and coherency
- PPE caches are **write-back**, 128-byte lines, and **hardware-coherent with respect to SPE DMA and RSX IO-mapped reads** — the Cell's EIB maintains coherency for cacheable main memory. In the normal PSL1GHT flow you do NOT manually flush caches for the RSX to see your vertex data; the mapped-memory path is coherent. [MEDIUM — coherency of IO-mapped regions is the documented Cell behavior, but if you see stale vertex data on hardware, test by inserting `__asm__ volatile("sync")` / cache-flush loops and comparing]
- You DO need memory barriers for ordering between PPE stores and RSX doorbell/label writes: use `__asm__ volatile("sync" ::: "memory")` (or `eieio` for IO ordering) before signaling the RSX that data is ready. [MEDIUM]
- SPE side: Local Store has no cache and no coherency — DMA is explicit; use MFC fences/barriers (`mfc_getf/putf`, tag waits) to order transfers. [HIGH]
- Explicit PPE cache ops exist (`dcbf`, `dcbst`, `dcbz`, `icbi` via inline asm). `dcbz` (zero a 128-byte line) is a cheap way to avoid read-for-ownership on buffers you will fully overwrite. Use `icbi`+`isync` only if you self-modify code (JITs — e.g., a PPC JIT must flush D-cache and invalidate I-cache over the emitted range, then `isync`). See "Executable memory / JIT code caches" below for where such a cache may legally live. [HIGH]

### Executable memory / JIT code caches
- **JIT works on PS3 homebrew, but only from `.text`.** Measured on retail hardware (CFW, PSL1GHT) 2026-09-20: a buffer placed in the executable segment with `__attribute__((section(".text")))` was written to at runtime, cache-synced, and called successfully. This reverses the common assumption that GameOS homebrew cannot self-modify. [HIGH — measured on hardware]
- **Nothing in the RW segment is executable.** `.bss` arrays, `malloc`, and `sysMemoryAllocate(size, SYS_MEMORY_PAGE_SIZE_1M, &addr)` were all tested and all faulted on the call. The libogc/Wii habit of executing out of a plain `.bss` array does NOT port. [HIGH — measured on hardware]
- **There is no execute flag to ask for.** The lv2 memory API offers only `SYS_MEMORY_PROT_READ_ONLY` and `SYS_MEMORY_PROT_READ_WRITE` (`sys/memory.h`). No mapping call grants execute, and PSL1GHT has no working `mprotect`. So you cannot make a runtime allocation executable — you must reserve the space in the image at build time. [HIGH]
- **lv2 does not enforce W^X on the loaded image.** `readelf -l` marks the text segment `R E` with no write bit, yet stores into it succeed. Do not trust segment flags to predict runtime page protection on this platform; measure. [HIGH — measured on hardware]
- **Sizing:** the cache is part of the ELF, so the binary grows by the full cache size (a 6 MB cache took a 4 MB ELF to 10.4 MB). The SELF/PKG compresses the zero-filled array to almost nothing, so install size is barely affected. [HIGH]
- **Cache-sync sequence** for emitted code, on 128-byte lines (Cell PPE line size — NOT the Wii's 32): `dcbst` over the range, then `sync`, then `icbi` over the range, then `isync`. [HIGH]
- **ELFv1 calling convention applies to generated code.** A function pointer is an OPD descriptor `{entry, toc, env}`, not code. To call generated code, build a descriptor and call through it. To emit a `bl` to a C function, dereference that function's OPD at emit time and branch to the real entry — branching to the symbol address lands in `.opd` data. Frames are 112 bytes minimum, back chain at `0(sp)`, and the callee saves your LR at `16(sp)`, not `4(sp)`. Link with `-Wl,--no-multi-toc` and never write `r2` in generated code. [HIGH]
- **A bad JIT block kills only the faulting thread, not the process.** Other threads keep running, so a fault in generated code presents as a hang, not a crash, with no message. Budget for that when debugging: journal progress to disk or UDP *before* each risky call, because you will get no post-mortem. [HIGH — measured on hardware]

### Stack
- Main PPU thread stack defaults are modest; PSL1GHT lets you set the main thread stack via the `SYS_PROCESS_PARAM(priority, stacksize)` macro in your source (e.g., `SYS_PROCESS_PARAM(1001, 0x100000)` for 1 MB). Set this explicitly — engine ports with deep recursion (BSP traversal, zone allocators on stack) will silently smash a default-sized stack. [HIGH]
- Secondary threads: `sysThreadCreate(..., stack_size, ...)` takes an explicit stack size. Do not go below 64 KB for anything nontrivial. [HIGH]

### DMA (SPE) requirements
- Size: 1/2/4/8/16 bytes for tiny transfers (naturally aligned), else multiples of 16 bytes up to 16 KB per element. [HIGH]
- Alignment: source and destination must share the same alignment mod 16; 128-byte alignment for full-speed transfers. [HIGH]

### Anti-patterns (memory)
- Do NOT allocate general-purpose data in RSX local memory just because it's "free" — CPU reads from it are 16 MB/s. [HIGH]
- Do NOT assume `malloc` never fails at 150 MB+ working sets; check every allocation, you are on a fixed 256 MB budget. [HIGH]
- Do NOT place SPE DMA buffers at unaligned addresses; the MFC raises interrupts/errors on bad alignment (crash or silent data corruption depending on handler). [HIGH]
- Do NOT forget `icbi`/`isync` after emitting code (JIT) — stale I-cache executes garbage and the crash appears far from the cause. [HIGH]
- Do NOT put a JIT code cache in `.bss`, `malloc`, or any lv2 mapping — those pages are never executable. Use a `section(".text")` array. [HIGH]
- Do NOT branch a generated `bl` to a C function's symbol address on PPC64 ELFv1 — that address is the OPD descriptor, which is data. Dereference it first. [HIGH]

---

## 6. AUDIO

- There is no homebrew-accessible dedicated audio DSP; the system audio service consumes PCM you generate on PPE/SPE and handles output (HDMI/optical/analog, including system-side downmix/encode). [HIGH]
- API: PSL1GHT `audio` library (LV2 audio ports).

### Canonical setup
1. `audioInit()`. [HIGH]
2. Fill `audioPortParam`: `numChannels` = `AUDIO_PORT_2CH` or `AUDIO_PORT_8CH`; `numBlocks` = `AUDIO_BLOCK_8` (8 is the common choice; 16 max). [HIGH]
3. `audioPortOpen(&params, &portNum)`, `audioGetPortConfig(portNum, &config)` — config gives you the ring-buffer address (`config.audioDataStart`), block count, and channel count. [HIGH]
4. `audioCreateNotifyEventQueue(&queue, &key)` + `audioSetNotifyEventQueue(key)` — the system signals this event queue **once per audio block consumed**. [HIGH]
5. `audioPortStart(portNum)`. [HIGH]
6. Loop: `sysEventQueueReceive(queue, &event, timeout)` → compute the block index just freed → write the next block. [HIGH]

### Format — fixed, non-negotiable
- Sample rate: **48000 Hz only.** Resample all other content (44.1 kHz music included) yourself. [HIGH]
- Sample format: **32-bit float, interleaved**, range −1.0..1.0. [HIGH]
- Block size: **256 samples per channel per block** (i.e., one block ≈ 5.33 ms). With 8 blocks queued you have ~42 ms of ring buffer. [HIGH]
- Channels: 2 or 8 (7.1). Open 2ch unless you genuinely mix surround. [HIGH]

### Compressed formats
- Nothing hardware-decodes for you at this layer. Decode Ogg/MP3/ADPCM in software (stb_vorbis, libvorbis, libmpg123 ports exist in ps3dev portlibs). Budget PPE time or push decode to an SPE. [HIGH — portlib availability MEDIUM; verify the ps3dev/ps3libraries repo has the codec you need]

### Avoiding glitches
- Service the event queue with real-time discipline: a dedicated PPU thread (SMT sibling is fine) that ONLY receives the event and memcpys a pre-mixed block. Miss one 5.3 ms deadline and you click. [HIGH]
- Mix ahead into your own ring buffer on a normal-priority thread; the audio thread should never mix, decode, or take locks with unbounded hold times. [HIGH]

### Anti-patterns (audio)
- Do NOT decode or mix inside the event-servicing loop. [HIGH]
- Do NOT assume 44.1 kHz output exists. It does not. [HIGH]
- Do NOT write int16 PCM into the port buffer — it expects float32 and you will get loud garbage. [HIGH]

---

## 7. STORAGE / IO

### Accessible media and mount points
| Mount | Media | Notes |
|---|---|---|
| `/dev_hdd0` | Internal HDD | Your primary read/write storage. [HIGH] |
| `/dev_usb000` … `/dev_usb006` | USB mass storage | Enumerated in plug order; FAT32 natively. [HIGH] |
| `/dev_bdvd` | Blu-ray/DVD in drive | Read-only. [HIGH] |
| `/dev_flash` | System firmware flash | Mounted read-only normally. NEVER remount/write casually — brick risk. [HIGH] |
| `/app_home` | Host/dev redirection | Only meaningful under debug launchers/specific loaders; do NOT depend on it. [HIGH] |

- USB filesystems: FAT12/16/32 are supported by LV2. FAT32's 4 GB file cap applies. NTFS/exFAT require the community NTFS library (ps3ntfs) with its own open/read API — not transparent through `fopen`. [HIGH]
- Internal HDD filesystem is Sony's encrypted CFS/UFS variant; you see it only through the VFS. Path handling on `/dev_hdd0` is effectively **case-preserving; treat it as case-SENSITIVE in your code** and normalize all asset paths to one case — mismatched case works on FAT USB sticks and then breaks on HDD. [MEDIUM — verify with a two-file case-collision test on your firmware]

### File API
- Standard newlib `fopen/fread/fwrite/opendir` work over the LV2 VFS in PSL1GHT; the native `sysLv2Fs*` calls (`sysLv2FsOpen`, `sysLv2FsRead`, `sysLv2FsOpenDir`) are available when you need explicit control or better error codes. [HIGH]
- Absolute paths only (`/dev_hdd0/...`); there is no meaningful CWD guarantee when launched from XMB loaders. Derive your data root at runtime (see below), never hardcode a single device. [HIGH]

### Homebrew app structure (what the loader needs)
- Homebrew is installed as a **PKG** that unpacks to `/dev_hdd0/game/<TITLEID>/`:
  - `PARAM.SFO` — metadata: TITLE, TITLE_ID (9 chars, convention `TEST00000`-style for homebrew), APP_VER, category HG. Generated by the native `sfo` tool. [HIGH]
  - `ICON0.PNG` — 320×176 icon shown in XMB. [HIGH]
  - `USRDIR/EBOOT.BIN` — your signed-as-fself executable. [HIGH]
  - Your assets under `USRDIR/` (e.g., `USRDIR/baseq3/...`); resolve them relative to the app dir. Loaders pass the app path in `argv[0]`-adjacent conventions inconsistently — the robust pattern is: probe `/dev_hdd0/game/<TITLEID>/USRDIR/` first, then fall back to scanning `/dev_usb%03d/<appfolder>/`. [MEDIUM]
- Optional: `PIC1.PNG` (1920×1080 XMB background), `SND0.AT3` (XMB sound — requires ATRAC3, usually skipped in homebrew). [HIGH]

### Network
- Full BSD-style TCP/UDP sockets over Ethernet or Wi-Fi: `netInitialize()`, then `netSocket/netBind/netConnect/...` (PSL1GHT prefixes) or the plain names via compatibility headers. `netctl` reports link/IP state. [HIGH]
- select/poll support exists (`netPoll`, `netSelect`); non-blocking via `netSetSockOpt(..., SO_NBIO, ...)`. [MEDIUM — flag name varies; check `net/net.h` in your PSL1GHT checkout]
- A practical use you should default to: **UDP debug logging** to a PC (`netcat -ul 18194`) — this is the platform's de facto printf channel (§9). [HIGH]

### Anti-patterns (storage)
- Do NOT write to `/dev_flash` or remount it read-write. [HIGH]
- Do NOT assume `/dev_usb000` exists or is the stick the user means; enumerate. [HIGH]
- Do NOT stream many small synchronous reads during gameplay from HDD or USB — LV2 file ops have high per-call latency; batch into large reads and cache. [MEDIUM]
- Do NOT exceed 4 GB per file on FAT32 USB paks. [HIGH]

---

## 8. BUILD SYSTEM

### Toolchain
- **ps3toolchain** (github.com/ps3dev/ps3toolchain): builds GCC for two targets — PPU: `powerpc64-ps3-elf-`, SPU: `spu-`. Plus **PSL1GHT** for libraries/rules and **ps3libraries** for portlibs (zlib, libpng, freetype, vorbis…). Use the current git HEAD or the ps3dev Docker image (`ghcr.io/ps3dev/ps3dev`) — the Docker image is the fastest reliable setup. [HIGH]
- Environment variables (required by every Makefile in the ecosystem):
```sh
export PS3DEV=/usr/local/ps3dev
export PSL1GHT=$PS3DEV
export PATH=$PATH:$PS3DEV/bin:$PS3DEV/ppu/bin:$PS3DEV/spu/bin
```
[HIGH]

### Compiler / linker flags (PPU)
- Baseline CFLAGS: `-mcpu=cell -mhard-float -fmodulo-sched -ffunction-sections -fdata-sections -O2` — big-endian and 64-bit-PPC ABI (with 32-bit longs — the PS3 PPU ABI is ILP32 on a 64-bit ISA) are the target's defaults; do not fight them. Pointers are 32-bit. [HIGH for flags; HIGH for ILP32 ABI]
- `-maltivec` where you use VMX intrinsics. [HIGH]
- Never `-msoft-float` (the PPE has a real FPU) and never little-endian flags. [HIGH]
- Linking: PSL1GHT's `ppu_rules` supplies crt/linker script. Typical libs for a game: `-lrsx -lgcm_sys -lio -laudio -lsysutil -lsysmodule -lnet -lnetctl -lrt -llv2 -lm`. Order matters (gnu ld single-pass); keep `-llv2` late. [HIGH — exact set varies per app]

### Output pipeline
1. Compile/link → `app.elf` (64-bit big-endian PPC ELF). [HIGH]
2. `sprxlinker app.elf` (fixes up stub libraries). [HIGH]
3. SELF creation, two paths (both tools live in PSL1GHT `tools/`): the **fself** tool produces a fake-signed SELF (the `%.self` rule also emits a `.fake.self`) — this is the fast-iteration artifact you FTP as `EBOOT.BIN` on CFW/HEN. The stock `make pkg` rule instead builds `EBOOT.BIN` with **`make_self_npdrm`** (geohot tools, in `tools/geohot/`) using `$(CONTENTID)`. [HIGH]
4. `sfo --title "My App" --appid TEST00000 -f sfo.xml PARAM.SFO` and `pkg --contentid UP0001-TEST00000_00-0000000000000000 pkgdir/ app.pkg` → installable PKG. (These are the native C tools; the old `sfo.py`/`pkg.py` accepted the same arguments.) The rule additionally runs `package_finalize` to emit a `.gnpdrm.pkg` variant alongside the plain `.pkg`. The stock PSL1GHT Makefile rules expose this as `make pkg`. [HIGH]
- Deployment options: install PKG from USB via XMB "Install Package Files"; or FTP `EBOOT.BIN` directly over an existing install (multiMAN/webMAN ftp server) — the fast iteration path: rebuild fself, `curl -T EBOOT.BIN ftp://PS3IP/dev_hdd0/game/TEST00000/USRDIR/`, relaunch. [HIGH]

### Minimal Makefile skeleton
```make
ifeq ($(strip $(PSL1GHT)),)
$(error "PSL1GHT is not set")
endif
include $(PSL1GHT)/ppu_rules

TARGET   := myapp
TITLE    := My App
APPID    := TEST00000
CONTENTID:= UP0001-$(APPID)_00-0000000000000000

CFILES   := $(wildcard source/*.c)
VCGFILES := $(wildcard shaders/*.vcg)
FCGFILES := $(wildcard shaders/*.fcg)

CFLAGS   += -O2 -mcpu=cell -Iinclude
LIBS     := -lrsx -lgcm_sys -lio -laudio -lsysutil -lsysmodule -lrt -llv2 -lm
# standard ppu_rules targets: $(TARGET).elf, $(TARGET).self, pkg
```
[MEDIUM — treat as a shape reference; reconcile against a current PSL1GHT sample Makefile, they drift]

### Asset pipeline
- Shaders: `.vcg`/`.fcg` → `cgcomp` at build time (automatic via ppu_rules; produces `.vpo`/`.fpo` binaries you `bin2s` into the ELF or load from disk). [HIGH]
- Textures: pre-swizzle and pre-generate mipmaps offline where possible, or do it once at load; store DXT for bulk content. Remember **big-endian**: any 16/32-bit texel or header data authored on PC must be byte-swapped at cook time or load time. [HIGH]
- Audio: pre-resample to 48 kHz float or int16 (convert at load). [HIGH]
- All binary asset formats: define endian-explicit readers (`read_le32`/`read_be32`), never `fread` straight into structs. [HIGH]

### Anti-patterns (build)
- Do NOT pass x86-isms (`-msse`, `-m32`) or expect little-endian bitfield layouts. [HIGH]
- Do NOT link with plain `gcc`/system ld; always the `powerpc64-ps3-elf-` prefix and PSL1GHT crt. [HIGH]
- Do NOT skip `sprxlinker`; you get an EBOOT that crashes at boot resolving stubs. [MEDIUM]
- Do NOT `fread` packed structs from PC-authored files without byte-swapping. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary emulator: **RPCS3**. Accuracy is high for retail games and good for PSL1GHT homebrew; it loads unsigned ELFs/fselfs directly (File → Boot ELF), which makes it the fastest iteration loop you have. [HIGH]

### What RPCS3 gets RIGHT (safe to rely on)
- LV2 syscall semantics, filesystem layout emulation (`dev_hdd0` mapped to a host folder), pad/audio/network services — functional bring-up and logic bugs reproduce well. [HIGH]
- Its **debugger and logging**: PPU disassembly, breakpoints, memory viewer, RSX capture/debugger, and TTY output — capabilities you simply don't have on retail hardware. Use RPCS3 as your GDB substitute. [HIGH]
- Endianness/ABI: it executes your real big-endian PPU code, so endian bugs do reproduce. [HIGH]

### What RPCS3 gets WRONG or hides (must test on hardware)
- **Performance. Never profile on RPCS3.** Its PPU/SPU recompilers and host-GPU RSX implementation have a completely different cost model (shader patching, texture cache behavior, FlexIO penalties, LHS stalls all differ or vanish). Framerate on RPCS3 predicts nothing. [HIGH]
- RSX edge behavior: real-hardware texture-cache thrashing, command-buffer wrap hazards, precise flip timing, and partial-surface/pitch mistakes can render "fine" in RPCS3 and corrupt or hang on hardware. [MEDIUM]
- Memory pressure: RPCS3 is often more forgiving about allocation limits and uninitialized reads. [MEDIUM]
- Timing races between PPU threads, SPU DMA completion, and RSX labels are laxer under emulation. [MEDIUM]

### Hardware debugging options
- No GDB stub for retail homebrew. Your channels, in order of usefulness:
  1. **UDP log sink**: trivial `netSocket`+`sendto` printf clone to a PC running `nc -ul 18194`. Add this in session 1 of any port. [HIGH]
  2. On-screen debug text (once video is up) — bitmap font blit to framebuffer. [HIGH]
  3. File logging to `/dev_hdd0/tmp/` with `fflush`+`fsync` per line (survives hard crashes up to the last sync). [HIGH]
  4. webMAN/ps3mapi peek of memory on a live console for post-mortem inspection. [MEDIUM]
- Crash behavior on hardware: an unhandled PPU exception typically freezes the app or kicks you to XMB; LV2 can write minidumps under some CFW configurations but do NOT design your workflow around them — design around the UDP log. [MEDIUM]

### Anti-patterns (emulator)
- Do NOT optimize based on RPCS3 FPS. [HIGH]
- Do NOT ship after emulator-only validation; every rendering and timing milestone needs a real-hardware pass. [HIGH]
- Do NOT interpret an RPCS3-only crash as necessarily real (recompiler quirks exist) — but treat hardware-only crashes as always real. [MEDIUM]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, app appears to run (log output continues) | After `rsxInit`/`videoConfigure` | Display buffers never registered, wrong pitch, or flip never issued: missing `gcmSetDisplayBuffer`, pitch not 64-aligned, or no `gcmSetFlip`/`rsxFlushBuffer` | Verify order: videoConfigure → rsxMemalign color buffers → rsxAddressToOffset → gcmSetDisplayBuffer(id,…) → per-frame flip protocol; log every offset | [HIGH] |
| Black screen, instant return to XMB | At launch | EBOOT not a valid fself (skipped `sprxlinker`/`fself`), or PARAM.SFO category wrong | Rebuild via `make pkg`; confirm fself step ran; test the same ELF in RPCS3 to separate packaging from code | [HIGH] |
| Textures scrambled / diagonal shearing | First textured draw | Linear-vs-swizzled mismatch, wrong pitch, or forgot `rsxAddressToOffset` (passed a pointer where an offset belongs) | Audit texture setup: layout flag, pitch=width*bpp (linear), offsets everywhere the API says "offset" | [HIGH] |
| Textures have wrong colors (red/blue swapped or garbage bands) | PC-authored assets | Endianness: texel or header data not byte-swapped for big-endian | Swap at cook or load; never fread structs raw | [HIGH] |
| Severe frame drops only in texture-heavy scenes; RPCS3 fine | Real hardware | Missing mipmap chains and/or linear layouts thrashing the RSX texture cache | Upload full mip chains, swizzle static textures | [HIGH] |
| Audio clicks/pops every few seconds | Under load | Missed 5.33 ms block deadline: mixing/decoding inside the event-servicing thread, or lock contention | Dedicated feeder thread that only memcpys pre-mixed blocks; mix ahead elsewhere | [HIGH] |
| Silence, no error codes | Audio init "succeeded" | Wrote int16 into float32 port buffer, or wrote blocks without pacing to the notify queue | Convert to float −1..1; drive writes strictly off `sysEventQueueReceive` | [HIGH] |
| Crash on boot before `main` output | Static initializers / early | Stack too small for init path, or missing `SYS_PROCESS_PARAM`, or unresolved sprx stubs | Add `SYS_PROCESS_PARAM(1001, 0x100000)`; run `sprxlinker`; bisect static ctors | [MEDIUM] |
| Crash/garbage after loading large files | Big asset loads (>~150 MB working set) | Unchecked `malloc` failure on the ~200 MB budget; or loaded into RSX-mapped region then CPU-processed at 16 MB/s (appears as a hang) | Check every allocation; keep CPU-processed data in main heap; probe real heap ceiling at boot | [HIGH] |
| Input dead after suspend/BT sleep although game runs | Long sessions | Cached pad presence from boot; pad re-enumerated on another port | Re-scan `ioPadGetInfo` every frame, all 7 ports | [HIGH] |
| `fopen` fails on HDD, same path works from USB | Asset paths | Case mismatch: FAT is case-insensitive, HDD VFS effectively is not | Normalize asset paths to one case; add a case-insensitive fallback scan in the file layer | [MEDIUM] |
| Everything works but ~20–30 FPS where PC logic predicts 60 | General perf | PPE in-order costs: LHS stalls, float↔int through memory, branchy hot loops; or vertex streams left in main memory over FlexIO | Profile with timestamps over UDP; hoist hot data to GDDR3; apply LHS/AltiVec fixes; consider SPE offload | [HIGH] |
| Intermittent GPU hang after minutes of play | Long frames / heavy draws | Command ring buffer wrap without proper jump/fence handling | Segment the CB or check space before writes; stress-test with worst-case frame | [MEDIUM] |
| Image displayed but stretched/squashed | First light on TV | videoConfigure aspect/resolution mismatch (e.g., forcing 720p on a 576i-only display, or wrong `videoConfiguration.aspect`) | Honor `videoGetState` current mode at bring-up; only later offer mode switching | [HIGH] |
| Exception with LV2 error 0x80010009 / 0x80010002 from fs or sys calls | Any syscall path | EINVAL/EFAULT-class: bad flags, unaligned or freed buffer passed to LV2 | Log every syscall return; validate args against PSL1GHT headers | [MEDIUM] |

---

## 11. ANTI-PATTERNS (platform-wide)

1. Do NOT assume little-endian anywhere: files, network, packed pixels, bitfields. [HIGH]
2. Do NOT read GDDR3 (framebuffer, RSX-local anything) from the CPU. [HIGH]
3. Do NOT plan for runtime shader compilation; all shaders are compiled offline with cgcomp. [HIGH]
4. Do NOT treat the PPE like an out-of-order x86 core: LHS stalls, microcoded misalignment, and branch costs will halve your framerate. [HIGH]
5. Do NOT leave 3D-scene textures mipless or linear. [HIGH]
6. Do NOT do work in the audio notify thread beyond copying a pre-mixed block. [HIGH]
7. Do NOT assume more than ~190 MB of main RAM without probing. [MEDIUM]
8. Do NOT hardcode `/dev_usb000` or port-0-only controller logic. [HIGH]
9. Do NOT write to `/dev_flash`, ever. [HIGH]
10. Do NOT `fread` packed structs directly from PC-authored binary files. [HIGH]
11. Do NOT profile or validate performance on RPCS3. [HIGH]
12. Do NOT update fragment-program constants per draw in hot paths (they're patched into microcode). [HIGH]
13. Do NOT emit JIT code without D-cache flush + I-cache invalidate + `isync` over the range. [HIGH]
14. Do NOT let SPE DMA buffers be unaligned or cross the 16 KB-per-transfer limit. [HIGH]
15. Do NOT block the render loop on file I/O; LV2 per-call latency is high — batch reads, load on a worker thread. [MEDIUM]

---

## 12. PORTING DECISION TREE

Follow in order. Each step gates the next.

1. **Endianness & ABI audit (before any PS3 code).**
   - Do: grep the codebase for memcpy-into-struct file loads, pointer/int casts (pointers are 32-bit here), bitfield assumptions, and `#ifdef __BIG_ENDIAN__` coverage; fix loaders to explicit byte-order readers.
   - Why first: everything downstream produces garbage data otherwise, and the bugs masquerade as renderer/audio faults.
   - Skip it and: you will chase "corrupt asset" ghosts through every later stage. [HIGH]
2. **Toolchain bring-up + null main.**
   - Do: build a hello-world fself with ps3toolchain/PSL1GHT, package it, boot it on hardware AND RPCS3; wire the UDP logger now.
   - Why: proves the entire build→deploy→observe loop before real code enters it.
   - Skip it and: you cannot attribute the first real failure to code vs packaging. [HIGH]
3. **Stub the platform layer, compile everything.**
   - Do: replace the engine's sys_/platform files (video, input, audio, fs, net, threads) with stubs; get the full codebase compiling big-endian ILP32.
   - Why: surfaces the port's true compile-level blockers (inline asm, SSE intrinsics, LP64 assumptions) in one pass.
   - Skip it and: you interleave compiler firefighting with logic debugging forever. [HIGH]
4. **Filesystem + asset loading (headless).**
   - Do: implement fs layer over `/dev_hdd0/game/<ID>/USRDIR`, load core assets, verify checksums over the UDP log — no rendering yet.
   - Why: isolates endian/asset bugs while the output channel is still just text.
   - Skip it and: your first triangle debugging session is also your first pak-file debugging session. [HIGH]
5. **Video bring-up: cleared framebuffer at correct resolution/aspect.**
   - Do: rsxInit → videoConfigure → double-buffered clear-color flip loop at vsync.
   - Why: this is the platform's riskiest boilerplate; do it with zero engine involvement.
   - Skip it and: renderer bugs and display-pipeline bugs are indistinguishable. [HIGH]
6. **Input loop.**
   - Do: ioPad polling mapped to the engine's input abstraction; on-screen or UDP echo of buttons.
   - Why: cheap, and you need it to navigate menus for all later testing.
   - Skip it and: every later hardware test requires code changes to auto-navigate. [HIGH]
7. **Renderer: minimal path first (untextured → textured → lightmapped → full).**
   - Do: one vertex + one fragment program compiled via cgcomp; interleaved vertex buffers in GDDR3; grow toward the engine's full material system as offline-compiled permutations.
   - Why: cgcomp limitations and RSX state bugs must be found on trivial scenes.
   - Skip it and: you debug shader miscompiles inside a 200-material scene. [HIGH]
8. **Audio.**
   - Do: 48 kHz float ring with notify-queue pacing; engine mixer feeds it.
   - Why after video: audio is self-contained and its failure modes (§6) don't block visual milestones.
   - Skip it and: nothing else breaks, but retrofitting the 48 kHz/float/256-block model into an engine that assumed SDL-style callbacks late is painful. [HIGH]
9. **Performance pass on real hardware only.**
   - Do: frame timestamps over UDP; fix in this order — texture mips/swizzle, vertex data residency (GDDR3 vs FlexIO), fragment-constant churn, PPE LHS/branch hotspots, then SPE offload for the biggest remaining CPU cost.
   - Why last: optimizing before correctness wastes the anti-loop budget; and the cost model only exists on hardware.
   - Skip it and: you ship 25 FPS, or worse, you "optimized" against RPCS3. [HIGH]
10. **Release hardening.**
    - Do: allocation-failure paths, controller hot-plug, USB vs HDD install paths, 50 Hz/analog display fallback, long-session soak (CB wrap, memory creep).
    - Why: these are exactly the failures that only appear on other people's consoles.
    - Skip it and: your issue tracker becomes this table's §10. [HIGH]

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PS3 HOMEBREW PRIMER (PSL1GHT stack)
CPU: Cell @3.2GHz = 1 PPE (PPC64 ISA, ILP32 ABI, 2-thread, IN-ORDER, BIG-ENDIAN,
     AltiVec, 128B cache lines) + 6 usable SPEs (256KB LS each, explicit DMA<=16KB).
GPU: RSX = NV47/G70-class @500MHz, SM3.0, Cg shaders compiled OFFLINE (cgcomp).
     No runtime shader compile. Fragment constants are patched into microcode ($$).
RAM: 256MB XDR (main, ~190-213MB usable) + 256MB GDDR3 (RSX local).
     CPU read of GDDR3 = 16 MB/s. NEVER read back. RSX can read XDR via FlexIO (slower).
SDK: ps3toolchain + PSL1GHT. CC=ppu-gcc (symlink to powerpc64-ps3-elf-gcc), SPU=spu-gcc.
     ENV: PS3DEV=/usr/local/ps3dev, PSL1GHT=$PS3DEV. Docker: ghcr.io/ps3dev/ps3toolchain/ps3dev.
GFX: rsxInit(64KB CB,1MB host) -> videoConfigure -> rsxMemalign(64) buffers ->
     rsxAddressToOffset -> gcmSetDisplayBuffer -> loop{draw; gcmSetFlip; rsxFlushBuffer;
     gcmSetWaitFlip}. Textures: swizzle + FULL MIP CHAINS or the tex cache thrashes.
PAD: ioPadInit(7); poll ioPadGetInfo+ioPadGetData every frame, all 7 ports. DS3 only.
AUD: audioPortOpen 48000Hz float32 interleaved, 256-sample blocks, 8 blocks,
     pace via audioCreateNotifyEventQueue + sysEventQueueReceive. No 44.1kHz.
FS : /dev_hdd0/game/<TITLEID>/USRDIR/EBOOT.BIN (+PARAM.SFO, ICON0.PNG 320x176).
     USB=/dev_usb000..006 FAT32 (4GB cap). Treat HDD paths as case-sensitive.
BLD: elf -> sprxlinker -> make_self_npdrm EBOOT.BIN -> pkg (native sfo/pkg, `make pkg`).
     FTP iteration uses the fself (.fake.self) as EBOOT.BIN instead.
     Iterate: FTP EBOOT.BIN via webMAN to USRDIR. Debug: UDP printf -> `nc -ul 18194`.
EMU: RPCS3 = logic/debugger, NEVER performance. All perf + timing on real hardware.
TOP TRAPS: endianness, LHS stalls on PPE, mipless textures, CPU-reads-VRAM,
     runtime shaders impossible, unchecked malloc on 200MB budget.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this SKILLS.MD is used in an active development session (not just reference), you must follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify platform-specific blockers, and plan the implementation.
- **STATE: IMPLEMENTATION** — Write/modify code based on this SKILLS.MD and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output the exact build commands and ask the user to compile and flash/run on real hardware.
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back results. Classify the result as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES
1. **ANALYSIS → IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change. You must include:
   - Exact `make` / build command
   - Expected output file name and location (e.g., `EBOOT.BIN`, `app.pkg`)
   - How to transfer to the PS3 (PKG install from USB, or FTP EBOOT.BIN via webMAN/multiMAN)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color framebuffer"]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including any error codes, black screen, crashes]
   === STOP ===
   ```
   After this header, you STOP generating. You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE → VALIDATION**: Triggered ONLY by user message containing "SUCCESS" or "FAILURE".
   - If user says "SUCCESS": Move to NEXT_GOAL.
   - If user says "FAILURE": Move to DEBUG_PROTOCOL.
   - If user says anything else (e.g., "it kind of works", "almost"): Ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: You must output a **DEBUG BUILD PROTOCOL**:
   - A minimal C test case that isolates the failure (e.g., a bare rsx clear-and-flip loop, a lone audioPort sine, an ioPad UDP echo).
   - OR a checklist of exactly 3 specific diagnostic steps (e.g., "Log the value returned by rsxAddressToOffset for each display buffer", "Verify pitch is a multiple of 64", "Confirm sysEventQueueReceive is being signaled by printing a counter every 64 blocks").
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the debug protocol, you return to WAITING_FOR_HARDWARE state.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: You propose the next milestone from the goal stack below. You do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [RPCS3 side-by-side comparison / UDP register-value trace / webMAN ps3mapi memory peek / PSX-Place or ps3dev community consultation].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide RPCS3 logs / UDP traces / memory dumps for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this default:
1. Initialize video output (solid color framebuffer, vsynced double-buffer flip)
2. Initialize controller input (read button presses, UDP/on-screen echo)
3. Initialize audio output (play 440 Hz sine via 48 kHz float port)
4. Load assets from HDD/USB (checksum-verified over UDP log)
5. Render main menu framebuffer (textured 2D quads via cgcomp shaders)
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
