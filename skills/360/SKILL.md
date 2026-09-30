---
name: "360"
description: Xbox 360 (Xenon CPU / Xenos GPU) homebrew development expertise using the open-source libxenon/Free60 toolchain and XeLL loader. Use this skill whenever the user mentions Xbox 360, X360, Xenon, Xenos, libxenon, XeLL, Free60, RGH/JTAG homebrew, eDRAM, or wants to port, build, debug, or optimize any code targeting the Xbox 360 — including engine ports (ioQuake3, UT99, emulators), graphics backends, audio, input, storage, networking, or toolchain/build issues. When used in an active development session, obey the hardware validation state machine in Section 14.
---

# SKILLS_360.md — Xbox 360 (Xenon) Homebrew Expertise Document

You are developing homebrew for the Microsoft Xbox 360 using the **open-source libxenon SDK (Free60 project)** and the **XeLL** loader. You must NOT use, reference, or assume access to the leaked Microsoft XDK. All code must compile with the community toolchain. Follow every rule below; append the stated confidence tag when repeating a claim to the user.

---

## 1. HARDWARE ARCHITECTURE

### CPU — "Xenon" (Microsoft XCPU, IBM PowerPC)
- 3 identical PowerPC cores at **3.2 GHz**, each 2-way SMT → **6 hardware threads**. [HIGH]
- The core is derived from the same PPE lineage as the Cell Broadband Engine's PPE (PS3). Compiler tuning for `cell` is the accepted community practice. [HIGH]
- **64-bit PowerPC ISA, BIG-ENDIAN.** Homebrew is built as 32-bit code with 64-bit instructions enabled (`-m32 -mpowerpc64`). You must byte-swap all little-endian assets and network data. [HIGH]
- **Strictly in-order, dual-issue.** There is no out-of-order machinery to hide latency. Load-Hit-Store (writing memory then reading it back quickly) stalls ~40–50 cycles and is the #1 CPU performance killer on this platform. [HIGH]
- Caches: 32 KB L1I + 32 KB L1D per core; **1 MB shared L2**; **128-byte cache lines**. [HIGH]
- FPU: scalar double-precision FPU with fused multiply-add per core. [HIGH]
- SIMD: **VMX128** — AltiVec extended to 128 vector registers per thread plus custom ops (dot product, D3D pack/unpack). GCC only emits standard AltiVec (32 registers); VMX128-specific instructions require inline assembly. [MEDIUM]
- Quirks: expensive integer divide; avoid microcoded instructions (`lswx`, `stswx`, misaligned VMX loads); branch mispredicts are costly on the in-order pipeline. [HIGH]

### GPU — "Xenos" (ATI/AMD C1, R500-derived)
- 500 MHz, **48 unified shader ALUs** (VLIW; unified vertex/pixel — the first unified-shader GPU shipped). [HIGH]
- **10 MB eDRAM daughter die** containing the ROPs, with 256 GB/s internal bandwidth. **Color and depth render targets live in eDRAM, not main RAM.** Finished frames must be *resolved* (copied) from eDRAM to a main-RAM texture for scanout or sampling. [HIGH]
- 1280×720 with no MSAA fits in eDRAM in one pass (~7 MB color+depth). 720p with 4×MSAA or 1080p requires **predicated tiling** (render the frame in tiles) — avoid in homebrew; libxenon does not automate tiling. [HIGH]
- Video output up to 1080p through the ANA/HANA scaler chip; render at 720p and let the scaler upscale. [HIGH]
- Texture formats: ARGB8888, RGB565, DXT1/3/5 and more; textures are stored **tiled/swizzled**, not linear — use the Xe library's texture creation path, which handles layout. [MEDIUM]
- Fully programmable shaders (no fixed function). Shader microcode is a custom Xenos ISA. [HIGH]

### RAM
- **512 MB GDDR3, unified memory architecture**, 700 MHz (1.4 GT/s effective), 128-bit bus, **22.4 GB/s**. CPU and GPU share this pool; Xenos is also the northbridge/memory controller. [HIGH]
- CPU↔GPU front-side bus: ~21.6 GB/s aggregate (10.8 GB/s each way). [MEDIUM]
- No virtual memory paging in homebrew context; libxenon runs with flat mappings and full privileges after XeLL. [MEDIUM]

### Co-processors / support silicon
- Hardware **XMA audio decoder** exists but is NOT exposed by libxenon — treat audio as CPU-mixed PCM only. [MEDIUM]
- **SMC** (System Management Controller): power, fans, RTC, IR receiver, front LEDs. libxenon exposes basic SMC messaging (e.g., power off, LED control). [MEDIUM]
- Southbridge handles USB 2.0, SATA (HDD/DVD), 100 Mbit Ethernet, internal Wi-Fi (slim). [HIGH]

### Security / DRM
- Boot chain: eFuses + 1BL→CB→CD signed bootloaders; at runtime a **hypervisor** enforces executable signing and W^X. Retail consoles run only signed XEX. [HIGH]
- Homebrew entry points:
  - **JTAG/SMC hack** — kernels ≤ 2.0.7371 only. [HIGH]
  - **RGH 1 / 1.2 / 2 / 3 / EXT+CLK** — hardware glitch of the CPU reset line to defeat the CB hash check; works on all Phat/Slim revisions with a soldered glitch chip (or RGH3 with wires only). [HIGH]
  - **BadUpdate** (released 2025) — software-only, **non-persistent** hypervisor exploit via a game-save vector; must be re-run each boot and has a per-attempt success rate. Verify current state with a web search before advising on it. [MEDIUM]
- After a successful hack, **XeLL** (Xenon Linux Loader) runs unsigned ELFs with full hardware access; the retail hypervisor is out of the picture for libxenon apps. [HIGH]
- You cannot access: Xbox Live, XMA decoder documentation, retail kernel services. Do NOT design around them.

---

## 2. OFFICIAL VS HOMEBREW SDK

- **Official:** Microsoft XDK (Direct3D9-derived API, XAudio, XAM services). It is leaked, proprietary, and you must NOT use it, reference its headers, or assume the user has it.
- **Homebrew:** **libxenon** (Free60 Project, GitHub `Free60Project/libxenon`), GPL-licensed, with a custom GCC/binutils/newlib toolchain (`xenon-gcc`) and the **XeLL** loader. [HIGH]

### libxenon CAN:
- Initialize video, text console, and the **Xe** GPU command-buffer API (vertex/pixel shaders, textures, render states, resolve). [MEDIUM]
- Read wired USB Xbox 360 controllers, USB mass storage (FAT32), internal HDD (XTAF/FATX), DVD drive. [MEDIUM]
- Network via **lwIP** (TCP/UDP, DHCP), including TFTP re-launch of ELFs for fast iteration. [MEDIUM]
- Output 48 kHz stereo PCM audio. [MEDIUM]
- Run code on all 6 hardware threads via its threading API. [MEDIUM]

### libxenon CANNOT:
- Compile HLSL/GLSL shaders at runtime — **there is no open-source runtime shader compiler for Xenos.** You must design around a small, fixed set of precompiled shader microcode blobs (libxenon ships working examples). This is the single biggest constraint on porting modern renderers. [MEDIUM]
- Use the XMA hardware decoder, Kinect, wireless controllers (internal RF module has no open driver — verify, sparse docs). [LOW — test: call the input API with only a wireless pad synced and see if data arrives]
- Provide OpenGL/D3D. Any GL-based engine needs a custom Xe backend (the libxenon SDL port shows the pattern). [MEDIUM]

---

## 3. GRAPHICS PIPELINE

- **API:** `xenos/xenos.h` for mode setting + text console; `xenos/xe.h` (the "Xe" library) for real rendering. There is no GL, no D3D. [MEDIUM]
- **Init:** call `xenos_init(VIDEO_MODE_AUTO)` early; `console_init()` gives you a printf framebuffer console — your best friend for bring-up. [MEDIUM]
- **Frame flow (Xe):**
  1. `Xe_Init()` once; create shaders from precompiled microcode blobs; create vertex buffers and textures via Xe allocators (they return GPU-visible memory and handle tiled texture layout).
  2. Per frame: set render target → set shaders/state → draw primitives → **`Xe_Resolve()`** to copy the eDRAM target into the main-RAM framebuffer → `Xe_Sync()` to wait for the GPU. [MEDIUM]
- **Nothing appears on screen until you resolve.** Rendering happens in eDRAM; scanout reads main RAM. Forgetting the resolve = black screen with a "working" render loop. [HIGH]
- Double buffering is your responsibility (two resolve targets, flip). [MEDIUM]
- **VSync/refresh:** 59.94 Hz (NTSC-timing modes) and 50 Hz PAL modes exist; HDMI/VGA/component are all routed through ANA/HANA. Use `VIDEO_MODE_AUTO` and read back the actual mode instead of assuming resolution. [MEDIUM]
- **Shaders:** Xenos microcode. Community practice: reuse/adapt the precompiled vertex+pixel shader binaries shipped with libxenon samples (basic transform, textured, colored). Budget for **very few shader permutations**. If a port needs many shaders (e.g., Q3e GLSL paths), pick the fixed-function-style minimal path instead. [MEDIUM]
- **Textures:** create through Xe; write pixel data to the texture's CPU pointer, then **flush the CPU data cache** before the GPU samples it. Max practical size 4096×4096; prefer DXT for bandwidth. [MEDIUM]
- Depth buffer and standard blending are available in eDRAM; stencil available. [MEDIUM]
- **Anti-patterns:**
  - Do NOT attempt predicated tiling; stay ≤ 720p no-MSAA so the whole frame fits in 10 MB eDRAM.
  - Do NOT assume you can generate shaders at runtime.
  - Do NOT write to a texture the GPU is currently sampling; fence with `Xe_Sync()`.

---

## 4. INPUT

- Supported by libxenon: **wired USB Xbox 360 controllers** (and most XInput-class wired pads). [MEDIUM]
- Wireless controllers via the internal RF board: assume **unsupported** in libxenon. [LOW — test: sync a wireless pad, poll all ports, check for data]
- Model: **polled.** Call `usb_do_poll()` (or the equivalent periodic USB service call) in your main loop, then `get_controller_data(&ctrl, port)` filling `struct controller_data_s` with digital buttons (`.a .b .x .y .start .back .rb .lb`, D-pad), analog sticks (`.s1_x .s1_y .s2_x .s2_y`, signed 16-bit), and triggers (`.lt .rt`, 0–255). Verify exact field names against the installed libxenon `input/input.h` before writing code. [MEDIUM]
- No accelerometer/gyro/pointer hardware in standard pads.
- Rumble: the controller supports it over USB; libxenon exposure is inconsistent across forks. [LOW — test: grep the toolchain's `input.h` for a rumble/set function and call it]
- Multiple ports: iterate ports 0–3 every frame; controllers can appear/disappear on USB hot-plug. Do NOT bind logic to port 0 only.
- Anti-pattern: Do NOT block waiting for input during video init; poll non-blocking.

---

## 5. MEMORY LAYOUT

- Physical RAM: 512 MB starting at physical 0. libxenon maps it with a **cached view around 0x80000000** and an **uncached view around 0xA0000000** (classic PowerPC convention); GPU-visible allocations come from the Xe allocators, which return correctly aligned physical-contiguous memory. Verify exact bases in `$DEVKITXENON/usr/include` headers before hardcoding. [MEDIUM]
- Heap: standard newlib `malloc` over the remaining RAM after the ELF image. You realistically have ~450+ MB free — memory pressure is rarely your problem on this console. [MEDIUM]
- **Cache architecture:** write-back L1/L2, **128-byte lines**, and the GPU is **NOT coherent** with CPU caches. [HIGH]
  - Before the GPU reads CPU-written data (vertices, textures, command data): **flush** the range (`memdcbst`/`dcbst`-based helper in libxenon).
  - After the GPU writes data the CPU will read (resolved frames): **invalidate** the range.
  - Symptoms of missed flushes: stale/partial geometry, 128-byte-granular texture corruption. [HIGH]
- DMA/GPU alignment: align GPU-visible buffers to at least 128 bytes (cache line); use Xe allocators instead of `malloc` for anything the GPU touches. [MEDIUM]
- Stack: fixed size set by the CRT/linker script (`app.lds`); recursive or stack-hungry code (Q3 VM, script interpreters) should be checked against it — increase in the linker script if needed. [LOW — test: fill stack guard pattern, check high-water mark]
- Anti-pattern: Do NOT pass `malloc`ed pointers straight to the GPU without flushing, and do NOT assume x86-style cache coherency ever.

---

## 6. AUDIO

- No open access to the XMA hardware decoder; audio is **CPU-mixed PCM pushed to the audio hardware** via libxenon's `xenon_sound`. [MEDIUM]
- Format: **48 kHz, 16-bit, stereo, interleaved, BIG-ENDIAN samples.** Byte-swap any little-endian source audio (WAV, ioq3 mixer output) before submitting. [MEDIUM]
- Model: **push/poll, no callback.** Loop: check buffered amount with `xenon_sound_get_free()` / `xenon_sound_get_unplayed()`, submit the next chunk with `xenon_sound_submit(buf, len)` when there is room. [MEDIUM]
- Glitch avoidance: keep ~2–4 frames of audio queued (e.g., 2048–4096 samples); submit from the main loop every frame or from a dedicated thread on another hardware thread. Underruns = crackle/silence; oversized queues = latency. [MEDIUM]
- No separate audio RAM; buffers live in main RAM. [MEDIUM]
- Anti-patterns: Do NOT assume little-endian samples; do NOT decode compressed audio (Vorbis/MP3) on the render thread — pin it to another hardware thread.

---

## 7. STORAGE / IO

- Accessible media: **USB mass storage (FAT32)**, **internal SATA HDD (XTAF/FATX)**, **DVD drive**, **Ethernet**. [MEDIUM]
- Mounts (libxenon device names — verify against your toolchain revision): `uda:/` (USB), `sda:/` (HDD partitions), `dvd:/`. FAT paths are case-insensitive; write your code case-correct anyway for portability. [MEDIUM]
- Init: filesystem drivers register during libxenon startup; call the USB poll periodically or storage may not enumerate before first access. Retry mounting for ~2 s at boot before declaring "no device". [MEDIUM]
- **XeLL app convention:** XeLL searches attached FAT USB storage, DVD, and network for **`xenon.elf`** (also accepts `xenon.z`/vmlinux for Linux) and boots it. Deployment = copy your ELF to the USB root as `xenon.elf`. [MEDIUM]
- **TFTP netboot:** XeLL fetches an ELF over the network — this is the fast iteration path (rebuild → `tftp put` / serve file → power cycle). Use it. [MEDIUM]
- Network stack: **lwIP** — `network_init()`, DHCP, BSD-ish sockets subset (TCP/UDP). 100 Mbit Ethernet on all models; Wi-Fi only external/slim-internal and not reliably supported by libxenon — use Ethernet. [MEDIUM]
- Anti-patterns:
  - **Do NOT write to NAND flash.** It holds the bootloaders/kernel; corruption = brick (recoverable only with a hardware programmer).
  - Do NOT assume the HDD exists; many RGH consoles run diskless with USB only.

---

## 8. BUILD SYSTEM

- Toolchain: **libxenon toolchain** built from the Free60 scripts (`Free60Project/libxenon` → `toolchain/build-xenon-toolchain`), producing `xenon-gcc`, `xenon-g++`, `xenon-objcopy`, plus newlib and libxenon itself. [MEDIUM]
- Environment: `export DEVKITXENON=/usr/local/xenon` and add `$DEVKITXENON/bin` to `PATH` (default install prefix used by the build script). [MEDIUM]
- Cross prefix: `xenon-` (custom PowerPC64 ELF target). [MEDIUM]
- Representative compiler flags (mirror `$DEVKITXENON/rules` — always prefer what the installed rules file says over this list): [MEDIUM]
  ```
  -m32 -mpowerpc64 -mcpu=cell -mtune=cell -maltivec -fno-pic -mno-fused-madd? (verify) 
  -D__XENON__ -I$(DEVKITXENON)/usr/include
  ```
  The `-m32 -mpowerpc64` pair (32-bit ABI, 64-bit instructions) is the platform signature. Big-endian is the default for this target — never force `-mlittle-endian`.
- Linker: `-T $(DEVKITXENON)/app.lds -L$(DEVKITXENON)/usr/lib -lxenon -lm` (add `-lfat`, `-lzlib` etc. per toolchain packaging). [MEDIUM]
- Output: ELF. Some XeLL builds want a converted 32-bit ELF: `xenon-objcopy -O elf32-powerpc app.elf app.elf32`. Ship both and try `.elf32` first if plain ELF fails to boot. [MEDIUM]
- Minimal Makefile skeleton:
  ```make
  include $(DEVKITXENON)/rules
  TARGET  := myapp
  OBJS    := main.o
  all: $(TARGET).elf32
  $(TARGET).elf32: $(TARGET).elf
  	xenon-objcopy -O elf32-powerpc $< $@
  $(TARGET).elf: $(OBJS)
  	$(CC) $(CFLAGS) $(OBJS) $(LDFLAGS) -o $@
  ```
- Asset pipeline: byte-swap little-endian binary assets at build time where possible (cheaper than runtime swaps); pre-convert textures to DXT; pre-resample audio to 48 kHz s16be.
- Anti-patterns: Do NOT use a generic `powerpc-linux-gnu` toolchain (wrong CRT, no libxenon); do NOT enable `-mstrict-align`-violating packed struct tricks — misaligned VMX/float access traps or micro-codes.

---

## 9. EMULATOR VS HARDWARE

- **Xenia** is the primary Xbox 360 emulator, but it emulates **retail XEX titles with an HLE kernel — it does NOT run libxenon ELFs.** For libxenon homebrew there is effectively **no usable emulator**: this is a hardware-first platform. [MEDIUM]
- What you may safely reason from Xenia: general Xenos behavior (eDRAM/resolve semantics, texture tiling) as documented in its source — Xenia's GPU docs are among the best public Xenos references. [MEDIUM]
- What only hardware tells you: cache-coherency bugs, real GPU timing, USB enumeration order, audio pacing, XeLL loading quirks.
- **Hardware debugging options:**
  - `printf` → on-screen console (after `console_init`) and **UART serial** (solder header on the motherboard, 115200 8N1) — XeLL and libxenon both log there. This is your primary trace channel. [MEDIUM]
  - TFTP relaunch for <30 s edit-run cycles. [MEDIUM]
  - GDB stub support in libxenon exists in some forks but is unreliable; do not plan around it. [LOW]
- Crash data: libxenon's exception handler prints a register/stack dump to screen and UART on unhandled exceptions — always photograph/copy it; the faulting address + LR localize most bugs. [MEDIUM]
- Anti-pattern: Do NOT "verify" a rendering or timing fix by code inspection alone — every graphics/audio milestone goes through the Section 14 hardware gate.

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, app clearly running (UART prints) | Xe rendering | Missing `Xe_Resolve()` — frame stays in eDRAM | Resolve to main-RAM target each frame before flip | [HIGH] |
| Black screen, no UART output | Boot | ELF not loaded (wrong name/format) or crash in CRT | Name it `xenon.elf` at USB root; try the `elf32-powerpc` objcopy variant | [MEDIUM] |
| Geometry flickers / vertices stale or garbage | CPU-written vertex buffers | CPU cache not flushed before GPU read | Flush dcache over the buffer range after writing, before draw | [HIGH] |
| Texture corruption in 128-byte-wide bands | Texture upload | Partial cache flush or linear data in tiled texture | Flush full range; upload only via Xe texture path | [MEDIUM] |
| Colors swapped / image looks "negative-ish" | Asset loading | Little-endian pixel data on big-endian GPU format | Byte-swap per-texel at load or pre-swap assets at build | [HIGH] |
| Audio = harsh static, correct rhythm | Sound submit | Little-endian PCM submitted (samples need big-endian) | Byte-swap 16-bit samples before `xenon_sound_submit` | [MEDIUM] |
| Audio crackles every few seconds | Sound pacing | Buffer underrun (submits tied to uneven frame time) | Keep 2–4 frames queued; submit from a dedicated thread | [MEDIUM] |
| Crash after loading a large file | File I/O + heap | Reading into undersized/unaligned buffer, or stack buffer | Heap-allocate, check `fread` returns; raise stack in `app.lds` if recursive parser | [MEDIUM] |
| Controller silent | Input | USB not being polled / wireless pad (unsupported) | Call the USB poll every frame; use a wired pad; scan ports 0–3 | [MEDIUM] |
| "No device" / files not found at boot | Storage | FS accessed before USB enumeration completes | Retry mount/poll loop for ~2 s before first access | [MEDIUM] |
| Exception dump with LR inside libc memcpy | Alignment | Misaligned vector/float access from packed structs | Remove packing; copy through byte-wise memcpy into aligned struct | [MEDIUM] |
| Runs at half expected speed | Performance | Load-hit-store storms or float↔int conversions in hot loop | Restructure to avoid write-then-read; keep data in registers; profile via UART timestamps | [HIGH] |
| Wrong aspect / cropped image | Video mode | Hardcoded 640×480 or ignoring actual mode from `VIDEO_MODE_AUTO` | Query the mode struct after init; scale UI from real width/height | [MEDIUM] |

---

## 11. ANTI-PATTERNS

1. Do NOT assume little-endian anywhere — Xenon is big-endian; swap file, network, and audio data explicitly.
2. Do NOT hand `malloc` memory to the GPU without a data-cache flush; the GPU is not cache-coherent.
3. Do NOT expect a runtime shader compiler — design for a tiny fixed set of precompiled Xenos shader blobs.
4. Do NOT skip `Xe_Resolve()`; eDRAM contents are invisible until resolved to main RAM.
5. Do NOT target 1080p or 4×MSAA at 720p — that requires predicated tiling, which homebrew must avoid.
6. Do NOT write-then-immediately-read the same memory in hot loops; load-hit-store stalls cripple the in-order PPE.
7. Do NOT write to NAND flash under any circumstances.
8. Do NOT use OpenGL/Direct3D idioms or headers — neither API exists; use the Xe library.
9. Do NOT block on input or storage during startup; poll with timeouts.
10. Do NOT assume wireless controllers or Wi-Fi work; require wired pads and Ethernet.
11. Do NOT rely on Xenia to validate libxenon homebrew — it cannot run it; use real hardware.
12. Do NOT tune performance for a single thread — you have 6 hardware threads; move audio/IO off the main thread.
13. Do NOT hardcode video resolution; query the mode after `xenos_init(VIDEO_MODE_AUTO)`.
14. Do NOT use packed structs holding floats/vectors — alignment traps and microcoded accesses follow.
15. Do NOT reference leaked XDK APIs, headers, paths, or build flags in any output.

---

## 12. PORTING DECISION TREE

1. **Confirm toolchain + "hello UART/console" boots via XeLL (TFTP path set up).** Priority: everything downstream depends on a working edit-build-boot loop; skipping it means every later failure is ambiguous (code bug vs. deploy bug).
2. **Endianness audit of the codebase.** Big-endian breaks silently: file format loaders, network code, audio samples, hash functions. Skip it and you'll chase "corrupt data" ghosts for weeks. ioquake3 is endian-clean [HIGH]; most PC-era code is not.
3. **Stub the platform layer (video=console text, input=one wired pad, FS=uda:/).** Gets the engine's main loop alive and printing before any GPU work. Skipping means debugging engine logic and GPU bring-up simultaneously.
4. **Filesystem + asset loading from USB.** Large-file reads exercise heap, alignment, and endian code early. Skipping delays the discovery of loader bugs until they masquerade as renderer bugs.
5. **Renderer: minimal Xe backend — clear, one textured triangle, resolve, flip.** This validates shaders/eDRAM/flush discipline in isolation. Skip it and your first "full scene" frame is undebuggable.
6. **Renderer: port the engine's fixed-function-style path onto ≤4 shader permutations.** For ioquake3: vertex-color + single-texture + lightmap-multiply covers most of the world. Skipping the permutation budget analysis leads to an unshippable shader explosion.
7. **Audio: 48 kHz s16be push loop on a dedicated thread.** Comes after video because silent games are testable; stuttering ones poison perf measurements.
8. **Input mapping + hot-plug across ports 0–3.**
9. **Performance pass: LHS elimination in hot loops, move decode work to spare hardware threads, DXT textures.** Last, because optimizing before correctness on an in-order CPU wastes effort on code that will change.
10. **Robustness: exception-dump review, storage-retry, mode-query for PAL/NTSC displays.**

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Xbox 360 — CPU "Xenon": 3× PPC @3.2GHz, 6 threads, IN-ORDER, BIG-ENDIAN,
  1MB L2, 128B cache lines, VMX128 (GCC = plain AltiVec). LHS stalls = perf killer.
GPU "Xenos": 500MHz, 48 unified shaders, 10MB eDRAM (render targets live there;
  MUST Xe_Resolve() to main RAM every frame). 720p no-MSAA max practical. No GL/D3D.
RAM: 512MB GDDR3 unified, 22.4GB/s. GPU NOT cache-coherent: flush dcache before GPU
  reads, invalidate after GPU writes. GPU buffers via Xe allocators, 128B aligned.
SDK: libxenon (Free60) + XeLL loader. xenon-gcc, DEVKITXENON=/usr/local/xenon.
  Flags: -m32 -mpowerpc64 -mcpu=cell -maltivec (check $DEVKITXENON/rules).
  Output: ELF (+ objcopy -O elf32-powerpc fallback). Deploy: xenon.elf on FAT32 USB
  root or TFTP netboot (fast path). NO runtime shader compiler — precompiled blobs only.
INPUT: wired USB 360 pads, polled (usb poll + get_controller_data, ports 0-3).
  Wireless pads/Wi-Fi: assume unsupported.
AUDIO: xenon_sound push loop, 48kHz s16 stereo BIG-ENDIAN, keep 2-4 frames queued.
STORAGE: uda:/ (USB FAT32), sda:/ (HDD XTAF), dvd:/. NEVER write NAND.
DEBUG: on-screen console + UART 115200; exception handler dumps regs. No emulator
  runs libxenon ELFs (Xenia = retail XEX only) → hardware validation mandatory.
ENTRY: JTAG (≤7371), RGH1/1.2/2/3, BadUpdate (SW-only, non-persistent) → XeLL.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this document is used in an active development session (not just reference), you must follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify platform-specific blockers (endianness, cache coherency, shader budget), and plan the implementation.
- **STATE: IMPLEMENTATION** — Write/modify code based on this document and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output the exact build commands and ask the user to compile and run on real hardware.
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back results. Classify the result as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES

1. **ANALYSIS → IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change. You must include:
   - Exact `make` command (toolchain: xenon-gcc via `$DEVKITXENON/rules`)
   - Expected output file name and location (e.g., `myapp.elf32`)
   - How to transfer to the console (copy as `xenon.elf` to FAT32 USB root, or TFTP netboot via XeLL)
   - What the user should observe on screen/UART/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color framebuffer via Xe resolve"]
   BUILD: [Command]
   DEPLOY: [USB xenon.elf / TFTP]
   OBSERVE: [Specific expected behavior on screen/UART/audio]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including UART output, exception dumps, black screen, RRoD-free hangs]
   === STOP ===
   ```
   After this header, you STOP generating. You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE → VALIDATION**: Triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - "SUCCESS" → NEXT_GOAL.
   - "FAILURE" → DEBUG_PROTOCOL.
   - Anything else ("it kind of works", "almost") → ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: You must output a **DEBUG BUILD PROTOCOL**:
   - A minimal C test case that isolates the failure (e.g., resolve-only clear screen; dcache-flush A/B test; UART-only boot probe), OR
   - A checklist of 3 specific diagnostic steps (e.g., "Confirm UART prints reach main()", "Verify Xe_Resolve target address is the scanout framebuffer", "Check `get_controller_data` return on ports 0–3").
   - Ask the user to run the diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the debug protocol, return to WAITING_FOR_HARDWARE.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: Propose the next milestone from the goal stack. Do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [UART trace capture / minimal libxenon sample comparison / Xenia GPU-docs cross-check / Free60 community sources].`
- Ask the user whether to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide UART logs / exception dumps for further analysis.

### GOAL STACK (User-Defined or Default)
If the user does not provide a GOAL_STACK, use this default:
1. Boot `xenon.elf` via XeLL; confirm console/UART "hello" output
2. Initialize video output (solid color via Xe clear + resolve)
3. Initialize controller input (print button presses to console)
4. Initialize audio output (play big-endian sine wave, no crackle for 10 s)
5. Load assets from USB (`uda:/`)
6. Render main menu framebuffer (textured quad + text)
7. Main menu input loop
8. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must NOT implement future goals speculatively.
