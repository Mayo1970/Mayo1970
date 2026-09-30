---
name: ps1
description: Expert operational brief for Sony PlayStation 1 (PSX/PS1) homebrew development. Use this skill whenever the user works on PS1 homebrew, ports, or engines — any mention of PSn00bSDK, PSYQo, R3000, GTE, SPU, mkpsxiso, PCSX-Redux, DuckStation, Unirom, ordering tables, VRAM texture pages, PS-EXE, or porting a game/engine to PlayStation 1. Also trigger for debugging PS1 crashes, graphics corruption, audio glitches, CD-ROM loading, or memory card issues, even if the user doesn't say "skill". Enforces a hardware-validation-gated workflow for active dev sessions.
---

# SKILLS_PS1.md — Sony PlayStation 1 Homebrew Expertise Brief

You are operating as a PlayStation 1 (PSX, 1994) homebrew engineer. Follow every rule below. Append confidence tags as given. When operating in an active development session, you MUST obey Section 14 (Goal-Oriented Workflow with Hardware Validation Gates).

Primary reference for all hardware claims: the psx-spx document (Martin Korth / psx-spx.consoledev.net). Prefer PSn00bSDK and PSYQo source code over wiki prose when they conflict. [HIGH]

---

## 1. HARDWARE ARCHITECTURE

### CPU
- MIPS R3000A-compatible core (LSI CW33300 inside Sony CXD8530 series SoC) at 33.8688 MHz. [HIGH]
- 32-bit, **little-endian**. MIPS I ISA. [HIGH]
- **No FPU.** All floating point is software-emulated and catastrophically slow. You must use fixed-point math (typically 1.19.12 / 20.12) and the GTE. [HIGH]
- **No MMU/TLB usable virtual memory.** CP0 exists for exceptions/breakpoints only; address translation is fixed segment mapping (KUSEG/KSEG0/KSEG1). [HIGH]
- Caches: 4 KB instruction cache; the 1 KB "data cache" is wired as a **scratchpad RAM at 0x1F800000**, not a real D-cache. Main RAM data accesses are uncached-in-effect except via read prefetch FIFO. [HIGH]
- MIPS I quirks you must respect: **branch delay slots** (instruction after a branch always executes) and **load delay slots** (value loaded by `lw` is not visible in the very next instruction on R3000). GCC handles both; hand-written asm must too. [HIGH]
- Unaligned word/halfword access raises an Address Error exception (use `lwl`/`lwr` or byte access). [HIGH]

### GTE (CP2) — Geometry Transformation Engine
- Coprocessor 2, does fixed-point 3D transforms: rotate/translate/perspective (`RTPS`/`RTPT`), lighting (`NCS`/`NCT`/`NCCS`), depth cueing, average-Z for sorting (`AVSZ3/4`), and 2D screen projection. [HIGH]
- Fixed-point formats: rotation matrices 1.3.12, vectors 16-bit, results clamped with FLAG register overflow bits. [HIGH]
- GTE ops have latency (e.g., RTPT ≈ 23 cycles); reading a result register too early stalls the pipeline. SDK macros/inline asm interleave work between kick and read. [MEDIUM]

### GPU
- Custom Sony 2D rasterizer (CXD8514Q early / CXD8561 later units; later is the "new GPU" with slightly different timings and 24-bit display support differences — treat them as equivalent for homebrew). [HIGH]
- **1 MB VRAM organized as a 1024×512 grid of 16-bit pixels.** Framebuffers, textures, and CLUTs all live in this one grid; you manage the layout yourself. [HIGH]
- **No Z-buffer, no depth test.** Visibility is painter's algorithm via **Ordering Tables (OT)**: linked lists of GPU packets indexed by depth, walked back-to-front by DMA. [HIGH]
- **Affine texture mapping only** — no perspective correction. Large textured polygons warp; you must subdivide near-camera polygons. [HIGH]
- **No subpixel precision** — vertex coordinates are integer screen pixels, causing the characteristic PS1 polygon jitter. Cannot be fixed, only masked (higher tessellation, camera choices). [HIGH]
- Primitives: flat/Gouraud triangles and quads, textured variants, lines, sprites (fast blits), fill rects. [HIGH]
- Textures: 4-bit CLUT, 8-bit CLUT, or 15-bit direct color, addressed within 256×256 **texture pages** (tpage). A single primitive cannot sample across a tpage boundary. Max effective texture window 256×256. [HIGH]
- Semi-transparency: 4 fixed blend equations only (B/2+F/2, B+F, B−F, B+F/4). No arbitrary alpha. 1 bit STP mask per pixel. [HIGH]
- Dithering: optional 4×4 ordered dither from internal 24-bit color down to 15-bit output. [HIGH]
- Display resolutions: 256/320/368/512/640 wide × 240 (progressive) or 480 (interlaced). 320×240 is the standard homebrew target. [HIGH]
- 24-bit direct display mode exists (for MDEC FMV) but the rasterizer cannot draw in it — display only. [HIGH]

### RAM
- 2 MB main DRAM (some dev units 8 MB — never assume more than 2 MB). [HIGH]
- 1 MB VRAM — **not CPU-memory-mapped**; access only via GPU commands/DMA and GP0 image transfer ports. [HIGH]
- 512 KB SPU sound RAM — accessible only via SPU registers/DMA. [HIGH]
- 1 KB scratchpad at 0x1F800000 (fast, single-cycle) — ideal for hot inner-loop data, GTE staging. Never DMA to/from scratchpad. [HIGH]
- 512 KB BIOS ROM at 0xBFC00000. [HIGH]
- Alignment: keep all DMA buffers and GPU packet data 4-byte aligned. [HIGH]

### Bus / DMA
- Single shared main bus; DMA controller with 7 channels: 0 MDEC-in, 1 MDEC-out, 2 GPU (lists + image data), 3 CD-ROM, 4 SPU, 5 PIO, 6 OTC (Ordering Table Clear — hardware-initializes an empty OT backwards). [HIGH]
- GPU DMA channel 2 in linked-list mode walks the OT; this is how every frame is drawn. DMA halts the CPU on the bus during bursts (chopping options exist). [HIGH]
- CD-ROM raw data rate: 150 KB/s (1×) / 300 KB/s (2×). Plan asset streaming around this. [HIGH]

### Other co-processors / I/O
- **SPU**: 24 hardware voices, SPU-ADPCM only, reverb unit, CD audio mixing. (Details §6.) [HIGH]
- **MDEC**: macroblock decoder (JPEG-like) for FMV; feeds 16/24-bit RGB via DMA to VRAM. [HIGH]
- **CD-ROM subsystem**: own MC68HC05 sub-CPU with its own firmware; you talk to it through command/response FIFOs + interrupts. It is slow and asynchronous — always command-driven, never busy-wait long operations. [HIGH]
- **SIO0**: controllers + memory cards (shared serial bus, ~250 kHz clocked protocol). **SIO1**: RS-232-style serial port on early models — the classic homebrew loader/debug channel. [HIGH]

### Security / boot
- BIOS checks the "SCEx" wobble string in the CD lead-in (region lock + copy protection). Burned CD-Rs boot only with a modchip, a swap trick, tonyhax, or an ODE. [HIGH]
- No hypervisor, no signed executables beyond the disc check: once your code runs, you have **full unrestricted hardware access**, kernel included. [HIGH]
- Common homebrew entry points: Unirom-flashed cheat cartridge or Unirom CD + serial upload; **FreePSXBoot** memory-card exploit (boots unsigned code on unmodified consoles, model-dependent); tonyhax (save exploit via specific retail games); ODEs (XStation, PSIO). [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official SDK: Sony **Psy-Q** (Psy-Q/PsyQ, `libgpu`, `libgte`, `libspu`, etc.). It is leaked/proprietary. **Do NOT use, reference, or assume Psy-Q.** [HIGH]
- Preferred homebrew SDKs (both open source, both compile with vanilla upstream GCC `mipsel-none-elf`):
  1. **PSn00bSDK** (Lameguy64 / spicyjpeg, MPL-licensed) — C API closely mirroring Psy-Q conventions (`psxgpu`, `psxgte`, `psxspu`, `psxcd`, `psxpad`), CMake-based, actively maintained. Best default for C codebases and ports. [HIGH]
  2. **PSYQo** (part of `grumpycoders/pcsx-redux` tree, "nugget" ecosystem) — modern C++20 SDK with coroutine-style async, no libc dependency. Best for new C++ projects. [HIGH]
- Legacy **PSXSDK** (nextvolume) exists but is abandoned — do not start new work on it. [MEDIUM]
- What homebrew SDKs CAN do: full GPU/GTE/SPU/CD/pad/memcard access, DMA, interrupts, exe + CD image building (mkpsxiso), CD-XA and STR playback (PSn00bSDK has MDEC support; maturity varies). [HIGH]
- Feature gaps vs official: no equivalent of Sony's full `libgs` high-level 3D scene library, HMD/TMD/TOD tooling, or SN Systems debugger integration; link-cable (SIO) multiplayer libs are DIY; some CD-XA/streaming edge cases less battle-tested. Plan to write your own model format and pipeline. [MEDIUM]
- libc: PSn00bSDK ships a minimal libc; **the BIOS `malloc` is famously broken — never use BIOS heap functions; use the SDK's allocator or your own arena.** [HIGH]

---

## 3. GRAPHICS PIPELINE

- API: **GPU command packets** submitted through ordering tables, via the SDK (`psxgpu` in PSn00bSDK). There is no OpenGL, no shaders, no depth buffer. Do NOT attempt a GL translation layer. [HIGH]
- Initialization (PSn00bSDK idiom): `ResetGraph(0)` → define two `DISPENV`/`DRAWENV` pairs (double buffering, e.g. framebuffer A at VRAM (0,0)–(320,240), B at (0,240)) → `PutDispEnv`/`PutDrawEnv` → `SetDispMask(1)`. Forgetting `SetDispMask(1)` = black screen with a working program behind it. [HIGH]
- Frame flow per frame:
  1. Clear OT with `ClearOTagR` (or DMA ch6 OTC). [HIGH]
  2. Build primitives in a CPU-side packet buffer, transform with GTE, `AddPrim` into OT bucket = average Z. [HIGH]
  3. `DrawSync(0)` → `VSync(0)` → swap DISPENV/DRAWENV → `DrawOTag` (kick DMA on the *reversed* OT so it draws back-to-front). [HIGH]
- Double-buffer VRAM layout at 320×240: two framebuffers stacked at y=0 and y=240 consume 320×480; the remaining 704×512 of VRAM holds textures and CLUTs. **Never upload a texture over an active framebuffer region.** [HIGH]
- VSync: NTSC 60 Hz (263 scanlines), PAL 50 Hz (314 scanlines). PAL consoles + PAL video mode = 50 Hz game logic unless you decouple timing. Set video mode explicitly (`SetVideoMode(MODE_NTSC/MODE_PAL)`). [HIGH]
- Texture upload: `LoadImage(rect, data)` (GP0 image transfer / DMA) then `DrawSync`. TIM is the conventional texture file format (tools: `img2tim`, PSn00bSDK's converters). [HIGH]
- Texture constraints: 4/8-bit CLUT or 15-bit; CLUTs are 16×1 or 256×1 strips in VRAM; primitives carry tpage+clut IDs; **no mipmaps, no filtering (nearest only), no wrapping across tpage boundaries** (texture window register gives limited repeat within a page). [HIGH]
- Blending: only the 4 fixed semi-transparency modes; per-primitive on/off; black (0,0,0) pixels with STP=0 are transparent in textures. [HIGH]
- Anti-patterns:
  - Do NOT poll or busy-wait the GPU per primitive; batch everything through the OT and one `DrawOTag`. [HIGH]
  - Do NOT sort by writing your own painter's sort on CPU; the OT *is* the sort (bucket by AVSZ). [HIGH]
  - Do NOT draw huge textured floors/walls as single quads — subdivide or accept severe affine warping. [HIGH]
  - Do NOT exceed ~1500–2500 textured polys/frame at 60 fps expectations; budget like it's 1996. [MEDIUM]

---

## 4. INPUT

- Port hardware: two SIO0 controller ports, each also hosting a memory card slot on the same serial bus; multitap expands to 4 pads per port. [HIGH]
- Controller types you must expect: Digital pad (ID 0x41), DualShock/analog in digital mode (0x41), analog mode (ID 0x73, two sticks + rumble), analog "flightstick" (0x53), mouse (0x12), neGcon (0x23), Guncon (0x63), multitap (0x80 wrapper). Check the ID byte every poll — do NOT assume a fixed type. [HIGH]
- Model: **polled, not event-driven.** The kernel/SDK polls pads during the VSync interrupt into buffers you register: PSn00bSDK `InitPAD(buf1, 34, buf2, 34)` → `StartPAD()` → read the raw buffer each frame; or use the higher-level `psxpad` helpers. Poll/read once per frame. [HIGH]
- Buttons arrive as an active-low 16-bit mask (0 = pressed) — invert before use. Analog sticks: 4 bytes, 0x00–0xFF, center ≈ 0x80; apply your own deadzone (~±0x10–0x20). [HIGH]
- Rumble (DualShock): requires entering "config mode" (0x43) and unlocking motor mapping (0x4D) before motor bytes in the poll command take effect; PSn00bSDK exposes this in its pad library. Mark exact command sequencing for verification against psx-spx SIO0 docs. [MEDIUM]
- Disconnect handling: a poll returning 0xFF ID / no-ack means no pad; treat every frame's read as potentially absent and neutralize inputs. Re-init is not required — polling just resumes when reconnected. [HIGH]
- Anti-patterns:
  - Do NOT bit-bang SIO0 yourself mid-frame while the SDK's VSync pad handler is active — bus conflicts with memory cards. [MEDIUM]
  - Do NOT read analog bytes from a pad reporting ID 0x41 (digital) — the bytes are not present. [HIGH]

---

## 5. MEMORY LAYOUT

| Region | Physical base | KSEG0 (cached fetch) | KSEG1 (uncached) | Size | Use |
|---|---|---|---|---|---|
| Main RAM | 0x00000000 | 0x80000000 | 0xA0000000 | 2 MB | code+data; kernel owns first 64 KB (0x0000–0xFFFF) |
| Scratchpad | 0x1F800000 | — (data only) | — | 1 KB | hot data, GTE staging; no DMA, no code |
| I/O ports | 0x1F801000 | — | via 0xBF801xxx | 8 KB | GPU/SPU/DMA/timers/SIO/CD registers |
| BIOS ROM | 0x1FC00000 | 0x9FC00000 | 0xBFC00000 | 512 KB | kernel |

All [HIGH] per psx-spx.

- PS-EXE default load/entry: **0x80010000** (right above the kernel's 64 KB). Text+data+bss+stack+heap all share the remaining ~1.94 MB. [HIGH]
- Default stack: BIOS sets SP to 0x801FFF00 (top of 2 MB); the EXE header can override. Stack size is "whatever you don't use" — there is no guard page; overflow silently corrupts your heap/BSS. Keep deep recursion out. [HIGH]
- Cache behavior: only instruction fetches are cached (4 KB I-cache). **Self-modifying or freshly loaded code (overlays!) requires an I-cache flush: call BIOS `FlushCache()` (A0h function 0x44) after copying code into RAM.** Data coherency with DMA needs no flush (data isn't cached) — but the write queue exists; SDK sync functions handle it. [HIGH]
- KSEG1 (0xA0xxxxxx) mirrors RAM uncached — use it only for MMIO-style access; running code from KSEG1 disables the I-cache and roughly halves speed. [HIGH]
- Allocation: no banked memory decisions to make (one RAM). Use the SDK heap (`InitHeap` + SDK malloc) or your own bump/arena allocators. **Never the raw BIOS malloc.** [HIGH]
- DMA: 4-byte alignment, sizes in 32-bit words; linked-list packets must have the 24-bit next-pointer/size header word the SDK macros build for you. [HIGH]
- Anti-patterns:
  - Do NOT place DMA buffers or GPU packets in scratchpad. [HIGH]
  - Do NOT touch 0x00000000–0x0000FFFF (kernel) unless you are intentionally replacing the kernel. [HIGH]
  - Do NOT assume `.bss` is zeroed by hardware — the SDK crt0 zeroes it; custom crt0s must too. [MEDIUM]

---

## 6. AUDIO

- Hardware: SPU with **24 voices**, each playing **SPU-ADPCM** (4-bit, 16-byte blocks = 28 samples, with per-block loop/end flags) from 512 KB SPU RAM. Per-voice: pitch (variable rate up to 44.1 kHz ×4), volume L/R, ADSR envelope. Global reverb with configurable presets. Output 44.1 kHz 16-bit stereo. [HIGH]
- **No raw PCM voice playback.** Everything sample-based must be encoded to SPU-ADPCM (VAG container). Tools: PSn00bSDK/community `vag` encoders, `es-ps2-vag-tool`, ffmpeg-based pipelines. [HIGH]
- Music options, pick one:
  1. Sequenced (tracker-style) music with instrument samples in SPU RAM — smallest footprint. Community players exist (e.g., PSn00bSDK examples, hitmod/xm-style players); quality/maturity varies [MEDIUM].
  2. **CD-DA** audio tracks — zero CPU, but stops during data seeks. [HIGH]
  3. **CD-XA ADPCM** streaming — interleaved audio+data, plays while you stream assets; supported by PSn00bSDK's `psxcd`. [HIGH]
- Buffering model: for one-shot SFX, upload VAG to SPU RAM (`SpuSetTransfer`-style DMA on channel 4) once, then key voices on/off — no per-frame callback exists or is needed. For streamed BGM from CD you double-buffer SPU RAM regions and feed from CD interrupts. [HIGH]
- First 4 KB of SPU RAM is reserved (capture/CD areas); allocate uploads above 0x1010 as the SDK does. [MEDIUM]
- Glitch avoidance: finish SPU DMA before keying a voice in that region; always set the ADPCM loop-end flag on the final block (unterminated samples play garbage through SPU RAM); don't hammer voice key-on more than once per tick. [HIGH]
- Anti-patterns:
  - Do NOT stream raw PCM through the CPU per-sample — there is no efficient path; encode to ADPCM. [HIGH]
  - Do NOT upload to SPU RAM regions currently being played. [HIGH]

---

## 7. STORAGE / IO

- Media reachable from homebrew: CD-ROM (ISO9660, mode 2 form 1 data + XA + CD-DA), memory cards (128 KB, 15 usable blocks × 8 KB, custom filesystem), SIO1 serial, and ODE-provided virtual discs. **No SD, no USB, no HDD, no network stack** (the Yaroze/serial and rare link-cable setups are DIY territory) [HIGH].
- CD filesystem: ISO9660 level 1 — 8.3 uppercase filenames, `;1` version suffix (`\DATA\LEVEL1.BIN;1`). PSn00bSDK: `CdInit()` → `CdSearchFile()` → `CdControl(CdlSetloc…)`/`CdRead()` async with completion callbacks or sync wrappers. [HIGH]
- Boot requirements on disc: root must contain **SYSTEM.CNF** (`BOOT = cdrom:\MAIN.EXE;1`, plus TCB/EVENT/STACK lines) and the license data region (mkpsxiso injects a license file). Unmodified retail consoles still refuse CD-Rs (SCEx wobble) — boot via modchip/tonyhax/FreePSXBoot/ODE. [HIGH]
- Build CD images with **mkpsxiso** (XML manifest → BIN+CUE). [HIGH]
- Memory cards: accessed over SIO0 with frame-based read/write protocol; 128-byte sectors; SDK helpers exist (PSn00bSDK memcard support is present but younger — verify writes on hardware) [MEDIUM]. Writes are slow (~seconds per block) — never block the game loop synchronously on saves. [HIGH]
- Homebrew loader conventions: with **Unirom**, executables are uploaded over SIO1 serial (or swap-loaded from CD) using **nops** (NotPSXSerial): `nops /fast /exe game.exe COMx`. There is no "apps folder" convention like later consoles — deliverable is a PS-EXE and/or a BIN/CUE. [HIGH]
- Anti-patterns:
  - Do NOT lowercase or long-name your CD paths. [HIGH]
  - Do NOT seek-then-read synchronously mid-gameplay; CD seeks cost 0.1–1 s. Prefetch into RAM. [HIGH]
  - Do NOT stream data and expect CD-DA music to keep playing — CD-DA and data reads share the single laser; use XA interleave instead. [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: upstream GCC + binutils for target **`mipsel-none-elf`** (GCC 12+ works; PSn00bSDK docs pin known-good versions; prebuilt toolchains ship with PSn00bSDK releases and PCSX-Redux "mips" tooling). [HIGH]
- Cross prefix: `mipsel-none-elf-` (gcc/ld/objcopy). [HIGH]
- Essential compiler flags (PSn00bSDK CMake sets these — do not fight them):
  - `-march=r3000` (or `-march=mips1`) `-mtune=r3000` — MIPS I only. [HIGH]
  - `-EL` little-endian (default for mipsel). [HIGH]
  - `-msoft-float` — no FPU; also avoid `double` entirely in hot code. [HIGH]
  - `-mno-abicalls -fno-pic -G0` — no PIC/GOT, disable gp-relative small data unless you configure it deliberately. [HIGH]
  - `-fdata-sections -ffunction-sections` + `--gc-sections`, `-Os` or `-O2`. [MEDIUM]
- Output pipeline: compile/link to ELF → convert to **PS-EXE** (`.exe` with 2 KB "PS-X EXE" header carrying load addr/entry/SP) using PSn00bSDK's `elf2x` (or `elf2exe`) → optionally pack into BIN/CUE with **mkpsxiso**. [HIGH]
- Recommended setup: PSn00bSDK via CMake presets (`cmake --preset default && cmake --build build`); env var **`PSN00BSDK_LIBS`** must point at the installed SDK libs; toolchain on PATH. [HIGH]
- Asset pipeline (preprocess at build time, never at runtime):
  - Textures → TIM (quantize to 4/8-bit CLUT; pack VRAM layout deliberately). [HIGH]
  - Audio SFX/instruments → VAG (SPU-ADPCM). BGM → XA or CD-DA tracks in the mkpsxiso manifest. [HIGH]
  - 3D models → your own fixed-point binary format (16-bit vertices, precomputed normals for GTE lighting). No standard homebrew model format — budget time for a converter. [MEDIUM]
- Anti-patterns:
  - Do NOT enable `-mhard-float`, `-mips2`+, or 64-bit types in hot paths. [HIGH]
  - Do NOT link desktop libc/newlib expecting syscalls to exist. [HIGH]
  - Do NOT ship an ELF to loaders that expect PS-EXE (Unirom/`nops` wants the converted `.exe`). [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Development emulator: **PCSX-Redux** — built for homebrew dev: GDB server, memory/VRAM/GPU debuggers, Lua scripting, web API, direct PS-EXE loading, "fastboot". Use it as your primary loop. [HIGH]
- Accuracy emulator: **DuckStation** — very accurate, good for sanity-checking rendering/timing; also loads EXEs directly. **no$psx** is valuable mainly for its debug views and because its author wrote psx-spx. [HIGH]
- Safe to rely on in emulators: CPU/GTE arithmetic results, GPU packet semantics and VRAM layout, basic CD filesystem reads, pad protocol basics. [HIGH]
- MUST verify on real hardware:
  - Exact DMA/GPU timing and frame budget — emulator FPS is not hardware FPS. [HIGH]
  - I-cache effects (many emulators don't emulate the I-cache; missing `FlushCache()` bugs appear ONLY on hardware — the classic "works in emulator, crashes on console"). [HIGH]
  - SPU edge cases (loop flags, reverb, voice stealing), CD seek latencies and XA interleave timing, memory card write timing, uninitialized-memory reads (emulators zero RAM; hardware has garbage). [HIGH]
- Hardware debugging options: Unirom + serial (SIO1) gives `printf`-over-serial, memory peek/poke, EXE upload, and a **GDB stub** (nops bridge). Older CommsLink/Xplorer carts also work. [HIGH]
- Crash forensics: install an exception handler that dumps EPC/Cause/BadVaddr + register file over serial or renders them on screen ("blue screen of your own making"); PSn00bSDK examples show custom exception hooks. [MEDIUM]
- Anti-patterns:
  - Do NOT tune performance against emulator frame rates. [HIGH]
  - Do NOT rely on RAM being zeroed at boot. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, program otherwise running | After graphics init | `SetDispMask(1)` never called, or DISPENV/DRAWENV rects wrong | Call `SetDispMask(1)` after first frame; verify env rects match video mode | [HIGH] |
| Black screen on PAL console | NTSC-only init | Video mode mismatch (NTSC mode on PAL set can also show rolling/BW picture) | `SetVideoMode()` per region; detect via BIOS/GPU status | [HIGH] |
| Polygons flicker/missing randomly | 3D scene | OT overflow (Z index out of range) or packet buffer overrun into live OT | Clamp AVSZ result to OT length; enlarge packet buffer; double-buffer packets | [HIGH] |
| Textures corrupted/garbage colors | After loading new textures | Texture uploaded over active framebuffer, wrong CLUT coords, or missing `DrawSync` before `LoadImage` | Audit VRAM layout map; `DrawSync(0)` before uploads; verify tpage/clut values | [HIGH] |
| Works in emulator, crashes on hardware after loading overlay/code | Dynamic code loading | Missing I-cache flush | Call BIOS `FlushCache()` after every code copy | [HIGH] |
| Exception: Address Error (Cause excode 4/5) | Random crash, BadVaddr unaligned | Unaligned `lw`/`sw` from packed structs or pointer casts | 4-byte-align structs; use memcpy for packed data | [HIGH] |
| Crash/garbage after large CD read | Streaming assets | Read overran destination buffer (CD reads are whole 2048-byte sectors) or read into stack region | Round buffers up to sector multiples; check load addresses vs SP | [HIGH] |
| Audio: clicks or garbage noise loop | After SFX plays | Missing ADPCM loop/end flag on last block, or voice keyed before SPU DMA finished | Re-encode VAG properly; wait for transfer completion before key-on | [HIGH] |
| Audio silent | SPU init | SPU master volume / CD input volume left at 0, or upload below 0x1010 | Init SPU with SDK defaults; set master vol; relocate samples | [MEDIUM] |
| Pad input dead | Boot | `StartPAD()` not called, or VSync IRQs disabled so pad polling never runs | Init pads after `ResetGraph`; keep VSync callback alive | [HIGH] |
| "File not found" on CD | CdSearchFile fails | Lowercase name, missing `;1`, or path not in mkpsxiso manifest | Uppercase 8.3 + `;1`; rebuild image; dump ISO to verify | [HIGH] |
| 10–20 fps, expected 60 | Perf | Software float creep (`float`/`double` in loops), drawing without OT batching, or code running from KSEG1 | Grep for float ops; batch via OT; run from 0x8000xxxx | [HIGH] |
| Screen squashed/letterboxed wrong | Display | 480i env with 240p rects (or vice versa), PAL 256-line centering | Match DISPENV `.screen` to mode; adjust for PAL offsets | [MEDIUM] |
| Random heap corruption over minutes | Long sessions | Stack (top of RAM) growing into heap/BSS — no guard exists | Move SP/define stack size in EXE header; reduce recursion; add canary word | [HIGH] |
| Hangs on second frame | Render loop | `DrawSync`/`VSync` order wrong or OT drawn while still being built (single-buffered packets) | Use two OT+packet buffers, swap per frame | [HIGH] |

---

## 11. ANTI-PATTERNS

1. Do NOT use `float` or `double` anywhere hot — there is no FPU; use 20.12 fixed point and the GTE. [HIGH]
2. Do NOT expect a Z-buffer; you must bucket every primitive into an ordering table. [HIGH]
3. Do NOT use the BIOS `malloc` — it is broken; use the SDK heap or arenas. [HIGH]
4. Do NOT copy code into RAM (overlays, loaders) without calling `FlushCache()`. [HIGH]
5. Do NOT dereference unaligned pointers; MIPS raises Address Error exceptions. [HIGH]
6. Do NOT upload textures without `DrawSync(0)` first, or over live framebuffer VRAM. [HIGH]
7. Do NOT assume the connected pad is a DualShock — check the ID byte every poll. [HIGH]
8. Do NOT block the main loop on CD seeks or memory-card writes — both are asynchronous, seconds-scale operations. [HIGH]
9. Do NOT let a textured quad span a 256×256 texture page boundary. [HIGH]
10. Do NOT render big near-camera polygons untessellated — affine warping will be severe. [HIGH]
11. Do NOT assume RAM is zeroed or that `.bss` clearing happens without a proper crt0. [HIGH]
12. Do NOT put DMA buffers in the 1 KB scratchpad or run code from it. [HIGH]
13. Do NOT busy-wait on the GPU per primitive; build the whole frame, then one `DrawOTag`. [HIGH]
14. Do NOT trust emulator timing, emulator-zeroed memory, or emulator I-cache leniency. [HIGH]
15. Do NOT target 640×480; 320×240 progressive is the platform's native sweet spot. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Audit the engine's math** — grep for `float`/`double` and division in hot paths. *Why first:* soft-float makes ports unplayably slow; converting to fixed point is the largest structural change and dictates everything downstream. *Skip it and:* you'll "finish" the port at 5 fps and rewrite anyway. [HIGH]
2. **Memory budget to 2 MB** — inventory code size, largest level's assets, audio. *Why:* if the working set can't fit 2 MB + 1 MB VRAM + 512 KB SPU, you need streaming/overlay architecture designed NOW. *Skip it and:* you hit an untraceable OOM wall mid-project. [HIGH]
3. **Stand up the platform layer on PSn00bSDK** — video init, solid-color double-buffered frame, pad read, timer. Validate on hardware (Section 14 gates). *Why:* everything else stacks on a proven main loop. *Skip it and:* you debug engine and platform bugs simultaneously. [HIGH]
4. **Replace the renderer, don't wrap it** — map the engine's draw calls to OT + GPU packets; move transform/lighting onto the GTE. *Why:* GL-style immediate mode translation layers waste the machine (your OpenGX lesson applies doubly here — there is not even a GL-shaped GPU to translate to). *Skip it and:* permanent 3–5× slowdown. [HIGH]
5. **Asset pipeline conversion** — textures to 4/8-bit CLUT TIMs with a planned VRAM atlas; audio to VAG/XA; models to fixed-point binaries. *Why:* runtime conversion is impossible (no CPU/RAM headroom). *Skip it and:* nothing fits and load times explode. [HIGH]
6. **File I/O → CD layer** — replace fopen-style access with async `CdRead` + a pack file; sector-align everything; design around 300 KB/s and slow seeks (your `FS_ReadFile` interception pattern from the PS4 procgen brief maps well here). *Skip it and:* stutters at every load. [HIGH]
7. **Audio port** — SFX to SPU voices; BGM decision (sequenced vs XA vs CD-DA) driven by whether gameplay streams data. *Skip it and:* late discovery that CD-DA + streaming conflict forces a music re-encode. [HIGH]
8. **Memory card saves** — last, async, with the SDK's memcard flow; hardware-verify writes. *Skip it and:* only cosmetic loss until release hardening. [MEDIUM]
9. **Hardware performance pass** — profile with hardware timers (root counters), move hot data to scratchpad, tune OT depth, check DMA chop. *Skip it and:* you ship emulator-tuned performance. [HIGH]

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PS1 / PSX — R3000A MIPS I @ 33.8688 MHz, little-endian, NO FPU, NO MMU, 4KB I-cache,
1KB scratchpad @ 0x1F800000. GTE (CP2) = fixed-point 3D coprocessor (RTPT/AVSZ/NCS).
RAM: 2MB main (kernel owns first 64KB; EXE loads @ 0x80010000; SP top @ 0x801FFF00)
VRAM: 1MB = 1024x512 x16bpp grid (framebuffers+textures+CLUTs; GPU/DMA access only)
SPU RAM: 512KB, 24 voices, SPU-ADPCM (VAG) only. CD: 2x = 300KB/s, ISO9660 8.3;1.
GPU: 2D rasterizer, NO Z-buffer (ordering tables), affine textures (subdivide!), int
vertices (jitter), 4/8bit CLUT tex in 256x256 tpages, 4 fixed blend modes, 320x240std.
NTSC 60Hz / PAL 50Hz. DMA ch: 2=GPU-list, 3=CD, 4=SPU, 6=OT-clear.
SDK: PSn00bSDK (C, CMake, $PSN00BSDK_LIBS) or PSYQo (C++20). NEVER leaked Psy-Q.
Toolchain: mipsel-none-elf-gcc  -march=r3000 -msoft-float -mno-abicalls -fno-pic -G0
ELF -> elf2x -> PS-EXE (.exe) -> mkpsxiso -> BIN/CUE. Emu: PCSX-Redux (dev/GDB),
DuckStation (accuracy). HW load: Unirom + nops over SIO1 serial / FreePSXBoot / ODE.
Frame: ClearOTagR -> build prims (GTE) -> AddPrim(ot[avgZ]) -> DrawSync -> VSync ->
swap envs -> DrawOTag(reversed). Golden rules: fixed-point everywhere, FlushCache()
after code copies, DrawSync before LoadImage, align everything to 4, async CD only,
BIOS malloc is broken, emulator success != hardware success.
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
   - Exact build command (e.g., `cmake --preset default && cmake --build ./build`)
   - Expected output file name and location (e.g., `build/game.exe` PS-EXE, or `build/game.bin`/`.cue`)
   - How to transfer to the target hardware (nops over serial: `nops /fast /exe build/game.exe COM3` — or burn/ODE-mount the BIN/CUE)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color double-buffered framebuffer"]
   BUILD: [Command]
   DEPLOY: [Method: nops serial / CD-R / ODE / FreePSXBoot card]
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
   - A minimal C/assembly test case that isolates the failure (e.g., fill VRAM rect via GP0 only; print GPU STAT register over serial),
   - OR a checklist of 3 specific diagnostic steps (e.g., "Confirm SetDispMask(1) is reached via serial printf", "Dump the DISPENV rect values", "Verify the EXE load address doesn't overlap the kernel 64KB").
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the debug protocol, you return to WAITING_FOR_HARDWARE state.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: You propose the next milestone from the goal stack below. You do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [Unirom GDB stub over SIO1 / serial register dump / PCSX-Redux side-by-side comparison / psxdev community forum].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide emulator logs / register dumps for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this default:
1. Initialize video output (solid color double-buffered framebuffer, correct NTSC/PAL mode)
2. Initialize controller input (read and display button presses)
3. Initialize audio output (upload a VAG, key one SPU voice)
4. Load assets from CD (CdSearchFile + async CdRead of a test file)
5. Render main menu framebuffer (TIM upload + sprite primitives via OT)
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
