---
name: xbone
description: Persistent expertise for developing homebrew on the Microsoft Xbox One family (One / One S / One X) via the sanctioned Developer Mode + UWP path — NOT jailbreak, NOT leaked XDK. Invoke with /xbone when porting an engine or writing homebrew targeting Xbox One in Dev Mode: covers Jaguar x86-64 / GCN hardware, the UWP app-vs-game resource split, DirectX 11 (App) / DirectX 12 (Game), ANGLE for OpenGL engines, XAudio2, Windows.Gaming.Input, MSVC/MSIX build, and Device Portal deploy. Includes failure-mode tables, anti-patterns, a porting decision tree, and a hardware-validation state machine.
---

# SKILL: Xbox One Homebrew (Developer Mode / UWP)

You are working on homebrew for the **Microsoft Xbox One family** (Xbox One 2013, Xbox One S 2016, Xbox One X 2017). You will target the **legal, Microsoft-sanctioned Developer Mode + Universal Windows Platform (UWP)** path only. You must NOT reference the leaked Durango XDK/GDK, its APIs, paths, or build flags. All code must build with the public Windows 10/11 SDK + MSVC toolchain.

Read this before writing any code. This platform is NOT a bare-metal retro console. It is a **locked, virtualized x86-64 PC running a Windows-derived OS**. Most classic console-porting concerns (endianness, manual cache flushing, raw memory maps, DMA alignment, co-processor DSP programming) DO NOT APPLY, and you must not invent them. Where a section would be N/A on this platform, you will say so explicitly.

---

## 1. HARDWARE ARCHITECTURE

You must treat the hardware as informational context only. **In Dev Mode/UWP you do NOT get raw hardware access** — the OS and hypervisor virtualize everything and hand your app a resource budget (see §5).

### CPU
- Xbox One / One S: AMD "Jaguar" 8-core, **x86-64**, ~1.6 GHz base / **1.75 GHz** for titles. [HIGH]
- Xbox One X: 8-core evolved Jaguar (Puma+) at **2.3 GHz**. [HIGH]
- **Endianness: little-endian.** There are NO byte-swap concerns porting from PC. [HIGH]
- Bit-width: 64-bit. All Dev Mode apps/games **must be built x64. x86 (32-bit) is NOT permitted.** [HIGH]
- SIMD: SSE/SSE2/SSE3/SSSE3/SSE4.x + AVX available (compiler `/arch:AVX` is safe; AVX2 [MEDIUM] — verify per SKU). FPU is standard x87/SSE hardware float; **do NOT use soft-float.** [HIGH]
- Cache: standard L1/L2, hardware-coherent. **You never manually flush or invalidate cache.** [HIGH]
- Quirk: your process is scheduled by the OS across a **limited core budget**, not all 8 cores (see §5). [HIGH]

### GPU
- Xbox One / One S: AMD GCN (GCN2-class), 12 CUs, ~768 shader ALUs, 853 MHz (One S ~914 MHz). [HIGH]
- Xbox One X: AMD Polaris-class GCN4, 40 CUs, 1172 MHz, ~6 TFLOPS. [HIGH]
- You access the GPU **only through DirectX**, never registers:
  - **App-mode UWP: DirectX 11, Hardware Feature Level 10.1.** [HIGH]
  - **Game-mode UWP: DirectX 12 (FL 11.0), and DirectX 11 (FL 10.1).** [HIGH]
- Programmable pipeline: HLSL shaders (SM 5.0 via D3D11/12). GLSL is NOT supported natively — see §3 (ANGLE). [HIGH]
- Max output: 1080p (One/S UWP), 4K on One X titles [MEDIUM — App mode is typically 1080p, the OS scales]. [MEDIUM]

### RAM
- Physical: 8 GB DDR3 + 32 MB ESRAM (One/S); 12 GB GDDR5 (One X). [HIGH]
- **You cannot see or address ESRAM or the physical map. It is fully abstracted.** Do NOT write ESRAM tiling code — it is inaccessible from UWP. [HIGH]
- What you actually get is a **foreground memory budget** (see §5): **1 GB (App) / 5 GB (Game).** [HIGH]
- Virtual memory: **YES** — full paging, standard `new`/`malloc`/`VirtualAlloc`. No manual bank management. [HIGH]
- Alignment: standard x86-64 (16-byte for SIMD). No console-specific alignment traps. [HIGH]

### Bus / DMA / Co-processors
- N/A for homebrew. You do not program DMA channels, audio DSPs, or I/O processors directly. XAudio2 (§6) and DirectX (§3) sit on top of the OS abstraction. [HIGH]

### Security / DRM
- Two-OS hypervisor design: Host OS + a virtualized Game/Exclusive partition. Retail is fully locked. [HIGH]
- **Developer Mode is the sanctioned homebrew route:** register a Microsoft Partner Center dev account, install the "Dev Mode Activation" app, switch the console into Dev Mode. Retail games/store are disabled while in Dev Mode. [HIGH]
- You get a **sandboxed UWP app**, not kernel/hypervisor access. There is no signature bypass and you must not seek one. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- **Official (do NOT use):** Microsoft XDK / Microsoft GDK for Xbox — gated behind ID@Xbox / concept approval and (for full-hardware devkits) NDA. You must NOT reference leaked copies, their headers, or `d3d11_x`/`d3d12_x` "monolithic" APIs. [HIGH]
- **Sanctioned homebrew stack (use this):**
  - Visual Studio 2019/2022 (Community is fine) + **Windows 10/11 SDK** + MSVC (v142/v143). [HIGH]
  - Target: **UWP application** (`Windows.Universal`), x64. [HIGH]
  - Deploy: **Xbox Device Portal** (browser) or VS remote deploy; file push via Device Portal or a Dev-Mode FTP app. [HIGH]
- **What the UWP path CAN do:** DirectX 11/12, XAudio2, Windows.Gaming.Input, WinSock/networking, sandboxed file I/O, C++/WinRT. [HIGH]
- **What it CANNOT do vs official:** full 8-core CPU, full GPU in App mode, >5 GB RAM, raw ESRAM, low-level GPU/audio DSP access, files >2 GB (Dev Mode limit), retail store distribution without cert. [HIGH]
- **Critical gap:** App mode caps you at DX11/FL10.1, ~45% GPU, 1 GB. For a demanding engine you want **Game mode** (DX12, full GPU, 5 GB, 4+2 cores). Decide this at project setup — it changes packaging.

---

## 3. GRAPHICS PIPELINE

- **Exact API:**
  - Native path: **Direct3D 11** (App mode) or **Direct3D 12** (Game mode). HLSL shaders. [HIGH]
  - **OpenGL / GLES engines (e.g. ioquake3): use ANGLE.** ANGLE translates GL ES 2.0/3.x → D3D11 and has a maintained UWP target. This is the standard bring-up path for GL renderers on Xbox UWP. [MEDIUM — build ANGLE for UWP/x64; confirm your engine's GL feature use maps to GLES]
  - Desktop OpenGL (WGL, GL 3.x compat profile) is **NOT available.** Do NOT link a desktop GL driver. [HIGH]
- **Video init:** create a `CoreWindow` (UWP app model) → DXGI swap chain via `CreateSwapChainForCoreWindow`. There is no manual video-mode/registers step. [HIGH]
- **Framebuffer flow:** render to a back buffer, `Present()` the DXGI swap chain. Double/triple buffering is a swap-chain buffer-count setting, not manual page flipping. [HIGH]
- **VSync/refresh:** `Present(1, 0)` for vsync. Region (NTSC/PAL) is irrelevant — the OS owns display timing; output is typically 60 Hz 1080p, HDMI-negotiated. [HIGH]
- **Textures:** standard DXGI formats (RGBA8, BC1–BC7 compression). Max 2D texture size 16384 at FL11, 8192 at FL10.1. **Prefer BCn compression** to fit the memory budget. [HIGH]
- **Depth/stencil/blend:** full D3D depth-stencil + programmable blend. [HIGH]
- **Anti-patterns:**
  - Do NOT call desktop OpenGL/WGL or Vulkan — neither is available in UWP. [HIGH]
  - Do NOT assume Game-mode DX12 features while packaged as an App — App mode is DX11/FL10.1 only. [HIGH]
  - Do NOT hardcode 720p/1080p — query the `CoreWindow`/swap-chain size; the OS may scale. [MEDIUM]

---

## 4. INPUT

- **Controllers:** Xbox One Wireless Controller (+ compatible pads), via UWP. [HIGH]
- **API: `Windows.Gaming.Input`** (`Gamepad` class) is the recommended model; legacy `XInput` also works. Do NOT use raw HID. [HIGH]
- **Model: polled.** Enumerate `Gamepad.Gamepads`; each frame call `GetCurrentReading()` for buttons, triggers (0–1 analog), and dual analog sticks (−1..1). [HIGH]
- **Hot-plug:** subscribe to `Gamepad.GamepadAdded` / `GamepadRemoved`. **You must handle multiple pads and disconnect/reconnect** — do not cache a single index. [HIGH]
- **Rumble:** `Gamepad.Vibration` (4 motors: left/right + left/right trigger impulse). [HIGH]
- **Accelerometer/gyro/pointer:** N/A for the standard pad. Kinect is not exposed to UWP homebrew — do not target it. [MEDIUM]
- **Anti-patterns:**
  - Do NOT assume controller 0 always exists — the list can be empty or reorder. [HIGH]
  - Do NOT block on input; poll once per frame and move on. [HIGH]

---

## 5. MEMORY LAYOUT / RESOURCE BUDGET

There is **no exposed physical memory map.** What matters is the **UWP foreground resource budget:**

| Resource | App submission | Game submission |
|---|---|---|
| Foreground RAM | **1 GB** | **5 GB** | [HIGH]
| Background RAM | 128 MB (games are suspended/terminated in background) | — | [HIGH]
| CPU | share of 2–4 cores | **4 exclusive + 2 shared cores** | [HIGH]
| GPU | ~45% share | **full GPU** | [HIGH]
| DirectX | DX11 FL10.1 | DX12 FL11.0 / DX11 FL10.1 | [HIGH]

- **Debugger exception:** memory caps are **NOT enforced when launched from the Visual Studio debugger** — a build that runs in-debugger can still OOM-crash when launched standalone. Always validate standalone. [HIGH]
- Allocation: normal `new`/`malloc`/`HeapAlloc`. No bank selection, no cached/uncached pointers. [HIGH]
- Stack: default OS thread stack (~1 MB); grow with the thread-creation stack-size arg or linker `/STACK`. [HIGH]
- **Cache flush/invalidate: N/A** — coherent hardware; you never call flush/invalidate. [HIGH]
- **DMA alignment: N/A** for homebrew. [HIGH]
- **Anti-patterns:**
  - Do NOT ship as an "App" if your engine needs >1 GB or full GPU — you must package as a **Game**. [HIGH]
  - Do NOT trust in-debugger memory headroom — it hides the real cap. [HIGH]
  - Do NOT try to allocate/tile ESRAM — it is invisible to UWP. [HIGH]

---

## 6. AUDIO

- **API: XAudio2 (2.9, ships with the OS).** This is the recommended homebrew audio path. WASAPI is also available. [HIGH]
- Formats: PCM (int16/float32) and ADPCM; you decode Vorbis/Opus/MP3 in software and feed PCM. [HIGH]
- Sample rate / channels: 48 kHz stereo (up to 7.1) standard. [HIGH]
- **Model: callback/streaming via source voices** — submit buffers to an `IXAudio2SourceVoice`; use the `IXAudio2VoiceCallback` (OnBufferEnd) to queue the next buffer. [HIGH]
- Audio RAM: N/A — shares main budget. [HIGH]
- Glitch avoidance: keep 2–4 queued buffers of ~10–20 ms; do the mixing/decoding on a worker thread, not in the callback. [HIGH]
- **Anti-patterns:**
  - Do NOT do file I/O, decode, or heavy math inside the XAudio2 callback. [HIGH]
  - Do NOT open the audio device on the UI thread and starve it. [MEDIUM]

---

## 7. STORAGE / IO

- **Sandboxed UWP file model.** Your writable root is `ApplicationData.Current.LocalFolder` (on disk under the package's `LocalState`). [HIGH]
- Recommended layout: keep assets/roms/config under `LocalState\` (e.g. `LocalState\baseq3\`). Push them with the **Xbox Device Portal** file manager or a Dev-Mode FTP app before first run. [HIGH]
- **Dev Mode hard limit: individual files must be ≤ 2 GB.** Split larger assets. [HIGH]
- Broader access: `broadFileSystemAccess` / known-folder capabilities exist but are restricted and prompt-gated; **prefer LocalState.** [MEDIUM]
- Path conventions: Windows-style `\`, case-insensitive. [HIGH]
- Loader/package requirements: `Package.appxmanifest` (identity, capabilities, entry point), a logo/splash asset set, and an x64 executable — packaged as **MSIX/APPX**. [HIGH]
- Network: full WinSock2 / UWP `Windows.Networking` (TCP/UDP, Wi-Fi/Ethernet). Add the `internetClient`(+`Server`/`privateNetworkClientServer`) capabilities in the manifest or sockets silently fail. [HIGH]
- **Anti-patterns:**
  - Do NOT assume a raw C:\ filesystem or hardcode absolute paths — use `LocalFolder`. [HIGH]
  - Do NOT forget network capabilities in the manifest. [HIGH]
  - Do NOT ship single asset files >2 GB in Dev Mode. [HIGH]

---

## 8. BUILD SYSTEM

- **Toolchain:** Visual Studio 2019/2022 + MSVC (v142/v143) + **Windows 10/11 SDK** (recent). [HIGH]
- **Target:** UWP app, **Configuration x64** only. (ARM/x86 are invalid for Xbox.) [HIGH]
- **Language:** C++17/20 with **C++/WinRT** for OS APIs. [HIGH]
- **Key compiler flags:** `/std:c++17` (or `/20`), `/EHsc`, `/O2` release, `/arch:AVX` [MEDIUM]. Do NOT use soft-float. [HIGH]
- **Linker/libs:** link `d3d11.lib`/`d3d12.lib`, `dxgi.lib`, `xaudio2.lib`, `windowsapp.lib` (WinRT). For GL engines, link your **ANGLE** UWP build (`libEGL`/`libGLESv2`). [MEDIUM]
- **Output:** an `.msix`/`.appx` (or `.msixbundle`) package. [HIGH]
- **Deploy:** Xbox Device Portal → "Add" app, or VS "Remote Machine" deploy pointed at the console's Dev Mode IP + pairing PIN. [HIGH]
- **Asset pipeline:** pre-convert textures to BCn/DDS, audio to 48 kHz PCM/Vorbis, and keep total footprint within the 5 GB (Game) budget. [HIGH]
- **Anti-patterns:**
  - Do NOT target x86 or leave the default Win32 project type — it must be a UWP x64 package. [HIGH]
  - Do NOT reference XDK/GDK monolithic `*_x` DirectX headers. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- **There is no separate Xbox One emulator for homebrew dev.** (xemu = original Xbox; Xenia = Xbox 360.) [HIGH]
- **Your "emulator" is a Windows 10/11 PC:** the same UWP project runs on desktop, so iterate renderer/audio/input logic on PC first. [HIGH]
- **What PC gets RIGHT:** DirectX behavior, HLSL, XAudio2, input logic, file/sandbox API shape. [MEDIUM]
- **What PC gets WRONG / hides:** the **memory budget** (PC has far more RAM), CPU core/GPU share limits, Jaguar's much lower per-core throughput, the 2 GB file limit, and standalone-vs-debugger memory enforcement. **These MUST be validated on the console.** [HIGH]
- **On-console debugging:** VS remote debugger over the network (breakpoints, watch); Device Portal shows running processes, crash dumps, and performance counters. [HIGH]
- **Crash dumps:** pull from Device Portal (crash dump collection) for post-mortem. [MEDIUM]
- **Anti-pattern:** Do NOT judge performance or memory from the PC build or the in-debugger run — only standalone-on-console numbers are real. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, app "runs" | DX init | Swap chain not created via `CreateSwapChainForCoreWindow`, or never `Present()` | Use CoreWindow swap-chain path; present each frame | HIGH |
| Corrupted / missing textures | Renderer | Uploaded desktop-GL/GLSL assets to a GLES/D3D path | Convert to BCn/DDS; run shaders through ANGLE/HLSL | MEDIUM |
| Silent or stuttering audio | XAudio2 | Heavy work in OnBufferEnd, or too few queued buffers | Decode on worker thread; queue 2–4 buffers | HIGH |
| Instant crash on standalone launch (fine in debugger) | Memory | Exceeds 1 GB App budget; debugger waives the cap | Repackage as Game (5 GB) or cut memory | HIGH |
| OOM after loading large level | Memory | Working set > foreground budget | Stream assets; compress textures; Game mode | HIGH |
| Controller does nothing | Input | Cached gamepad index; no Added/Removed handling | Enumerate `Gamepad.Gamepads` each frame; handle hot-plug | HIGH |
| "File not found" for pushed assets | Storage | Reading from a raw path, not `LocalFolder` | Use `ApplicationData.Current.LocalFolder`; push via Device Portal | HIGH |
| Asset fails to copy / load | Storage | Single file > 2 GB Dev Mode limit | Split the file | HIGH |
| Sockets never connect | Network | Missing manifest network capability | Add `internetClient`(+server) capabilities | HIGH |
| Poor FPS on One base, fine on PC/One X | CPU/GPU | Jaguar cores + App-mode GPU share | Move to Game mode; multithread across 4+2 cores; reduce draw calls | HIGH |
| Deploy rejected / won't install | Packaging | Built x86/ARM or non-UWP; bad manifest | Rebuild UWP x64; validate `Package.appxmanifest` | HIGH |
| Wrong aspect / letterbox | Display | Hardcoded resolution vs actual CoreWindow size | Query swap-chain size; render to it | MEDIUM |

---

## 11. ANTI-PATTERNS

1. Do NOT reference or link the leaked XDK/GDK, its headers, paths, or build flags.
2. Do NOT use desktop OpenGL, WGL, or Vulkan — route GL engines through ANGLE or port to Direct3D.
3. Do NOT build x86/32-bit or non-UWP — Xbox requires UWP **x64**.
4. Do NOT trust in-debugger memory headroom; the budget is enforced only standalone.
5. Do NOT ship as an "App" if you need >1 GB RAM or full GPU — package as a **Game**.
6. Do NOT assume DX12/FL11 features in App mode — App mode is DX11/FL10.1.
7. Do NOT do decode/IO/heavy math inside the XAudio2 callback.
8. Do NOT cache a single controller index or ignore hot-plug events.
9. Do NOT hardcode absolute filesystem paths — use `LocalFolder`.
10. Do NOT forget to declare network capabilities in the manifest.
11. Do NOT ship any single asset file larger than 2 GB in Dev Mode.
12. Do NOT write byte-swap or big-endian code — this platform is little-endian x86-64.
13. Do NOT write manual cache flush/invalidate or DMA code — the OS/hardware handle coherency.
14. Do NOT try to tile or address ESRAM — it is invisible to UWP.
15. Do NOT benchmark on the PC build — Jaguar cores and the GPU share make console numbers very different.

---

## 12. PORTING DECISION TREE

1. **Decide App vs Game submission FIRST.** Why: it sets your RAM (1 vs 5 GB), GPU (45% vs full), and DirectX tier. Skip it and you'll rebuild packaging late and hit OOM/perf walls. → For any real engine (ioquake3), choose **Game**.
2. **Stand up an empty UWP x64 project that clears the screen (DX11).** Why: proves CoreWindow + swap chain + Present + deploy pipeline before any engine code. Skip it and every later bug is ambiguous.
3. **Get the renderer translating: build ANGLE for UWP (GL engines) or stub the D3D backend.** Why: rendering is the highest-risk subsystem. Skip it and you can't see anything to validate.
4. **Wire input via Windows.Gaming.Input.** Why: needed to drive/validate everything interactively. Skip it and you can't leave a static frame.
5. **Wire audio via XAudio2 on a worker thread.** Why: audio callback timing bugs are easier to fix in isolation than mixed with gameplay. 
6. **Fix the file layer to LocalFolder + Device Portal push; respect the 2 GB file cap.** Why: the engine can't load assets until paths and pushing work. Skip it and you get "file not found" everywhere.
7. **Fit the memory budget (standalone, not in-debugger).** Why: the debugger hides the cap. Skip it and it "works" for you and crashes for users.
8. **Profile on real hardware and multithread across 4+2 cores.** Why: Jaguar single-core is weak; only console numbers are real. Skip it and PC-tuned code will stutter on base Xbox One.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
XBOX ONE HOMEBREW = DEV MODE + UWP (legal path; no jailbreak, no XDK)
CPU  : AMD Jaguar x86-64, 8c (1.75GHz One / 2.3GHz One X), little-endian, SSE/AVX
GPU  : AMD GCN (12CU One / 40CU One X) — access ONLY via DirectX
DX   : App=DX11 FL10.1  |  Game=DX12 FL11.0 (+DX11)   GLSL? use ANGLE (GLES->D3D11)
RAM  : foreground budget 1GB(App)/5GB(Game); NO ESRAM/no raw map; full virtual mem
CPU$ : App=2-4 shared cores/45% GPU ; Game=4 excl+2 shared cores/full GPU
BUILD: VS2022 + MSVC + Win SDK, UWP **x64 only**, C++/WinRT, output MSIX/APPX
INPUT: Windows.Gaming.Input (Gamepad, poll+hotplug)     AUDIO: XAudio2 (worker thread)
IO   : ApplicationData LocalFolder; push via Device Portal; **2GB/file cap**; net caps in manifest
DEPLOY: Xbox Device Portal (browser) or VS Remote Machine (Dev Mode IP + PIN)
TEST : iterate on Windows PC (same UWP); VALIDATE budget/perf STANDALONE on console
NO   : desktop GL/Vulkan, x86, soft-float, byte-swap, cache flush, DMA, ESRAM, XDK/GDK
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this SKILL is used in an **active development session**, you MUST follow this state machine. Do NOT skip the WAITING_FOR_HARDWARE gate.

### STATES
- **ANALYSIS** — Examine the codebase, identify UWP/Dev-Mode blockers, list files to modify.
- **IMPLEMENTATION** — Write/modify code from this SKILL + training data. No hardware testing here.
- **BUILD_REQUEST** — Output exact build/package/deploy commands and ask the user to run on real hardware.
- **WAITING_FOR_HARDWARE** — STOP. Output only the test protocol. No code, no speculative fixes, no debug loops.
- **VALIDATION** — User reports SUCCESS / FAILURE.
- **NEXT_GOAL** — On SUCCESS, propose the next milestone and ask for confirmation.

### TRANSITIONS
1. **ANALYSIS → IMPLEMENTATION:** only after you have listed the specific files to modify.
2. **IMPLEMENTATION → BUILD_REQUEST:** only after a complete, compilable change, including: exact `msbuild`/VS build command, output `.msix` name/location, deploy method (Device Portal or VS Remote), and what to observe.
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE:** you MUST output exactly:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name]
   BUILD: [Command]
   DEPLOY: [Device Portal / VS Remote Machine]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [what you saw]
     - FAILURE: [what you saw, incl. error codes / black screen / crash / OOM]
   === STOP ===
   ```
   Then STOP. Offer no fixes, no guesses.
4. **WAITING_FOR_HARDWARE → VALIDATION:** triggered only by a message containing "SUCCESS" or "FAILURE". Anything vague ("almost", "kind of") → ask for the SUCCESS/FAILURE binary; do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL:** output a minimal isolating test OR 3 specific diagnostics (e.g. "Confirm the crash happens standalone, not just in-debugger → memory budget?", "Log `MemoryManager.AppMemoryUsage` at load", "Verify swap chain came from `CreateSwapChainForCoreWindow`"). Ask the user to run it and report back. Do NOT rewrite the whole implementation; do NOT guess-and-patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE.**
7. **VALIDATION (SUCCESS) → NEXT_GOAL:** propose the next goal; do not implement until confirmed.

### ANTI-LOOP
If the same goal fails validation **more than 2 times**, STOP and output:
`ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [VS remote debugger / Device Portal crash dump / standalone-vs-debugger memory test / Emu Shack Discord].`
Then ask the user to (a) mark the goal BLOCKED and skip, or (b) provide a crash dump / MemoryManager logs for further analysis.

### DEFAULT GOAL STACK (if user provides none)
1. Initialize video output (clear swap chain to a solid color)
2. Initialize controller input (read button presses via Windows.Gaming.Input)
3. Initialize audio output (play a sine wave via XAudio2)
4. Load assets from LocalFolder
5. Render main menu framebuffer
6. Main menu input loop
7. Transition to game state

Work ONLY on the active goal (top of stack). Do NOT implement future goals speculatively.
