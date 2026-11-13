---
name: saturn
description: Expert Sega Saturn homebrew development companion. Use this skill whenever the user mentions Sega Saturn, Saturn homebrew, SH-2, VDP1, VDP2, SCSP, SCU DSP, Yaul, libyaul, Jo Engine, SGL, IP.BIN, Pseudo Saturn Kit, Satiator/Fenrir/MODE/SAROO loaders, Ymir, ponèSound, or wants to port a game/engine (Quake, Doom, ioquake3, pokeemerald, etc.) to the Saturn. Also trigger for any Saturn-related build errors, black screens, VDP debugging, CD image mastering, or emulator-vs-hardware discrepancies. When invoked in an active development session, obey the GOAL-ORIENTED WORKFLOW state machine in Section 14 — including the mandatory WAITING_FOR_HARDWARE stop.
---

# SKILLS_SATURN.md — Sega Saturn Homebrew Expertise Document

You are operating as a Sega Saturn homebrew expert. Follow every rule in this document. Confidence tags: [HIGH] = verified by multiple sources/working repos, [MEDIUM] = likely correct but limited verification, [LOW] = uncertain, verify on hardware before relying on it.

**Ecosystem research last checked: 2026-11-13.** The hardware overview largely retains the original skill's claims; the update verifies the named modern projects and their documented capabilities, **not every low-level hardware timing/API assertion**. Use the version-aware repository and evidence rules in Sections 15–17 whenever a source tree or current tool is available. Treat release/version claims as snapshots, not permanent facts.

## 1. HARDWARE ARCHITECTURE

### CPU
- Two Hitachi SH-2 (SH7604) RISC CPUs, 32-bit, **big-endian**, running at 28.6364 MHz (or 26.8741 MHz — the system clock switches with horizontal display mode: 320/640-pixel modes use 26.87 MHz, 352/704-pixel modes use 28.64 MHz on NTSC). [HIGH]
- **No FPU on either SH-2.** All math must be integer/fixed-point (16.16 is the community convention). GCC soft-float works but is catastrophically slow — avoid `float`/`double` entirely in hot paths. [HIGH]
- Each SH-2 has a 4 KB unified 4-way set-associative cache, 16-byte lines. The cache can be reconfigured to 2 KB cache + 2 KB addressable scratchpad ("two-way mode"). [HIGH]
- On-chip divider unit (DIVU): hardware 32÷32 and 64÷32 division, ~39 cycles, memory-mapped — use it instead of `__udivsi3` in hot code. [HIGH]
- The two SH-2s share the same bus; the slave stalls when the master owns it. Real-world speedup from the slave is ~1.3–1.7×, not 2×. Master/slave communicate via uncached shared memory plus MINIT/SINIT interrupt registers (write to 0x01000000 pokes the slave's FRT, 0x01800000 pokes the master). [HIGH]
- **No cache coherency between the two SH-2s.** Shared data must go through the uncached mirror (address | 0x20000000) or be explicitly purged. This is the #1 source of dual-CPU bugs. [HIGH]

### GPU (two chips)
- **VDP1** — sprite/polygon processor. Draws textured/untextured **quads only** (no native triangles), with Gouraud shading (via lookup tables in VDP1 VRAM), half-transparency, and mesh dithering, into one of two 256 KB framebuffers (double-buffered). 512 KB dedicated VDP1 VRAM holds command tables + textures. [HIGH]
- **VDP2** — background/compositing processor. Up to 4 tile/bitmap scroll planes (NBG0–NBG3) plus rotation planes (RBG0, RBG1) for Mode-7-style effects; handles priority, color calculation (blending), line/window effects, and final video output. 512 KB VDP2 VRAM (4 banks with per-bank cycle-access budgets) + 4 KB CRAM for palettes. [HIGH]
- Resolutions: 320×224/240 and 352×224/240 progressive; 640/704 wide and 448/480 interlaced modes exist. NTSC 59.94 Hz, PAL 50 Hz. [HIGH]
- No shaders, no Z-buffer, no perspective-correct texturing. Depth is handled by painter's-algorithm draw order in the VDP1 command list. [HIGH]
- VDP1 pixel formats: 16-bit RGB555 (bit 15 = RGB flag) and 4/8-bit palettized ("color bank"/"lookup table" modes). VDP1 has no texture size limit beyond VRAM, but quads sample textures with per-edge stepping — expect distortion vs. affine PC rasterizers. [HIGH]

### RAM
- 1 MB **High Work RAM** (SDRAM, fast, burst-capable) at 0x06000000 — put code, stack, and hot data here. [HIGH]
- 1 MB **Low Work RAM** (DRAM, slower, higher latency) at 0x00200000 — bulk/cold data. [HIGH]
- 512 KB VDP1 VRAM, 2×256 KB VDP1 framebuffers, 512 KB VDP2 VRAM, 4 KB CRAM, 512 KB sound RAM, 512 KB CD-block buffer RAM (only reachable through CD-block commands), 32 KB battery-backed backup RAM. Total ≈ 4.5 MB but heavily partitioned — you cannot treat it as one pool. [HIGH]
- No MMU in use, no virtual memory. Flat physical addressing. [HIGH]
- SH-2 requires natural alignment: 16-bit accesses on 2-byte boundaries, 32-bit on 4-byte. Misaligned access raises an address-error exception (does NOT silently work like x86). [HIGH]
- Expansion cartridge RAM: 1 MB or 4 MB DRAM carts map into A-bus (CS0, around 0x02000000/0x02400000 region depending on cart) — usable by homebrew if present, but never assume it. [MEDIUM]

### Bus topology
- SH-2s ↔ SCU (System Control Unit) ↔ B-bus (VDP1, VDP2, SCSP) and A-bus (cartridge, CD block). The SCU is the traffic cop; CPUs cannot burst-write B-bus devices efficiently on their own. [HIGH]
- SCU DMA: 3 levels (0/1/2) with different max transfer sizes; level 0 is the general workhorse. Use SCU DMA for WRAM→VDP1/VDP2 VRAM transfers instead of memcpy. **SCU DMA cannot read from the SH-2 cache region and has known restrictions writing B-bus (destination add must be set correctly, and DMA from Low WRAM to B-bus has documented erratum-level quirks — prefer High WRAM sources).** [MEDIUM]
- The CD block (its own SH-1 CPU + 512 KB buffer) streams sectors; you fetch them via CD-block register commands or DMA. [HIGH]

### Co-processors
- **SCU DSP**: 32-bit fixed-point DSP with tiny instruction RAM (256 instructions) and 4×64-word data RAM, programmed in its own assembly, VLIW-ish (parallel ALU/X/Y/D1 fields). Powerful for matrix math but notoriously hard; most homebrew ignores it. [HIGH]
- **SCSP sound system**: Yamaha SCSP (YMF292) + Motorola 68EC000 @ ~11.3 MHz sound CPU + 512 KB sound RAM. The 68K runs a sound driver; the main CPU talks to it through shared sound RAM. [HIGH]
- **SMPC** (System Manager & Peripheral Control): manages joypads, reset, RTC, and powering the slave SH-2 / 68K on and off. [HIGH]

### Security / DRM
- Boot ROM checks a "security ring" pattern physically pressed into the disc lead-in — impossible to burn on CD-R. Homebrew therefore needs one of: Pseudo Saturn Kit (free firmware flashed onto an Action Replay cart), an ODE (Satiator, Fenrir, MODE), a modchip, or the swap trick. [HIGH]
- No hypervisor, no signed executables beyond the disc check — once booted, homebrew has full hardware access, including the BIOS ROM (readable at 0x00000000). [HIGH]

## 2. OFFICIAL VS HOMEBREW SDK

- Official SDKs were Sega's **SGL** (Sega Graphics Library, C, gouraud/3D-oriented) and **SBL** (Saturn Basic Library, lower level). Both are leaked/proprietary. **You must NOT use, reference APIs from, or provide build instructions for SGL/SBL.** [HIGH]
- **Preferred: Yaul (libyaul)** — open-source Sega Saturn SDK and toolchain ecosystem, including VDP1/VDP2/SCU-DMA/SMPC/CD abstractions and supporting libraries. Its repository includes `libmic3d` (3D), `libfiber`, `libmenu`, `libtga`, `libbcl`, and tools; consult their actual READMEs and checked-out headers before deciding you need a custom subsystem. For **Yaul-specific examples**, use `yaul-org/libyaul-examples` and the matching SDK revision. [HIGH] [Source](https://github.com/yaul-org/libyaul)
- **Jo Engine** — open-source MIT-licensed C 2D/3D engine with sprites, audio, level/cell-and-portal tooling, model conversion, profiling, software/voxel rendering, and input/storage helpers documented in its 2026 API docs. The online documentation labels itself **2026.11.2** and says it talks directly to the hardware without Sega libraries; the GitHub releases page still lists **v2025.1** as the latest tagged stable release **as checked 2026-11-13**. Treat documentation and a tagged release as potentially different code states; **audit the exact checkout and linked objects before claiming it is SGL-free**. Jo APIs are **not Yaul APIs**. [MEDIUM] [Docs](https://jo-engine.org/doxygen/) · [Releases](https://github.com/johannes-fetz/joengine/releases)
- **SaturnRingLib (SRL)** — its own README calls it an **SGL wrapper in C++**, **not** an SGL-independent SDK. Given this skill's clean-room/no-SGL requirement, do not recommend integrating SRL or its SGL playback stack. You may mention it for ecosystem comparison, explicitly labeled SGL-dependent; never copy SGL API or build instructions. [HIGH] [Source](https://github.com/ReyeMe/SaturnRingLib)
- **Audio ecosystem** — open-source 68K sound work exists, notably `ponut64/SCSP_poneSound` (ponèSound) with PCM, ADX decoding and streaming; do not say that CD-DA or handwritten SCSP register control are the only practical routes. Sequenced SEQ/TON tooling such as `jfsantos/mid2seq` currently describes SGL-driver playback and is **not** a drop-in clean-room runtime. [HIGH] [ponèSound](https://github.com/ponut64/SCSP_poneSound) · [mid2seq](https://github.com/jfsantos/mid2seq)
- **Remaining limitations are project-dependent**: a portable OpenGL-style renderer, general-purpose sequencer without driver integration, high-level DSP toolchain, and high-performance fixed-point geometry remain substantial undertakings; do not dismiss existing Yaul/Jo libraries before inspecting them. [MEDIUM]
- **SDK separation rule**: detect whether the target project uses Yaul, Jo Engine, SRL/SGL, or custom hardware code **before providing function names or build commands**. Default to Yaul for *new clean-room examples*, but respect a user's existing open-source SDK when compatible with the no-SGL rule.

## 3. GRAPHICS PIPELINE

- API: Yaul's `vdp1`/`vdp2` modules, or raw registers (VDP1 regs at 0x05D00000, VDP2 regs at 0x05F80000). There is no OpenGL, no ANGLE, no portable GPU API. **Do NOT attempt an OpenGL translation layer — the quad-based, Z-bufferless VDP1 cannot express GL semantics.** [HIGH]
- Initialization (Yaul): `vdp2_tvmd_display_res_set(VDP2_TVMD_INTERLACE_NONE, VDP2_TVMD_HORZ_NORMAL_A, VDP2_TVMD_VERT_224)` then `vdp2_tvmd_display_set()`; VDP1 environment via `vdp1_env_default_set()`. Yaul examples repo (`yaul-org/libyaul-examples`) has canonical init code — mirror it rather than improvising. [HIGH]
- Frame flow: build a VDP1 **command table** in VDP1 VRAM each frame (system clip → user clip → local coords → draw commands → end command). VDP1 rasterizes it into the back framebuffer; framebuffers swap on VBlank (auto or manual change mode). VDP2 then composites the VDP1 layer with its scroll planes for output. Effective double buffering is built-in; there is no triple buffering. [HIGH]
- If your VDP1 command list takes longer than one frame to draw, you drop to 30/20 fps in whole-frame steps. Budget polygon counts accordingly (a few hundred textured quads per frame at 60 fps is realistic; low thousands at 30). [MEDIUM]
- VSync: wait via `vdp2_tvmd_vblank_in_wait()` / VBlank interrupt. NTSC 59.94 Hz, PAL 50 Hz — never hardcode 60. [HIGH]
- Texture formats: 4-bit and 8-bit palettized (CRAM or VDP1 lookup tables) and RGB555. No compression (no S3TC/anything). Texture width for VDP1 sprites must be a multiple of 8 pixels. [HIGH]
- Transparency: color 0 / RGB msb conventions for transparent pixels; **half-transparency on distorted (textured, warped) quads double-draws pixels along edges, causing visible seams — this is a hardware behavior, not your bug.** Use mesh mode or VDP2 color calculation as alternatives. [HIGH]
- No depth buffer, no stencil. Sort your VDP1 commands back-to-front. Blending exists as VDP1 half-transparency and VDP2 color calculation ratios. [HIGH]
- **Prefer VDP2** for backgrounds, tiled HUDs, sky, and suitable fullscreen bitmaps to avoid expensive VDP1 overdraw, **but it is not free**: VRAM bank access cycles, pattern/character fetch windows, layer combinations, color calculation, and resolution constrain what can be shown concurrently. Budget VDP2 banks/cycles just as carefully as VDP1 drawing time. [HIGH]
- Anti-pattern: do NOT render a large static UI as hundreds of VDP1 sprites when a feasible VDP2 tilemap plane would save fill rate. Do NOT clear the framebuffer with CPU writes — use VDP1 erase/write or a background plane on VDP2. [HIGH]
- **3D implementation check**: before writing a new geometry engine, inspect Yaul's `libmic3d`, the actual Jo Engine version's 3D/model/level tools, and the application's performance/format needs. These libraries do not change the underlying VDP1 quad/Z-buffer constraints. [MEDIUM] [Yaul](https://github.com/yaul-org/libyaul) · [Jo Engine](https://jo-engine.org/doxygen/)

## 4. INPUT

- Controller types: standard digital pad (D-pad, A/B/C, X/Y/Z, L/R, Start), 3D Control Pad (analog stick + analog triggers, has digital/analog mode switch), Arcade Racer wheel, Mission Stick, Shuttle Mouse, keyboard, Virtua Gun (light gun), and the 6-port Multitap on each of the 2 ports (12 pads max). [HIGH]
- Model: **polled**, via the SMPC `INTBACK` command, which returns peripheral-type IDs plus data for everything connected (multitap-aware). Yaul wraps this as `smpc_peripheral_*` (`smpc_peripheral_process()` in the VBlank handler, then read `smpc_peripheral_digital_port(1, &digital)`). [HIGH]
- Each peripheral report includes a type ID byte — **check it every frame**; the 3D pad reports a different ID in analog vs. digital mode, and users hot-swap controllers. [HIGH]
- Analog: 3D Control Pad returns 8-bit X/Y plus 8-bit L/R trigger values in analog mode. [HIGH]
- Direct TH/TR pin-toggling reads ("manual mode") are faster than INTBACK and used by some engines, but break multitap/odd peripherals — only do this if INTBACK latency is a measured problem. [MEDIUM]
- No rumble on any first-party Saturn controller. No accelerometers/gyros. [HIGH]
- Disconnect handling: INTBACK simply reports "not connected" for the port; treat it as all-buttons-released and do not crash on type-ID change. [HIGH]
- Anti-pattern: do NOT assume port 1 is a digital pad; do NOT read the pad more than once per frame via INTBACK (it is slow, ~ms-scale). [MEDIUM]

## 5. MEMORY LAYOUT

| Address | Region | Notes |
|---|---|---|
| 0x00000000 | Boot ROM (512 KB) | readable [HIGH] |
| 0x00100000 | SMPC registers | [HIGH] |
| 0x00180000 | Backup RAM (32 KB) | odd-byte addressing (8-bit device on 16-bit bus) [HIGH] |
| 0x00200000 | Low Work RAM (1 MB DRAM) | slow [HIGH] |
| 0x01000000 / 0x01800000 | SINIT / MINIT | slave/master interrupt pokes [HIGH] |
| 0x02000000 | A-bus CS0 (cartridge) | [HIGH] |
| 0x05800000 | CD block registers | [HIGH] |
| 0x05A00000 | Sound RAM (512 KB) | 68K must be halted for safe bulk CPU writes [MEDIUM] |
| 0x05B00000 | SCSP registers | [HIGH] |
| 0x05C00000 | VDP1 VRAM (512 KB) | [HIGH] |
| 0x05C80000 | VDP1 framebuffer window | [HIGH] |
| 0x05D00000 | VDP1 registers | [HIGH] |
| 0x05E00000 | VDP2 VRAM (512 KB) | [HIGH] |
| 0x05F00000 | VDP2 CRAM (4 KB) | [HIGH] |
| 0x05F80000 | VDP2 registers | [HIGH] |
| 0x05FE0000 | SCU registers (DMA, DSP, interrupts) | [HIGH] |
| 0x06000000 | High Work RAM (1 MB SDRAM) | code + stack live here [HIGH] |
| addr \| 0x20000000 | Cache-through mirror of everything | uncached access [HIGH] |
| 0xFFFFFE00+ | SH-2 on-chip registers (DIVU, FRT, cache CCR…) | per-CPU [HIGH] |

- Allocation: Yaul's linker script places `.text/.data/.bss` in High WRAM starting at 0x06004000 (below that is BIOS/Yaul reserved space including vector table area); Low WRAM is yours to manage manually (declare a section or hand out addresses from 0x00200000). [MEDIUM]
- Default stack: master SP is set by IP.BIN (conventionally 0x06002000 region top-of-stack for early boot, then your runtime sets its own; Yaul configures master/slave stacks in its startup). Increase by editing the linker script / Yaul crt0 constants, not by guessing. [MEDIUM]
- Cache: SH-2 caches are effectively used **write-through** on Saturn (CCR configuration by convention); still, DMA engines and the other CPU never snoop the cache. Rules: (1) data written by DMA/VDP/CD into WRAM must be read through the uncached mirror or after a cache purge; (2) buffers you build for SCU DMA to read should be fine under write-through, but any write-back configuration requires explicit purge. Yaul: `cpu_cache_purge()` (whole cache) / line purges via the 0x40000000 purge area. When in doubt, purge. [MEDIUM]
- SCU DMA alignment: source/destination generally 4-byte aligned; B-bus writes have destination add-value requirements (add=2 for VDP2 VRAM quirks in some modes). Indirect-mode DMA tables must be 4-byte aligned. [MEDIUM] — verify with a small DMA-pattern test on hardware if a transfer corrupts.
- Anti-patterns: do NOT put the slave SH-2's working set in the same cache lines the master writes; do NOT bulk-memcpy to B-bus (VDP VRAM) with the CPU when SCU DMA exists; do NOT allocate hot code in Low WRAM.

## 6. AUDIO

- Hardware: Yamaha SCSP — 32 slots, each playing 8/16-bit PCM (or FM operator mode) at up to 44.1 kHz, with per-slot envelopes, LFO, panning, and a 128-step effects DSP (reverb/echo). Driven by the 68EC000 from 512 KB sound RAM. [HIGH]
- **Choose an audio design before implementation**: (a) small SFX via SCSP PCM slot control; (b) a **proven 68K sound driver** for effects and/or software-decoded compressed samples; (c) streamed PCM/ADX when CD bandwidth and buffering allow; (d) CD-DA for disc-track music when competing data reads are acceptable. Avoid building a new driver before evaluating tested sources. [MEDIUM]
- **ponèSound (`ponut64/SCSP_poneSound`)** is an MIT-licensed 68K SCSP driver with a core PCM library (`pcmsys`) and separate streaming library (`pcmstm`). Its README documents 8-bit signed/16-bit big-endian signed PCM, real-time ADX decompression on the sound CPU, and limited simultaneous streams/bitrates. It includes Jo Engine demos. These are **implementation-specific limits**, not capabilities of every Saturn sound driver; integrate only after checking the driver's current headers, memory needs, and sample format. [HIGH] [Source](https://github.com/ponut64/SCSP_poneSound)
- **CD-DA is cheap for SH-2 computation, not free for the optical drive**: ponèSound documents that CD-DA playback occupies CD access bandwidth while running, so concurrent game asset reads require an explicit plan. Do not promise seamless simultaneous CD-DA and CD data streaming. [HIGH] [Source](https://github.com/ponut64/SCSP_poneSound)
- Formats: PCM 8/16-bit signed big-endian in sound RAM; CD-DA from disc. The SCSP has **no native ADPCM slot decoding** (unlike the PS1 SPU); software decoding of ADX/ADPCM by the 68K or another processor is possible, with measured CPU/memory cost. [MEDIUM]
- **Sequenced music / FM research**: `Teuthida255/Tethys` offers experimental SCSP instrument editing based on a modified ponèSound driver; its README notes that its fork removed ADX support. `jfsantos/mid2seq` provides MIDI-to-SEQ, SoundFont-to-TON, and FM-patch tools, but its advertised playback path uses the SGL sound driver, so **do not present it as a compliant Yaul runtime**. Some of its bundled SCSP/WASM editor components have separate restrictive licensing; audit the relevant LICENSE files. [MEDIUM] [Tethys](https://github.com/Teuthida255/Tethys) · [mid2seq](https://github.com/jfsantos/mid2seq)
- Buffering for streamed audio: maintain and monitor a ring buffer in sound RAM; choose refill scheduling based on the driver and measured throughput (not an unconditional VBlank-only rule). Size for worst-case CD seek, bus contention, and decode jitter — a fixed number of video frames is only an initial estimate. [MEDIUM]
- The 68K and main CPUs contend for sound RAM; coordinate large uploads using the selected driver's protocol or safely suspend/restart the 68K as documented. Do not halt a live driver indiscriminately. [MEDIUM]
- Anti-patterns: do NOT compute audio in float on the SH-2 hot path; do NOT concurrently modify driver-owned SCSP registers or sound RAM; do NOT schedule CD streams and heavy sound-RAM copies without measuring bandwidth.

## 7. STORAGE / IO

- Media: CD-ROM (2× drive, ~300 KB/s ideal sequential throughput, Mode 1/Mode 2 sectors + CD-DA). Satiator, Fenrir, and MODE are optical-disc replacement/drive-intercept solutions with implementation-specific compatibility. Most software still uses the standard CD-block-facing workflow, but seek/timing behavior may differ; do not assume every loader exposes the same extras or hot-reload features. [MEDIUM]
- Filesystem: ISO 9660. Yaul provides `cdfs`/CD-block APIs (`cd_block_init()`, sector reads, ISO9660 directory walking). 8.3-style names are the safe convention; treat paths as case-insensitive-uppercase on disc. [MEDIUM]
- Disc structure the loader/BIOS requires: first 16 sectors = **IP.BIN** (system ID "SEGA SEGASATURN", region codes, master stack pointer, first-read file address/size, security code block), then ISO 9660 with your first-read binary (conventionally the file IP.BIN points at, loaded to 0x06004000). **Do not assume a universal image target or filename:** inspect the checked-out SDK example/project Makefile and verify the generated CUE/ISO/BIN paths. [HIGH]
- Backup storage: 32 KB internal battery-backed RAM at 0x00180000 (odd bytes only — an 8-bit device on a 16-bit bus, so usable capacity is what the BIOS backup library formats), plus backup RAM cartridges. Use the BIOS backup functions or a tested library; the on-disc format is shared with retail games. [MEDIUM]
- **Network/serial**: Saturn is **offline by default**, not incapable of experimental communications. `R11/saturn-libs` documents a prototype NetLink/modem UART/AT transport and host-side tests, but its Saturn target uses SGL include/build paths, so it is **research-only under this skill's no-SGL rule**; do not claim production readiness or clean-room compatibility. `Save-Game-Copier` 3.7.0 (May 2026 pre-release) lists experimental *write-only* modem and serial-link support — not a general Internet stack. [MEDIUM] [NetLink project](https://github.com/R11/saturn-libs) · [Save Game Copier releases](https://github.com/slinga-homebrew/Save-Game-Copier/releases)
- **SAROO**: open-source Saturn *cartridge-slot* SD disc loader (`tpunix/SAROO`), not a conventional drive-bay ODE; its repo describes CD-block emulation via FPGA/MCU, with 1 MB/4 MB expansion-RAM functionality. Test the actual SAROO hardware revision and firmware; compatibility, timing, and supported features may differ from an optical drive, Satiator, Fenrir, or MODE. [HIGH] [Source](https://github.com/tpunix/SAROO)
- **Save Game Copier** (`slinga-homebrew/Save-Game-Copier`) provides practical backup-RAM/cartridge/loader save-file transfer examples. Release 3.7.1 (May 14, 2026) is a **pre-release**, not evidence that every experimental transfer mode is stable. [HIGH] [Source](https://github.com/slinga-homebrew/Save-Game-Copier/releases)
- **Development transfer**: USB-capable development carts, CD/ODE/loader image deployment, and device-specific workflows can accelerate iteration, but **verify upload, debug, and hot-reload support per hardware+firmware**. Simply supporting SD-based images does not imply live code injection. [MEDIUM]
- Anti-patterns: do NOT assume seek is cheap (2× CD seeks are hundreds of ms — batch reads, lay out files contiguously); do NOT write to backup RAM without the proper block format (the BIOS memory manager will report it corrupted).

## 8. BUILD SYSTEM

- Toolchain: **Yaul's GCC cross toolchain, target `sh2eb-elf`** (big-endian SH-2, ELF). Install via Yaul's prebuilt packages (MSYS2 pacman packages on Windows, tarballs/AUR on Linux) or build from `yaul-org` repos. Recent GCC (11+) builds are provided. [HIGH]
- Cross prefix: `sh2eb-elf-` (`sh2eb-elf-gcc`, `sh2eb-elf-objcopy`, …). [HIGH]
- Compiler flags (Yaul's defaults do this for you): `-m2 -mb` (SH-2, big-endian), `-O2 -fomit-frame-pointer`, no `-m4`/`-msoft-float` confusion — SH-2 has no FPU so soft-float is implied; simply avoid floats. [HIGH]
- Linker: Yaul's provided linker scripts (place image base at 0x06004000) + `-nostartfiles` with Yaul crt0. [MEDIUM]
- Environment: Yaul documents `YAUL_INSTALL_ROOT`, `YAUL_PROG_SH_PREFIX`, `YAUL_ARCH_SH_PREFIX`, and `YAUL_BUILD_ROOT`, loaded from a `.yaul.env` environment file for some installation flows. **Variable names and make targets change by revision**; inspect the target repo, matching examples, and local shell before writing build commands. Do not assume `SH_PREFIX` or `yaul.env.mk` are universal. [HIGH] [Source](https://github.com/yaul-org/libyaul)
- Output flow: ELF → SH-2 binary (often using an `objcopy` step) → boot IP.BIN and CD image generated by the SDK/project tooling. **`make image` and `build/game.iso` are examples, not guaranteed targets/outputs**. Copy the exact commands and file names from the checked-out project and report them literally. [HIGH]
- Build reproducibility: record SDK repository/branch/commit, cross-toolchain version, `Makefile` target, linker script, output map, first-read binary name, region/IP metadata, and image hashes when available. Never mix Jo Engine and Yaul compiler flags, includes, or entry points. [HIGH]
- Asset pipeline: convert textures offline to RGB555 or palettized formats with correct byte order (big-endian 16-bit) and 8-pixel-multiple widths; convert audio to raw big-endian PCM at your target rate; convert models to fixed-point (16.16) vertex data. Never convert at runtime on the SH-2. [MEDIUM]
- Anti-patterns: do NOT compile with `-msoft-float`-dependent code paths assuming acceptable performance; do NOT link C library functions that drag in doubles (`printf("%f")` etc.); do NOT forget `-mb` if hand-rolling flags — a little-endian build produces a garbage image that boots to a black screen.

## 9. EMULATOR VS HARDWARE

- **Mednafen** (ss core): useful reference for CPU, graphics, and SCSP emulation, but **no emulator proves real-hardware correctness**. A Mednafen failure is a strong diagnostic signal, not a theorem about all consoles. [MEDIUM]
- **Ymir** (`ymir-emu/Ymir`) is a major modern accuracy/debugging option: SH-2 disassembly, breakpoints/watchpoints and call/data stack tracing, memory dumps, VDP2 VRAM-access timing and overlay views, CD partitions, and SCSP-related debugging. **Version checkpoint (2026-11-13): stable v0.4.1 released 2026-10-18**; v0.4.2/Vulkan was listed *in development*, and nightlies are experimental. Prefer its debugger for detailed subsystem inspection and cross-check tricky behavior against Mednafen/hardware. [HIGH] [Changelog](https://github.com/ymir-emu/Ymir/blob/main/CHANGELOG.md) · [Releases](https://github.com/ymir-emu/Ymir/releases)
- **Kronos** (Yabause fork) offers SH-2/VDP diagnostics and convenient iteration. **Yaba Sanshiro / classic Yabause** can be helpful for quick smoke tests but are not sufficient validation for cache/DMA/timing-sensitive code. Avoid a permanent absolute accuracy ranking across versions; prefer **differential testing** of Mednafen, Ymir, Kronos, and physical hardware. [MEDIUM]
- Emulators commonly differ on exact bus contention/DMA timing, cache effects, CD seek latency, indirect SCU DMA edge cases, uninitialized-VRAM contents, peripheral quirks, and cartridge-loader timing. Record which engine/version and BIOS/region were used. **Emulator-only success is not hardware validation.** [MEDIUM]
- **Hardware GDB is possible with a suitable transport**: `yaul-org/libyaul-gdbstub` is an open-source standalone stub; its README describes compiling `gdbstub.bin`, placing it in writable RAM, and supplying a byte-oriented device driver (e.g. a supported USB cartridge). It is **not a turnkey GDB service on every Saturn**; verify peripheral, load address, vector ownership, memory budget, and GDB transport. [HIGH] [Source](https://github.com/yaul-org/libyaul-gdbstub)
- Hardware debugging alternatives: USB development carts, serial/debug output where supported, exception handlers dumping PC/SR/registers via VDP2 text, and minimal register/VRAM test ROMs. Follow *the exact currently checked-out* Yaul exception setup rather than assuming every version presents the same panic screen. [MEDIUM]
- **Profiling**: emulator instruction traces, memory dumps, and VDP diagnostics are useful for localization, but final throughput claims (VDP1 draw completion, VDP2 access cycles, DMA overlap, CD/audio streaming, and master/slave contention) require measured hardware or a clearly marked emulation-only result. Do not profile exclusively in emulator and present it as hardware FPS. [HIGH]

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen at boot, BIOS logo shows | After IP.BIN hands off | First-read binary not loaded to/linked at 0x06004000, or wrong entry point | Match linker base address to IP.BIN first-read address; verify with Kronos debugger PC | [HIGH] |
| Black screen, no BIOS logo | Real hardware, burned CD-R | Security ring absent — disc rejected | Use Pseudo Saturn Kit / ODE / modchip; media is fine in emulator | [HIGH] |
| Boots in emulator, black screen on hardware | Any | Stale cache / uncached-mirror violation, or uninitialized VDP2 TVMD display bit | Purge cache around DMA'd data; ensure `vdp2_tvmd_display_set()`; test in Mednafen not Yabause | [MEDIUM] |
| Garbage/striped textures | VDP1 sprites | Wrong byte order (little-endian asset), width not multiple of 8, or bad character address alignment | Re-export assets big-endian; pad widths to 8 px | [HIGH] |
| Colors wrong / everything magenta-green | Any 16-bit art | RGB555 channel order or missing RGB flag bit 15 | Set bit 15 for RGB pixels; verify 0bMBBBBBGGGGGRRRRR layout | [MEDIUM] |
| Half-transparent quads show bright seams | VDP1 distorted sprites | Hardware double-draws edge pixels on warped quads | Use mesh mode, avoid half-transparency on distorted sprites, or composite via VDP2 | [HIGH] |
| Audio silent | SCSP direct-drive | 68K left halted with no driver while code expects driver, or sound RAM upload while 68K running corrupted it | Halt 68K during upload, resume after; verify slot key-on bits | [MEDIUM] |
| Audio crackles every few seconds | Streaming PCM | Ring-buffer underrun — refill tied to game loop that occasionally exceeds frame | Refill in VBlank interrupt; enlarge buffer to ≥4 frames | [MEDIUM] |
| Crash (address error panic) after loading a large file | CD reads into WRAM | Read destination overlaps stack/heap, or odd-address access into loaded struct | Check link map vs. load buffers; ensure struct fields naturally aligned (SH-2 faults on misalignment) | [HIGH] |
| Input dead / stuck | SMPC INTBACK | Peripheral processing not called each VBlank, or type-ID change (3D pad flipped modes) unhandled | Call `smpc_peripheral_process()` in VBlank; branch on type ID | [MEDIUM] |
| "File not found" from ISO | cdfs | Filename case/8.3 mismatch or file outside ISO9660 primary descriptor | Uppercase 8.3 names; rebuild image with Yaul tooling | [MEDIUM] |
| Slow: 60→30→20 fps steps | VDP1-heavy scenes | Command table exceeds one frame draw time | Cut quads, move layers to VDP2, shrink textures (fill-rate bound) | [HIGH] |
| Wrong aspect / overscan cut | PAL console or 352-px mode | Hardcoded NTSC 320×224 assumptions | Query/set TVMD per region; design safe areas | [MEDIUM] |
| Random corruption only with slave CPU enabled | Dual-CPU code | No cache coherency between SH-2s | Share data via 0x20000000 mirror; purge before handoff | [HIGH] |
| Background disappears/flickers while VDP1 looks correct | VDP2 multi-layer scene | VRAM bank cycle conflict, inaccessible fetch pattern, or unsupported layer combination | Inspect VDP2 access timing/banks (Ymir if available), then isolate one layer at a time | [MEDIUM] |
| CD music plays but asset loading stalls | CD-DA + CD data | CD-DA consumes optical drive access time | Choose load windows, preload assets, or use buffered/decoded music instead | [HIGH] |
| GDB stub appears unresponsive | Real hardware with USB/dev cart | Stub not in writable memory, unset stack, or wrong/missing byte-device driver | Check libyaul-gdbstub README and actual transport driver before debugging program state | [MEDIUM] |
| Image boots via emulator but fails under SAROO | Cartridge loader | Firmware, timing, cartridge configuration or CD emulation difference | Record SAROO hardware/firmware, compare another loader and real drive when available | [MEDIUM] |

## 11. ANTI-PATTERNS

1. Do NOT use `float` or `double` anywhere performance matters — there is no FPU; use 16.16 fixed-point. [HIGH]
2. Do NOT assume little-endian: the Saturn is big-endian; all binary assets and network-style byte twiddling must respect it. [HIGH]
3. Do NOT perform misaligned loads/stores — SH-2 raises an address-error exception instead of tolerating them. [HIGH]
4. Do NOT share cached memory between the two SH-2s without purging or using the uncached mirror. [HIGH]
5. Do NOT render triangles naively — VDP1 draws quads; degenerate quads (two identical verts) work but waste fill rate and distort textures. [HIGH]
6. Do NOT expect a Z-buffer or perspective-correct texturing; sort quads yourself. [HIGH]
7. Do NOT use CPU memcpy into VDP1/VDP2 VRAM for bulk data — use SCU DMA. [MEDIUM]
8. Do NOT put everything on VDP1; offload backgrounds, skies, and HUD to VDP2 planes. [HIGH]
9. Do NOT touch sound RAM in bulk while the 68K is running. [MEDIUM]
10. Do NOT treat the 4.5 MB total RAM as one heap — it is seven special-purpose pools. [HIGH]
11. Do NOT treat any emulator as hardware validation; use Mednafen and/or Ymir for diagnostics, differential checks where useful, and physical hardware for timing/cache/DMA claims. [HIGH]
12. Do NOT hardcode NTSC timing (59.94 Hz) or 320×224. [HIGH]
13. Do NOT scatter small files across the ISO — CD seeks at 2× are brutal; pack and read contiguously. [MEDIUM]
14. Do NOT reference or link SGL/SBL (leaked Sega SDKs) — use Yaul; flag any feature gap instead. [HIGH]
15. Do NOT enable the slave SH-2 expecting 2× speedup — plan for ~1.5× on bus-friendly workloads. [MEDIUM]
16. Do NOT label SRL or an SGL-backed SEQ/TON player as a clean-room/SGL-free SDK; verify transitive dependencies and content/tool licensing. [HIGH]
17. Do NOT claim VDP2 has zero cost; validate bank/cycle constraints and enabled layer combinations. [HIGH]
18. Do NOT claim an SDK API or `make` target exists until verified against the chosen revision's headers/examples/Makefile. [HIGH]

## 12. PORTING DECISION TREE

1. **Audit floating-point usage.** Priority #1 because no FPU exists; a float-heavy engine (Quake-era or later) needs a fixed-point conversion plan before anything else. Skip it and the port will "work" at single-digit fps or not link cleanly. [HIGH]
2. **Audit memory footprint against 2 MB work RAM (+1 MB VRAM budgets).** Decide what pool each subsystem lives in. Skip it and you discover mid-port that assets + code + heap cannot coexist. [HIGH]
3. **Fix endianness and alignment.** Big-endian + strict alignment breaks file loaders and packed structs written for x86. Skip it and get address-error panics and garbled assets. [HIGH]
4. **Stand up a reproducible booting example using the project's actual SDK** (Yaul by default for new clean-room projects; otherwise inspect Jo Engine/custom SDK). Verify the CUE/image in Mednafen/Ymir and, when available, the selected hardware loader. Record the toolchain, source revision, and outputs; do not mix APIs. [HIGH]
5. **Design the renderer around VDP1 quads + VDP2 planes** — map the engine's drawing model (BSP surfaces, sprites, UI) onto quad lists and scroll planes; decide the sorting strategy replacing the Z-buffer. Skip it and you get a CPU software renderer at unusable speed. [HIGH]
6. **Replace file I/O with CD-block/ISO9660 reads with contiguous layout.** Skip it and loads take minutes and stutter. [MEDIUM]
7. **Audio: choose a CD-DA vs. SCSP PCM vs. proven ponèSound/streamed-ADX design early.** Budget disc access, RAM, sound-CPU processing, and concurrent loading; treat SGL-dependent SEQ/TON toolchains as incompatible unless an independent legal playback backend is verified. [MEDIUM]
8. **Only then consider the slave SH-2 and SCU DSP** for transform/mixing workloads, with uncached-mirror communication. Doing this earlier multiplies debugging surface before the single-CPU path is proven. [HIGH]

## 13. QUICK REFERENCE CHEAT SHEET

```
SEGA SATURN: 2× Hitachi SH-2 @ 28.6/26.9 MHz, 32-bit, BIG-ENDIAN, NO FPU (16.16 fixed-point).
Caches: 4 KB each, no coherency between CPUs -> share via addr|0x20000000 or purge.
RAM: 1 MB High WRAM @0x06000000 (fast, code here) + 1 MB Low WRAM @0x00200000 (slow)
     + VDP1 VRAM 512K @0x05C00000 + 2×256K FB + VDP2 VRAM 512K @0x05E00000 + CRAM 4K
     + Sound RAM 512K @0x05A00000 + CD buffer 512K + Backup 32K. No MMU. Strict alignment.
GPU: VDP1 = QUADS ONLY (no tris, no Z-buffer, painter's sort), gouraud, RGB555/4-8bit pal,
     double-buffered FB. VDP2 = 4 scroll planes + rotation planes, compositing -> put
     backgrounds/HUD/sky here when BANK/CYCLE budget permits. 320×224@59.94 NTSC / 50 PAL.
AUDIO: SCSP 32 PCM slots (8/16-bit BE) + 68EC000 sound CPU; ponèSound PCM/ADX/streams,
       or CD-DA music (CD-DA competes with CD data reads).
INPUT: polled via SMPC INTBACK each VBlank; check peripheral type IDs (pads/3D pad/multitap).
SDK: Yaul (libyaul) + sh2eb-elf-gcc (-m2 -mb). NEVER SGL/SBL (leaked). ELF->BIN->ISO+IP.BIN,
     common image base 0x06004000; verify actual linker/IP settings and Makefile targets.
DEBUG: Ymir (modern SH-2/VDP2/CD/SCSP diagnostics), Mednafen (cross-check), Kronos; HW final.
BOOT: disc security ring -> Pseudo Saturn Kit / ODE (Satiator, Fenrir, MODE) / modchip;
      SAROO is a cartridge-slot SD/CD-block loader with expansion RAM (firmware-specific).
DMA: SCU DMA for WRAM->VRAM; 4-byte alignment; B-bus quirks; never CPU-memcpy to VRAM.
TOP BUGS: stale cache (works in emu, dies on HW), misalignment faults, endianness in assets,
     half-transparency seams on distorted quads, 60->30 fps cliffs from VDP1 overdraw.
```

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this skill is used in an active development session (not just reference), you MUST follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify Saturn-specific blockers (floats, endianness, alignment, memory pools, renderer model), and plan the implementation.
- **STATE: IMPLEMENTATION** — Write/modify code based on this document and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output the exact build commands and ask the user to compile and run on real hardware (or Mednafen if the user designates it as the validation target).
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back results. Classify as **SUCCESS, PARTIAL, or FAILURE**, without silently treating a partial as success.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES

1. **ANALYSIS → IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change. You must include:
   - Exact build command **confirmed against the project's Makefile/README** (for example `make clean && make image` **only if it really exists**)
   - Expected output file name and location **confirmed against the project** (do not invent `build/game.iso`)
   - How to deploy on the designated target (CD-R with suitable boot method, Satiator/Fenrir/MODE/SAROO with their actual supported image structure, or CUE in Mednafen/Ymir)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color VDP2 back screen"]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Expected behavior observed]
     - PARTIAL: [What works and what doesn't; actual behavior]
     - FAILURE: [Observed behavior, error codes/PC/registers if available]
   === STOP ===
   ```
   After this header, you STOP generating. You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE → VALIDATION**: Triggered when the user reports **SUCCESS**, **PARTIAL**, or **FAILURE**, or clearly describes their outcome in natural language. Do not force a binary answer when a partial observation provides real evidence.
   - **SUCCESS**: next goal only after the designated test passes; explicitly distinguish "emulator-validated" from "hardware-validated".
   - **PARTIAL**: record which expected observations passed and failed; move to a focused DEBUG_PROTOCOL for the missing behavior.
   - **FAILURE**: move to DEBUG_PROTOCOL.
   - If the outcome is unclear, ask **one** targeted question about the observable result. Do NOT proceed to a new milestone.
5. **VALIDATION (PARTIAL/FAILURE) → DEBUG_PROTOCOL**: Output a **DEBUG BUILD PROTOCOL**:
   - A minimal C/assembly test case that isolates the failure (e.g., write a known pattern to VDP2 CRAM and set a single back-screen color), OR
   - A checklist of exactly 3 specific diagnostic steps (e.g., "Confirm the panic screen shows an address-error PC inside the loader", "Read the framebuffer pointer through 0x25C80000 uncached and report the first 4 words", "Boot the same .cue in Mednafen and report whether it differs from hardware").
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the debug protocol, return to WAITING_FOR_HARDWARE. Collect the project commit, exact build output, designated target and version, BIOS/region, loader/firmware (if any), screenshot/log, and relevant PC/SR/register dump **only when available and useful**.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: Propose the next milestone from the goal stack below. Do NOT implement it until the user confirms. An emulator pass is not a physical-hardware pass; if hardware testing is unavailable, state the limitation rather than blocking unrelated analysis forever.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires more evidence or hardware-specific debugging. Recommend: [Ymir SH-2/VDP2/CD debugger / Mednafen differential / Kronos debugger / Yaul GDB stub with supported transport / hardware memory dump / SegaXtreme Saturn development forum].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide emulator logs / register dumps for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this default:
1. Initialize video output (solid-color VDP2 back screen)
2. Initialize controller input (read button presses via SMPC, display state)
3. Initialize audio output (play a PCM sine wave on one SCSP slot)
4. Load assets from CD (ISO9660 file read into High WRAM, verify checksum on screen)
5. Render main menu framebuffer (VDP1 sprites + VDP2 background)
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must NOT implement future goals speculatively.

## BEHAVIORAL CONSTRAINTS (ALWAYS ACTIVE)

- Do NOT provide generic PC/mobile advice unless explicitly marked cross-platform.
- Do NOT hallucinate APIs or hardware features. The Saturn has no native OpenGL, programmable shaders, FPU, Z-buffer, or MMU/virtual-memory platform. Network/serial projects exist experimentally; **do not** claim either universal Internet connectivity or that communication is categorically impossible. Give the Saturn-native alternative.
- Do NOT omit "obvious" platform-specific details: cache purging, big-endianness, strict alignment, memory-pool partitioning, quad-based rendering.
- If you circle on a topic or hit contradictory information, STOP and output a DEBUG BUILD PROTOCOL: a minimal C/assembly test case that definitively resolves the uncertainty on hardware or Mednafen.
- PREFER verifiable open-source, non-SGL homebrew SDKs (Yaul by default; compatible Jo Engine checkouts may be used when independently audited) over proprietary/leaked Sega SDKs. All code examples must **target the repository's actually selected community toolchain**. Do NOT reference leaked SDK APIs, paths, or build flags. If a feature gap exists, flag it explicitly and propose a tested alternative or minimal hardware test marked [LOW].
- Never assume the most recently updated documentation is identical to the latest tagged release; inspect branch/tag/commit and transitive licenses. Cross-check modern tools against Section 15 and the source URLs in Section 17.


## 15. MODERN SATURN PROJECTS AND TOOL SELECTION (CHECKED 2026-11-13)

**Purpose:** consult the original project repository and the *actual version in use* before implementing a subsystem. "Present in GitHub" is evidence of an implementation, **not** proof it is mature, up-to-date, commercially licensable, Yaul-compatible, or tested on a given hardware loader. Do not mix code across SDK families without an explicit port and license audit.

| Project | Use / verified evidence | Compatibility and cautions | Priority |
|---|---|---|---|
| [yaul-org/libyaul](https://github.com/yaul-org/libyaul) | Open-source Saturn SDK; contains `libmic3d`, `libfiber`, `libmenu`, `libtga`, `libbcl`, tools | Prefer matching examples, checked-out headers, build config and license | Core |
| [yaul-org/libyaul-gdbstub](https://github.com/yaul-org/libyaul-gdbstub) | Standalone SH-2 GDB stub, `gdbstub.bin`, selectable byte-device transport | Requires suitable writable RAM, stack, debugging transport, integration | High |
| [ymir-emu/Ymir](https://github.com/ymir-emu/Ymir) | Integrated SH-2, VDP/VRAM, CD-block and memory debugging; stable v0.4.1 on 2026-10-18 | Nightlies and v0.4.2 development notes are **not** stable release claims; emulation ≠ hardware | Core |
| [johannes-fetz/joengine](https://github.com/johannes-fetz/joengine) | C 2D/3D engine; docs list model/level systems, profile tools, software/voxel paths | Docs version **2026.11.2** vs last tagged **v2025.1** in checked release listing; inspect selected checkout for Sega-library dependency | High |
| [ponut64/SCSP_poneSound](https://github.com/ponut64/SCSP_poneSound) | MIT 68K audio driver; PCM, ADX decompression, stream libraries, Jo demos | Driver-specific rates/channels/buffers; test concurrent CD load and integration with chosen SDK | High |
| [tpunix/SAROO](https://github.com/tpunix/SAROO) | Open-source cartridge-slot SD HD loader/CD-block emulation and 1 MB/4 MB RAM | Firmware and hardware revision affect compatibility; not conventional drive-bay ODE | High |
| [slinga-homebrew/Save-Game-Copier](https://github.com/slinga-homebrew/Save-Game-Copier) | Save backup/copying and loader transfer examples; 2026 pre-releases for experimental serial/modem | 3.7.0/3.7.1 experimental modes not general networking; test specific media | Medium |
| [Teuthida255/Tethys](https://github.com/Teuthida255/Tethys) | Experimental SCSP instrument editor built from a modified ponèSound driver | Jo Engine based; README says ADX support removed in its modified driver; research reference | Experimental |
| [jfsantos/mid2seq](https://github.com/jfsantos/mid2seq) | MIDI→SEQ, SF2→TON, FM patch/editor conversion tools | Describes **SGL sound-driver playback**; not clean-room runtime; review separate WASM/editor licensing | Research only |
| [ReyeMe/SaturnRingLib](https://github.com/ReyeMe/SaturnRingLib) | Modern C++ wrapper and tooling | Explicitly **SGL-dependent** in README: **do not integrate under this skill's no-SGL policy** | Comparison only |
| [R11/saturn-libs](https://github.com/R11/saturn-libs) | Experimental peripheral, backup, NetLink code + host-side TDD | Saturn build docs reference SGL headers/tooling; not automatically compliant with Yaul/clean-room goals | Research only |

### Recommended selection rules

1. **New general-purpose clean-room game** → start with Yaul + a matching examples revision; evaluate `libmic3d` before starting a 3D transform library.
2. **Existing Jo Engine project** → stay on its own API/build conventions, inspect the selected checkout's dependencies; do not import Yaul calls into Jo code as though names were interchangeable.
3. **Hard-to-debug VDP2/VDP1/SH-2/CD behavior** → compare Ymir and Mednafen; use Ymir's subsystem inspectors; verify on hardware for timing/DMA/cache issues.
4. **Audio needs beyond simple SFX/CD-DA** → review ponèSound before writing a new 68K driver; choose real sample rates, channels, memory and throughput based on its source revision.
5. **Hardware reload/save workflows** → distinguish drive ODE, drive-interceptor, cartridge loader (SAROO), USB cart, backup cartridge and actual disc. Log model and firmware. Never assume all can hot-patch a running program.
6. **Networking/SEQ/FM experiments** → keep SGL-backed projects clearly labeled. The existence of research code doesn't waive clean-room, dependency, or license constraints.

## 16. SOURCE-AWARE DEVELOPMENT AND VALIDATION RULES (ALWAYS ACTIVE)

### Before writing any project-specific code

1. **Inventory the project**: identify SDK/engine, commit or tag, target hardware/loader, toolchain prefix/version, Makefile/build script, linker script, image builder, boot/entry settings, and existing example tests. Ask for missing *essential* details only when they cannot be inferred from the codebase.
2. **Source-of-truth hierarchy for APIs**: checked-out project's header and example > matching version's official docs/README > unpinned upstream docs > community posts > prior model knowledge. If unavailable, explicitly label pseudocode instead of presenting invented functions as compilable C.
3. **Search for working implementations**: look for graphics (Yaul `libmic3d`, Jo's 3D modules), SCSP drivers (ponèSound), debugger (Ymir/GDB stub), and storage/save implementations before designing a replacement. Check licensing and interface compatibility **before copying anything**.
4. **Pin source versions**: describe *which* Jo Engine revision, Yaul branch/commit, driver checkout, emulator build, and target firmware your claim applies to. Never infer a stable release from a web site's documentation version.
5. **Plan performance with real budgets**: VDP1 command completion/fill-rate; VDP2 access cycles and bank allocation; SCU DMA alignments and cache rules; SH-2 master/slave bus pressure; SCSP sound-RAM/68K timing; CD seek + CD-DA/asset-stream competition.
6. **Do not overclaim builds**: a proposed patch is "intended to compile" until actually built. Report build commands and output paths only after inspecting the source tree. Do not claim physical testing, screenshots, or timing measurements you have not performed.

### Three distinct verification levels

- **BUILD VERIFIED** — actual compile/link/image-generation succeeded, with command, toolchain version, output filenames and errors/warnings captured. This does *not* imply an emulator/hardware boot.
- **EMULATOR VERIFIED** — specified image ran in an identified emulator+version+BIOS region, with observed behavior. Name Ymir, Mednafen, or Kronos as applicable; still not a physical-hardware pass.
- **HARDWARE VERIFIED** — user or tester observed the specified behavior on a physical Saturn; record region/model, optical/cart loader and firmware when relevant, and the exact test outcome.

### Debug evidence contract

On **PARTIAL** or **FAILURE**, collect what is available from: `expected vs. observed`, SDK revision, exact command and full compiler/linker error, image path, program counter/SR/registers, Ymir/Mednafen/Kronos version, BIOS/region, cartridge/ODE/SAROO revision and firmware, VRAM/DMA/SCSP snapshots, and timing or audio underrun records. **Choose the smallest discriminating test**; do not simultaneously rewrite unrelated rendering, audio, and loader systems. Your Section 14 mandatory `WAITING_FOR_HARDWARE` stop still applies to the active development milestone, and "hardware" means the *designated test target*, clearly named.

### License and dependency guardrail

- Open-source code is **not** automatically independent of Sega-proprietary libraries. A wrapper's own permissive license does not resolve transitive library rights.
- Under this skill's policy, **do not integrate** SGL/SBL, SGL-backed SRL, or SGL-dependent sound playback into clean-room Yaul projects. For potentially independent newer Jo Engine checkouts, inspect linked archives/imported headers and release revision before concluding compliance.
- Check sound tools separately from sound runtimes. `mid2seq`'s conversion sources and its bundled SCSP/WASM preview/editor files may have different license terms; read the relevant LICENSE files for commercial redistribution.

## 17. RESEARCH LINKS, VERIFIED CHANGE NOTES, AND NEXT REVIEW

**Checked on 2026-11-13.** These entries are source references, **not assertions that every project has shipped a new stable version on that date**. Always recheck releases and commits before calling anything "latest."

- **Ymir** — [release listing](https://github.com/ymir-emu/Ymir/releases), [CHANGELOG](https://github.com/ymir-emu/Ymir/blob/main/CHANGELOG.md): stable 0.4.1 dated 2026-10-18; 0.4.0 brought D3D12/Metal backend work and SH-2/VDP2 diagnostics; 0.4.2/Vulkan listed as in development at this review. Its built-in debugger is a material omission in the original §9.
- **Jo Engine** — [online API docs](https://jo-engine.org/doxygen/), [tagged releases](https://github.com/johannes-fetz/joengine/releases): docs identify 2026.11.2 and claim direct hardware without Sega libraries; last tagged stable release on checked listing: v2025.1. **Do not conflate docs with stable binaries or assume every checkout is SGL-free.**
- **Yaul** — [main SDK repository](https://github.com/yaul-org/libyaul), [GDB stub](https://github.com/yaul-org/libyaul-gdbstub): `libmic3d`/other sublibraries and practical SH-2 GDB stub provide alternatives to unnecessary reinvention; inspect exact code/version for interfaces.
- **Audio** — [ponèSound](https://github.com/ponut64/SCSP_poneSound) README describes PCM, software ADX decoding, streamed samples, limitations, and CD-DA drive contention. [Tethys](https://github.com/Teuthida255/Tethys) is experimental music/instrument research. [mid2seq](https://github.com/jfsantos/mid2seq) provides offline SEQ/TON tools but explicitly describes an SGL-driver playback pipeline and separately licensed WASM portions.
- **Cartridge/deployment/save** — [SAROO](https://github.com/tpunix/SAROO) documents SD loading and 1/4 MB RAM via cartridge-port hardware. [Save Game Copier releases](https://github.com/slinga-homebrew/Save-Game-Copier/releases) include May 2026 experimental modem/serial features and May 14 3.7.1 Satiator fix (pre-release).
- **Compatibility caveats** — [SaturnRingLib README](https://github.com/ReyeMe/SaturnRingLib) explicitly states SGL wrapper. [R11/saturn-libs](https://github.com/R11/saturn-libs) demonstrates experimental NetLink/host tests but documents SGL target builds, unsuitable as a clean-room drop-in.

### 2026-11-13 change summary

1. Updated §2 SDK classification, Jo version/release caveat, and SRL/SGL compliance warning; added Yaul auxiliary libraries.
2. Corrected §3 VDP2 "free" oversimplification, added bank/cycle scheduling, and pointed to existing 3D libraries.
3. Expanded §6 with ponèSound PCM/ADX streaming, CD-DA data-access tradeoffs, experimental instruments, and license cautions.
4. Updated §7 for SAROO, Save Game Copier, experimental NetLink/serial, and loader/firmware distinctions.
5. Corrected §8's assumptions about universal SDK Make targets, path names, and environment variables.
6. Expanded §9 to include Ymir's 2026 debugger, community Yaul GDB stub, and cross-emulator + hardware evidence.
7. Updated failure modes, anti-patterns, decision tree, and quick reference to match the above.
8. Resolved §14's **PARTIAL-vs-binary** validation contradiction, preserved the mandatory stop, and added explicit build/emulator/hardware evidence labels.
9. Added §§15–17 as a version-aware project reference, code-verification protocol, and durable links for future refreshes.
