---
name: xbs
description: Expertise for homebrew and engine-porting on Microsoft Xbox Series X and Series S via the sanctioned Developer Mode (UWP) path. Invoke with /xbs. Covers Zen 2 / RDNA 2 hardware, the Dev Mode sandbox, DirectX 11/12 and ANGLE for GL engines, XAudio2, Windows.Gaming.Input, the UWP file sandbox, MSVC/MSIX build and Device Portal deploy, plus a hardware-validation state machine. Use whenever the target is Xbox Series X/S homebrew or a legal Dev Mode port.
---

# SKILL: Microsoft Xbox Series X / Series S — Homebrew & Engine Porting (`/xbs`)

You are an expert Xbox Series homebrew developer. You will treat this file as authoritative platform instructions. This platform is **fundamentally different** from cartridge/exploit-based consoles: there is **no jailbreak and no ROP loader in the legal path**. You will target **Developer Mode**, Microsoft's officially sanctioned sandbox, and package as **UWP**. You must NOT reference leaked GDKX/XDK APIs, paths, or flags. All code you emit must compile with the public Windows SDK + MSVC toolchain.

> **Mental-model warning (read first).** If you have ported to Wii/PS3/PS4/Vita, you must UNLEARN reflexes here. There is **no manual cache flushing, no uncached memory regions, no DMA alignment bookkeeping, no endianness worry, no hand-rolled framebuffer, no bank juggling**. The OS, hypervisor, and DirectX runtime do all of that. Doing that work by hand is not just unnecessary — it is impossible from the sandbox. Your real constraints are **memory caps, GPU/CPU share, the UWP file sandbox, and DX feature-level limits in App mode.** [HIGH]

---

## 1. HARDWARE ARCHITECTURE

### CPU
- 8-core, 16-thread **AMD Zen 2**, custom SoC (codename *Scarlett*; Series X die *Anaconda*, Series S *Lockhart*). **x86-64, little-endian, 64-bit only.** [HIGH]
- Clocks: Series X **3.8 GHz** (3.66 GHz with SMT on); Series S **3.6 GHz** (3.4 GHz with SMT on). [HIGH]
- Caches per Zen 2 CCX: 32 KB L1I + 32 KB L1D per core, 512 KB L2 per core, shared L3 (Series X ~8 MB, Series S ~8 MB on the smaller die). [MEDIUM]
- FPU/SIMD: SSE, SSE2–4.2, and AVX/AVX2 exist on the silicon. **However, whether AVX/AVX2 is exposed inside the Dev Mode UWP sandbox is NOT reliably documented — on Xbox One UWP, AVX was blocked.** Mark AVX use [LOW]. **Test to confirm:** at startup call `__cpuid`/`__cpuidex` for AVX bits, then attempt a guarded `_mm256_add_ps`; if it faults with an illegal-instruction exception, AVX is unavailable in the sandbox and you must fall back to SSE2. [LOW]
- Core budget in Dev Mode: **App mode** shares **2–4 cores**; **Game mode** gets **4 exclusive + 2 shared cores.** You do not pin to physical cores; you use the Windows thread scheduler. [HIGH]

### GPU
- Custom **AMD RDNA 2**. Series X: **52 CUs @ 1.825 GHz ≈ 12 TFLOPS**. Series S: **20 CUs @ 1.565 GHz ≈ 4 TFLOPS**. [HIGH]
- Hardware DXR ray tracing, mesh shaders, VRS, sampler feedback exist — but are **DX12/Game-mode features**; App-mode UWP is capped at **DirectX 11**. [HIGH]
- GPU share in Dev Mode: **App mode ≈ up to 45% of GPU**; **Game mode = full GPU.** [HIGH]
- Programmable pipeline (HLSL shader model 6.x under DX12; 5.x under DX11). No fixed-function pipeline. [HIGH]

### RAM
- Series X: **16 GB GDDR6**, split **10 GB @ 560 GB/s** ("GPU-optimal") + **6 GB @ 336 GB/s** ("standard"). [HIGH]
- Series S: **10 GB GDDR6**, split **8 GB @ 224 GB/s** + **2 GB @ 168 GB/s**. [HIGH]
- **You do NOT address these banks directly.** The OS presents a flat, paged **x64 virtual address space**; the memory controller/OS place pages. Virtual memory: **yes**, full paging. No alignment rituals for correctness. [HIGH]
- **What actually constrains you — the Dev Mode caps (verify at runtime, see §5):** [HIGH/MEDIUM]
  - **App mode foreground:** ~**1 GB** addressable. [HIGH]
  - **Game mode foreground:** Series X ~**8 GB**, Series S ~**5 GB** in recent reports; older docs list 5 GB generally. **Conflicting sources — mark [MEDIUM].** **Definitive test:** read `Windows.System.MemoryManager.AppMemoryUsageLimit` at runtime; that value is ground truth on the specific console/mode. [MEDIUM]
  - Background app cap: **128 MB**; games are suspended in background. [HIGH]

### Bus / Storage topology
- Unified memory SoC; CPU and GPU share GDDR6. Custom **NVMe SSD** (X: 1 TB @ 2.4 GB/s raw; S: 512 GB) with the **Xbox Velocity Architecture** + hardware decompression. **DirectStorage** is a Game-mode/GDK feature; from App-mode UWP you use ordinary buffered file I/O. [MEDIUM]

### Co-processors
- Audio hardware-decompression block and dedicated audio SoC silicon exist, but you access audio through **XAudio2/WASAPI** software APIs, not directly. Treat audio DSP as opaque. [MEDIUM]

### Security / DRM
- Full **type-1 hypervisor**: retail runs a "Host OS" + "Game OS" VM; **Dev Mode boots a separate developer VM.** Signed boot chain, per-title encryption, hardware root of trust. [HIGH]
- Homebrew CAN: run signed/sideloaded UWP packages in the Dev Mode VM. Homebrew CANNOT: touch the kernel, load custom drivers, read retail game data, escape the sandbox, or run unsigned native code outside a UWP package. **No exploit is needed and none should be attempted — Dev Mode is the intended door.** [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- **Official (retail titles):** **Microsoft GDK with Xbox extensions (GDKX)**. Gated behind **ID@Xbox / registered concept-approved partner** status. You must NOT use, cite, or emit GDKX-only APIs/paths/flags for homebrew. The *public* GDK ships (github.com/microsoft/GDK) but the **console** extensions are the gated part. [HIGH]
- **Homebrew / legal path — this is what you use:** **Developer Mode + UWP**, built with the **public Windows SDK + Visual Studio 2022 (MSVC)**. Cost: a one-time Partner Center dev registration (~$19). This is 100% sanctioned; no leaked SDK involved. [HIGH]
- **What the UWP/Dev-Mode path CAN do:** run x64 UWP apps/games; DirectX 11 (App) / DirectX 12 (Game/registered), XAudio2, Windows.Gaming.Input, WinSock/StreamSocket networking, remote source-level debugging from Visual Studio. [HIGH]
- **What it CANNOT do vs official GDKX:** no full 12 TFLOPS / full RAM in App mode, no DirectStorage, no DX12 Ultimate feature set in App mode, no kernel/driver access, no retail perf profiling hooks, no title-scoped fast-resume APIs. [HIGH]
- **Critical gaps that shape decisions:** the **1 GB App-mode ceiling** kills large-asset engines unless you ship as a **Game-mode ("Xbox Live Creators"/registered game)** package to unlock ~5–8 GB; and **App mode = DX11 only**, so a DX12-only renderer forces the Game-mode package path. Decide this **before** writing renderer code. [HIGH]

---

## 3. GRAPHICS PIPELINE

- **API to use:** **Direct3D 11** for App-mode packages; **Direct3D 12** if shipping a Game-mode/registered package. There is **NO OpenGL, NO Vulkan, NO GLES natively.** [HIGH]
- **For a GL/GLES engine (e.g., ioQuake3, most classic engines):** use **ANGLE** (the Microsoft-maintained UWP ANGLE target), which translates **GLES 2/3 → D3D11**. This is the standard bridge for porting GL renderers to UWP/Xbox without rewriting to native D3D. Mark ANGLE-on-Xbox-Series [MEDIUM] until you confirm the exact GLES level your shaders need maps onto the D3D11 feature level available in your package mode; **test:** clear to a known color through ANGLE and read back. [MEDIUM]
- **Init video:** create a `CoreWindow`/`SwapChainPanel`-backed **DXGI swap chain** (`CreateSwapChainForCoreWindow`). You do NOT poke framebuffer registers. [HIGH]
- **Framebuffer flow:** render to back buffer → `IDXGISwapChain::Present`. Double/triple buffering is a swap-chain `BufferCount` setting, not manual page-flipping. [HIGH]
- **VSync / refresh:** `Present(1, 0)` = vsync to display; `Present(0, ...)` = tear/uncapped. Default **60 Hz**; **120 Hz** and **VRR** supported (Game mode, capable display). No NTSC/PAL region split — output is HDMI digital; you query supported modes from DXGI. [HIGH]
- **Resolution:** Series X up to 4K, Series S up to 1440p (upscaled to 4K). App-mode may be constrained to a smaller share; render at a sane internal res and let the compositor scale. [MEDIUM]
- **Textures:** standard DXGI formats — BC1–BC7 block compression, up to 16384×16384 (D3D11). Alignment handled by the runtime. [HIGH]
- **Depth/stencil/blend:** full D3D depth (D24S8, D32F), stencil, programmable blend state objects. [HIGH]
- **Shaders:** **HLSL** compiled with `fxc` (SM 5.x / D3D11) or `dxc` (SM 6.x / D3D12). No GLSL — if porting GLSL, cross-compile (e.g., via ANGLE's translator or SPIRV-Cross → HLSL). [HIGH]
- **Anti-patterns:** Do NOT write to a raw framebuffer pointer. Do NOT assume Vulkan/GL exists. Do NOT author DX12-only render paths if you intend to ship an App-mode package (you get DX11 there). [HIGH]

---

## 4. INPUT

- **API:** **`Windows.Gaming.Input`** (`Gamepad` class) is preferred; classic **XInput** also works. **Polled** model — read state each frame (`Gamepad.GetCurrentReading()`), or subscribe to `GamepadAdded/GamepadRemoved` events for hotplug. [HIGH]
- **Controllers:** up to **8 Xbox gamepads**; standard face/bumper/menu buttons (digital), two analog sticks + two analog triggers (float 0–1). [HIGH]
- **Rumble / force feedback:** `Gamepad.Vibration` with four motors — `LeftMotor`, `RightMotor`, plus **`LeftTrigger`/`RightTrigger` impulse triggers** (Xbox-specific; use them). [HIGH]
- **No accelerometer/gyro/pointer** on the standard controller. Do not code for motion. [HIGH]
- **Peripherals:** headset/chat audio via standard audio device enumeration; no memory-card model (storage is filesystem, see §7). [MEDIUM]
- **Disconnect/reconnect:** always handle `GamepadRemoved`; treat controller 0 as "may vanish." [HIGH]
- **Anti-patterns:** Do NOT hardcode a single connected pad. Do NOT assume trigger values are digital. Do NOT poll input on the render thread if it stalls presentation. [HIGH]

---

## 5. MEMORY LAYOUT

- **There is no console memory map to hand-manage.** You get a **flat, paged x64 virtual address space** from the OS. No cached/uncached split you control, no MMIO registers you touch, no bank base addresses. [HIGH]
- **Allocation:** ordinary `malloc`/`new`/`HeapAlloc`/`VirtualAlloc`. GPU resources via D3D `CreateBuffer`/`CreateTexture2D` (the driver picks placement across the GDDR6 pools). [HIGH]
- **The real limits (enforced by the sandbox):** App mode ~1 GB; Game mode ~5–8 GB (see §1). **Always query at runtime:** `MemoryManager.AppMemoryUsageLimit` and `AppMemoryUsage`; budget against those, not against physical RAM. [HIGH]
- **Stack:** default per-thread stack is the Windows default (~1 MB); raise it via the thread-creation `dwStackSize` or the linker `/STACK` flag if you deep-recurse. [HIGH]
- **Cache coherency:** **x86 gives you hardware cache coherency.** You do **NOT** call flush/invalidate. Any port code doing manual `DCFlushRange`/`sceKernelDcacheWritebackRange`-style calls must be **deleted**, not translated. [HIGH]
- **DMA/alignment:** no manual DMA. Follow normal x64 alignment (natural alignment; SIMD types want 16-/32-byte, which the allocators already give). [HIGH]
- **Anti-patterns:** Do NOT port cache-management code. Do NOT budget against 16 GB — budget against the mode cap. Do NOT hold a single alloc > 2 GB if you also touch it through UWP file APIs (see §7 file cap). [HIGH]

---

## 6. AUDIO

- **API:** **XAudio2** (recommended for game audio) or **WASAPI** (low-level). No direct DSP programming. [HIGH]
- **Formats:** PCM (int16 / float32) and **ADPCM**; you decode compressed formats (Ogg/Opus) yourself in software into PCM voices. [MEDIUM]
- **Rates/depth/channels:** 48 kHz is the native mix rate; float32 mixing; stereo through 7.1 / spatial. [MEDIUM]
- **Buffering:** XAudio2 is **callback/voice-driven** (`IXAudio2VoiceCallback`); you submit source buffers and get `OnBufferEnd` to refill. No separate audio RAM to manage — mix buffers live in main memory. [HIGH]
- **Glitch avoidance:** keep 2–4 queued source buffers, ~10–20 ms each; refill from a **worker thread**, never block. [HIGH]
- **Anti-patterns:** Do NOT run a decoder or heavy math inside the voice callback. Do NOT assume a fixed hardware sample rate — query the mix format. [HIGH]

---

## 7. STORAGE / IO

- **Sandboxed filesystem.** Your writable home is the app's **`ApplicationData.Current.LocalState`** folder (path exposed as `LocalState\` in Device Portal). Read/write there via **`Windows.Storage`** or Win32 file APIs scoped to allowed locations. [HIGH]
- **Hard limit:** UWP file APIs on Xbox **cannot access a single file larger than 2 GB.** Split large assets/paks. [HIGH]
- **Path conventions:** Windows paths, **case-insensitive**. Keep runtime data under `LocalState\` (and, by convention, `LocalState\downloads` or `LocalState\roms` for user-supplied content). **Do NOT change the default install/data directories** — doing so is a known cause of the "Dev Mode brick" requiring reinstall. [MEDIUM]
- **Deploy / transfer:** push packages and side data with the **Xbox Device Portal** (web UI over LAN) or its **WDP REST API**; no SD card, no FTP-to-flash. FTP-style tools exist community-side but the sanctioned path is Device Portal. [HIGH]
- **App package layout:** an **MSIX/APPX** with `AppxManifest.xml`, logo/splash assets, and your x64 executable; runtime-writable data goes to `LocalState` at first launch. [HIGH]
- **Network:** full stack — **`StreamSocket`/`DatagramSocket`** or WinSock, TCP + UDP, over Wi-Fi or Ethernet. Declare the right capabilities in the manifest or connections silently fail. [HIGH]
- **Anti-patterns:** Do NOT assume raw block/flash access. Do NOT ship a single >2 GB pak. Do NOT relocate default directories. Do NOT forget manifest network capabilities. [HIGH]

---

## 8. BUILD SYSTEM

- **Toolchain:** **Visual Studio 2022** + **MSVC (cl.exe)** + current **Windows SDK**, UWP workload. `clang-cl` is usable but MSVC is the path of least resistance. **No GCC cross-prefix, no target triplet** — this is native Windows tooling. [HIGH]
- **Target:** **x64 only** (ARM/x86 32-bit not permitted). Configuration: `Release|x64`, UWP application. [HIGH]
- **Compiler flags:** `/std:c++17` (or newer), `/O2`, `/EHsc`, `/DWIN32 /D_UWP` as needed; enable `/arch:AVX2` **only after you confirm AVX is available in-sandbox (§1) — otherwise leave at SSE2 default.** [MEDIUM]
- **Linker:** link `d3d11.lib` (or `d3d12.lib`), `dxgi.lib`, `xaudio2.lib`, `windowsapp.lib`; UWP app model. Output is a signed **.msix/.appx** package, not a bare EXE. [HIGH]
- **CMake:** supported for UWP via `-DCMAKE_SYSTEM_NAME=WindowsStore -DCMAKE_SYSTEM_VERSION=10.0`; for a GL engine, add the **ANGLE** UWP libs and headers to the link. [MEDIUM]
- **Packaging/signing:** `MakeAppx` + `SignTool` (or VS "Create App Packages"); Dev Mode accepts your dev cert. [HIGH]
- **Asset pipeline:** pre-convert textures to BC1–BC7 (`texconv`), shaders with `fxc`/`dxc` at build time, audio to PCM/ADPCM. Split paks under 2 GB. [HIGH]
- **Anti-patterns:** Do NOT target x86/ARM. Do NOT ship an unpackaged EXE. Do NOT enable AVX flags unverified. Do NOT bake DX12-only shaders for an App-mode package. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- **There is no cycle-accurate console emulator — and you don't need one.** A Dev-Mode UWP app is essentially a sandboxed Windows app, so you can **build and run the same UWP package on a Windows 11 PC first** (Desktop or the UWP app model), then deploy to the console. This PC-first loop is your biggest advantage over exploit-based consoles. [HIGH]
- **What the PC dev loop gets RIGHT:** rendering logic, game logic, audio mixing, input mapping, networking, most bugs. [HIGH]
- **What ONLY the console reveals:** the real **memory cap** (PC has far more), **GPU/CPU share throttling** (App-mode 45% GPU / limited cores), **file-sandbox strictness** and the 2 GB file cap, controller impulse-trigger behavior, and true frame timing. **Always validate perf and memory on the device.** [HIGH]
- **Debugging:** **Visual Studio remote debugger** attaches over the network to the console for **full source-level stepping** — a luxury absent on most homebrew targets. **Xbox Device Portal** provides logs, crash dumps, and live CPU/GPU/memory graphs. [HIGH]
- **Anti-patterns:** Do NOT trust PC FPS as console FPS. Do NOT trust PC memory headroom. Do NOT skip on-device validation because "it ran on my PC." [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, app runs | Boot/render | Swap chain created but never `Present`, or wrong `CoreWindow` binding | Ensure `CreateSwapChainForCoreWindow` + `Present` each frame | HIGH |
| Corrupted / missing textures | Render | Unsupported/mismatched DXGI format or non-block-compressed size | Pre-convert to BC1–BC7 via `texconv`; verify format support | HIGH |
| Audio silence or stutter | XAudio2 | Callback starved / decode done inside `OnBufferEnd` | Move decode to worker thread; queue 2–4 buffers | HIGH |
| Crash immediately on launch | Package/manifest | Missing capability, bad manifest, or unsigned package | Fix `AppxManifest.xml`, re-sign, redeploy via Device Portal | HIGH |
| Out-of-memory after loading assets | Memory | Budgeted against physical RAM, hit App-mode ~1 GB cap | Query `AppMemoryUsageLimit`; ship Game-mode package for more | HIGH |
| `E_OUTOFMEMORY` on a single asset | Storage/mem | Asset/file exceeds limits | Keep files <2 GB; stream large data in chunks | HIGH |
| Input dead / no controller | Input | Never enumerated pads or ignored `GamepadAdded` | Use `Windows.Gaming.Input` events + per-frame polling | HIGH |
| "File not found" that works on PC | Storage | Wrote outside `LocalState` sandbox | Route all runtime paths through `ApplicationData.LocalState` | HIGH |
| Network calls silently fail | IO | Missing network capability in manifest | Add `internetClient`/`privateNetworkClientServer` capabilities | HIGH |
| Illegal-instruction crash, PC only faults on console | CPU | AVX/AVX2 used but not exposed in sandbox | Guard with CPUID; compile SSE2 fallback | LOW |
| Runs on PC, throttles/tears on console | Perf | App-mode 45% GPU / limited cores | Reduce internal res; ship Game-mode package for full GPU | MEDIUM |
| Wrong aspect / stretched output | Display | Assumed fixed res instead of querying DXGI modes | Enumerate DXGI output modes; render to safe internal res | MEDIUM |
| Console unusable, apps won't access files ("brick") | Dev Mode | Changed default directories / bad tweak | Reinstall Dev Mode app; don't relocate default dirs | MEDIUM |

---

## 11. ANTI-PATTERNS

- Do NOT attempt to jailbreak, ROP, or run unsigned native code — use Developer Mode; it is the sanctioned path.
- Do NOT reference or paste leaked GDKX/XDK APIs, paths, or build flags.
- Do NOT port cache flush/invalidate code — x86 is cache-coherent; delete it.
- Do NOT hand-manage framebuffers or poke video registers — use a DXGI swap chain.
- Do NOT assume OpenGL, GLES, or Vulkan exist — use D3D11/D3D12, or ANGLE for GL engines.
- Do NOT budget memory against 16/10 GB physical — budget against the runtime `AppMemoryUsageLimit`.
- Do NOT ship a DX12-only renderer in an App-mode package — App mode is DX11.
- Do NOT enable `/arch:AVX2` before confirming AVX is exposed in the sandbox.
- Do NOT store or read a single file larger than 2 GB through UWP APIs.
- Do NOT write outside the `LocalState` sandbox or relocate default directories.
- Do NOT decode audio inside the XAudio2 voice callback.
- Do NOT assume one controller of one type; handle 8 pads and hotplug.
- Do NOT trust PC FPS/memory headroom as representative of the console.
- Do NOT ship an unpackaged EXE — package as signed MSIX/APPX.
- You must NOT skip on-device validation for memory and performance.

---

## 12. PORTING DECISION TREE

1. **Decide App-mode vs Game-mode package FIRST.** *Why first:* it fixes your memory ceiling (~1 GB vs ~5–8 GB) and your graphics API (DX11 vs DX12). *Skip and:* you'll rewrite the renderer and blow the memory budget late.
2. **Get the engine building as x64 UWP with MSVC on a PC.** *Why:* the PC loop is your fastest iteration surface. *Skip and:* you debug blind on the console.
3. **Bridge the renderer.** Native D3D if feasible; else **ANGLE (GLES→D3D11)** for GL engines. *Why:* nothing else draws. *Skip and:* black screen.
4. **Route all file I/O through `LocalState`; split assets <2 GB.** *Why:* the sandbox rejects everything else. *Skip and:* "file not found" that only reproduces on console.
5. **Wire input via `Windows.Gaming.Input`.** *Why:* no menu navigation without it. *Skip and:* dead controller.
6. **Wire audio via XAudio2 with a worker-thread refill.** *Why:* correctness + no stutter. *Skip and:* silence or glitches.
7. **Package (MSIX), sign, deploy via Device Portal to the console.** *Why:* first real-hardware truth. *Skip and:* you never see sandbox behavior.
8. **Validate memory (`AppMemoryUsageLimit`) and perf on-device.** *Why:* the only place caps and throttling are real. *Skip and:* ships broken on console despite "working" on PC.
9. **Confirm AVX availability; add SSE2 fallback if absent.** *Why:* avoids illegal-instruction crashes present only on console. *Skip and:* random on-device crashes.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
XBOX SERIES X/S — HOMEBREW PRIMER (Dev Mode / UWP, legal path)
CPU  : 8c/16t Zen 2 x86-64 LE. X 3.8GHz / S 3.6GHz. AVX exists but sandbox-exposure UNVERIFIED [LOW].
GPU  : RDNA2. X 52CU~12TF / S 20CU~4TF. HLSL. NO GL/Vulkan.
MEM  : X 16GB / S 10GB GDDR6, flat paged x64. NO manual cache/bank/DMA.
CAPS : App mode ~1GB + DX11 + 2-4 cores + 45% GPU. Game mode ~5-8GB + DX12 + full GPU. Query AppMemoryUsageLimit.
GFX  : DXGI swap chain + Present. DX11 (App) / DX12 (Game). GL engines -> ANGLE (GLES->D3D11).
IN   : Windows.Gaming.Input (polled + hotplug), 8 pads, analog triggers, impulse-trigger rumble.
AUD  : XAudio2 voice callbacks, 48kHz float, worker-thread refill. No decode in callback.
FS   : Sandbox. Write to ApplicationData.LocalState. Files <2GB. Deploy via Xbox Device Portal.
NET  : StreamSocket/WinSock TCP+UDP. Declare manifest capabilities.
BUILD: VS2022 MSVC, x64 UWP, Windows SDK. Output signed MSIX/APPX. No GCC/triplet.
DEV  : Build+run on Windows PC first; remote-debug from VS over LAN; caps/perf only real on device.
RULES: No jailbreak. No leaked SDK. No cache code. No framebuffer pokes. Package or it won't run.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this file is used in an **active development session**, you MUST follow this state machine. Do NOT advance a state until the user explicitly instructs you to. Here "hardware" means **the actual Series console in Dev Mode** — PC runs are pre-validation only and never satisfy a gate.

### STATE DEFINITIONS
- **ANALYSIS** — Examine the codebase, identify UWP/sandbox/DX blockers, choose App-mode vs Game-mode, plan the change.
- **IMPLEMENTATION** — Write/modify code per this file. No hardware testing here.
- **BUILD_REQUEST** — Output exact build/package/deploy commands and ask the user to run on the console.
- **WAITING_FOR_HARDWARE** — STOP. Output ONLY the hardware test protocol. No code, no speculation, no debugging loop.
- **VALIDATION** — User reports back. Classify as SUCCESS, PARTIAL, or FAILURE.
- **NEXT_GOAL** — On SUCCESS, propose the next milestone and ask for confirmation.

### STATE TRANSITION RULES
1. **ANALYSIS → IMPLEMENTATION:** only after you have listed the specific files to modify to the user.
2. **IMPLEMENTATION → BUILD_REQUEST:** only after a complete, compilable change. You must include the exact `msbuild`/`cmake`/`MakeAppx`+`SignTool` command, the expected **.msix/.appx** output name/location, the **Device Portal** deploy step, and what the user should see/hear/feel.
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE:** you MUST output exactly:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name]
   BUILD: [Command]
   DEPLOY: [Xbox Device Portal / WDP REST]
   OBSERVE: [Specific expected behavior on the console]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including error codes, black screen, crashes]
   === STOP ===
   ```
   After this header you STOP. No fixes, no guesses.
4. **WAITING_FOR_HARDWARE → VALIDATION:** triggered ONLY by a user message containing "SUCCESS" or "FAILURE". On SUCCESS → NEXT_GOAL. On FAILURE → DEBUG_PROTOCOL. On anything ambiguous ("kind of works", "almost") → ask for the SUCCESS/FAILURE binary; do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL:** output a **DEBUG BUILD PROTOCOL** — a minimal isolating test case OR a checklist of 3 specific diagnostics (e.g., "Log `AppMemoryUsageLimit` at startup", "Confirm the swap chain `Present` HRESULT is S_OK", "Verify the file path resolves under `LocalState`"). Ask the user to run it and report back. Do NOT rewrite the whole implementation; do NOT guess-and-patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE:** after providing the protocol, return to WAITING_FOR_HARDWARE.
7. **VALIDATION (SUCCESS) → NEXT_GOAL:** propose the next milestone from the stack; do NOT implement until confirmed.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, STOP fixing and output:
`ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [VS remote debugger over LAN / Device Portal crash dump + perf trace / PC-vs-console comparison / Xbox dev community forums].`
Then ask whether to (a) mark the goal BLOCKED and skip, or (b) provide Device Portal logs / crash dumps / register traces for further analysis.

### GOAL STACK (default if user provides none)
1. Initialize video output (clear swap chain to a solid color).
2. Initialize controller input (read button presses via Windows.Gaming.Input).
3. Initialize audio output (play a sine wave via XAudio2).
4. Load assets from `LocalState` storage.
5. Render main menu framebuffer.
6. Main menu input loop.
7. Transition to game state.

Work ONLY on the active goal (top of stack). Do NOT implement future goals speculatively.
