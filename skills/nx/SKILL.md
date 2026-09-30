---
name: nx
description: Nintendo Switch (NX) homebrew development expertise — libnx, devkitA64, deko3d, EGL/OpenGL via mesa, NRO packaging, Atmosphère, hbmenu, applet-vs-application memory modes, and hardware-validated porting workflows. Use this skill whenever the user mentions Nintendo Switch, NX, Tegra X1, libnx, deko3d, nxlink, Atmosphère, hbmenu, NRO/NSP, Joy-Con input, or porting/debugging any engine (ioQuake3, SDL2 games, emulators) to Switch. Also trigger on /nx. Enforces the HARDWARE TEST REQUIRED state machine for all active development sessions.
---

# SKILLS_NX.md — Nintendo Switch Homebrew Expertise Document

You are developing homebrew for the Nintendo Switch. You will follow this document as authoritative platform guidance. You must obey the confidence tags, the anti-patterns, and the GOAL-ORIENTED WORKFLOW in Section 14 during active development sessions.

---

## 1. HARDWARE ARCHITECTURE

### CPU
- SoC: NVIDIA Tegra X1 — **Erista (T210)** on launch units, **Mariko (T210B01 / T214)** on Switch Lite, revised HAC-001(-01), and OLED. [HIGH]
- CPU cluster: 4× ARM **Cortex-A57**, AArch64 (ARMv8-A), **little-endian**, 64-bit. A 4× Cortex-A53 companion cluster exists in silicon but is **disabled by Horizon OS** — you will never run on it. [HIGH]
- Clock: **1020 MHz** default for both handheld and docked. Applications may select 1785 MHz "boost mode" during loading; homebrew can overclock via sys-clk (Atmosphère sysmodule) up to 1785 MHz safely on both Erista and Mariko. [HIGH]
- Core availability: Horizon reserves **core 3 for the OS**. Application threads run on cores 0–2 by default; you can place a thread on core 3 as an application only via preemptive scheduling with `svcSetThreadCoreMask` and core mask permissions — treat cores 0–2 as your budget. [HIGH]
- Caches: 48 KB L1I + 32 KB L1D per core, 2 MB shared L2. Cache line size 64 bytes. [HIGH]
- FPU/SIMD: full **NEON (AdvSIMD)** and VFPv4, hard-float ABI. Doubles are native. You must NOT build soft-float. [HIGH]
- Quirks: unaligned access is legal for normal memory but faults on device memory; `dmb`/`dsb` barriers matter for GPU-shared buffers. [HIGH]

### GPU
- **NVIDIA Maxwell GM20B**, 256 CUDA cores (2 SMs), second-gen Maxwell — feature parity with desktop GTX 900 series. [HIGH]
- Clocks: 307.2 / 384 / 460.8 MHz handheld profiles, **768 MHz docked**. Overclockable to 921 MHz via sys-clk. [HIGH]
- Fully programmable pipeline: vertex/tessellation/geometry/fragment/compute shaders. No fixed-function anything. [HIGH]
- Output: 1280×720 internal panel (handheld), up to 1920×1080@60 docked over HDMI. The compositor scales arbitrary layer sizes. [HIGH]
- Framebuffer: linear or block-linear surfaces in **unified main RAM** — there is no dedicated VRAM. [HIGH]
- Texture capability: full desktop-class — BCn (DXT/BC1–BC7), ASTC (hardware-decoded on Maxwell? — ASTC is decoded by a dedicated unit and is slower than BCn; prefer BCn) [MEDIUM], max texture size 16384×16384, mipmaps, sRGB, cubemaps, 3D textures, arrays. [HIGH]

### RAM
- **4 GB LPDDR4** (Erista: 1331/1600 MHz profiles; Mariko: LPDDR4X), unified CPU+GPU. [HIGH]
- Memory pools: applications get roughly **3.2 GB**; the OS and applets take the rest. [MEDIUM]
- **CRITICAL:** homebrew launched in **applet mode** (album/hbmenu over applet) gets only ~**450 MB**. Homebrew launched as **application** (title takeover: holding R while launching a game, or a forwarder) gets the full application pool. This is the single most common Switch homebrew failure source. [HIGH]
- Full MMU with virtual memory and ASLR. You never deal with physical addresses. [HIGH]
- Alignment: GPU memory blocks require 4 KB (0x1000) alignment minimum; deko3d exposes `DK_MEMBLOCK_ALIGNMENT`. Code regions require 0x200000 alignment at the kernel level (handled by loader). [HIGH]

### Bus / DMA
- Unified memory: CPU and GPU contend for the same LPDDR4 bandwidth (~25.6 GB/s at max clock). Bandwidth, not ALU, is the usual bottleneck at 1080p. [HIGH]
- GPU DMA/copy engines are exposed through the nvdrv channel (deko3d transfer commands / glTexSubImage paths). You do not program DMA registers directly — everything goes through the `nvdrv` service. [HIGH]

### Co-processors
- **APE (Audio Processing Engine)**: audio mixing/effects run on a dedicated ADSP (Cortex-A9 + accelerators) driven by the `audren` (audio renderer) service. You submit mix commands; the ADSP does the work. [MEDIUM]
- **NVDEC/NVENC**: hardware video decode (H.264/VP8/VP9...) exists and is reachable from homebrew via nvdrv (used by homebrew media players); the ffmpeg portlib has a Tegra NVDEC path used by some ports. Treat as advanced/optional. [MEDIUM]
- **TSEC / Falcon** security processors: not accessible from userland homebrew; irrelevant to app development. [HIGH]

### Security / What homebrew can access
- Boot chain: BootROM → package1 → TrustZone secure monitor → package2 → Horizon kernel. Erista is exploitable via **fusée-gelée** (RCM, unpatchable in hardware); Mariko/OLED require a modchip. Runtime CFW: **Atmosphère**. [HIGH]
- Horizon is a microkernel: **no direct hardware register access** from userland. Everything is a service: `vi` (display), `nvdrv` (GPU), `hid` (input), `audout`/`audren` (audio), `fsp-srv` (filesystem), `bsd`/`nifm` (network). libnx wraps all of these. [HIGH]
- Homebrew NROs run inside a host process with the host's permissions. In application mode you have essentially full app-level access; you cannot touch other processes, kernel memory, or (safely) system NAND. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: **NintendoSDK** with the proprietary **NVN** graphics API. You must NOT use, reference, or assume it. All guidance below is community-toolchain only. [HIGH]
- Homebrew stack (all open source):
  - **devkitA64** (devkitPro GCC toolchain for AArch64) [HIGH]
  - **libnx** (ISC license) — kernel SVC wrappers, all system services, threading, filesystem, input, audio, applet lifecycle [HIGH]
  - **switch-tools** — `elf2nro`, `nacptool`, `build_pfs0`, etc. [HIGH]
  - **deko3d** (MIT) — low-level Vulkan-style GPU API targeting Maxwell natively; fastest option [HIGH]
  - **mesa/nouveau portlib** — real hardware-accelerated **OpenGL 4.3 / OpenGL ES 3.2 / EGL**; the easiest path for ports like ioQuake3 [HIGH]
  - **SDL2 portlib** — video (EGL-backed), audio, joystick/touch; the standard porting substrate [HIGH]
  - Hundreds of prebuilt portlibs via `dkp-pacman` (zlib, libpng, freetype, ffmpeg, curl+mbedtls, openal-soft, etc.) [HIGH]
- CAN do vs official: full GPU (GL or deko3d), full input incl. gyro/HD rumble, audio in/out, sockets, romfs, SD access, multithreading, JIT (via `svcMapProcessCodeMemory` / libnx `jitCreate` — works in application mode) [HIGH]
- CANNOT do / gaps:
  - No NVN — GL via nouveau is ~10–30% slower than NVN/deko3d in draw-call-heavy scenes [MEDIUM]
  - No official account/network services (NEX), no eShop, no game card access
  - No official multimedia middleware — use ffmpeg portlib
  - `jitCreate` fails or is restricted in applet mode on some firmware — JIT-dependent ports (Quake3e, dynarecs) should require application mode [MEDIUM]

---

## 3. GRAPHICS PIPELINE

### Choose ONE of three paths
1. **SDL2 (+ OpenGL/GLES through it)** — default for ports. SDL creates the EGL context, handles docked/handheld resize, maps input. Use unless you have a reason not to. [HIGH]
2. **Raw EGL + OpenGL 4.3 / GLES 3.2** (mesa/nouveau) — for engines with their own GL backend (ioQuake3's `opengl2` renderer runs as-is). Create an `NWindow` from `nwindowGetDefault()`, then standard `eglGetDisplay(EGL_DEFAULT_DISPLAY)` → `eglInitialize` → config → `eglCreateWindowSurface`. [HIGH]
3. **deko3d** — Vulkan-like explicit API, lowest overhead, best performance, most work. Shaders written in GLSL and compiled **offline** with `uam` (deko3d's shader compiler) into Maxwell binaries. [HIGH]
4. Pure-2D path: libnx `framebufferCreate` on the default NWindow gives you a CPU-writable RGBA8/RGB565 linear framebuffer with `framebufferBegin`/`framebufferEnd` — perfect for Goal 1 (solid color). [HIGH]

### Display / buffering
- Video out goes through the `vi` service compositor; libnx `NWindow` wraps a native window with **double or triple buffering** negotiated via the buffer queue (`nwindowSetSwapInterval`, EGL swap interval, or framebuffer API's internal queue). [HIGH]
- Refresh: **60 Hz always**. There is no NTSC/PAL split; PAL50 does not exist on this platform. [HIGH]
- VSync: `eglSwapInterval(dpy, 1)` or deko3d's `dkQueuePresentImage`; with the raw framebuffer API, `framebufferEnd` implicitly syncs to the queue. [HIGH]

### Docked/handheld handling — mandatory
- Register an applet hook (`appletHook`) and watch `AppletHookType_OnOperationMode`; query `appletGetOperationMode()`. On change, resize your GL viewport/swapchain: 1280×720 handheld, up to 1920×1080 docked. Rendering 720p docked works (compositor upscales) but looks soft. [HIGH]
- SDL2 delivers this as a window resize event. [HIGH]

### Textures / formats
- GL exposes BC1–BC7 (via `GL_EXT_texture_compression_s3tc` + BPTC), ETC2/EAC, ASTC. Prefer **BC1/BC3 for ports** (ioQuake3's DXT path works unchanged). [HIGH]
- Depth: D16/D24S8/D32F all available. Full stencil, full blending, MRT, FBOs — desktop-class. [HIGH]

### Anti-patterns (graphics)
- Do NOT ship without handling operation-mode change; docked users will get a stretched or letterboxed 720p layer. [HIGH]
- Do NOT compile shaders at runtime in deko3d — there is no runtime GLSL compiler; `uam` is offline only. (Mesa GL *does* compile GLSL at runtime, which is why GL is the porting path.) [HIGH]
- Do NOT assume dedicated VRAM; every texture upload eats the same bandwidth your CPU uses. [HIGH]

---

## 4. INPUT

### Controllers supported
Joy-Con pair (handheld/attached/detached), single sideways Joy-Con, Pro Controller (USB/BT), third-party USB HID pads, GameCube adapter, touchscreen, and per-controller sixaxis (accel+gyro). [HIGH]

### libnx model — polled
```c
padConfigureInput(1, HidNpadStyleSet_NpadStandard); // up to 8 players
PadState pad;
padInitializeDefault(&pad);        // No1 + handheld merged
// per frame:
padUpdate(&pad);
u64 down = padGetButtonsDown(&pad); // edge
u64 held = padGetButtons(&pad);     // level
HidAnalogStickState l = padGetStickPos(&pad, 0);
```
- Sticks are s32 in **[-32768, 32767]** (`JOYSTICK_MAX`); apply your own deadzone (~10%). [HIGH]
- Touch: `hidInitializeTouchScreen()` then `hidGetTouchScreenStates()` — up to 16 touch points, 1280×720 coordinate space regardless of dock state. [HIGH]
- Sixaxis: `padGetSixAxisSensorHandles` → `hidStartSixAxisSensor` → `hidGetSixAxisSensorStates` (accel in G, gyro in rad/s, plus fused rotation matrix). [HIGH]
- HD rumble: `hidInitializeVibrationDevices` + `hidSendVibrationValues` with `HidVibrationValue {amp_low, freq_low, amp_high, freq_high}`; neutral frequencies are 160/320 Hz. [HIGH]

### Disconnect/reconnect
- `padUpdate` transparently tracks style changes (Joy-Cons detached mid-game switch styleset). If you need the user to reconfigure, invoke the **controller applet**: `hidLaShowControllerSupportForSystem` / `hidLaShowControllerSupport`. [HIGH]

### Anti-patterns (input)
- Do NOT read `hid` shared memory directly; always go through `padUpdate`. [HIGH]
- Do NOT assume handheld mode: `padInitializeDefault` merges No1+Handheld precisely so both work — keep it. [HIGH]
- Do NOT hardcode Joy-Con button positions for sideways single Joy-Con play without remapping; SL/SR become your shoulders. [HIGH]

---

## 5. MEMORY LAYOUT

- Virtual memory only. No memory map to memorize; the kernel gives you: code regions, a **heap** (grown via `svcSetHeapSize`, wrapped by newlib `malloc`), thread stacks, and mappable "alias/stack" regions. libnx configures a sane default heap automatically (all available memory minus reserve). [HIGH]
- **Applet vs application pool** (repeat because it kills ports): ~450 MB applet vs ~3.2 GB application. Query with `svcGetInfo(InfoType_TotalMemorySize / UsedMemorySize)` at boot and print it. If a port needs >400 MB, detect applet mode (`appletGetAppletType() == AppletType_LibraryApplet`) and show an error telling the user to launch via title takeover. [HIGH]
- Stacks: main thread stack default is set by the homebrew environment (typically **1–8 MB**, hbloader default 8 MB) [MEDIUM — verify with `.nacp`/hbl config; test: recurse to measure]. New threads: you pass the stack explicitly to `threadCreate` — align to 4 KB. [HIGH]
- Caches: write-back. For CPU-written, GPU-read memory **outside** coherent GL/deko3d pools, flush with `armDCacheFlush(ptr, size)`; invalidate GPU-written CPU-read with `armDCacheClean`/invalidate variants. Mesa/SDL handle this internally — you only care in deko3d with non-coherent memblocks or when using `jitCreate` (use `jitTransitionToExecutable`, which handles I-cache). [HIGH]
- Instruction cache: after writing code (dynarec), you must transition via libnx JIT API — do NOT hand-roll `IC IVAU` loops unless you know the JIT type in use. [HIGH]
- GPU memblocks (deko3d): size and alignment multiple of 0x1000; images additionally need `DK_IMAGE_LINEAR_STRIDE_ALIGNMENT` (linear) or block-linear layout from `dkImageLayoutMaker`. [HIGH]

### Anti-patterns (memory)
- Do NOT assume `malloc` failure is impossible at 4 GB — in applet mode you have 450 MB and ioQuake3 with high `com_hunkMegs` will die. [HIGH]
- Do NOT `memcpy` into GPU-mapped write-combined memory byte-by-byte in hot loops; use bulk copies. [MEDIUM]

---

## 6. AUDIO

- Hardware path: `audout` (simple output) and `audren` (renderer with mixing/effects on the ADSP). [HIGH]
- **Recommended:** SDL2 audio (backed by audren) or **audout** directly for engines with their own mixer (ioQuake3 mixes internally → audout is perfect). [HIGH]
- audout fixed format: **PCM16, 48000 Hz, stereo**. Resample anything else yourself (or let SDL do it). [HIGH]
- Model: submit/reclaim buffer queue —
```c
audoutInitialize(); audoutStartAudioOut();
// loop: fill AudioOutBuffer (data 0x1000-aligned, size multiple of 0x1000? — buffer_size must be data_size rounded up; keep both 4KB-aligned to be safe [MEDIUM])
audoutAppendAudioOutBuffer(&buf);
audoutWaitPlayFinish(&released, &count, timeout); // or audoutGetReleasedAudioOutBuffer
```
- Latency: 2–4 buffers of 1024–2048 frames (≈21–43 ms each) is glitch-free; a single small buffer will underrun during loads. [HIGH]
- Feed audio from a **dedicated thread** pinned to a different core than your render loop (`threadCreate(..., core 1 or 2)`). [HIGH]
- No separate audio RAM — buffers live in main RAM. [HIGH]

### Anti-patterns (audio)
- Do NOT decode Vorbis/MP3 inside the buffer-feed loop's timing window; decode ahead into a ring. [HIGH]
- Do NOT call `audoutWaitPlayFinish` on the main thread with infinite timeout — it will stall rendering. [HIGH]

---

## 7. STORAGE / IO

- Media: **microSD** (`sdmc:/`, auto-mounted by libnx), embedded **RomFS** in the NRO (`romfs:/` after `romfsInit()`), save data (application mode only, via `fsdev` mounts), USB mass storage via `libusbhsfs` portlib (Atmosphère required). [HIGH]
- Filesystem: SD should be **FAT32**. exFAT works but the stock exFAT driver has a history of corrupting cards — warn users, prefer FAT32. [HIGH]
- Paths: case-insensitive on FAT32, case-preserving; **RomFS is case-sensitive**. Forward slashes. Standard C stdio (`fopen("sdmc:/switch/mygame/config.cfg","rb")`) works via devoptab. [HIGH]
- Layout expected by **hbmenu**:
```
sdmc:/switch/<appname>/<appname>.nro   (icon+NACP embedded in NRO)
sdmc:/switch/<appname>/...data...
```
hbmenu scans `/switch` recursively; NRO needs embedded 256×256 **JPEG** icon and NACP (name/author/version) via `nacptool`+`elf2nro` — the devkitPro Makefile does this from `APP_TITLE/APP_AUTHOR/APP_VERSION/ICON`. [HIGH]
- `argv[0]` is set by hbloader to the NRO's own path — use it to locate your data directory instead of hardcoding. [HIGH]
- Network: full BSD sockets — `socketInitializeDefault()`, then standard `socket/connect/bind`. Wi-Fi (and USB-C Ethernet adapters). curl/mbedtls portlibs for HTTPS. [HIGH]
- **nxlink**: `nxlink -s app.nro` uploads over Wi-Fi to hbmenu's netloader AND redirects stdout back to your PC — this is your primary dev loop. Enable with `socketInitializeDefault(); nxlinkStdio();` in debug builds. [HIGH]

### Anti-patterns (storage)
- Do NOT write to NAND/system partitions. Ever. [HIGH]
- Do NOT assume `romfs:/` exists unless you embedded a romfs and called `romfsInit`. [HIGH]
- Do NOT do thousands of tiny `fopen`s at load; FAT32 over SDIO has high per-op latency — pack files (pk3s are already ideal). [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **devkitA64** from devkitPro. Install via `dkp-pacman -S switch-dev`; portlibs e.g. `dkp-pacman -S switch-sdl2 switch-mesa switch-glm switch-libdrm_nouveau switch-zlib switch-libpng`. [HIGH]
- Environment (mandatory):
```
export DEVKITPRO=/opt/devkitpro
export DEVKITA64=$DEVKITPRO/devkitA64
PATH=$DEVKITPRO/tools/bin:$DEVKITA64/bin:$PATH
```
- Prefix / triplet: `aarch64-none-elf-` [HIGH]
- Compiler flags (from switch-examples, do not improvise):
```
ARCH  = -march=armv8-a+crc+crypto -mtune=cortex-a57 -mtp=soft -fPIE
CFLAGS = $(ARCH) -g -Wall -O2 -ffunction-sections -D__SWITCH__
LDFLAGS = -specs=$(DEVKITPRO)/libnx/switch.specs $(ARCH) -Wl,-Map,$(notdir $*.map)
LIBS = -lnx        # GL adds: -lEGL -lglapi -ldrm_nouveau -lm
```
`-mtp=soft` and `-fPIE` are NOT optional — NROs are position-independent and libnx owns TPIDR. [HIGH]
- Output flow: `ELF → elf2nro (embeds NACP via nacptool + icon.jpg) → .nro`. Application-mode forwarders/NSPs are a packaging concern, not a build concern — develop with NRO + title takeover. [HIGH]
- Use the template Makefile from `$DEVKITPRO/examples/switch/templates/application` — it already wires nacptool/elf2nro, romfs (`ROMFS := romfs` dir), and portlibs pkg-config. Prefer it over hand-rolled CMake; if the engine is CMake-based, use `$DEVKITPRO/cmake/Switch.cmake` toolchain file (`-DCMAKE_TOOLCHAIN_FILE=$DEVKITPRO/cmake/Switch.cmake`). [HIGH]
- Asset pipeline:
  - Textures for deko3d: precompress to BCn (e.g. `tex-convert`/AMD Compressonator offline); GL ports can keep engine-native formats. [MEDIUM]
  - Shaders for deko3d: compile offline with **uam** to `.dksh`. [HIGH]
  - Audio: pre-resample to 48 kHz PCM16 where feasible. [HIGH]
  - Small static assets → RomFS dir (baked into NRO); large game data → `sdmc:/switch/<app>/`. [HIGH]

### Anti-patterns (build)
- Do NOT build with `-mcpu=native`, soft-float, or without `-fPIE`; the NRO will crash at load. [HIGH]
- Do NOT link desktop mesa headers/libs; only the `switch-mesa` portlib. [HIGH]
- Do NOT strip the NACP/icon step; hbmenu will show a blank entry or skip the NRO. [MEDIUM]

---

## 9. EMULATOR VS HARDWARE

- Landscape (verify currency — this scene moved fast): **yuzu** was discontinued March 2024 and **Ryujinx** ceased development October 2024 after Nintendo action; community forks of Ryujinx (e.g. Ryubing/"Ryujinx reborn" lineage) continue and remain the best homebrew-testing option. [MEDIUM — check current fork status before recommending a download]
- Ryujinx-lineage gets RIGHT: libnx service calls, GL/nouveau rendering, hbmenu/NRO loading with applet-vs-application memory emulation, sockets, most of hid. Safe for logic bring-up. [MEDIUM]
- Emulators get WRONG / hide:
  - Real GPU timing and memory bandwidth (emulator FPS is meaningless) [HIGH]
  - Cache/coherency bugs (emulated memory is always coherent) [HIGH]
  - Exact audout buffer timing; SD card latency; thermal/clock throttling [HIGH]
  - Sixaxis/HD-rumble fidelity, dock transitions [MEDIUM]
- Hardware debugging:
  - **nxlink stdout** over Wi-Fi — first line of defense. [HIGH]
  - **Atmosphère crash reports**: on crash, `sdmc:/atmosphere/crash_reports/*.log` contains PC, LR, all registers, and a stack trace you can symbolicate against your `.elf`/`.map` with `aarch64-none-elf-addr2line`. Always request this file on FAILURE. [HIGH]
  - **Atmosphère GDB stub**: enable in `system_settings.ini` (`enable_standalone_gdbstub`), then `aarch64-none-elf-gdb app.elf` + `target extended-remote <switch-ip>:22225`. Full breakpoints/watchpoints on hardware. [HIGH]
- Anti-pattern: Do NOT optimize based on emulator FPS, and do NOT ship after emulator-only testing — coherency and memory-pool behavior differ. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, instant return to hbmenu | App start | Uncaught crash before first frame; check crash report | Read `sdmc:/atmosphere/crash_reports/`, addr2line the PC against your .elf | [HIGH] |
| Crash ~0x??? in malloc / abort after loading big assets | Applet mode | ~450 MB applet pool exhausted | Launch via title takeover (hold R on a game) or forwarder; detect `AppletType_LibraryApplet` and warn | [HIGH] |
| "2168-0002 (0x4a8)" / data abort in crash report | Any | Null/wild pointer deref (classic segfault) | Symbolicate PC+LR; check recently added platform code first | [HIGH] |
| NRO missing from hbmenu | Deployment | Wrong path, or NACP/icon malformed | Place at `sdmc:/switch/<name>/<name>.nro`; rebuild with template Makefile (nacptool step) | [HIGH] |
| Textures fine, then corruption after minutes | deko3d / custom memory | Missing `armDCacheFlush` on CPU-written GPU buffer, or fence misuse | Flush after CPU writes; check dkFence waits | [HIGH] |
| Audio crackles during level load | audout | Buffer underrun — feed loop starved by main thread I/O | Move audio feed to own thread on another core; ≥3 buffers of ≥1024 frames | [HIGH] |
| Silence, audoutAppendAudioOutBuffer returns error | audout | Buffer/data size not 0x1000-aligned, or audout not started | Align buffer to 4 KB, call audoutStartAudioOut | [MEDIUM] |
| Input dead after Joy-Con detach | hid | App configured only one style set / no pad update | Use padConfigureInput + padInitializeDefault; call padUpdate every frame | [HIGH] |
| sdmc fopen fails, works in emulator | SD | Case mismatch vs RomFS expectations, or exFAT card issues | Match case exactly; reformat card FAT32 | [HIGH] |
| 10–20 FPS docked, fine handheld | GL | Rendering native 1080p over bandwidth budget, or GL driver overhead | Render 720p and let compositor scale, or reduce draw calls; try sys-clk GPU 768→921 | [MEDIUM] |
| Stretched/soft image when docking mid-game | vi/EGL | Operation-mode change unhandled | appletHook OnOperationMode → recreate/resize surface | [HIGH] |
| jitCreate fails (rc != 0) | Dynarec/JIT ports | Applet mode restriction or old hbloader | Require application mode; update Atmosphère + hbloader | [MEDIUM] |
| App hangs on exit, hbmenu never returns | Lifecycle | Threads not joined / services not exited | `appletMainLoop()` respected? join threads, `romfsExit`, `socketExit` before return | [HIGH] |

---

## 11. ANTI-PATTERNS

1. Do NOT assume 4 GB of RAM — detect applet mode and its ~450 MB pool before allocating.
2. Do NOT bypass libnx services to poke hardware registers; Horizon is a microkernel and there is nothing to poke.
3. Do NOT build without `-fPIE -mtp=soft -march=armv8-a+crc+crypto`; the resulting NRO is not loadable/correct.
4. Do NOT compile deko3d shaders at runtime — there is no on-device GLSL compiler for deko3d; use `uam` offline.
5. Do NOT hardcode 1280×720 or ignore `AppletHookType_OnOperationMode`.
6. Do NOT block the main thread on audio waits or run the audio feed inside the render loop.
7. Do NOT use exFAT-formatted SD cards for anything you care about.
8. Do NOT write outside `sdmc:/switch/` and your own save mount; never touch NAND.
9. Do NOT spawn threads without explicit stacks and core masks — plan cores 0–2, leave 3 alone.
10. Do NOT self-modify code without libnx `jitCreate`/`jitTransitionToExecutable`; W^X is enforced.
11. Do NOT poll input by reading HID shared memory or stale libnx <4.0 `hidScanInput` APIs; use the PadState API.
12. Do NOT trust emulator performance or emulator-only "it works".
13. Do NOT skip `socketInitializeDefault()` before any network call, or call it twice.
14. Do NOT ship an NRO without embedded NACP + 256×256 JPEG icon.
15. Do NOT assume the working directory is your app folder — derive paths from `argv[0]`.

---

## 12. PORTING DECISION TREE

1. **Toolchain sanity: build and run a template NRO.**
   Why first: validates devkitA64, switch-tools, SD layout, and your deploy loop (nxlink).
   Skip → every later failure is ambiguous between toolchain and code.
2. **Decide graphics path: SDL2 → raw EGL/GL → deko3d (in that order of preference).**
   Why: it dictates the entire backend effort. ioQuake3-class engines: raw EGL + existing GL renderer.
   Skip → you'll write a deko3d backend nobody asked for, or fight SDL where the engine wants raw GL.
3. **Memory audit: measure peak allocation on PC; compare to 450 MB applet pool.**
   Why: determines whether you must mandate title takeover from day one.
   Skip → mysterious OOM crashes misattributed to code bugs.
4. **Filesystem: route all paths through `sdmc:/switch/<app>/` + romfs; wire `argv[0]`.**
   Why: asset loading precedes everything visible.
   Skip → black screen with no clue (engine silently fails to find data).
5. **Main loop integration: `appletMainLoop()`, operation-mode hook, exit handling.**
   Why: lifecycle correctness is cheap now, painful retrofitted.
   Skip → hangs on exit, dock breakage.
6. **Input mapping via PadState (+ touch as mouse if the engine wants one).**
   Why: needed to progress past menus; trivial with libnx.
   Skip → untestable game states.
7. **Audio: feed engine mixer output to audout on a dedicated thread.**
   Why: last of the core subsystems; isolable.
   Skip → silence or crackle, but game runs — fine to defer.
8. **Performance pass on real hardware: profile handheld clocks first (worst case), then docked.**
   Why: handheld GPU at 307–460 MHz is the binding constraint.
   Skip → a port that only works docked.
9. **JIT/dynarec enablement if applicable (`jitCreate`, application mode required).**
   Why last: pure optimization on Switch since the interpreter path already runs on a 1 GHz 4-core A57.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Nintendo Switch (NX) — Tegra X1 (Erista/Mariko)
CPU: 4x Cortex-A57 @ 1020MHz (core 3 = OS), AArch64 LE, NEON, hard-float
GPU: Maxwell GM20B 256cc @ 307-460MHz handheld / 768MHz docked, fully programmable
RAM: 4GB LPDDR4 unified — APPLICATION ~3.2GB vs APPLET ~450MB (title takeover matters!)
OS: Horizon microkernel; all HW via services (vi/nvdrv/hid/audout/fsp-srv); CFW = Atmosphère
TOOLCHAIN: devkitA64, prefix aarch64-none-elf-, DEVKITPRO=/opt/devkitpro
FLAGS: -march=armv8-a+crc+crypto -mtune=cortex-a57 -mtp=soft -fPIE -D__SWITCH__
LINK: -specs=$DEVKITPRO/libnx/switch.specs -lnx (+ -lEGL -lglapi -ldrm_nouveau for GL)
OUTPUT: ELF -> elf2nro -> app.nro -> sdmc:/switch/<app>/<app>.nro (NACP + 256x256 JPEG icon)
GRAPHICS: SDL2 or EGL+OpenGL4.3/GLES3.2 (mesa/nouveau) or deko3d (offline uam shaders)
DISPLAY: 720p handheld / up to 1080p docked, 60Hz, handle appletHook OnOperationMode
INPUT: padConfigureInput+padInitializeDefault, padUpdate per frame; sticks ±32767; touch 1280x720
AUDIO: audout = PCM16 48kHz stereo, 4KB-aligned buffers, feed from own thread (core 1/2)
FS: sdmc:/ (FAT32! not exFAT), romfs:/ (romfsInit), stdio works; argv[0] = own NRO path
NET: socketInitializeDefault(); BSD sockets; DEV LOOP: nxlink -s app.nro (+ nxlinkStdio)
DEBUG: sdmc:/atmosphere/crash_reports/*.log + addr2line; Atmosphère GDB stub port 22225
EMU: Ryujinx forks (yuzu/Ryujinx upstream dead 2024) — logic only, never perf
JIT: libnx jitCreate — application mode; W^X enforced otherwise
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this SKILL.md is used in an active development session (not just reference), you must follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify platform-specific blockers, and plan the implementation.
- **STATE: IMPLEMENTATION** — Write/modify code based on this SKILL.md and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output the exact build commands and ask the user to compile and run on real hardware.
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back results. Classify the result as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES

1. **ANALYSIS → IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change. You must include:
   - Exact `make` / `cmake` build command
   - Expected output file name and location (e.g., `app.nro`)
   - How to transfer to the target hardware (SD copy to `sdmc:/switch/<app>/`, or `nxlink -s app.nro`)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color framebuffer"]
   BUILD: [Command]
   DEPLOY: [nxlink -s app.nro | SD copy path]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including crash report contents from sdmc:/atmosphere/crash_reports/ if present]
   === STOP ===
   ```
   After this header, you STOP generating. You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE → VALIDATION**: Triggered ONLY by user message containing "SUCCESS" or "FAILURE".
   - If user says "SUCCESS": Move to NEXT_GOAL.
   - If user says "FAILURE": Move to DEBUG_PROTOCOL.
   - If user says anything else (e.g., "it kind of works", "almost"): Ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: You must output a **DEBUG BUILD PROTOCOL**:
   - A minimal C test case that isolates the failure (e.g., framebufferCreate solid-fill NRO, audout sine NRO),
   - OR a checklist of 3 specific diagnostic steps (e.g., "Print svcGetInfo TotalMemorySize at boot — is it ~450MB?", "Attach nxlink stdout and confirm the fopen path", "Pull the newest crash_reports log and paste the PC/LR lines").
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the debug protocol, you return to WAITING_FOR_HARDWARE state.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: You propose the next milestone from the goal stack below. You do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [Atmosphère GDB stub on port 22225 / nxlink stdout trace / crash_reports symbolication / emulator-vs-hardware differential / GBAtemp-Switchbrew community].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide crash reports / GDB output / emulator logs for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this default:
1. Initialize video output (solid color via libnx framebuffer API)
2. Initialize controller input (print button presses over nxlink stdout)
3. Initialize audio output (48kHz sine wave via audout)
4. Load assets from sdmc:/ and romfs:/
5. Render main menu framebuffer (or first GL frame)
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
