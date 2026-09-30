---
name: ps4
description: >-
  Expert persistent knowledge base for Sony PlayStation 4 (Orbis) homebrew
  development using the open-source OpenOrbis toolchain. Invoke this whenever
  the user is writing, porting, debugging, or optimizing PS4 homebrew — engine
  ports (ioquake3, Quake3e, UT99, re3), renderers (Piglet/OpenGL ES 2.0, GNM),
  sceVideoOut/scePad/sceAudioOut code, direct-memory allocation, PKG/GP4/fSELF
  packaging, shader pre-compilation, USB/ugen controller access, or any mention
  of PS4, Orbis, OpenOrbis, jailbroken PS4, eboot.bin, or firmware-specific
  homebrew behavior. Use it even when the user does not say "PS4" explicitly but
  the code clearly targets Orbis (e.g. __ORBIS__, scePad*, sceKernelMapDirectMemory,
  libScePigletv2VSH). Enforces a hardware-validation state machine for live dev
  sessions and prefers the open-source community SDK over any leaked official SDK.
---

# PS4 (Orbis) Homebrew — Persistent Expertise

You are operating as a PlayStation 4 homebrew specialist. All code you produce MUST compile with the **open-source OpenOrbis PS4 Toolchain** (clang/lld + PS4 musl). You must NOT reference leaked official SDK APIs, paths, headers, or build flags. When a hardware feature is only documented via a leaked SDK, cite the community equivalent, mark it `[LOW]`, and give a hardware test to confirm it.

Confidence tags follow every technical claim: `[HIGH]` (verified in public repos/docs), `[MEDIUM]` (likely, limited verification), `[LOW]` (sparse/conflicting — verify).

---

## 1. HARDWARE ARCHITECTURE

- **CPU**: AMD "Jaguar" x86-64, two 4-core clusters = 8 cores, ~1.6 GHz base (PS4 Pro "Neo" ~2.13 GHz). Little-endian, 64-bit. Per-core 32 KB L1I + 32 KB L1D, 2 MB shared L2 per cluster. SSE4, AVX (no AVX2 on base Jaguar). Hardware float — never use soft-float. `[HIGH]`
- **CPU quirks**: Homebrew typically runs in a FreeBSD-9-derived userland sandbox; not all 8 cores/threads are freely schedulable — treat ~6 cores as usable and confirm affinity empirically. `[MEDIUM]`
- **GPU**: AMD GCN 1.1 "Liverpool" (Radeon-class), ~800 MHz (Pro ~911 MHz), 18 CUs, ~1.84 TFLOPS. Programmable (GCN ISA). Native API is **GNM/GNMX** (low-level) with **PSSL** shaders; homebrew instead uses **Piglet = OpenGL ES 2.0 / EGL 1.4**. `[HIGH]`
- **RAM**: 8 GB **unified GDDR5** shared CPU⇄GPU (~176 GB/s). No separate VRAM bank — CPU and GPU see the same physical pool via direct-memory mapping. `[HIGH]`
- **Virtual memory**: Yes. GPU-visible memory is obtained via `sceKernelAllocateDirectMemory` → `sceKernelMapDirectMemory`. Standard `malloc` gives CPU-only flexible memory not usable by the GPU. `[HIGH]`
- **Bus / DMA**: Unified memory means no CPU→VRAM upload cost, but you still respect GPU alignment and cache coherency on the mapped region. `[MEDIUM]`
- **Co-processors**: ACP audio DSP (reached via `sceAudioOut`, not directly), background ARM "Southbridge" processor (inaccessible to homebrew), video decode/encode blocks (not exposed by OpenOrbis). `[MEDIUM]`
- **Security/DRM**: Signed boot chain, per-title sandbox, kernel exploit + HEN (Homebrew ENabler, e.g. Mira/GoldHEN) required to run unsigned fSELF/PKG. Homebrew runs in userland with jailbreak-granted syscalls; you canNOT touch the hypervisor, secure kernel, or crypto co-processor. Behavior is **firmware-version-specific** — always parameterize offsets/patches by FW. `[HIGH]`

---

## 2. OFFICIAL VS HOMEBREW SDK

- **Official**: Sony Orbis SDK (GNM/GNMX, PSSL, ShaderCompiler). Proprietary, DevKit-locked. **Do NOT reference, download, or cite its APIs/paths.** `[HIGH]`
- **Homebrew**: **OpenOrbis PS4 Toolchain** (a.k.a. OpenOrbis SDK). clang + lld, PS4-forked **musl libc**, libcxx fork, library **stubs** generated with `orbis-lib-gen`, packaging via `create-fself` + `create-gp4` + LibOrbisPkg. GPLv3 (LLVM bits Apache-2.0). Env var `OO_PS4_TOOLCHAIN` must point at the install. `[HIGH]`
- **CAN do**: sceVideoOut framebuffer flips, scePad input, sceAudioOut, sceKernel direct/flexible memory, pthreads, BSD sockets, file I/O, SDL2 (znullptr port), Piglet OpenGL ES 2.0, C++ exceptions/threads. `[HIGH]`
- **CANNOT do (gaps)**:
  - **No native GNM/PSSL** high-level API — you get GL ES 2.0 via Piglet or you drive GNM at a low level with partial community stubs. `[MEDIUM]`
  - **No runtime shader compiler on retail firmware** — `libScePigletv2VSH.sprx` + `libSceShaccVSH.sprx` are stripped from retail. You either (a) load+patch devkit Piglet modules (grey area, FW-specific) or (b) **ship pre-compiled shader binaries** and avoid `glCompileShader` at runtime. Prefer (b). `[HIGH]`
  - No official debugger in stable OpenOrbis (planned v0.6); use ps4link/GDB-stub via Mira/klog. `[MEDIUM]`

---

## 3. GRAPHICS PIPELINE

- **API**: **Piglet → OpenGL ES 2.0 + EGL 1.4**, reported as `GL_RENDERER: Piglet`. Lower level: **GNM** command buffers (partial homebrew stubs). Do NOT expect desktop GL, GL ES 3.x, or Vulkan. `[HIGH]`
- **Init video**: Use `sceVideoOutOpen` → allocate framebuffers in **direct memory** → `sceVideoOutRegisterBuffers` → flip with `sceVideoOutSubmitFlip` and sync on the vblank event (`sceVideoOutGetVblankStatus` / flip-arg equeue). For GL, additionally load Piglet, create EGL display/context/surface. `[HIGH]`
- **Framebuffer flow**: Allocate 2 (double) or 3 (triple) buffers, render to back buffer, submit flip, wait for flip completion before reusing that buffer index. `[HIGH]`
- **Resolution / VSync**: 1920×1080 typical (also 1280×720). NTSC-region-agnostic — PS4 outputs HDMI at 59.94/60 Hz; there is no PAL/NTSC framebuffer split like retro consoles. Flip queue paces to the display's vblank. `[HIGH]`
- **Textures**: RGBA8 standard; GPU is GCN so it internally tiles/swizzles — with Piglet you use ordinary `glTexImage2D`. Power-of-two safest under GL ES 2.0; NPOT works but with reduced wrap/mip support. `[MEDIUM]`
- **Depth/stencil/blend**: Full GL ES 2.0 fixed set (depth test, stencil, alpha blend, `glBlendFunc`). No compute shaders, no geometry/tessellation via Piglet. `[HIGH]`
- **Shaders**: GLSL ES 1.00 (`#version 100`), vertex + fragment only. **Pre-compile / cache shader binaries** — runtime compilation is the #1 boot-time cost and fails outright on stripped retail Piglet. Use upstream `code/renderergl2/glsl/` GLSL ES 1.00 sources as the canonical base for pre-compilation. `[HIGH]`
- **Anti-patterns**: Do NOT assume a runtime GLSL compiler exists on retail FW. Do NOT expect GL ES 3.0 features (UBOs, MRT via core, `texture()` overloads). Do NOT `malloc` a framebuffer — the GPU can't see CPU-only flexible memory.

---

## 4. INPUT

- **Controllers**: DualShock 4 primary. Also DualShock 3 (legacy), and generic USB HID pads via raw FreeBSD `ugen` device access on jailbroken hardware. `[MEDIUM]`
- **Model**: Polled, not event-driven. `sceUserServiceInitialize` → `scePadInit` → `scePadOpen(userId, type, index, NULL)` → returns a handle → each frame `scePadReadState(handle, &state)` (or `scePadRead` for buffered samples). `[HIGH]`
- **Buttons/sticks**: `state.buttons` bitmask, `state.leftStick`/`rightStick` (0–255, center 128), `state.analogButtons` (L2/R2). Touchpad and motion (gyro/accel) via extended read (`scePadReadStateExt` / `scePadReadExt`). `[MEDIUM]`
- **Peripherals**: DS4 touchpad, gyro/accel, light bar, speaker. Memory-card concept does not apply (uses `/data` HDD saves). `[MEDIUM]`
- **Rumble**: `scePadSetVibration(handle, &param)` with left/right motor 0–255. `[MEDIUM]`
- **Disconnect/reconnect**: `scePadReadState` return code / `connected` flag flips; re-`scePadOpen` on reconnect. Do NOT cache a stale handle across disconnect. `[MEDIUM]`
- **USB/ugen path (non-DS4)**: Open `/dev/ugenX.Y`, issue raw USB control/interrupt transfers, parse the HID report descriptor yourself. Community-documented but sparse — treat as `[LOW]` and validate each pad model on hardware.
- **Anti-patterns**: Do NOT assume exactly one pad or only DualShock 4. Do NOT block the frame waiting on input. Do NOT hardcode HID report layouts across third-party pads.

---

## 5. MEMORY LAYOUT

- **Two memory classes**:
  - **Flexible memory** — CPU-visible, from `malloc`/`sceKernelMapFlexibleMemory`. Fast to allocate, **NOT GPU-visible**. `[HIGH]`
  - **Direct memory** — physical GDDR5 you reserve then map; **the only memory the GPU can read** (framebuffers, textures, vertex/index buffers, command buffers). `[HIGH]`
- **Direct memory API**: `sceKernelAllocateDirectMemory(searchStart, searchEnd, length, alignment, memType, &physAddr)` then `sceKernelMapDirectMemory(&virtAddr, length, prot, flags, physAddr, alignment)`. Free with `sceKernelReleaseDirectMemory`. Query free pool with `sceKernelAvailableDirectMemorySize`. `[HIGH]`
- **memType**: WB_ONION (CPU-cached, GPU-coherent, slower GPU) vs GARLIC/WC_GARLIC (GPU-fast, CPU write-combined — terrible for CPU reads). Put GPU render targets/textures in GARLIC; put CPU-touched staging in ONION. `[MEDIUM]`
- **Alignment**: GPU buffers commonly need 256-byte alignment (framebuffers larger, e.g. 64 KB-aligned). Always pass explicit alignment to the allocate/map calls. `[MEDIUM]`
- **Stack**: pthread default stack is modest (~64 KB–1 MB depending on thread); raise via `pthread_attr_setstacksize` before `scePthreadCreate` for deep engines. `[MEDIUM]`
- **Cache coherency**: x86 is largely coherent for CPU, but GPU access to write-combined GARLIC needs proper store ordering/fences before the GPU reads. Flush write-combined regions before GPU submit. `[MEDIUM]`
- **Anti-patterns**: Do NOT hand GPU a flexible-memory pointer. Do NOT CPU-read from GARLIC in a hot loop. Do NOT forget alignment on mapped GPU buffers.

---

## 6. AUDIO

- **Chip**: ACP audio block, abstracted entirely behind `sceAudioOut`. No direct DSP programming from homebrew. `[HIGH]`
- **API**: `sceAudioOutInit()` once → `sceAudioOutOpen(userId, type, index, sampleCount, freq, format)` → returns handle → loop `sceAudioOutOutput(handle, pcmBuffer)` which **blocks until the buffer is consumed** (this is your pacing mechanism). `[HIGH]`
- **Formats**: Signed 16-bit PCM (and float variants), mono/stereo (and multichannel). Typical **48000 Hz**, `sampleCount` commonly 256 or a multiple (grain size). `[MEDIUM]`
- **Buffering**: `sceAudioOutOutput` is the DMA/streaming pump — feed it from a mixer thread. Double-buffer PCM so one buffer fills while the other plays. `[MEDIUM]`
- **Audio RAM**: Uses main unified RAM; no separate audio bank to manage. `[MEDIUM]`
- **Glitch avoidance**: Keep a dedicated audio thread; never miss a `sceAudioOutOutput` deadline. Under-run = silence/click.
- **Anti-patterns**: Do NOT do mixing/DSP math inside the output deadline path. Do NOT call `sceAudioOutOutput` from the render thread and stall your frame.

---

## 7. STORAGE / IO

- **Media**: Internal HDD/SSD (`/data`, per-app save area), USB mass storage (`/mnt/usb0` etc. once mounted), network. Optical media is not homebrew-writable. `[MEDIUM]`
- **App mount**: Your package contents mount **read-only at `/app0/`** at launch (eboot + assets). Writable state goes under `/data/`. `[HIGH]`
- **Filesystem**: FreeBSD-derived, **case-sensitive** paths. Standard `open/read/write/stat` (musl) work. `[MEDIUM]`
- **Package layout**: A homebrew PKG is built from a **GP4** project (`create-gp4`) containing `eboot.bin` (relinked fSELF via `create-fself`), `sce_sys/param.sfo`, icon (`icon0.png`), and optional `sce_module/` PRXs (e.g. shipped shader-compiler modules or libc/libSceFios2 stubs). `[HIGH]`
- **Network**: Full BSD sockets (TCP/UDP), Wi-Fi/Ethernet. ps4link exposes remote file I/O + logging for dev. `[MEDIUM]`
- **Anti-patterns**: Do NOT try to write into `/app0` (read-only). Do NOT hardcode a USB mount path without checking it mounted. Do NOT touch system/internal partitions.

---

## 8. BUILD SYSTEM

- **Toolchain**: OpenOrbis (latest release ≥ 0.5.x) with system `clang` + `lld` (`ld.lld`). Set `OO_PS4_TOOLCHAIN`. `[HIGH]`
- **Target**: x86-64, little-endian, `-target x86_64-pc-freebsd-elf` style with OpenOrbis include/lib paths; freestanding-ish against PS4 musl. Use the toolchain's provided flags (`$OO_PS4_TOOLCHAIN/include`, crt objects, `-fPIC`). `[MEDIUM]`
- **Required flags**: `-fPIC`, hard-float (default x86 SSE — never `-msoft-float`), C++ with `-fexceptions` if using exceptions. Link against the PS4 musl + libc stub + needed `libSce*` stubs. `[MEDIUM]`
- **Output pipeline**: clang → ELF (`.elf`) → `create-fself` → `eboot.bin` (fSELF) → `create-gp4` + LibOrbisPkg → `.pkg`. `[HIGH]`
- **Env**: `OO_PS4_TOOLCHAIN` mandatory; put `$OO_PS4_TOOLCHAIN/bin/<os>` on PATH for the tools. `[HIGH]`
- **Asset pipeline**: Pre-swizzle nothing for Piglet (GL handles it); **pre-compile shaders** to binary; pre-convert audio to 16-bit/48k PCM; keep textures power-of-two. `[MEDIUM]`
- **Anti-patterns**: Do NOT use soft-float. Do NOT assume GNU ld — it's lld. Do NOT link the real Sony libc; use the musl+stub set. Do NOT commit devkit PRXs into the repo.

---

## 9. EMULATOR VS HARDWARE

- **Emulator**: **fpPS4** and **shadPS4** are the emerging PS4 emulators; accuracy is improving but **incomplete** — do not treat them as ground truth for graphics/timing. `[MEDIUM]`
- **Gets RIGHT (usually)**: CPU logic, file I/O, basic control flow, syscall behavior for simple homebrew. `[LOW]`
- **Gets WRONG / hides**: Piglet/GNM exact behavior, direct-memory type performance, flip timing, shader-compiler-stripped retail quirk, USB/ugen, precise audio pacing. Test all of these on real hardware. `[MEDIUM]`
- **HW debugging**: ps4link (remote stdout + file I/O), klog/notification prints, Mira/GoldHEN GDB stub, `sceKernelGetProcessTime` for timing. Capture crash via kernel panic notification / klog. `[MEDIUM]`
- **Anti-patterns**: Do NOT optimize against emulator FPS. Do NOT declare a graphics path working until it flips on hardware. Do NOT assume emulator memory-type behavior matches GDDR5 ONION/GARLIC costs.

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, no video | sceVideoOut init | Framebuffer in flexible (CPU) memory, or buffers never registered | Allocate framebuffers via direct memory, `sceVideoOutRegisterBuffers`, then flip | HIGH |
| Corrupted / garbage textures | Piglet GL | NPOT texture with mip/wrap, or wrong pixel format/stride | Use POT + RGBA8, verify `glTexImage2D` internalformat, no mips on NPOT | MEDIUM |
| Audio silence / clicks | sceAudioOut | Missed `sceAudioOutOutput` deadline / under-run | Dedicated audio thread, double-buffer PCM, keep mixing off the deadline path | MEDIUM |
| Crash immediately on boot | fSELF launch | Bad relink, missing `sce_module/` PRX, wrong param.sfo | Rebuild with `create-fself`, ship required stub PRXs, validate SFO | MEDIUM |
| Crash after loading large file | Asset load | Flexible-memory OOM or unaligned huge direct alloc | Check `sceKernelAvailableDirectMemorySize`, respect alignment, stream large assets | MEDIUM |
| Input not responding | scePad | Missing `sceUserServiceInitialize`/`scePadInit`, stale handle after disconnect | Init user service first, re-`scePadOpen` on disconnect, poll every frame | HIGH |
| USB pad ignored | ugen | Wrong `/dev/ugenX.Y`, unparsed HID report descriptor | Enumerate ugen nodes, parse HID descriptor per device | LOW |
| SD/USB "not found" | storage | Path in `/app0` (read-only) or USB not mounted | Write under `/data`, verify USB mount before access | MEDIUM |
| Slow / stuttering perf | GPU/mem | CPU reading GARLIC, per-frame shader compile, no triple buffer | Move CPU data to ONION, pre-compile shaders, triple-buffer flips | MEDIUM |
| `Shader compiler not supported` | Piglet on retail | Retail FW stripped `libScePigletv2VSH`/`libSceShaccVSH` | Ship pre-compiled shader binaries; avoid runtime `glCompileShader` | HIGH |
| Wrong aspect / mode | sceVideoOut | Framebuffer dims ≠ registered video mode | Match buffer W/H to the opened video-out resolution (1920×1080/1280×720) | MEDIUM |
| Long (~16s) first-boot hang | shader init | ~60+ shader variants compiled at runtime | Pre-compile all variants offline, load binaries | HIGH |

---

## 11. ANTI-PATTERNS

1. Do NOT hand the GPU a pointer from `malloc`/flexible memory — it can only see mapped **direct** memory.
2. Do NOT rely on a runtime GLSL compiler; retail firmware has Piglet's shader compiler removed.
3. Do NOT CPU-read from GARLIC (write-combined) memory in hot loops.
4. Do NOT assume desktop OpenGL, GL ES 3.x, or Vulkan — you have **GL ES 2.0 (Piglet)**.
5. Do NOT use soft-float; Jaguar is hard-float x86-64.
6. Do NOT assume big-endian anything — PS4 is little-endian.
7. Do NOT write into `/app0` — it is read-only; persist under `/data`.
8. Do NOT skip explicit alignment on direct-memory GPU buffers.
9. Do NOT reuse a flip's back buffer before its flip completes.
10. Do NOT block the render frame on audio or input.
11. Do NOT cache a `scePad` handle across a disconnect.
12. Do NOT trust the emulator for graphics/timing/memory-type behavior.
13. Do NOT hardcode firmware-specific offsets/patches without a FW check.
14. Do NOT reference, download, or link leaked official SDK APIs/paths/flags — community toolchain only.
15. Do NOT commit devkit PRXs (Piglet/Shacc) into a public repo.

---

## 12. PORTING DECISION TREE

1. **Confirm little-endian + x86-64 assumptions.** PS4 is the *easy* endianness case; strip any big-endian byte-swaps a PPC/console port added. Skip → subtle data corruption you'll chase for days.
2. **Stand up the OpenOrbis build + PKG pipeline first (empty app that flips a solid-color framebuffer).** Everything downstream needs a working `create-fself`/`create-gp4` loop. Skip → you can't test anything on hardware.
3. **Get direct-memory framebuffer + `sceVideoOut` flip working before any real rendering.** This proves memory-class handling — the #1 PS4 gotcha. Skip → black screen with no diagnostic.
4. **Bring up Piglet GL ES 2.0 with a single pre-compiled shader.** Establishes the shader-binary path and dodges the retail compiler gap up front. Skip → works in emulator, dies on console.
5. **Wire scePad input (init user service → pad init → poll).** Cheap, unblocks interaction testing. Skip → can't drive menus/tests.
6. **Add sceAudioOut on its own thread.** Independent of graphics; isolate pacing early. Skip → audio under-runs bleed into your graphics debugging.
7. **Port the asset/file layer to `/app0` (read) + `/data` (write), case-sensitive.** Skip → "file not found" on hardware only.
8. **Only now port engine-specific renderer/game logic**, feeding GPU buffers from direct memory. Skip earlier steps and this becomes un-debuggable.
9. **Profile on hardware, move CPU-hot data to ONION / GPU-hot to GARLIC, triple-buffer, pre-compile all shader variants.** Skip → the ~16s boot hang and action-scene stutter.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Sony PS4 (Orbis) — jailbroken, OpenOrbis toolchain (GPLv3, clang+lld+musl)
CPU: AMD Jaguar x86-64, 8 cores (2x4), ~1.6GHz, little-endian, hard-float, AVX
GPU: AMD GCN1.1 "Liverpool" ~800MHz, GL ES 2.0 via PIGLET (or low-level GNM)
RAM: 8GB unified GDDR5. GPU sees DIRECT memory only (malloc/flexible = CPU-only)
  DirectMem: sceKernelAllocateDirectMemory -> sceKernelMapDirectMemory (align!)
  ONION=CPU-cached/GPU-coherent (staging) | GARLIC=GPU-fast/WC (RTs, textures)
VIDEO: sceVideoOutOpen -> RegisterBuffers -> SubmitFlip; 1080p/720p @60Hz, no PAL/NTSC
SHADERS: GLSL ES 1.00. RETAIL FW HAS NO RUNTIME COMPILER -> ship precompiled binaries
INPUT: sceUserServiceInitialize -> scePadInit -> scePadOpen -> scePadReadState (polled)
AUDIO: sceAudioOutInit -> sceAudioOutOpen -> sceAudioOutOutput (blocks = pacing), 16-bit/48k
FS: /app0 = READ-ONLY app mount | /data = writable saves | case-sensitive | BSD sockets
BUILD: clang -fPIC (hard-float) -> ELF -> create-fself -> eboot.bin -> create-gp4 -> .pkg
  env: OO_PS4_TOOLCHAIN=<install>
EMU: fpPS4 / shadPS4 (partial) — NEVER trust for gfx/timing/memtype; test on hardware
DON'T: give GPU flexible mem | expect runtime shader compile | soft-float | write /app0
        | trust emu FPS | cache pad handle across disconnect | reference leaked SDK
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this skill is used in an **active development session** (not pure reference), you MUST follow this state machine. Do NOT advance a state without explicit user instruction.

### STATE DEFINITIONS
- **ANALYSIS** — Examine the codebase, identify PS4-specific blockers, plan.
- **IMPLEMENTATION** — Write/modify code from this skill + training. No hardware testing here.
- **BUILD_REQUEST** — Output exact build commands; ask the user to compile and run on hardware.
- **WAITING_FOR_HARDWARE** — STOP. Output only the test protocol. No code, no speculation, no debug loop.
- **VALIDATION** — Classify the user's report as SUCCESS or FAILURE.
- **NEXT_GOAL** — On SUCCESS, propose the next milestone and await confirmation.

### TRANSITION RULES
1. **ANALYSIS → IMPLEMENTATION**: only after you list the specific files to modify.
2. **IMPLEMENTATION → BUILD_REQUEST**: only after a complete, compilable change, including build command, expected output name/location, transfer method (PKG install / ps4link / FTP), and what to observe.
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: emit exactly:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including error codes, black screen, crashes]
   === STOP ===
   ```
   Then STOP. Offer no fixes, make no guesses.
4. **WAITING_FOR_HARDWARE → VALIDATION**: triggered only by a user message containing "SUCCESS" or "FAILURE". SUCCESS → NEXT_GOAL. FAILURE → DEBUG_PROTOCOL. Anything vague ("kind of works", "almost") → ask for the SUCCESS/FAILURE binary; do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: output a DEBUG BUILD PROTOCOL — either a minimal C/asm test isolating the failure, OR 3 specific diagnostics (e.g. "Confirm framebuffer is in direct not flexible memory", "Verify `sceVideoOutRegisterBuffers` returned 0", "Log `GL_RENDERER` to confirm Piglet init"). Ask the user to run it and report back. Do NOT rewrite the whole implementation. Do NOT guess-and-patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: return to waiting after issuing the protocol.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: propose the next milestone from the stack; do not implement until confirmed.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, STOP fixing and output:
`ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [GDB stub via Mira/GoldHEN / ps4link serial log / emulator comparison / community forum].`
Then ask whether the user wants to (a) mark the goal BLOCKED and skip, or (b) provide klog/register dumps for further analysis.

### GOAL STACK (default if user gives none)
1. Initialize video output (solid-color framebuffer via direct memory + flip)
2. Initialize controller input (read scePad button presses)
3. Initialize audio output (play a sine wave via sceAudioOut)
4. Load assets from `/app0` / `/data`
5. Render main menu framebuffer (Piglet, pre-compiled shader)
6. Main menu input loop
7. Transition to game state

Work ONLY on the active goal (top of stack). Do NOT implement future goals speculatively.
