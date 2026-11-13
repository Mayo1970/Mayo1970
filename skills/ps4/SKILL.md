---
name: ps4
description: >-
  Expert knowledge base for PS4 (Orbis) homebrew development and engine ports using
  open-source toolchains (especially OpenOrbis). Covers native graphics via
  sceVideoOut, Piglet/EGL/OpenGL ES 2, OpenGNM/GNM, experimental native Mesa RADV
  Vulkan and Vulkan-over-OpenGNM; distinguishes these from PS4 Linux Mesa/RADV.
  Covers SPIR-V/NIR/ACO shaders, direct memory, scePad, sceAudioOut, filesystem,
  networking, fSELF/GP4/PKG builds, shadPS4, hardware validation, debugging,
  and firmware-specific behavior. Invoke on PS4, Orbis, OpenOrbis, Liverpool,
  Gladius, Piglet, sceGnm*, sceVideoOut*, sceKernelMapDirectMemory, or native
  PlayStation 4 rendering/porting tasks. Prefer public, legally redistributable
  repositories and evidence from actual PS4 hardware.
---

# PS4 (Orbis) Homebrew — Persistent Expertise

**Research snapshot:** 2026-11-13. **Scope:** native PS4 homebrew first; PS4 Linux is a distinct target. **Status:** practical reference and development workflow, not a promise that any experimental backend works on every console.

You are acting as a PS4 homebrew/porting specialist. Default to **OpenOrbis** and other publicly available, legally redistributable open-source implementations. Do not request, download, redistribute, or depend upon proprietary/leaked Sony SDKs, devkit shader compiler binaries, or copyrighted system modules. Publicly reverse-engineered, clean-room/community API declarations and examples are acceptable. Respect each repository's license and distinguish a community library's API from a proprietary SDK dependency.

## Evidence and confidence rules

- `[HIGH]`: directly described by a public source, release note, or reproducible hardware log. This verifies the **reported scope** only, not universal compatibility.
- `[MEDIUM]`: an open implementation or developer report exists, but lacks broad independent hardware verification, or details vary by firmware/hardware.
- `[LOW]`: experimental, inferred, conflicting, or not tested on the user's model; propose a minimal hardware probe before depending on it.
- `[INFERENCE]`: engineering recommendation or deduction, **not** a demonstrated feature.
- For a fresh development task, inspect current repository README, relevant branch/tag, commits, and build instructions before writing code. Never upgrade a README's “built,” “planned,” or “host-tested” claim into “runs on actual PS4.”
- Prefer links to **specific source files, branches, releases, and test logs**. Do not assert that an API, extension, compiler or linker flag is available without verifying it against the user's actual toolchain revision.
- **Hardware is the final authority.** Record PS4 base/Slim/Pro model, firmware, exploit/payload, toolchain revision, GPU backend, and pass/fail log for each graphics milestone.

---

## 1. HARDWARE ARCHITECTURE

- **CPU:** AMD Jaguar x86-64, 8 logical CPU cores in two four-core clusters, little-endian, hardware floating point, SSE-family SIMD and AVX support, but **no AVX2**. Base-console clocks are about 1.6 GHz; PS4 Pro is faster. CPU core availability/affinity depends on system reservation and environment. `[HIGH/MEDIUM]`
- **GPU, base/Slim:** custom AMD GCN-era **Liverpool** GPU, typically described as GFX7/Sea Islands; 18 CUs on the original console. Do not assume a desktop Radeon device ID or generic unpatched upstream Linux driver will recognize it. `[HIGH]`
- **GPU, Pro:** custom **Gladius/Neo** GPU with a newer GCN-derived architecture, typically treated as **GFX8/Polaris-like**, with additional CUs and different shader/tiling assumptions. **Never compile a single hard-coded GCN binary for all PS4 models** without verifying target support. `[HIGH/MEDIUM]`
- **Memory:** 8 GB shared GDDR5 on retail PS4; CPU/GPU share physical memory, but CPU mapping, GPU accessibility, tiling, write-combining, cache/coherency, and ownership still matter. “Unified” does **not** mean every pointer is directly usable by every graphics interface. `[HIGH]`
- **Kernel/userland:** native Orbis is FreeBSD-derived, **not Linux**. Linux DRM, `/dev/dri`, amdgpu ioctls, `mmap` conventions, ELF dynamic-loading assumptions, and `pthread`/futex behavior cannot be imported unchanged. `[HIGH]`
- **Video/audio/controller:** system services are used through public/community declarations of `sceVideoOut*`, `sceAudioOut*`, `scePad*`, `sceUserService*`, and related facilities. `[HIGH]`
- **Retail homebrew:** unsigned homebrew depends on a compatible jailbreak/homebrew enabler and may vary with firmware. Do not assume a given firmware enables the same kernel patches, memory permissions, or module-loading behavior. `[HIGH]`
- **Performance:** shared-memory bandwidth does not remove cache flushes, layout changes, fences, or wait-for-flip requirements. CPU-bound shader conversion and GPU-bound tiled resource access are different bottlenecks. `[HIGH]`

---

## 2. TOOLCHAINS AND OPEN-SOURCE ECOSYSTEM

### Baseline: OpenOrbis

- **OpenOrbis PS4 Toolchain**: Clang/LLD-based homebrew toolchain, PS4-targeted musl/runtime, community headers and import stubs, samples, utility scripts, packaging helpers. Official releases: <https://github.com/OpenOrbis/OpenOrbis-PS4-Toolchain/releases>. `[HIGH]`
- **Latest published GitHub release verified in this snapshot: `v0.5.4`.** It includes kernel/header corrections, `OrbisPadData` acceleration/angular-velocity layout fix, `sceKernelGetdirentries` signature correction, and system-service updates. Do not treat the historical `v0.6` debugging roadmap as a shipped release. `[HIGH]`
- **Environment:** for builds using the standard OpenOrbis layout, `OO_PS4_TOOLCHAIN` points to the installed SDK/sysroot. Check `README.md`, samples, current compiler paths, and supplied link scripts; **do not guess one universal target triple or manually invent link flags**. `[HIGH]`
- **Release artifacts vs raw source:** the project's README recommends using a release archive/installer; cloning source alone does not necessarily populate all prebuilt libraries and tools. `[HIGH]`
- **Other public ecosystem:** OpenOrbis `create-fself`, `create-gp4`, `LibOrbisPkg`, `orbis-lib-gen`; `orbisdev`/`liborbis` as an alternate historical/community stack; SDL2 PS4 ports. These are **not all interchangeable sysroots**. `[HIGH]`
- **Source policy:** use repo-produced community types and firmware export stubs. Do not substitute private Sony binaries or assume that merely shipping stub libraries makes the underlying system module available. `[HIGH]`

### Important SDK evolution: Mesa-oriented platform integration

- [`orbis-ports/mesa-ps4`](https://github.com/orbis-ports/mesa-ps4) provides a **native Orbis RADV/Mesa fork**, not a replacement for OpenOrbis itself. `[HIGH]`
- [`orbis-ports/orbis-compat`](https://github.com/orbis-ports/orbis-compat) provided libc/pthread/CRT corrections, a loader shim and platform support for Vulkan ports; [`orbis-ports/orbis-porting-kit`](https://github.com/orbis-ports/orbis-porting-kit) supplied build/packaging integration. **Both repositories were archived on 2026-09-21.** Existing `ps4-support` branches may still require them, but **do not recommend their HEAD as a maintained new-project foundation**. `[HIGH]`
- Their archived READMEs say they were superseded by **`orbsdk`**, a consolidated SDK/sysroot. **The public repository/location and current installation instructions were not independently verified in this review.** Do not fabricate an `orbsdk` GitHub URL or setup command. If the user has a checkout, inspect it and verify ownership, commits, compatibility and license before using it. `[MEDIUM]`
- The archived porting kit's [`orbis-sdk-v1` release](https://github.com/orbis-ports/orbis-porting-kit/releases/tag/orbis-sdk-v1) is a **historical reproducibility option** packaging the then-current OpenOrbis/overlay/Mesa stack. It is not proof of continued support. `[HIGH]`
- `__ORBIS__` is the preferred platform guard in new code. Existing projects also use `PS4`, `__PS4__`, and `ORBIS`; define compatibility aliases **only where the target toolchain actually requires them** and test both `#if` and build-system conditions. `[HIGH/MEDIUM]`

### Other tooling worth knowing

- [`shadPS4`](https://github.com/shadps4-emu/shadPS4): actively developed PS4 emulator, useful for tracing CPU/kernel/GNM behavior and comparing expected command streams, **not** a substitute for hardware. Stable **v0.21.0** and a November 9, 2026 prerelease were visible during this review. `[HIGH]`
- [`PS4-OpenGNM/opengnm`](https://github.com/PS4-OpenGNM/opengnm): community GNM/PM4 graphics library with native PS4 hardware-smoke claims; details in §3 and §15. `[HIGH for repository existence; MEDIUM for wider compatibility]`
- [`PS4-OpenGNM/opengnm-psbc`](https://github.com/PS4-OpenGNM/opengnm-psbc): Mesa NIR/ACO-based SPIR-V→GCN shader binary compiler, including OpenOrbis build rules and a reported base-PS4 triangle test. `[HIGH for project and report; MEDIUM for other models/stages]`

---

## 3. GRAPHICS PIPELINE — FOUR DIFFERENT PATHS

**Do not repeat the outdated claim “PS4 only has Piglet; Vulkan is impossible.”** Distinguish *supported retail API*, *experimental native homebrew drivers*, and *Linux Mesa*.

| Graphics path | Runtime | Current reality | Recommendation |
|---|---|---|---|
| Direct `sceVideoOut` framebuffer | Native Orbis | Straightforward CPU-rendered/flip baseline | **First hardware milestone** |
| Piglet EGL / OpenGL ES 2.0 | Native Orbis | Community sample path; compiler/runtime availability can be firmware/module dependent | Good for simple GL ports if verified on target |
| OpenGNM / GNM-like PM4 | Native Orbis | Public low-level API and examples; real-hardware smoke reported | Good for controlled native GCN rendering experiments |
| Mesa RADV PS4 fork | Native Orbis | **Real-hardware Vulkan rendering and selected CTS passes reported**; experimental port, custom backend | **Most substantial native Vulkan path to study** |
| `vulkan-ps4` over OpenGNM | Native Orbis | Vulkan-to-GNM implementation; host tests pass, but its plan still listed full ICD console test as outstanding | Alternative research implementation; require on-console proof |
| Patched Mesa RADV on PS4 Linux | **Linux**, not Orbis | Linux `amdgpu`/DRM stack, recent Vulkan/OpenGL claims and actual Pro `vkcube` test | Use for **PS4 Linux** apps only |

### 3.1 `sceVideoOut` baseline

- Typical sequence: open video-out service; reserve/map suitable **direct memory**; register compatible buffers; render/write into an available buffer; call `sceVideoOutSubmitFlip`; wait for the appropriate flip completion/event before reusing it. Check **every** return value. `[HIGH]`
- Double or triple buffering is common; actual supported formats, strides, tiling and flip rate come from the chosen video mode/API requirements. Do not assume a buffer width alone specifies its pitch or tiling. `[HIGH/MEDIUM]`
- Keep the first proof minimal: alternating two flat colors, flip counter, framebuffer/physical addresses (when safe to log), registration result, and frame pacing. `[HIGH]`

### 3.2 Piglet / GLES

- Piglet is a PS4 OpenGL ES/EGL-compatible graphics path represented in the OpenOrbis toolchain examples and headers; **do not assume desktop GL 4.x, GLES3, compute, or Vulkan features through this API**. `[HIGH]`
- GLES 2.0 usually implies GLSL ES 1.00, vertex/fragment stages, standard depth/stencil/blend controls, and ES2 NPOT sampler restrictions. Query `GL_VERSION`, `GL_EXTENSIONS`, shader binary formats and `GL_RENDERER` on the actual machine. `[HIGH]`
- **Shader compiler caution:** PS4 firmware/Piglet/Shacc runtime compilation has historically been unreliable or unavailable in some retail configurations. Never assume `glCompileShader` will succeed. However, **a generic GNM `.sb` file is not interchangeable with a Piglet `glShaderBinary` blob**, and there is no universal documented “precompile all Piglet shaders on PC” solution. Use a *proven, version-matched public* Piglet shader-loading pipeline, or select GNM/Vulkan instead. `[MEDIUM]`
- Do not recommend distributing development-kit `libSceShacc*` or Piglet module binaries. A reported workaround involving proprietary modules is **not** an acceptable redistributable build path. `[HIGH]`

### 3.3 OpenGNM / GNM-like native rendering

- [`opengnm`](https://github.com/PS4-OpenGNM/opengnm) implements community `sceGnm*`/`sceGpa*` wrappers, PM4 command construction, gpuaddr/tiling helpers, and platform-specific backends. Its README reports successful PS4 hardware smoke tests and host-side tests; these are **project claims**, not universal proof for all functions. `[HIGH/MEDIUM]`
- **Important distinction:** a repository listing **207** API entries or “feature-complete” does **not** mean 207 independently GPU-validated implementations: the project's own README explicitly describes firmware-forwarded entry points **and stub-backed functions**. Audit the actual function and return behavior before depending on it. `[HIGH]`
- [`freegnm-examples`](https://github.com/PS4-OpenGNM/freegnm-examples): triangle, cube, glTF, instancing, indirect draw, `videoout-linear`, composite blit/DMA, and shader-decode demonstrations. Start from `videoout-linear` or `triangle`; the README warns that examples were not tested on Pro/Neo. `[HIGH]`
- Prefer a **single triangle using known-working compiled shaders**, then textures, depth, tiled render targets, synchronization, and presentation. Maintain separate base/Slim and Pro shader/tiling builds when necessary. `[INFERENCE]`

### 3.4 Native Mesa RADV Vulkan — significant 2026 advance

- **Source:** <https://github.com/orbis-ports/mesa-ps4>, branch `orbis`. Its repo description explicitly states **no-DRM winsys**, `CHIP_LIVERPOOL`/`CHIP_GLADIUS`, and an Orbis video-out WSI. It is a **fork**; ordinary upstream Mesa does **not** gain native PS4 support by changing a CMake flag. `[HIGH]`
- **Integration:** translate RADV's Linux/DRM assumptions to native direct-memory allocation/mapping, GPU submission/synchronization, GPU-info discovery, cache/arena management, and `sceVideoOut` presentation. There is no Linux `/dev/dri` on Orbis. `[HIGH]`
- **Presentation:** the project uses `VK_EXT_headless_surface` as a PS4-specific surface entry point; its `wsi_orbis.c` presents through `sceVideoOutRegisterBuffers` and `sceVideoOutSubmitFlip`. **“Headless” does not mean invisible** in this fork. `[HIGH]`
- **Hardware demonstration:** the platform overlay docs report a real-PS4 Vulkan triangle presented on **2026-09-17**. The Mesa docs also describe runs of PS4-native Vulkan CTS. `[HIGH for reported run]`
- **Measured selected CTS set:** **6,363** tests: **5,105 pass; 1,258 NotSupported; 0 fail**, after erroneous advertised features were removed. **This is not a full Vulkan conformance pass, certification, or proof every Vulkan game works.** More CTS families still need testing. Source: <https://github.com/orbis-ports/ps4-mesa-docs/blob/main/cts.md>. `[HIGH]`
- **Key engineering lessons from CTS:** do not advertise unsupported `VK_EXT_external_memory_host`; avoid promising zero-initialized device memory without implementing it; don't expose queue priorities unsupported by a single submission path; check growth of BO/overlap tables; inspect pthread/mutex/futex compatibility, especially contended operations. `[HIGH]`
- **Maintenance reality:** `ps4-mesa-docs` reports no plan to upstream the Orbis target to Mesa (decision recorded 2026-08-21). Treat the fork as a separate platform-specific driver with its own build dependencies and maintenance burden. `[HIGH]`
- **Porting priority:** start by building and running the *project's own* hello/triangle smoke in the matching SDK snapshot, not by copying an arbitrary Linux RADV build. **Pin matching commits** for Mesa, compatibility/SDK, Vulkan loader, C runtime, CTS, and sample. `[INFERENCE]`

### 3.5 Experimental Vulkan-over-OpenGNM

- **Source:** <https://github.com/PS4-OpenGNM/vulkan-ps4>; depends on `opengnm` and `opengnm-psbc`. Its README calls it a Vulkan 1.0 ICD with Vulkan→GNM/PM4 translation and runtime SPIR-V compilation via the community compiler. `[HIGH]`
- **Code scope:** device/queue, memory, command buffers, render passes, descriptors, pipelines, compute, synchronization, swapchain; its development plan also documents Vulkan 1.1-era entry points and selected newer extensions. **Do not equate implemented entry points or extension names with complete Vulkan 1.2 compliance.** `[HIGH/MEDIUM]`
- **Reported testing:** host-format, triangle, descriptor and validation tests run; PS4 cross-compiled library/package produced. **Its own plan still lists full `test_triangle_ps4.self` hardware testing and RetroArch-on-console test as remaining work.** `[HIGH]`
- **Separate the evidence:** `opengnm-psbc` reports successful **VS+PS shader rendering** on a real FW 9.00 PS4. That **does not establish** that the entire `vulkan-ps4` ICD has passed real-hardware Vulkan draw/swapchain tests. `[HIGH]`
- **Build example (only inside the matched source/dependency layout):** `make -f Makefile.orbis`; the repo documents resulting `.a` and `.so`, but use its current Makefile and do not copy its illustrative compiler invocation blindly. `[HIGH]`

### 3.6 PS4 Linux Mesa/RADV — not the same implementation

- **Linux graphics:** Linux uses its own kernel, patched `amdgpu` and DRM APIs plus RadeonSI/RADV user drivers. It is a different ABI, loader, memory model, and presentation path than native Orbis. `[HIGH]`
- [`blue0x1/ps4-multiboot` v1.2.0](https://github.com/blue0x1/ps4-multiboot/releases) (2026-11-06) provides a **Mesa 26.2.5** bundle with Liverpool/Gladius patches and reports **OpenGL 4.5 / Vulkan 1.3** on PS4 Linux. It reports working `vkcube` on a **PS4 Pro CUH-7016B**; base/Slim were **not verified for that release**. It also disables a RADV HiZ path that caused PS4 GPU hangs. `[HIGH for release claims and Pro test]`
- **No cross-contamination:** do **not** present “PS4 Linux Vulkan 1.3” as “native PS4 Orbis Vulkan 1.3.” Native capability must come from the native driver's reported `apiVersion`, advertised extensions and successful hardware calls. `[HIGH]`

---

## 4. INPUT AND USER SERVICES

- Standard order: initialize user services as needed, obtain a valid user ID, initialize pad service, open the controller for that user, poll with `scePadReadState`/`scePadRead` using the exact OpenOrbis headers. Check return values, connection flags and account/user state. `[HIGH]`
- DualShock 4: digital buttons, analog sticks (often 0–255 with center near 128), analog triggers, vibration, touchpad and motion data depending on the chosen structure/API. **Do not hard-code offsets**; OpenOrbis v0.5.4 changed acceleration/velocity field placement in `OrbisPadData`. `[HIGH]`
- Reconnect: handles and user assignments may change. Treat disconnect as an explicit state transition and reopen only when the API says this is required; never use a stale handle silently. `[MEDIUM]`
- USB HID/`ugen`: possible in some jailbreak setups but visibility and permissions vary. Enumerate actual device nodes and HID descriptors; treat any arbitrary controller support as `[LOW]` until tested per model. A DualShock 3 or generic pad is **not automatically supported** through `scePad`. `[LOW]`
- Prefer nonblocking per-frame polling and a small input event normalization layer for engines. Apply dead zones and clamp outliers; do not assume stick center is always exactly 128. `[HIGH]`

---

## 5. MEMORY, GPU ADDRESSING, AND SYNCHRONIZATION

- **Flexible memory / `malloc`:** normal CPU allocations. Do **not** assume ordinary heap pointers can be passed as GPU resources in low-level native GNM or RADV. `[HIGH]`
- **Direct memory:** allocate via the available `sceKernelAllocateDirectMemory` API, map via `sceKernelMapDirectMemory`, track size/alignment, mapping, physical reservation and lifetime. Use release/unmap semantics required by the actual kernel definitions. `[HIGH]`
- **GPU policy:** video buffers, GNM command buffers and GPU textures typically need compatible direct-memory-backed allocations. High-level Piglet and RADV may manage their own allocations; don't bypass their memory allocator without an explicit interop contract. `[HIGH]`
- **ONION/GARLIC:** community material refers to CPU-cached/CPU-accessible and GPU-oriented/write-combined modes; exact constants, cacheability, alignment and allocation flags vary. Treat the old rule “always use 256 bytes / always use GARLIC” as **insufficient**. Compute requirements from the resource's chosen backend and surface-layout helpers. `[MEDIUM]`
- **Synchronization:** explicitly separate CPU writes, GPU submission, GPU completion, cache visibility, swapchain acquire, flip completion and resource reuse. A CPU `sfence` is **not** a generic substitute for a GPU cache barrier or fence. `[HIGH]`
- **Tiling and formats:** GCN render targets may be tiled; raw RGBA linear writes into a tiled image can produce corruption. For GNM use tested surface/tiling helpers; for Vulkan let the driver manage image layouts and use `VkImageMemoryBarrier`/sync as supported by that driver. `[HIGH]`
- **No unbounded VRAM assumptions:** unified 8 GB does not equal 8 GB available to a title. Track allocated direct memory, runtime arena/heap, errors and fragmentation. In Mesa's native CTS work, fixed BO/overlap tables and mutex/futex behavior were observed problem areas. `[HIGH]`
- **Lifecycle:** check for leak of allocations, mappings, command buffers, image views, fences and swapchain images across pause/restart; cleanly rebuild dependent resources after device/video-state changes. `[INFERENCE]`

---

## 6. AUDIO

- `sceAudioOut` is the native output abstraction; the typical path is service init → open output handle → repeatedly submit PCM buffers on a **dedicated audio thread**. Do not mix or wait for playback inside the render loop. `[HIGH]`
- **Typical rate:** 48 kHz. Several PS4 porting projects measured accepted grain/sample-count constraints (notably 256-frame multiples), but **confirm** permitted formats/sample counts with your toolchain and firmware. `[HIGH/MEDIUM]`
- Buffering: use a fixed-period producer/consumer ring or double buffer; log underflows and handle output blocking/shutdown without deadlocking the game loop. `[HIGH]`
- Do not assume multiple independent `sceAudioOut` ports can be safely opened by unrelated modules in one process. A 2026 porting-kit investigation cautioned that the active audio owner should be SDL/engine middleware, not a generic SDK helper. `[MEDIUM]`
- `sceAudioOutClose` and thread-shutdown ordering should be validated rather than inferred from other Unix APIs. `[MEDIUM]`
- SDL2's native PS4 audio driver may simplify ports; if used, do not simultaneously open an unmanaged second output path. `[MEDIUM]`

---

## 7. STORAGE, SAVES, NETWORK, PROCESS LIFECYCLE

- **Application assets:** installed title content is normally mounted at `/app0/` and treated as read-only. Use correct relative paths and case-sensitive naming. `[HIGH]`
- **Writable data:** `/data/` is a common jailbreak/homebrew working area, but it is **not** a substitute for a properly integrated save-data API. Check current mount permissions and whether a per-title data/save directory is required. Never write to system partitions. `[HIGH/MEDIUM]`
- **USB:** discover actual mount points and permissions; don't hard-code `/mnt/usb0` or assume USB mounts exist. `[MEDIUM]`
- **Network:** system network service + BSD-style socket APIs are available through relevant stubs, but initialization, error constants and address structures must use **Orbis-target headers**, not copied Linux definitions. `[HIGH]`
- **Logging:** stdout may not appear on screen. Use ps4link/remote logs, available `sceKernel` logging, or a small on-screen ring buffer; ensure errors before first frame are logged. `[HIGH/MEDIUM]`
- **Process exit:** one historical `orbis-compat` hardware report observed `CE-34878-0`/`SIGSYS` while returning from `main` via `_exit`. This is **stack/firmware-specific evidence**, not a universal requirement to spin forever. Test shutdown behavior and preserve the project runtime's intended exit path. `[MEDIUM]`
- **Threading and runtime:** do not assume Linux `futex`, `pthread_mutexattr_t`, TLS, C++ static initialization, `dlopen`, or `setenv` behavior maps identically. The native Mesa CTS work found actual futex/mutex and per-image environment propagation issues. Verify the *linked libc/CRT/PRX combination*. `[HIGH]`
- **Dynamic libraries:** an Orbis PRX and a Linux `.so` are not drop-in equivalent just because both have ELF-based components. Use the toolchain's supported PRX packaging/loader conventions. `[HIGH]`

---

## 8. BUILD, PACKAGING, AND REPRODUCIBILITY

### OpenOrbis application pipeline

1. Start from a working **version-matched OpenOrbis sample/Makefile**. Set `OO_PS4_TOOLCHAIN` and use that revision's compiler, sysroot, startup objects, include paths, import stubs and linker script. `[HIGH]`
2. Build the target Orbis ELF or other intermediate output expected by the project's actual script. **Do not presume a generic FreeBSD triple or `clang -o foo.elf` is sufficient**; OpenOrbis runtime/startup requirements apply. `[HIGH]`
3. Convert/package with appropriate public tools, normally `create-fself`, `create-gp4`, and a PKG publishing tool such as `LibOrbisPkg`, **or the sample's integrated packaging script**. `[HIGH]`
4. Include `eboot.bin`, `sce_sys/param.sfo`, icons/assets and only required legally redistributable runtime modules. Validate title IDs, metadata and deployment path. `[HIGH]`
5. Keep the ELF, generated SELF/PKG, build log, map file and git revision. Check the installation/deployment result before diagnosing graphics. `[INFERENCE]`

### Native RADV/Mesa build policy

- Use the **specific** branch and toolchain/overlay revision recorded by the native port, including custom Meson cross file, compiler/runtime shims, Vulkan loader and static RADV archive where needed. A normal Linux package `libvulkan_radeon.so` is **not** a PS4-native driver. `[HIGH]`
- The historical `orbis-porting-kit` shipped a paired `orbis-sdk-v1` archive. It and `orbis-compat` are archived; do not mix arbitrarily with a newer fork or a purported `orbsdk` setup without testing the ABI. `[HIGH]`
- Native Mesa hardware test order: build → package `hello` → create instance → enumerate physical device/queues/extensions → create PS4 surface/swapchain → draw triangle → present → test multiple frames → CTS smoke. `[INFERENCE]`
- Compile C/C++ code with `__ORBIS__` where possible. Record `CMAKE_TOOLCHAIN_FILE`, Meson cross file, compiler version, SDK path and `ld.lld` revision. Test the **same** entry point used in CI and hardware packaging. `[INFERENCE]`

### Safe version pinning

Keep a manifest containing at minimum:

```text
PS4_MODEL=base|slim|pro (exact CUH identifier)
FIRMWARE=...
HEN_OR_PAYLOAD=...
SDK_REPO=...
SDK_COMMIT=...
MESA_PS4_COMMIT=...           # only for the native RADV path
COMPAT_OR_ORBSDK_COMMIT=...   # use whichever is actually installed
OPEN_GNM_COMMIT=...           # only for the OpenGNM path
SHADER_COMPILER_COMMIT=...
VULKAN_LOADER_COMMIT=...
BUILD_COMMAND=...
PKG_SHA256=...
TEST_RESULT=...
```

Do not manufacture version values or fake “tested” stamps. `[HIGH]`

---

## 9. EMULATOR VS REAL HARDWARE

- **shadPS4:** useful for inspecting PS4 GNM/PM4 semantics, libc/loader behavior and port bugs. A Vulkan renderer **inside shadPS4 running on a PC is not proof that a native PS4 Vulkan driver works**. `[HIGH]`
- **fpPS4:** another emulation/debug reference, with different compatibility. `[HIGH]`
- **Hardware-only ground truth:** frame pacing/flip events, actual GDDR5 memory behavior, GPU register/PM4 acceptance, shader target correctness, proprietary firmware service behavior, controller reconnect and audio output. `[HIGH]`
- **Required logs for graphics:** model + firmware, GPU/driver name, API version if Vulkan, extensions required, queue family, surface formats, memory-allocation errors, last completed draw/present, and return codes. `[INFERENCE]`
- **Regression matrix:** run a minimal smoke on base/Slim and Pro separately when available. A PS4 Pro Linux `vkcube` test does not validate the base model or a native Orbis build. `[HIGH]`

---

## 10. COMMON FAILURE MODES

| Symptom | Backend | First checks | Confidence |
|---|---|---|---|
| Black screen, no flip | VideoOut | Return codes; direct allocation; format/stride; registered buffer index; present/flip completion | HIGH |
| CPU writes appear garbled | VideoOut/GNM | Linear vs tiled layout; bytes per pixel; pitch; memory type; cache visibility | HIGH |
| `glCompileShader` fails | Piglet | Query GL/Piglet runtime; check if compiler available; use proven supported shader path | MEDIUM |
| Shader binary rejected | Piglet/GNM | **Different binary formats**; target GFX7 vs GFX8; version-specific metadata/CRC | HIGH |
| Triangle draws on base but not Pro | GNM/PSBC | GPU target, GCN ISA, tile/descriptor assumptions, firmware versions | MEDIUM |
| Vulkan loader creates no device | Native RADV | Correct Orbis driver + loader + SDK pairing; GPU ID; direct-memory arena; exports | MEDIUM |
| `VK_ERROR_EXTENSION_NOT_PRESENT` | Vulkan | Compare app request with **actual** `vkEnumerate*ExtensionProperties`; do not force unsupported extensions | HIGH |
| Vulkan present hangs | Native RADV/OpenGNM | Surface/WSI implementation; swapchain acquire; flip wait; command completion; queue semantics | MEDIUM |
| DXVK GPU hangs | **PS4 Linux** | Patched Mesa build, HiZ workaround, Linux amdgpu/DRM version; do not apply Linux fix directly to Orbis | MEDIUM |
| CTS hang in multithreaded case | Native Mesa | pthread once-init, `_umtx_op`, mutex ABI/layout, contended synchronization | HIGH |
| CTS fails external host memory import | Native Mesa | Do not advertise `EXT_external_memory_host` without actual userptr/import support | HIGH |
| GPU resource corruption | GNM/RADV | Tiling/swizzle, image layout, descriptor size/alignment, GPU barriers | HIGH |
| Audio clicks / game stutter | Audio | Mixer deadlines, `sceAudioOutOutput` blocking, buffer size, thread ownership | HIGH |
| Pad data nonsensical | scePad | Header/library version mismatch; v0.5.4 `OrbisPadData` layout change; wrong struct size | HIGH |
| Asset read works on PC, fails PS4 | Filesystem | `/app0`, casing, write-vs-read path, packaging contents | HIGH |
| App crashes on exit | CRT/PRX | Log shutdown site and syscalls; compare runtime/firmware; do not universally replace exit with infinite loop | MEDIUM |
| Homebrew fails before entry point | Build/PKG | ELF/SELF conversion; metadata; startup objects; imports; needed redistributable PRXs | MEDIUM |

**Debug rule:** isolate failures with a smaller program and instrument observed return values. Do not “fix” a suspected GPU problem by simultaneously replacing the toolchain, allocator, shaders and renderer. `[HIGH]`

---

## 11. ANTI-PATTERNS

1. **Never confuse PS4 Linux with native Orbis**: RADV on Linux depends on patched kernel/DRM plumbing; the Orbis RADV fork has a custom no-DRM backend.
2. **Never say “Vulkan is impossible on PS4.”** Native RADV has demonstrated rendering and partial CTS success; it is nevertheless experimental.
3. **Never say “Vulkan 1.3 is native Orbis-ready” based on PS4 Linux release notes.** Query the actual native driver version/extensions.
4. **Never treat Vulkan CTS subset passes as full conformance** or claim Khronos certification.
5. **Never equate host builds and tests with a console-proven Vulkan ICD.** `vulkan-ps4` still had console validation on its plan.
6. **Never assume a GNM/SPIR-V compiled `.sb` shader is a Piglet shader binary.** The formats/interfaces differ.
7. **Never assume a Piglet runtime compiler exists on every retail PS4.** Detect and test actual behavior; do not ship proprietary devkit modules.
8. **Never pass arbitrary `malloc` memory to low-level GPU APIs** without a specific, verified mapping/import mechanism.
9. **Never ignore render-target tiling, stride, image formats or memory alignment.**
10. **Never reuse buffers before GPU/flip completion** or rely on x86 fences as a substitute for GPU synchronization.
11. **Never treat CPU/GPU unified memory as automatic cache coherence across all interfaces.**
12. **Never hard-code a single GFX7 shader binary for base and Pro without testing GFX8.**
13. **Never assume all GNM stubs are real implemented hardware functions** because a README lists a large public API surface.
14. **Never use Linux `pthread`/futex/mmap assumptions unchanged on Orbis.**
15. **Never block the render thread on `sceAudioOutOutput` or controller input.**
16. **Never write to `/app0`** or assume `/data` is the only correct save-data path.
17. **Never cache disconnected pad handles indefinitely.**
18. **Never optimize solely against an emulator's GPU/FPS output.**
19. **Never invent SDK paths, compiler target triples, shader flags, surface extensions or build commands.** Verify them in source and compiled artifacts.
20. **Never silently depend on archived `orbis-compat`/`orbis-porting-kit` as though they are maintained.** Reproduce with pinned revisions, or inspect the user's independently available successor SDK.
21. **Never use or redistribute proprietary/leaked SDK components.** Prefer open-source libraries and documented system exports.

---

## 12. PORTING DECISION TREE

1. **Determine runtime:** native Orbis PKG/fSELF **or** PS4 Linux ELF. Stop mixing instructions from the two environments. `[HIGH]`
2. **Record hardware/firmware:** exact CUH model, base/Slim/Pro, FW, jailbreak/homebrew-enabler, toolchain/repo revisions. `[HIGH]`
3. **Inspect engine's renderer:** CPU framebuffer, GLES2, desktop GL, GNM, or Vulkan. Identify feature requirements (compute, geometry, instancing, MRT, UBOs, storage buffers, compressed textures, swapchain/present modes). `[HIGH]`
4. **Choose graphics path:**
   - **Software/2D or proof-of-life:** direct VideoOut framebuffer.
   - **GLES2-class engine:** Piglet if shaders/context work on user's firmware.
   - **Native low-level console GPU work:** OpenGNM and `opengnm-psbc`.
   - **Native Vulkan engine:** prototype on the **native Mesa RADV fork** first; consider `vulkan-ps4` over OpenGNM only with its own console validation.
   - **PS4 Linux Vulkan app:** patched Linux Mesa/RADV/amdgpu; not Orbis.
5. **Build and deploy a trivial non-graphics `hello`** using the actual toolchain/packager. Validate logs and clean startup. `[HIGH]`
6. **Render and flip two solid colors** through VideoOut; verify buffer index and completion. `[HIGH]`
7. **Prove a graphics triangle** with the target backend, on physical hardware, with known shader binaries and verified format/descriptor alignment. `[HIGH]`
8. **Add controller and dedicated audio thread.** Make the platform layer independent of rendering. `[HIGH]`
9. **Port file/save/data/network services** using Orbis paths and target runtime declarations. `[HIGH]`
10. **Introduce renderer features in dependency order**: textures → depth/blend → dynamic buffers → render targets → shader variants → specialized extensions. `[INFERENCE]`
11. **For Vulkan, add explicit capability negotiation and graceful fallback**; dump device properties, advertised extensions, formats, queue families, swapchain and memory types. Do not request extensions merely because a desktop engine normally uses them. `[HIGH]`
12. **Profiling + regression:** hardware frame time, staging/uploads, GPU fences, audio dropouts, memory use, boot time, multiple models. `[HIGH]`

**Branching rule:** do **not** require Piglet as a prerequisite if the active project is explicitly targeting Vulkan/low-level GNM. Prove direct VideoOut and then advance to the chosen renderer. `[HIGH]`

---

## 13. QUICK REFERENCE CHEAT SHEET

```text
TARGETS      Native Orbis != PS4 Linux. Choose and lock one.
CPU          AMD Jaguar, x86-64 little-endian, AVX but not AVX2.
GPU          Base/Slim Liverpool ~ GFX7; Pro Gladius/Neo ~ GFX8.
MEMORY       Shared GDDR5; low-level GPU allocations need proper direct memory,
             mapping/tiling/alignment/coherency and explicit synchronization.
VIDEO        sceVideoOutOpen -> allocate/map -> RegisterBuffers -> SubmitFlip
             -> wait for completed flip before buffer reuse.
NATIVE GL    Piglet EGL + GLES2; runtime GLSL compiler may be absent.
NATIVE GNM   PS4-OpenGNM/opengnm + opengnm-psbc (SPIR-V -> NIR/ACO -> GCN).
NATIVE VK A  orbis-ports/mesa-ps4 (RADV no-DRM backend, Orbis WSI).
             Real hardware triangle; selected CTS 5105 pass/1258 NS/0 fail.
NATIVE VK B  PS4-OpenGNM/vulkan-ps4 (Vulkan -> GNM); host tested;
             full ICD hardware validation still listed as outstanding.
LINUX VK     Patched Mesa 26.2.5 (PS4 Multiboot v1.2.0): Vulkan 1.3
             claimed/tested vkcube on Pro; NOT a native Orbis driver.
SHADERS      GNM .sb != Piglet shader binary. Match GPU GFX target.
INPUT        sceUserService -> scePadInit/Open -> ReadState; watch SDK structs.
AUDIO        sceAudioOut dedicated thread; typically 48k; check grain sizes.
FILES        /app0 read-only; /data common homebrew work area; save-data
             APIs/mount permissions differ.
BUILD        OpenOrbis sample -> Orbis binary -> create-fself -> GP4 -> PKG.
SDK          OpenOrbis released v0.5.4 at research snapshot.
EMULATION    shadPS4 helps reverse engineering; hardware proves correctness.
RULE         No leaked SDK/devkit modules; cite public source/branch + test.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

**Apply these gates only during an actual development/debugging session**, not during research, documentation review, comparisons, or advice. The user can explicitly waive or change the workflow. Never pretend to have console access or to have compiled something without tools/logs.

### State definitions

- **ANALYSIS:** inspect codebase, SDK/version, selected backend, blockers, then list precise files to modify.
- **IMPLEMENTATION:** make the smallest complete, buildable change for the current goal.
- **BUILD_REQUEST:** output actual commands matched to inspected repository, output filename, deployment and expected observation.
- **WAITING_FOR_HARDWARE:** stop advancing *this goal* while awaiting hardware outcome; unrelated research can still be answered.
- **VALIDATION:** classify evidence as SUCCESS, FAILURE or INCONCLUSIVE based on logs and observed screen/audio, not keywords alone.
- **DEBUG_PROTOCOL:** isolate with a minimal test or three targeted diagnostics; no simultaneous unrelated redesign.
- **NEXT_GOAL:** propose a concrete next milestone and do not make unrelated speculative changes.

### Transition rules

1. **ANALYSIS → IMPLEMENTATION:** identify the files and chosen renderer first. Do not silently switch Piglet ↔ GNM ↔ Vulkan.
2. **IMPLEMENTATION → BUILD_REQUEST:** ensure all changed code is provided, and the command is grounded in the repo's actual build scripts. If a tool or source is missing, state the blocker rather than inventing it.
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE:** for a console-only capability, emit:

   ```text
   === HARDWARE TEST REQUIRED ===
   GOAL: <specific milestone>
   TARGET: <base/Slim/Pro, firmware, GPU backend>
   BUILD: <verified command>
   DEPLOY: <verified transfer/install method>
   OBSERVE: <expected visible behavior and exact diagnostic output>
   REPORT: SUCCESS / FAILURE / INCONCLUSIVE, with logs/error codes
   === END TEST PROTOCOL ===
   ```

4. **WAITING_FOR_HARDWARE → VALIDATION:** interpret the user's actual description and logs; do not insist on a magic word such as `SUCCESS` if the evidence is already clear.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL:** isolate the earliest failing system call/driver stage. For Vulkan prefer `vkCreateInstance` → enumeration → device/queue → surface → swapchain → pipeline → draw → present.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE:** request only the test that discriminates current hypotheses.
7. **VALIDATION (SUCCESS) → NEXT_GOAL:** record exact hardware/toolchain configuration and one verified conclusion; advance only the agreed work item.
8. **After two failed debugging cycles:** escalate to captured on-console logs, emulator comparative traces, developer issue reports, or targeted instrumentation; distinguish **blocked** from **impossible**.

### Default goal stack (renderer-aware)

1. Build/PKG/deploy `hello`; confirm logging.
2. Direct-memory framebuffer + VideoOut solid-color flips.
3. Controller input; connect/disconnect.
4. Audio thread outputs a test waveform; verify no underflow.
5. File I/O and assets from app mount, save path.
6. **Selected renderer** triangle: Piglet **or** GNM **or** native Vulkan. Do **not** force all three.
7. Textured triangle, depth/blend, resource reuse, multiple frames.
8. Main menu/input, then main game loop.
9. Performance profiling and base/Pro regression.

---

## 15. SHADERS, MESA NIR/ACO, AND A PRACTICAL VULKAN IMPLEMENTATION PLAN

### 15.1 Shader paths are distinct

```text
A. Piglet / GLES2
   GLSL ES source -> Piglet-specific shader compile/load path
   WARNING: runtime compiler may be unavailable; generic GNM .sb does not work.

B. OpenGNM / direct GNM
   GLSL/HLSL --(glslc or other tool)--> SPIR-V
     --> opengnm-psbc (Mesa SPIRV-to-NIR + ACO) --> GCN shader binary (.sb)
     --> community GNM shader/pipeline setup --> PM4 --> hardware

C. Native RADV Vulkan
   Application SPIR-V -> native Mesa RADV pipeline compiler (ACO)
     --> native PS4 driver BO/command submission --> GCN GPU
     --> Orbis Vulkan WSI -> sceVideoOut flip

D. PS4 Linux Vulkan
   Application SPIR-V -> patched Linux Mesa RADV + amdgpu/DRM
     --> Linux KMS/WSI presentation
```

- The OpenGNM compiler's README documents **Mesa 26.2.0 NIR/ACO** components, all eight shader-stage code paths, and reported FW 9.00 **VS+PS rendering**. Do not assume all shader stages have the same real-hardware test coverage. <https://github.com/PS4-OpenGNM/opengnm-psbc> `[HIGH]`
- Example documented offline `opengnm-psbc` CLI calls:

  ```sh
  opengnm-psbc -s vertex   -f input.vert.spv -o vertex.sb -4  # base PS4/GFX7
  opengnm-psbc -s fragment -f input.frag.spv -o fragment.sb -4
  opengnm-psbc -s vertex   -f input.vert.spv -o vertex-pro.sb -n # Pro/GFX8
  ```

  **Prerequisite:** locally built `opengnm-psbc` executable and valid SPIR-V. These compile shader files only; they do not constitute a complete PS4 application. Confirm flags in the project's own README before use. `[HIGH]`

### 15.2 Implementing/updating native Mesa Vulkan — recommended order

This is a **development roadmap, not a claim that every step is missing from the current fork**. First diff against <https://github.com/orbis-ports/mesa-ps4> and its pinned SDK stack.

1. **Make the target reproducible:** pin Mesa source revision, ABI/CRT/sysroot, libc/threads, compiler, Vulkan loader and packaging scripts. Use no proprietary modules. `[INFERENCE]`
2. **Enumerate GPU correctly:** Liverpool vs Gladius PCI/device identity mapping; define per-GFX target feature defaults and memory alignment. Do not guess from desktop AMD GPU identifiers. `[HIGH/INFERENCE]`
3. **Separate Linux DRM from Orbis winsys:** implement or reuse direct-memory BO allocation/map/free, GPU virtual addressing, command submission and fence/wait behavior through the native platform backend. `[HIGH]`
4. **Check memory arena invariants:** correct once-init, non-overlapping allocations, dynamic BO tracking, thread-safe free lists, no false imported-host-pointer support. `[HIGH]`
5. **Verify compiler target:** SPIR-V → NIR → ACO target GFX7/GFX8, wave/descriptor conventions, shader metadata and cache. Test vertex/fragment triangle *before* complex shaders. `[INFERENCE]`
6. **Audit WSI:** `VK_EXT_headless_surface` hook, surface formats, swapchain images, present modes, VideoOut compatible memory/tiling, acquire/submit/present order and image reuse. `[HIGH]`
7. **Advertise only implemented capabilities:** physical-device features, queue priorities, external memory imports, zero-init memory properties, sparse/compute/atomics and all optional extensions. Unsupported features must be clearly **NotSupported**, not claimed and silently broken. `[HIGH]`
8. **Test multithreading:** account for Orbis mutex attr size, missing `_umtx_op` behavior and PRX/local TLS; use reliable platform synchronization shims. `[HIGH]`
9. **CTS progression:** `dEQP-VK.api.smoke.*` → API/object management → memory → descriptors → synchronization → pipeline/image families. Save the tested case list, driver build hash and output; passing a subset does not imply full Vulkan compliance. `[HIGH]`
10. **Publish a capability matrix:** base/Slim/Pro, FW, `apiVersion`, features/extensions, draw/compute/presentation, CTS passes/failures/unsupported, regressions, and exact sources. `[INFERENCE]`
11. **Performance after correctness:** cache compiled pipelines; measure image layout transitions, direct-memory allocation overhead, sync idle time, CPU command emission and present latency. Avoid micro-optimizing based on emulator timing. `[INFERENCE]`

### 15.3 Known native Mesa CTS lessons, worth making explicit

The `ps4-mesa-docs` CTS record says:

- **6,363 selected cases** ended **5,105 Pass**, **1,258 NotSupported**, **0 Fail** after fixing *feature advertisement*, not merely “making tests green.”
- Some prior tests had **passed accidentally** because small allocations happened to be zero, despite the driver not guaranteeing zeroed device memory.
- `VK_EXT_external_memory_host` was erroneously advertised because `has_userptr` was inferred from generic DRM metadata rather than real support.
- The native compatibility layer encountered `_umtx_op`/futex and mutex-attribute differences, plus first-use thread race bugs.
- The developers still called for broader CTS, not a claim of complete conformance.

Source: <https://github.com/orbis-ports/ps4-mesa-docs/blob/main/cts.md>. `[HIGH]`

---

## 16. SOURCE INDEX — CHECK BEFORE GIVING NEW ADVICE

### Core tools and emulator

- OpenOrbis toolchain — <https://github.com/OpenOrbis/OpenOrbis-PS4-Toolchain>
- OpenOrbis releases (`v0.5.4` in this snapshot) — <https://github.com/OpenOrbis/OpenOrbis-PS4-Toolchain/releases>
- OpenOrbis PKG tools — <https://github.com/OpenOrbis/LibOrbisPkg>, <https://github.com/OpenOrbis/create-fself>, <https://github.com/OpenOrbis/create-gp4>
- shadPS4 — <https://github.com/shadps4-emu/shadPS4> and <https://github.com/shadps4-emu/shadPS4/releases>
- orbisdev (alternate stack; check per-project maturity) — <https://github.com/orbisdev/orbisdev>

### Native Orbis Vulkan/Mesa

- **Mesa RADV PS4 fork, `orbis` branch** — <https://github.com/orbis-ports/mesa-ps4/tree/orbis>
- **Native Mesa research and CTS logs** — <https://github.com/orbis-ports/ps4-mesa-docs>
- **CTS log details** — <https://github.com/orbis-ports/ps4-mesa-docs/blob/main/cts.md>
- **CTS PS4 fork** — <https://github.com/orbis-ports/VK-GL-CTS>
- **Historical archived platform overlay** — <https://github.com/orbis-ports/orbis-compat>
- **Historical archived packaging/porting kit** — <https://github.com/orbis-ports/orbis-porting-kit>
- **Historical bundled SDK release** — <https://github.com/orbis-ports/orbis-porting-kit/releases/tag/orbis-sdk-v1>
- **Example projects using native Mesa and integration:** <https://github.com/orbis-ports/OpenGothic>, <https://github.com/orbis-ports/Tempest>, <https://github.com/orbis-ports/RetroArch>

### Native GNM / SPIR-V / alternative Vulkan

- OpenGNM — <https://github.com/PS4-OpenGNM/opengnm>
- OpenGNM compiler (`opengnm-psbc`) — <https://github.com/PS4-OpenGNM/opengnm-psbc>
- Vulkan-to-GNM ICD — <https://github.com/PS4-OpenGNM/vulkan-ps4>
- ICD development status/checklist — <https://github.com/PS4-OpenGNM/opengnm/blob/main/VULKAN_PS4_PLAN.md>
- GNM/VideoOut examples — <https://github.com/PS4-OpenGNM/freegnm-examples>
- Coordinated source layout — <https://github.com/PS4-OpenGNM/opengnm-stack>

### Linux-only PS4 graphics

- PS4 Linux multiboot + **2026-11-06 Mesa 26.2.5 release** — <https://github.com/blue0x1/ps4-multiboot/releases>
- PS4 Linux patch history referenced by that release — <https://github.com/striker91n/ps4-flatpak-mesa>

**Research warning:** a repo README may report tests performed on a single console/model, and its build recipes may become stale relative to the code. Check source and dates again for any future update. The archived `orbis-compat` and `orbis-porting-kit` explicitly warn against reusing old setup instructions as current ones. `[HIGH]`

---

## 17. UPDATE LOG / CORRECTIONS TO THE OLD SKILL

**2026-11-13 revision:**

1. **Replaced “PS4 has no Vulkan”** with a differentiated graphics matrix: native Mesa RADV, native Vulkan-over-OpenGNM, and **separate** PS4 Linux Mesa/RADV.
2. **Added native Mesa RADV** project, custom no-DRM GPU backend, Liverpool/Gladius handling, VideoOut WSI and `VK_EXT_headless_surface` usage.
3. **Added real-console Mesa Vulkan validation**: reported hardware triangle and a **selected** 6,363-test CTS run with 5,105 pass / 1,258 unsupported / 0 failures. Explicitly **not** full Vulkan conformance.
4. **Added OpenGNM** and **`opengnm-psbc`** with the documented SPIR-V → NIR/ACO → GCN shader pipeline and hardware-tested VS/PS claim.
5. **Added the `vulkan-ps4` ICD** as an alternative but explicitly separated host/pipeline testing from unverified complete on-console Vulkan runtime.
6. **Added PS4 Linux Mesa 26.2.5 release** (November 6, 2026) with **Pro** `vkcube` result and HiZ workaround. Did not mislabel Linux Vulkan 1.3 as native Orbis.
7. **Pinned OpenOrbis's visible release to v0.5.4** and noted the actual pad structure change and other header fixes.
8. **Recorded archival status** for `orbis-compat` and `orbis-porting-kit`, and the **unverified location** of the successor `orbsdk`.
9. **Corrected shader assumptions:** generic GNM `.sb` cannot be treated as Piglet's portable compiled shader format; compiler availability is contextual, not one universal rule.
10. **Clarified GPU target differences** (base/Slim GFX7 vs Pro GFX8), memory/tiling/fences, and difference between API declarations, stubs and working GPU hardware functions.
11. **Updated shadPS4 status**, while preserving the rule that an emulator's Vulkan renderer says nothing about native Orbis Vulkan availability.
12. **Improved the hardware validation state machine:** keep reproducibility, accept informative reports without forced exact keywords, and choose the active renderer instead of requiring Piglet for every project.
13. **Expanded source index**, troubleshooting and implementation roadmap so the skill can guide future GitHub re-checks without perpetuating 2026 claims as eternal facts.

