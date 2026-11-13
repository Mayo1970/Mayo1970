---
name: wiiu
description: >-
  Expert Wii U homebrew development reference for native Cafe OS applications, ports,
  GX2/Latte graphics, devkitPPC/wut, CafeGLSL, Aroma, RPX/WUHB, AX audio,
  VPAD/KPAD, memory/cache coherency, SDL2, networking, Cemu, debugging and
  hardware validation. Trigger for Wii U, WiiU, GX2, wut, Cafe OS, Espresso,
  Latte, Aroma, .rpx, .wuhb, R700 shaders, Wii U engine ports and related tools.
  Use the development-goal and hardware-validation process in section 14 for
  changes requiring on-console verification; do not apply the stop gate to
  reference questions, source audits, docs or host-side-only work.
---

# Nintendo Wii U Homebrew Development — Engineering Skill

**Research refresh:** 2026-11-13. **Target:** native Wii U (Cafe OS) using open-source tools, principally wut/devkitPPC and Aroma. **Not** vWii/libogc development (except when explicitly comparing it).

## How to use this skill

- Be a practical Wii U systems/porting engineer. **Inspect the actual project and installed headers before assuming an API or build flag exists.** Favor official wut code, headers, examples, upstream release notes and hardware-tested projects over remembered snippets.
- Confidence labels: **[DOCUMENTED]** directly supported by upstream docs/headers; **[IMPLEMENTED]** demonstrated in identifiable public source; **[HARDWARE-REPORTED]** project author reports real-console testing, not independently reproduced; **[INFERRED]** plausible engineering guidance needing a test; **[UNVERIFIED]** do not rely on without validation. Labels are about the **claim**, not a blanket endorsement of the cited repository.
- Track evidence separately: **SOURCE-CHECKED / BUILD-VERIFIED / CEMU-VERIFIED / HARDWARE-VERIFIED**. Neither compiling nor passing in Cemu proves that hardware cache, synchronization, timing, memory, suspend/resume or controller behavior is correct.
- Do not present old prohibitions as hardware limits when a new implementation provides an alternative. Distinguish **possible**, **supported by a library**, **practical for this project**, and **measured on hardware**.
- Use *only* legitimately obtained game assets and open-source/community toolchains; do not require proprietary/leaked Nintendo Cafe SDK components. Keep legal compatibility of engine and port licenses in mind.
- The current links, examples, versions and recommendations below are a **snapshot**. Check the upstream repo and installed versions again before starting a new project or pinning CI.

---

## 1. HARDWARE ARCHITECTURE

### CPU — Espresso

- Tri-core, 32-bit, **big-endian** IBM PowerPC 750-family derivative running at about **1.243875 GHz**. [DOCUMENTED]
- Per-core L1 around 32 KiB instruction + 32 KiB data; asymmetric L2 arrangement commonly described as **512 KiB / 2 MiB / 512 KiB** (cores 0/1/2). Consider CPU affinity when profiling but **measure** before pinning the busiest thread to core 1. [DOCUMENTED / INFERRED optimization]
- Hardware floating point and paired-single instructions; **no AltiVec/VMX**. Do not blindly import AltiVec code paths. Verify compiler feature flags/ABI from wut build recipes. [DOCUMENTED]
- PowerPC write-back caches, 32-byte cache lines; CPU data caches are **not automatically coherent** with GPU/DMA consumers. Flush/invalidate and ownership rules matter on console even if Cemu appears correct. [DOCUMENTED]
- Limited per-core execution throughput and memory latency make instruction count, layout, batching and contention relevant. Avoid claims like "core 1 is always faster" without a benchmark. [INFERRED]

### GPU — Latte

- AMD R700/R6xx-family GPU, about **550 MHz**, commonly described as 320 shader ALUs, programmed by the native **GX2** API. The GPU's shader format/ISA is not desktop GLSL or SPIR-V; shader tools translate into GX2-compatible programs. [DOCUMENTED]
- Output targets: TV (system mode may be 480p/720p/1080 modes) and **DRC/GamePad** (854×480). They can present different content; **do not assume that copying TV output is the only valid design**. [DOCUMENTED]
- Supports tiled surfaces, depth/stencil, blending, mipmaps, compressed formats and programmable shaders. Check actual GX2 format/limit enums and hardware behavior for uncommon formats, geometry stages or large textures. In particular, **8192×8192 as a guaranteed maximum was previously only inferred from the GPU family**; mark it unverified until tested. [DOCUMENTED / UNVERIFIED limit]
- **MEM1** is ~32 MiB of fast GPU-friendly memory commonly used for render targets. MEM1 placement and fit are workload-dependent, not hard-coded by resolution alone. [DOCUMENTED]

### RAM, OS and coprocessors

- Physical memory: ~2 GiB DDR3 (MEM2); roughly ~1 GiB can be available to a foreground title under common arrangements, but actual headroom varies with title, plugins, heaps and allocations. Log **allocatable** memory rather than promising a precise free-size figure. [DOCUMENTED]
- **MEM1:** ~32 MiB; **foreground bucket (FG):** commonly described as ~40 MiB of foreground-lifetime memory used for scan buffers and other context-dependent resources; **MEM0:** system/kernel controlled. Verify heap APIs and release/acquire behavior for the chosen runtime. [DOCUMENTED]
- Latte mediates much system I/O. **Starbuck/IOSU** runs security/IO services; use public wut/libmocha wrappers rather than assuming general IOSU or NAND privileges. AX provides the console audio mixer; ordinary homebrew does not program the audio DSP directly. [DOCUMENTED]
- Aroma is a modular homebrew environment, not itself a guarantee of unlimited kernel/NAND rights. Applications, plugins (`.wps`) and modules (`.wms`) are **different targets with different lifecycles**. Do not conflate them. [DOCUMENTED]

### Render-memory arithmetic — corrected

- A 1920×1080, 4-byte-per-pixel RGBA8 buffer costs **8,294,400 bytes = 7.91 MiB** before surface padding/alignment. A same-size 32-bit depth buffer costs another **7.91 MiB**. The **pair is ~15.82 MiB**, not "16 MiB each".
- A 1280×720 RGBA8 + 32-bit depth pair is **~7.03 MiB** before padding. This does **not** account for TV/DRC double buffering, additional offscreen passes, multisampling, mip levels, compression metadata or driver allocations.
- **Do not categorically claim 1080p render targets cannot fit in MEM1.** Instead size every `GX2Surface`, inspect available memory, account for the DRC and benchmark 720p/1080p. The wut `libwhb` graphics helper demonstrates MEM1 allocation for TV and DRC render surfaces; failure/fallback handling is still required. [IMPLEMENTED: S03]

---

## 2. TOOLCHAINS, LIBRARIES AND ENVIRONMENTS

### Default modern stack

- **devkitPro/devkitPPC + wut**: maintained open-source Wii U headers, libraries, toolchain and RPX/WUHB build helpers. Install via devkitPro's package manager; a commonly documented command is `(sudo) (dkp-)pacman -Syu --needed wiiu-dev`. [DOCUMENTED: S01]
- **wut 1.9.2** was the newest upstream GitHub release confirmed on **2026-11-13** (released November 1, 2026). This is a dated reference, **not** an eternal minimum/version requirement. Its notes include new GX2/DMAE semaphore declarations, additional padscore/VPAD APIs, memory fixes and debug documentation. [DOCUMENTED: S02]
- `libwhb` (part of wut) provides **WHBProc, WHBGfx, logging, mounts and GFD shader loaders**. Prefer it for simple applications and proof-of-concept code. For advanced renderers, using raw GX2 is reasonable when the abstraction imposes measurable limitations. [IMPLEMENTED: S03]
- **CafeGLSL** supports both **offline desktop compilation** and runtime Wii U compilation of GLSL to GX2-compatible shaders. The old assertion "there is no offline homebrew shader compiler" is **false**. See section 3. [DOCUMENTED: S04]
- **Aroma** provides the main modern user-facing homebrew environment; **WUPS** plugins and **WUMS** modules need their own framework, lifecycle and version checks. [DOCUMENTED: S05]
- Other useful ports/libraries (check packages and versions): SDL2, SDL2_image, SDL2_mixer, SDL2_ttf, SDL2_net, zlib, libpng, curl, mbedTLS, PhysicsFS, libmocha, and optional storage/SMB libraries. Treat package availability, API coverage and hardware stability as distinct questions. [DOCUMENTED: S01]

### Version/ABI traps

- Build against headers/libraries matching your installed devkitPro packages. Rebuild **every** module with compatible versions after ABI-impacting changes; avoid mixing stale binaries and SDK fragments.
- wut newer releases have migrated library behavior, C++ threading/newlib support and build conventions. Do **not** paste legacy `-lwhb`, `-lwut`, `wut_enable_*` or `wut.mk` recipes blindly: use a current sample and the actual wut link rules. [DOCUMENTED: S01, S02]
- **Aroma Beta-27** was the latest confirmed release on 2026-11-13; Beta-26 documented a **devkitPPC r49-related plugin heap-corruption compatibility problem**. For affected builds use **WUPS 0.9.1+** (plugins) and **WUMS 0.3.6+** (modules), with an Aroma environment that supports the new builds. This concerns plugins/modules and should **not** be generalized as "every wut application crashes under r49". [DOCUMENTED: S05]
- Do not use proprietary Nintendo Cafe SDK files/headers or assume APIs are exposed simply because they appear in an emulator/decompilation. Prefer the wut API and cite reverse-engineered behavior with uncertainty.

---

## 3. GX2 GRAPHICS, SHADERS AND PRESENTATION

### Choose a rendering layer first

1. **Small program / diagnostic / simple UI:** start with `WHBProcInit()` and `WHBGfxInit()`, then use `WHBGfxBeginRender*`, `WHBGfxFinishRender*`, and `WHBGfxShutdown()` as appropriate. Inspect current samples and call order. [IMPLEMENTED: S03]
2. **Port or performance-critical 3D engine:** manage GX2 state, presentation, buffering, allocations and synchronization directly when needed. You must then own foreground transitions and all resource lifetimes. [INFERRED]
3. **OpenGL compatibility wrapper:** may be useful for experimentation or narrow APIs, but **do not assume general desktop OpenGL/ANGLE performance or compatibility**. Native GX2 is the safer default for substantial 3D ports. A small experimental wrapper such as `gx2gl` exists; treat it as exploratory, not as a universal solution. [IMPLEMENTED: S14]

### Raw GX2 initialization — conceptual sequence (inspect exact current signatures)

1. Initialize ProcUI/lifecycle control and GX2; configure a command-buffer pool when required.
2. Query TV/DRC scan modes and calculate scan buffer sizes/alignment using the GX2 sizing APIs (`GX2GetSystemTVScanMode`, `GX2CalcTVSize`, `GX2CalcDRCSize`, etc.).
3. Allocate **scan buffers from an appropriate foreground heap**, honoring the actual API's lifetime and alignment contract; bind them through `GX2SetTVBuffer`/`GX2SetDRCBuffer`.
4. Describe and size color/depth surfaces (`GX2CalcSurfaceSizeAndAlignment`), allocate with the **returned** alignment, set render targets, viewports and scissor/depth/blend states.
5. Create/bind context state(s) and ensure presentation modes are enabled.
6. Render required TV/DRC content, copy/resolve into scan buffers as appropriate, swap, flush and coordinate GPU/CPU completion using a deliberate frame-pacing policy.
7. Handle **foreground release/acquire, HOME overlay and exit**; free resources requiring foreground ownership **only after GPU work is safe**, recreate them on reacquire, and re-bind the necessary state. Missing ProcUI handling can hang the console.

**Do not treat this conceptual ordering as a copy-pastable, version-stable code listing.** Use current `libwhb` and `wiiu-examples` to obtain exact API arguments and initialization/teardown order. [IMPLEMENTED: S03, S07]

### The modern shader path — major correction

**Preferred release workflow:** write vertex + pixel GLSL, compile **offline** with [CafeGLSL](https://github.com/Exzap/CafeGLSL), bundle the resulting GFD `.gsh`, then load with wut WHB graphics helpers. [DOCUMENTED: S04]

```sh
# Run on host, using CafeGLSL's desktop compiler binary.
bin/glslcompiler -vs shader.vert -ps shader.frag -o shaders.gsh
# Optional: inspect generated R600 code using -v.
# To enforce a uniform style, use --uniform-mode auto|register|block.
```

```c
/* Illustrative: after loading shaders.gsh into a valid buffer. */
WHBGfxShaderGroup shaderGroup = {0};
WHBGfxLoadGFDShaderGroup(&shaderGroup, 0, shaderFileBuffer);
/* Also bind the appropriate shaders/fetch shader, attributes, textures,
   and use the reported GX2ShaderMode before drawing. */
```

- Compile **vertex/pixel shaders as a pair**, because a GX2 draw uses **one shader uniform mode for both stages**. For block-mode programs call `GX2SetShaderMode(GX2_SHADER_MODE_UNIFORM_BLOCK)`; for register-mode programs use its corresponding mode. The CafeGLSL CLI prints the required mode; **do not infer it from the GLSL source alone**. [DOCUMENTED: S04]
- `GX2SetVertexUniformReg`/pixel equivalent vs uniform-block bindings (`GX2Set*UniformBlock`) are different paths. CafeGLSL `--uniform-mode auto` can move loose uniforms into a block when needed; inspect the compiler's output/reflection and bind accordingly. [DOCUMENTED: S04]
- A `GX2FetchShader` maps the vertex attribute layout. Derive layouts and stride/format/endian handling from the actual assets/`GX2AttribStream` declarations; changing the vertex format requires an appropriate fetch shader.
- Shader **uniform-block payloads** often need per-32-bit-word conversion from the PowerPC CPU's big-endian representation to the GPU-expected representation. **Do not blindly byteswap every GPU buffer**: check the actual shader ABI, asset layout, and helper behavior, and test a matrix/color pattern. The Crema implementation documents such conversions for its uniform upload. [HARDWARE-REPORTED: S08]
- **Runtime** CafeGLSL still exists and can be appropriate for procedural shaders, tools or bring-up: `InitGLSLCompiler()`, `CompileShaderPair()`, `DestroyGLSLCompiler()`. It adds dependencies/compilation cost. Current CafeGLSL specifically documents **no geometry, tessellation or compute shader support**; do not assume a full desktop GL shader pipeline. [DOCUMENTED: S04]
- Hand-assembled R700 shaders and prebuilt `.gsh` binaries are alternatives for specialized needs. Their existence should not make offline GLSL compilation disappear from recommendations.

### Surfaces, tiling and cache

- Size and align every `GX2Surface` with `GX2CalcSurfaceSizeAndAlignment` and the surface's returned size/alignment. Prefer tiled texture layouts for sampling performance **where testing supports it**; use linear staging surfaces for simple CPU writes and `GX2CopySurface`/other supported transfers to tiled destinations. [IMPLEMENTED: S03, S08]
- For mipmapped textures, validate every mip level's dimensions, offset, format and tile mode. Sample with a matching sampler. Without mipmaps, heavily minified textured surfaces may thrash caches on hardware. [HARDWARE-REPORTED: S08]
- Do not assume surfaces are just contiguous linear RGBA. A CPU-side PNG loader must respect format conversion, tiling, endianness and alignment rather than treating the final GPU image as a flat bitmap.
- Do not modify a resource while a submitted GPU command still references it. Reuse dynamic vertex/index/uniform ring slots only after appropriate GPU completion or fences (not wall-clock sleeps). [DOCUMENTED: S02; HARDWARE-REPORTED: S08]

### Pipelining and render hazards

- **`GX2DrawDone()` serializes CPU and GPU completion**. It is useful for bring-up and shutdown but may destroy multi-frame overlap. The current `WHBGfxFinishRender()` does call `GX2DrawDone()`; profile before replacing the helper. [IMPLEMENTED: S03]
- For advanced pacing, track scan-buffer swaps/flips, reserve 2–3 safe dynamic-data slots, and use GX2/DMAE semaphores or documented sync primitives. Beware CPU/command-processor/GPU ownership differences. Do not label an OSTime delay a "fence". [DOCUMENTED: S02; HARDWARE-REPORTED: S08]
- On render-target → texture transitions, ensure preceding writes have completed and proper resource/cache transitions are performed; restore shader, sampler, depth and blend state after offscreen passes. Diagnose actual synchronization instead of guessing at a universal invalidation incantation. [HARDWARE-REPORTED: S09]
- Display lists: do not assume all GX2/GX2R operations can be safely recorded and replayed. A public engine reports hardware issues when replaying certain state/setup operations inside lists. Benchmark indexed batches, instancing and simple immediate command streams before adopting display lists. [HARDWARE-REPORTED: S08]
- Separate GamePad presentation from TV presentation when beneficial. A HUD/map/inventory screen often makes better use of DRC than blindly duplicating TV output. At minimum show deliberate content on both screens, and test screen-off/TV-only behavior explicitly.

---

## 4. INPUT — VPAD, KPAD/WPAD AND THIRD-PARTY CONTROLLERS

- The **GamePad** is polled through VPAD (typically channel 0); Wii Remote/extension and Pro-class devices are accessed through KPAD/WPAD APIs. Initialize the relevant services, poll once per frame/input tick, and build a **normalized input snapshot** for the engine. [DOCUMENTED]
- `VPADRead()` reports hold/trigger/release masks, analog sticks and touch/motion data. Distinguish valid samples, `NO_SAMPLES` and disconnect/invalid-controller results; do not treat a single missing sample as physical unplug. [DOCUMENTED]
- Calibrate touch input with the current VPAD touch API before treating raw coordinates as calibrated screen coordinates; account for the 854×480 DRC space and game UI transforms.
- KPAD extensions can change. Inspect the current `KPADStatus` extension/device type every read, handle Pro/Classic/Nunchuk, and test hot-plug/disconnect, repeated button edges, left/right analog deadzones and controller reassignments.
- Rumble, motion, pointer and GamePad-specific features are **optional**: isolate them behind capability checks instead of blocking the basic gamepad path.
- **Bloopair** maps many third-party Bluetooth devices into Wii U controller interfaces. This broadens real-world input compatibility but is **environment/user-install dependent**, not a promise that every HID device works natively with wut. Check controller-specific mappings. [DOCUMENTED: S12]
- For multiplayer, define player-slot ownership and input merging explicitly; don't just OR every controller into player 1. For network/physics engines, edge-trigger events must be consumed once per simulation step, even when a frame runs several fixed steps.

---

## 5. MEMORY, ENDIANNESS, CACHE COHERENCY AND THREADS

| Region | Approximate budget | Typical use | Critical constraint |
|---|---:|---|---|
| MEM2 (DDR3) | ~1 GiB foreground-title space in common configurations | Code, heap, assets, commands, streaming data | Measure actual free/fragmented budget |
| MEM1 | ~32 MiB | Color/depth render targets; selected hot GPU resources | Fit/alignment and competing allocations |
| FG/foreground bucket | ~40 MiB | TV/DRC scan buffers and foreground-lifetime resources | Release/recreate with ProcUI lifecycle |
| MEM0 | system-controlled | Kernel/services | Not ordinary userspace storage |

- **Never treat a physical region's headline capacity as the current allocatable amount.** Query the actual heap free/allocatable size where possible; log allocation failures and surface sizes.
- `malloc`/`memalign`, MEM exp/frame heaps, GX2R allocators and `WHBGfx` helper allocators are **not interchangeable**. Do not free something through a different allocator family. For raw GX2 buffers use the allocation/lock/unlock model specified by the API.
- **Cache coherency is a data-flow problem, not a universal two-call macro.** A correct raw CPU→GPU path must ensure CPU writes are visible in memory and that the GPU sees the new resource data; use the documented `DCFlushRange`, appropriate `GX2Invalidate` modes, and any GX2R helper semantics required for that resource. Some CPU-inclusive GX2 invalidation modes also handle CPU-cache operations; avoid redundant or wrongly ordered work. [DOCUMENTED: S03; IMPLEMENTED: S08]
- For GPU/DMA→CPU, **wait for the producer to finish**, perform the documented CPU cache invalidation for the destination, and only then read it. Never invalidate still-dirty CPU cache lines unrelated to the GPU transfer. 
- Cache line is roughly 32 bytes; do not flush shared half-lines from different threads. Honor specific GPU buffer/surface alignment, sometimes substantially stricter than a cache line (including 256-byte uniform-block alignment where required).
- Big-endian CPU effects vary by data type. File/network formats must be decoded according to **their own specifications** (many game archives are little-endian, network data is often big-endian); vertex attributes, shaders and PCM may each have a separate contract. **No unconditional byteswap-all-assets rule.**
- Explicit `OSCreateThread()` stacks, affinity, priorities, callback restrictions and thread-safe queues require platform-specific handling. Modern wut/newlib releases have improved C++ threading support, but check your project's current C++ library version and test real contention. Don't assume 8 MiB desktop stacks.
- Treat **foreground release** as a GPU/resource lifecycle event; synchronize outstanding GPU references before returning/freeing the relevant allocations. When returning from HOME, re-create and re-bind display state where necessary.

### Cache bug checklist

1. Who most recently **wrote** the bytes (CPU, GPU, DMAE, AX producer, loader)?
2. Who **reads** them next, and is the writer finished?
3. Which cache has the old copy, and what does the selected GX2/GX2R call already do?
4. Does the pointer/range include all mip levels, index data, uniform blocks or staging tiles?
5. Is the buffer slot being overwritten while the previous frame is in flight?
6. Was the path tested on **hardware**, not just Cemu?

---

## 6. AUDIO — AX / sndcore2 AND SDL2

- Native mixer: **AX** through `sndcore2` (`<sndcore2/*.h>`), commonly configured for a **48 kHz** output mix with TV/DRC routing. Voice count is commonly reported as up to **96**, but use API-returned resource limits rather than assuming 96 voices are always idle. [DOCUMENTED]
- Common formats: 8-bit PCM, 16-bit PCM and Nintendo ADPCM. **Sample byte order is format/API dependent**; for raw CPU-provided PCM16, verify the format's expected byte order against wut/examples before streaming large files.
- Use voice sample-rate conversion/routing (`AXSetVoiceSrcRatio`, mix matrix APIs) rather than pretending all source audio is 48 kHz.
- AX mixes in short frames (commonly **3 ms** at 48 kHz). Audio callbacks must not perform blocking filesystem/network work, expensive allocation, logging floods or shader work. Pre-fill ring buffers and communicate using bounded queues.
- For ports, **SDL2's Wii U audio backend** can simplify integration when supported by the installed SDL2 port. Direct AX is appropriate when fine control over mixing, effects or independent TV/DRC routes is needed.
- Flush/prepare audio buffers according to the AX resource contract; under-runs and starving the callback can crackle. A public project reports real-hardware AX timing and streaming experiments; treat its benchmarks as project-reported. [HARDWARE-REPORTED: S08]

---

## 7. STORAGE, CONTENT, NETWORK AND PERSISTENCE

### Assets and mount points

- Ordinary Aroma homebrew commonly loads from `sd:/wiiu/apps/<app>/<app>.wuhb` (or an equivalent app folder) and uses SD for mutable saves/mods. `.wuhb` packaging can include metadata, icon/splash graphics and embedded content. [DOCUMENTED]
- `WHBMountSdCard()` / wut devoptab and filesystem APIs expose SD paths; examples use paths such as **`fs:/vol/external01/`**. Native Cafe OS paths may also appear as **`/vol/...`** in some libraries. **Do not assume `sd:/`, `fs:/`, and `/vol/` are interchangeable**: log the actual mounted prefix and use absolute paths.
- WUHB embedded content is typically accessible through the title content mount, e.g. `fs:/vol/content/` or `/vol/content/` depending on API. Confirm the package's `CONTENT` mapping and mount semantics before hardcoding paths. [IMPLEMENTED: S08]
- SD FAT32 is case-insensitive/case-preserving in typical configurations. Do not distinguish files only by case. USB mount support depends on libraries, environment and filesystem drivers. Avoid writing NAND/system titles for game saves or ordinary tests.
- Filesystem operations involve IOSU/service overhead. **Batch small assets into a validated archive**, perform large reads, and use asynchronous/loader-thread prefetching for workloads where real hardware shows stalls. A public project measured substantial emulator-versus-console differences in small save-file latency. [HARDWARE-REPORTED: S08]

### Save-data correctness

- Use versioned binary formats, length checks, checksums and explicit endianness; never deserialize raw unvalidated host structs and trust padding or `sizeof` to remain stable.
- For important saves, write a **temporary file**, flush/close it, then replace the live save using a filesystem-supported safe strategy. Atomic rename semantics and crash durability must be **tested on the actual mount**; do not promise transactional guarantees for FAT without verification.
- Ensure saves can tolerate missing, partial, older and corrupt data. Log fallback behavior without creating write loops during startup.

### Networking

- `nsysnet`/BSD-style sockets exist in wut. Network bring-up is environment/version-specific; check available account/connection APIs and actual `AC*` signatures rather than pasting a fixed boot sequence.
- For common networking use cases, evaluate Wii U SDL2_net, curl or mbedTLS portlibs if available. They **do not turn a Wii U into a modern general-purpose TLS/OS environment by magic**; validate protocol versions, certificates, DNS, sockets and timeouts.
- Wii U is typically Wi-Fi unless a compatible LAN adapter is installed. Tests must include real networking conditions, packet loss, disconnect/reconnect and server protocol/endianness.

---

## 8. BUILD, PACKAGING AND FAST ITERATION

### Current defaults

- Recommended install: devkitPro package manager, devkitPPC, wut and Wii U portlibs. Environment variables commonly include `DEVKITPRO=/opt/devkitpro` and `DEVKITPPC=$DEVKITPRO/devkitPPC` on Unix-like installations; don't hardcode those on Windows/MSYS2.
- Cross compiler: `powerpc-eabi-gcc` / `powerpc-eabi-g++`; use the flags and specs provided by current **wut/devkitPro** samples. Cross-compile against the Wii U ABI, not native desktop headers.
- **CMake update:** `$DEVKITPRO/wut/share/wut.toolchain.cmake` is **deprecated**; it forwards to devkitPro CMake and prints a deprecation message. Prefer **`/opt/devkitpro/portlibs/wiiu/bin/powerpc-eabi-cmake`** or **`$DEVKITPRO/cmake/WiiU.cmake`** where suitable. [DOCUMENTED: S06]

```sh
# Example for an existing CMake-based Wii U project; the wrapper chooses Wii U tools.
/opt/devkitpro/portlibs/wiiu/bin/powerpc-eabi-cmake -S . -B build
cmake --build build -j4

# Alternatively, using your installed devkitPro CMake toolchain explicitly:
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE="$DEVKITPRO/cmake/WiiU.cmake"
cmake --build build -j4
```

- wut's documented helpers include `wut_create_rpx(target)` and `wut_create_rpl(target)`; **check the current `wiiu-examples` project for the exact `wut_create_wuhb` syntax/metadata and `CONTENT` arguments** in the installed version. CMake target naming, metadata and packaging vary by sample; do not guess a function signature. [DOCUMENTED: S01, S07]
- Makefile projects should use modern `wut_rules`/devkitPro samples instead of deprecated `wut.mk`. Link with library targets/specs prescribed by the installed tooling; old direct `-lwut -lwhb ...` link recipes may be unnecessary/obsolete.
- Normal pipeline: compile/link ELF → produce `.rpx` with wut tools → create `.wuhb` with `wuhbtool` or build helpers, optionally embedding title assets. Keep debug `.elf`/map files from the same build for crash symbolication.
- Test **Cemu for quick iteration**, then deploy by **wiiload** (if the matching Aroma plugin is present) or SD copy for release validation. Do not claim `wiiload` is always installed or always accepts the same artifact type/command on every environment.

### Asset build pipeline

- **Shaders:** precompile CafeGLSL pairs to `.gsh`; embed content and test every uniform mode on hardware. Generate reproducible shader logs/disassembly for debugging.
- **Textures:** precompute format conversions, mipmaps and useful linear/tiled representation when appropriate; include format, dimensions, alignment and data checks in a manifest.
- **Meshes:** bake vertex/index streams to the exact GX2 fetch-shader layout; define endian behavior explicitly. Compression versus low-CPU read cost is a measured tradeoff.
- **Audio:** convert samples to an AX-compatible format (PCM/ADPCM) and prepare source-rate metadata in advance.
- **CI:** pin image/package versions or record them in artifacts; retain the compiler version, linker map, Git revision and emitted binary hashes. Distinguish *build smoke test* from *Cemu test* and *hardware test*.

---

## 9. DEBUGGING, CEMU AND HARDWARE VALIDATION

### Cemu: excellent accelerator, not a console substitute

- Open-source **Cemu** runs Wii U homebrew and is suitable for UI logic, shader/compiler iteration, basic GX2 correctness and many I/O/CPU bugs. It can hide or behave differently around **CPU/GPU cache incoherence, DMA timing, allocations, controller behavior, ProcUI transitions, performance and race conditions**.
- **Cemu FPS is not Wii U FPS.** Record actual TV mode, rendering resolution, swap interval, vsync, frame count, CPU work and GPU wait times on physical hardware before optimizing.

### Logging and crash analysis

- Use `WHBLogPrintf`, `OSReport`, UDP/file/serial logging where configured, and concise diagnostics around each resource creation. Consider flush/receive reliability when the console crashes.
- For exceptions, preserve SRR0/program counter, exception type, GPRs, stack trace and build ID; map to **the matching unstripped `.elf`** via e.g. `powerpc-eabi-addr2line -f -C -e app.elf 0xADDRESS`.
- `OSFatal` may provide a last-resort visible message, but it is not a normal recovery strategy.
- **Aroma Beta-26+** added a configurable **TCP syslog/command server**, IOPShellModule commands and plugin heap diagnostics. These are **environment features**; check whether the relevant modules are installed/enabled and avoid exposing diagnostic network services unnecessarily. [DOCUMENTED: S05]

### Real GDB debugging — major correction

- The [Aroma-compatible GDBStub plugin](https://github.com/wiiu-env/gdbstub_plugin) supports **TCP port 3000**, up to 512 software breakpoints, one hardware breakpoint, one hardware watchpoint (8-byte accuracy), memory/register inspection, stepping and interrupting execution. It is **not** true that there is no practical GDB stub. [DOCUMENTED: S10]
- Use `-O0 -g` (or a carefully chosen debug configuration) and load **symbols from the matching ELF**. The README recommends a PowerPC architecture setting; in GDB, for example:

```gdb
file app.elf
set arch powerpc:750
set remotetimeout 30
set remote hardware-watchpoint-limit 1
set remote hardware-breakpoint-limit 1
# Replace the address below with your own console's IP:
target remote tcp:192.0.2.10:3000
```

- **Major operational warning:** loading the stub can cause the console to **wait unresponsively for a debugger**; the plugin remains active until shutdown. Do not install it as an always-on dependency for end users. Its setup requires a compatible Aroma/Mocha/kernel-module environment and an appropriate wut build (the README states **wut 1.2.0-2+**). [DOCUMENTED: S10]

### What to test on hardware

1. Launch `.wuhb` from actual SD/Aroma menu and also through development deployment.
2. TV mode/resolution, simultaneous DRC output and GamePad reconnect/off conditions.
3. Repeated HOME foreground release/reacquire and exit/relaunch (including while background I/O is active).
4. CPU-written textures, vertex/index streams, uniform blocks, framebuffer-as-texture and GPU readback.
5. Low-memory/OOM paths, large assets, load/save under realistic SD media, power interruption recovery where safely testable.
6. AX under mixed loads; every supported controller/hot-plug path; Wi-Fi disconnect/reconnect where relevant.
7. Stable frame pacing and thermal/long-session behavior using measurements from the console.

---

## 10. COMMON FAILURE MODES — DIFFERENTIAL DIAGNOSTICS

| Symptom | First suspects | Minimal next observation |
|---|---|---|
| Black TV/DRC, logs continue | Scan buffer allocation/mode, missing enable/copy/swap, incorrect surface target | Log scan mode, sizes, allocations, buffer setup, swap counters |
| HOME causes hang/black screen | ProcUI not pumped, pending GPU accesses, FG lifetime mishandled | Log release/acquire callbacks and completed GPU work |
| Cemu fine, hardware textures corrupted | Wrong cache/ownership transition, mip offsets, tile mode, incorrect staging transfer | Test one fixed test texture with all transitions logged |
| Exploded geometry / flicker | In-flight vertex/index overwrite, fetch shader mismatch, endian/stride/format error | Draw static triangle, then ring-buffer test with explicit synchronization |
| Uniforms wrong / colors swapped | Shader-mode mismatch, block binding/reflection mismatch, word endianness | Use a one-color and identity-matrix probe; log reported shader mode |
| GPU hang after display-list replay | Unsupported recorded state change or unsafe resource reuse | Replace list with direct commands; bisect recorded operations |
| GPU slowdown in textured scenes | No mips, linear textures, overdraw, repeated state changes or waiting every frame | Compare textured/tiled/mipped workload and GPU wait times |
| Audio silence/crackle | Device mix, voice allocation, wrong sample format, callback under-run | Play one short known-good 48-kHz sample and log callback cost |
| Save freezes frame / read stalls | SD/IOSU latency, too many opens, synchronous I/O | Time same operation in Cemu **and hardware**; batch reads |
| SD path works only from wiiload | CWD/mount assumption or absent bundled content | Print real mount root and enumerate expected files |
| C++ runtime/thread crash | ABI/package mismatch, old newlib/wut runtime, thread stack | Record versions; rebuild all objects; test minimal thread |
| Plugin crash on recent Aroma | WUPS/WUMS compatibility (not necessarily your app's heap) | Check Beta-26+ notes and plugin/module build versions |
| Symbolic crash address looks wrong | Wrong build ELF, stripped binary, relocation/address mismatch | Match build ID, ELF/map and runtime load address |

**Rule:** investigate an observed signature with a small distinguishing test; don't shotgun a dozen unverified changes at once.

---

## 11. ANTI-PATTERNS AND NON-NEGOTIABLE SAFETY RULES

1. **Never** infer a file/vertex/shader/audio format's byte order only from CPU endianness.
2. **Never** trust Cemu-only results for cache, frame timing, foreground lifecycle or performance.
3. **Never** overwrite GPU-in-flight dynamic data without fencing or proven retirement.
4. **Never** guess GX2 surface alignment, footprint, mip offset or scan-buffer allocation lifetime.
5. **Never** ignore ProcUI/HOME release/reacquire when owning GX2/foreground resources.
6. **Never** hardcode "1080p cannot fit MEM1"; calculate actual bytes and inspect headroom.
7. **Never** claim shaders require runtime compilation; desktop CafeGLSL exists.
8. **Never** assume one shader uniform mode applies to every compiled program.
9. **Never** use `GX2DrawDone()` everywhere without measuring CPU/GPU overlap impact.
10. **Never** classify all `GX2Invalidate` modes or GX2R calls as identical to an explicit `DCFlushRange` sequence.
11. **Never** assume every SDL/OpenGL feature on desktop is implemented or performant on Wii U.
12. **Never** perform slow I/O or resource allocation inside an AX callback or a tight render loop without benchmarking.
13. **Never** overwrite NAND/system titles as a normal development test or bypass OS permissions as a design assumption.
14. **Never** use an old devkitPro build recipe without checking current samples and API signatures.
15. **Never** let a GDB stub or debug TCP service accidentally ship as a permanent end-user prerequisite.
16. **Never** distribute proprietary game assets with a port unless you have the rights. Document asset extraction requirements cleanly.

---

## 12. ENGINE PORTING DECISION TREE

1. **Inventory the source:** language/ABI, OS calls, SDL/GL usage, asset formats/endian, build system, threads, audio callback, networking, game-data licensing, actual upstream status. List Wii U blockers before implementing.
2. **Choose architecture:** SDL2 where supported, WHBGfx for minimal graphics, raw GX2 for substantial custom rendering; offline CafeGLSL first for fixed shaders. Don't prematurely promise full desktop GL compatibility.
3. **Build bootable logging/null renderer** and verify current build toolchain, `.elf`/`.rpx`/`.wuhb`, launch under Cemu and hardware.
4. **Initialize TV/DRC output and ProcUI**; pass a simple clear-color, HOME suspend/resume, and exit test. Use helpers unless the engine needs custom management.
5. **Normalize VPAD/KPAD input**, including analog deadzones, touch mapping, reconnect and player-slot logic.
6. **Load assets from WUHB/SD**, with correct mounts, explicit endian readers and failure logging. Prebake expensive/portable asset formats.
7. **Bring up renderer in small steps:** triangle → textured quad → depth + mesh → world/lightmap → model → UI → two-screen presentation. Compile shader pairs offline and verify reported uniform modes.
8. **Audit synchronization:** CPU→GPU visibility, GX2R ownership, frame fences, framebuffer→texture hazards and uniform/vertex rings; test each path on real hardware.
9. **Integrate AX or SDL2 audio** and networking where required. Keep callbacks bounded and isolate I/O.
10. **Profile on hardware** with controlled settings: resolution, draw calls, state changes, tiling/mips, GPU drains, SD latency, fixed-timestep logic.
11. **Release hardening:** launch modes, packaged assets, versioned saves, HOME loops, mismatched controller setup, memory pressure, realistic SD cards, user documentation and licensing.

### Special notes for Quake 3–class ports

- There is a public **native Cafe OS/wut/GX2 ioQuake3 port**, [ioQuake3-U](https://github.com/Mayo1970/ioQuake3-U), whose author reports loading maps, world/lightmaps/models, bots and controller support. Inspect it as an implementation reference, **not** a guarantee that every feature or build flavor is complete. [IMPLEMENTED / HARDWARE-REPORTED: S11]
- Frontend/backend decoupling can help translate a legacy `renderergl1`-style draw stream to GX2 while keeping game logic largely intact. Audit every GL feature against your native backend rather than relying on compatibility wrappers.
- Test Quake-family byte-oriented network protocols, QVM loading, pk3/ZIP files, config persistence, little-endian game data and variable-frame-step input separately.
- Use benchmarks from projects such as Crema to form hypotheses about tiling/instancing/mips/frame pacing, then reproduce on your own scene and hardware.

---

## 13. QUICK REFERENCE

```text
TARGET     Native Wii U Cafe OS (not vWii) | Aroma | devkitPPC + wut
CPU        3x PowerPC Espresso ~1.24 GHz; big-endian; paired singles; no AltiVec
GPU        Latte/R700-family ~550 MHz | GX2 | program/fetch shader state
MEMORY     MEM2 title memory ~1 GiB typical | MEM1 ~32 MiB | FG ~40 MiB
1080P      RGBA8 7.91 MiB + 32-bit depth 7.91 MiB before alignment
SHADERS    CafeGLSL desktop CLI -> .gsh (preferred fixed shader pipeline)
           Runtime CafeGLSL when needed; VS+PS must share GX2ShaderMode
BUILD      devkitPro modern CMake / wut_rules; avoid deprecated toolchain path
DISPLAY    WHBProc + WHBGfx for small apps | raw GX2 for controlled engines
SYNC       Diagnose CPU/GPU writer/reader + cache, GX2R, in-flight lifetime
INPUT      VPAD GamePad; KPAD/WPAD Wii Remote/Pro/Classic; optional Bloopair
AUDIO      AX/sndcore2 48 kHz; SDL2 backend where supported
STORAGE    SD + WUHB bundled content, mounts and prefixes explicitly checked
DEBUG      WHBLog / addr2line + Aroma GDBStub over TCP 3000 when configured
TEST       Build -> Cemu logic check -> hardware cache/GPU/ProcUI/perf check
LAST CHECK 2026-11-13; verify upstream current versions before use
```

---

## 14. DEVELOPMENT WORKFLOW WITH HARDWARE VALIDATION GATES

**Scope:** apply to **active changes to running Wii U software** when behavior must be tested on the console. Do **not** block documentation edits, research, code review, shader source inspection or host-only builds behind an unnecessary hardware test.

### States

- **ANALYSIS:** examine the real repository and toolchain; identify exact files, failure signals, risks and planned test.
- **IMPLEMENTATION:** make a focused complete change; explain what changed and its evidence status.
- **BUILD_REQUEST:** report exact tested/intended commands, artifact path(s), deployment route and test expectations. Never imply a build succeeded unless it actually ran.
- **WAITING_FOR_HARDWARE:** when hardware is essential and unavailable to the assistant, state what the user should run and observe; stop making unvalidated follow-on implementation claims. Non-hardware research or clarification may still continue when genuinely useful.
- **VALIDATION:** classify user feedback as **SUCCESS / PARTIAL / FAILURE / INCONCLUSIVE** with observations and missing evidence.
- **DEBUG_PROTOCOL:** isolate an actual failed assertion with a minimal test or up to three high-information diagnostics, retaining the earlier known-good baseline.
- **NEXT_GOAL:** advance after the milestone is validated or the user explicitly accepts its risk/blocker.

### Transition rules

1. ANALYSIS → IMPLEMENTATION: identify the actual files/functions to change; say which details still require checking.
2. IMPLEMENTATION → BUILD_REQUEST: present complete build instructions (not guessed templates); separate commands actually run from suggested commands. Include output artifact(s), deployment route and expected TV/DRC/audio/controller behavior.
3. BUILD_REQUEST → WAITING_FOR_HARDWARE when a console-only result is required. Use the following adaptable report format:

```text
=== HARDWARE TEST REQUIRED ===
GOAL: <specific observable feature>
BUILD: <exact command(s); state if not executed by assistant>
ARTIFACT: <exact expected output path>
DEPLOY: <wiiload or SD path compatible with this setup>
OBSERVE: <specific TV / DRC / audio / input / HOME behavior>
REPORT BACK:
  SUCCESS: <observations + test conditions>
  PARTIAL: <what worked, what did not, logs>
  FAILURE: <repro, exception/SRR0, logs>
  INCONCLUSIVE: <why the test could not be completed>
=== END TEST PROTOCOL ===
```

4. Any meaningful user description of the test counts as feedback; **do not demand magic words** or discard partial results. Ask only for missing distinguishing facts needed to proceed.
5. On FAILURE or PARTIAL, propose **one** isolating change or at most **three** diagnostics. Prefer reproducibility, logs, matched build symbols and a controlled hardware-vs-Cemu comparison. Avoid simultaneous speculative rewrites.
6. A hardware-verified result may advance to NEXT_GOAL. If user explicitly requests to proceed despite absent hardware, label the change **UNVERIFIED ON HARDWARE**, preserve the known-good baseline, and explain the risk.
7. After repeated failed attempts at the **same hypothesis**, escalate to targeted GDB/log capture or a minimal standalone reproduction rather than infinite random patches. Never block unrelated analysis forever.

### Default milestone stack (change when the project provides its own)

1. Boot, log and exit cleanly.
2. Display clear-color on TV and DRC; survive HOME transitions.
3. Read controller input and normalize slots.
4. Load a validated asset from bundled content and SD.
5. Draw a textured triangle/quad and then depth-tested geometry.
6. Upload dynamic buffers with correct cache ownership/sync.
7. Produce short AX audio and optional streaming.
8. Implement main menu and transition to game state.
9. Verify longer session performance and release packaging.

---

## 15. ADVANCED GRAPHICS / SYSTEMS CHECKLIST

Use this section when the engine has progressed past "draws a triangle".

### Frame and GPU architecture

- Timestamp a consistent series: input/update, draw-submit, GPU-wait, flip/vsync, asset I/O. Profile with vsync both on and off to distinguish CPU vs GPU limits.
- Use contiguous state-batched draws and indexed geometry. Benchmark **instancing**, fetch-shader changes and texture/sampler state churn on console. [HARDWARE-REPORTED: S08]
- Distinguish command-buffer pressure, queued swaps, command processor stalls and actual shader/fill bottlenecks. CPU-side `GX2DrawDone()` time alone is not a complete GPU performance metric.
- Ensure dynamic uniform/vertex/index ring slots are safe for the maximum frames in flight. Test memory writes delayed by 1–2 frames to expose stale data.
- For postprocessing and render-to-texture, document each resource's write/read phase, depth-stencil state, resolved/tiled layout and cache operations. Start with simple ping-pong textures.

### Data and asset engineering

- Make asset formats self-describing: version, magic, endian, length, format, alignment and bounds validation. Bake textures, mipmaps and meshes offline when startup I/O/math is expensive.
- Consider separate streaming/loading queues that never block render or AX callbacks. Reconcile cancellation during HOME/exit before releasing memory.
- If a renderer uses a custom fixed-step simulation, consume button edges exactly once per step, cap catch-up frames and interpolate wrapped angles correctly; test on HOME resume.
- Use safe save replacement and validate power-loss/partial files on a throwaway SD test setup, not live system storage.

### Measurement discipline

- Published community figures are **contextual data**, not guaranteed specs. Crema's author reports real-hardware polygon/fill/AX/storage results, and libgui's author reports Wii U menu batching improvements; record the scene, resolution, build, timing and console conditions if using them. [HARDWARE-REPORTED: S08, S13]
- Do not estimate theoretical FPS from the advertised R700 shader count or desktop GPU benchmarks. Real workloads differ in memory, draw submission, shader mix and display mode.

---

## 16. UPSTREAM REFERENCES AND SOURCE PRIORITY

**Source priority:** (1) installed wut headers/source and examples, (2) upstream release notes/readmes and CafeGLSL documentation, (3) public tested ports, (4) emulator implementations/independent research, (5) assumptions awaiting a minimal hardware test. These URLs document the **2026-11-13 research snapshot**; recheck them for breaking changes.

| ID | Repository/source | Why it matters | Evidence type |
|---|---|---|---|
| S01 | [devkitPro/wut](https://github.com/devkitPro/wut) | SDK, installs, APIs, build helpers, C++ changes | Upstream |
| S02 | [wut releases](https://github.com/devkitPro/wut/releases) | Version 1.9.2 on Nov 1, 2026; GX2/DMAE semaphores, VPAD/padscore additions | Upstream release |
| S03 | [wut libwhb graphics implementation](https://github.com/devkitPro/wut/blob/master/libraries/libwhb/src/gfx.c) | Actual render/FG lifetime, MEM1 allocations, `GX2DrawDone()` use | Upstream code |
| S04 | [Exzap/CafeGLSL](https://github.com/Exzap/CafeGLSL) | Offline/runtime shader compilation, CLI, uniform mode, limitations | Upstream docs |
| S05 | [wiiu-env/Aroma releases](https://github.com/wiiu-env/Aroma/releases) | Beta-27 and Beta-26 compatibility/debug facilities | Upstream releases |
| S06 | [wut deprecated CMake toolchain file](https://github.com/devkitPro/wut/blob/master/share/wut.toolchain.cmake) | Points to devkitPro CMake wrapper / `WiiU.cmake` | Upstream code |
| S07 | [devkitPro/wiiu-examples](https://github.com/devkitPro/wiiu-examples) | Modern small, testable sample projects | Upstream samples |
| S08 | [vs-sr-dev/wiiu-crema](https://github.com/vs-sr-dev/wiiu-crema) | GX2R, uniform endian, mips, frame pacing, AX, asset pipeline, author benchmarks | Community + HW-reported |
| S09 | [banjo-kazooie-wiiu](https://github.com/D4ead3ye/banjo-kazooie-wiiu) | Engine-port bug cases: GX2 sync/render targets, HOME, saves | Community |
| S10 | [wiiu-env/gdbstub_plugin](https://github.com/wiiu-env/gdbstub_plugin) | Practical Aroma remote GDB; requirements/limitations | Upstream README |
| S11 | [Mayo1970/ioQuake3-U](https://github.com/Mayo1970/ioQuake3-U) | Native Wii U GX2 Quake 3 renderer and ports | Community + author status |
| S12 | [GaryOderNichts/Bloopair](https://github.com/GaryOderNichts/Bloopair) | Third-party Bluetooth input compatibility | Upstream |
| S13 | [dborth/libgui](https://github.com/dborth/libgui) | Cross-console UI abstraction; Wii U GX2 batching | Community + author benchmarks |
| S14 | [snickerbockers/gx2gl](https://github.com/snickerbockers/gx2gl) | Experimental GL-compatibility direction, **not** a production recommendation | Experimental |
| S15 | [HarbourMasters/Shipwright-WiiU](https://github.com/HarbourMasters/Shipwright-WiiU) | Larger C++/asset/renderer porting reference; verify branch/current Wii U status | Community |
| S16 | [Cemu project](https://github.com/cemu-project/Cemu) | Emulator tests and GX2 behavior research, **not** hardware performance proof | Upstream |

### Maintenance procedure

Before substantial changes, review: wut changelog/releases and headers; devkitPro examples/build-system conventions; CafeGLSL README and releases; Aroma/WUPS/WUMS release compatibility; GDB stub README; and at least one real-console port matching the target use case. Update stale links and date-sensitive notes, and **replace contradicted rules**, not merely append new exceptions.

