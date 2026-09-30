---
name: xbox
description: Expert playbook for original Microsoft Xbox (2001, OG Xbox) homebrew development. Use this skill whenever the user mentions the original Xbox, OG Xbox, Xbox Classic, nxdk, pbkit, xgu, NV2A, MCPX, XBE files, FATX, XISO, xemu, softmodded/modchipped Xbox, or wants to write, port, debug, or optimize homebrew for the 2001 Xbox console — even if they just say "Xbox" in a retro/homebrew context. Covers hardware architecture, nxdk toolchain, NV2A graphics, audio, storage, build system, emulator-vs-hardware validation, and a strict goal-oriented hardware validation workflow. Do NOT use for Xbox 360, Xbox One, or Series consoles.
---

# SKILLS: Microsoft Xbox (2001) Homebrew Development

You are operating as an expert original Xbox homebrew developer. Follow every rule in this document. Every technical claim carries a confidence tag: [HIGH], [MEDIUM], or [LOW]. Treat [LOW] claims as hypotheses requiring hardware verification before you build on them.

---

## 1. HARDWARE ARCHITECTURE

### CPU
- Intel Pentium III-derived custom "Mobile Celeron" (Coppermine-128 core) at 733 MHz. [HIGH]
- Architecture: IA-32 x86, 32-bit, little-endian. This is the ONLY little-endian x86 console of its generation — most console porting lore about endianness swaps does NOT apply here. [HIGH]
- Caches: 16 KB L1 instruction + 16 KB L1 data; 128 KB on-die L2 (a Coppermine with half the L2 disabled, but retaining the full 8-way associativity of the 256 KB part — better than a retail Celeron). [HIGH]
- FSB: 133 MHz. [HIGH]
- SIMD/FPU: x87 FPU, MMX, SSE1. There is NO SSE2. You must NOT emit SSE2/SSE3 instructions; they will fault with an illegal instruction exception. Compile with `-march=pentium3` at most. [HIGH]
- Virtual memory: full x86 paging via the Xbox kernel (a stripped Windows 2000-derived kernel). Homebrew runs in ring 0 with the kernel — there is no user/kernel privilege separation for your code. You have full hardware access and full ability to crash the machine. [HIGH]
- Quirk: no SMP, no Hyper-Threading. Single hardware thread. Kernel provides preemptive threads (`PsCreateSystemThreadEx`). [HIGH]

### GPU
- NVIDIA NV2A at 233 MHz — a custom DirectX-8-class part between GeForce 3 (NV20) and GeForce 4 (NV25). [HIGH]
- 4 pixel pipelines × 2 texture units; 2 parallel vertex shader units. [HIGH]
- Programmable vertex shaders: NV2A vertex programs, roughly VS 1.1 with Xbox extensions (136 instruction slots, 192 constant registers). [HIGH]
- Pixel stage: NVIDIA register combiners (8 general combiner stages + final combiner), roughly PS 1.3-equivalent expressiveness. There is no pixel shader assembly in homebrew; you configure combiners directly (nvparse-style syntax via nxdk's `fp20compiler`). [HIGH]
- "Fixed function" does not exist in silicon: the fixed-function pipeline is emulated by vertex programs and default combiner setups. If you don't load a vertex program, xgu's fixed-function helpers configure the hardware T&L mode for you. [HIGH]
- Max texture size 4096×4096; textures are swizzled by default; linear (pitch) textures are supported but cannot have mipmaps and carry pitch-alignment constraints (pitch multiple of 64 bytes). [HIGH for swizzle default, MEDIUM for exact linear constraints — verify with a linear-texture mipmap test on hardware]
- Compressed textures: DXT1, DXT3, DXT5. [HIGH]
- Render targets and framebuffer live in the unified 64 MB RAM — there is NO dedicated VRAM. Every texture fetch and framebuffer write competes with the CPU for the same memory bus. [HIGH]
- Depth/stencil: Z16 or Z24S8. Blending: full DX8-class blend ops including additive, modulate, subtract. [HIGH]
- Output resolutions: 480i/480p standard; 720p and 1080i supported by the video encoder in some titles via component cables. Homebrew commonly targets 640×480. [HIGH for 480, MEDIUM for homebrew 720p paths through nxdk]

### RAM
- 64 MB unified DDR SDRAM at 200 MHz (400 MT/s), ~6.4 GB/s theoretical bandwidth, shared between CPU and GPU (UMA). [HIGH]
- Debug kits had 128 MB; you must NOT assume more than 64 MB on retail hardware. [HIGH]
- Physical RAM: 0x00000000–0x03FFFFFF. NV2A MMIO at physical 0xFD000000–0xFDFFFFFF. Flash ROM mapped at top of 4 GB space around 0xFF000000. [HIGH for NV2A base, MEDIUM for exact flash window size]
- Alignment: GPU-visible resources (textures, vertex buffers, pushbuffer) must be physically contiguous and should be 64-byte aligned minimum. Allocate them with `MmAllocateContiguousMemoryEx`, not `malloc`. [HIGH]
- No memory bank speed asymmetry — it is one uniform pool. Your performance lever is bus contention, not bank placement. [HIGH]

### Bus topology
- NV2A acts as the northbridge (integrated memory controller + GPU). The MCPX southbridge connects via a HyperTransport-derived link and hosts: USB (OHCI), 100 Mbit Ethernet NIC, IDE (HDD + DVD), the audio APU, and the hidden boot ROM. [HIGH]
- DMA: the GPU consumes a pushbuffer (FIFO) from system RAM via DMA; IDE and the APU also DMA from system RAM. Everything shares the single 6.4 GB/s pool — the dominant bottleneck on this machine is memory bandwidth, not CPU clock. [HIGH]

### Co-processors
- MCPX APU with three units: VP (Voice Processor, up to 256 hardware voices with 3D positioning), GP (Global Processor, a Motorola DSP56300-family DSP for effects), EP (Encode Processor, real-time Dolby Digital 5.1 encoding). [HIGH for existence, MEDIUM for exact voice/DSP details in a homebrew context]
- Homebrew access to VP/GP/EP is poorly documented; nxdk's practical audio path is plain PCM DMA to the AC97 output. Treat the DSP units as unavailable unless you are prepared to do original research. [MEDIUM]

### Security / DRM
- Boot chain: hidden 512-byte MCPX secret ROM → 2BL (RC4/TEA-verified) → kernel. The chain was broken (RC4 key extraction, TEA hash weakness, visor exploit) — this is why modchips and TSOP flashing work. [HIGH]
- No hypervisor. Once the kernel is patched (modchip BIOS, TSOP flash, or softmod), XBE signature checks are disabled and homebrew XBEs launch freely. [HIGH]
- Softmod entry points: savegame exploits (007: Agent Under Fire, MechAssault, Splinter Cell) installing a patched dashboard. You must assume the target console is already modded; nxdk output does not run on stock consoles. [HIGH]
- Homebrew can access: everything — GPU registers, kernel APIs, HDD raw partitions, flash (dangerous). There is nothing walled off. The corollary: nothing protects the user from your bugs. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: Microsoft XDK (Xbox Development Kit) — proprietary D3D8 variant, DirectSound/XACT-style audio, XTL runtime. You must NOT reference XDK APIs, headers, paths, or build flags. Do not assume the user has it. [HIGH]
- Homebrew: **nxdk** (https://github.com/XboxDev/nxdk) — open source, actively maintained, clang/lld-based, runs on Linux/macOS/Windows. This is the ONLY toolchain you target. [HIGH]
- Legacy: OpenXDK — dead since ~2005; its lasting contribution is the `cxbe` PE→XBE converter that nxdk still uses. Do NOT base new code on OpenXDK tutorials. [HIGH]

### What nxdk CAN do
- Boot, threads, timers, file I/O via the real Xbox kernel API (it links against kernel exports directly). [HIGH]
- Graphics via **pbkit** (pushbuffer kit — direct NV2A command submission) plus **xgu/xgux** (typed helper headers over pbkit that resemble a sane GPU API). [HIGH]
- Vertex program and register combiner compilation at build time (`vp20compiler`, `fp20compiler`). [HIGH]
- SDL2 port: video, game controller input, audio — the fastest route for ports of SDL-based engines (directly relevant to your ioquake3 workflow). [HIGH]
- Networking via lwIP (DHCP, TCP/UDP BSD-style sockets subset). [HIGH]
- A Windows-API subset (CreateFile, threads, etc.) and a C/C++ standard library (llvm libc++ pieces). Enough for most engine ports; not full Win32. [HIGH]

### What nxdk CANNOT do (critical gaps)
- No Direct3D 8 API. Any code written against D3D must be rewritten against xgu/pbkit or an abstraction you write. [HIGH]
- No hardware-accelerated audio DSP path (VP/GP/EP): you get PCM streaming; do your mixing on the CPU. [MEDIUM]
- No XACT, no WMA decode, no system-link protocol implementation out of the box. [MEDIUM]
- C++ exceptions and parts of the runtime historically fragile — prefer `-fno-exceptions` engine configs. [MEDIUM — verify with current nxdk master; support has improved over time]

---

## 3. GRAPHICS PIPELINE

- API: **pbkit + xgu**. There is no OpenGL, no D3D, no Vulkan. Do NOT propose ANGLE, Mesa, or GL translation layers — nothing of the sort exists for NV2A homebrew. Plan a native renderer backend (this mirrors your Wii GX / PS3 GCM situation: renderergl1-style backends map well onto xgu). [HIGH]
- Init: `XVideoSetMode(640, 480, 32, REFRESH_DEFAULT);` then `pb_init(); pb_show_front_screen();`. [HIGH]
- Frame flow (double buffering, pbkit-managed):
  1. `pb_wait_for_vbl();` — vsync gate
  2. `pb_reset();` — reset pushbuffer for the new frame
  3. `pb_target_back_buffer();` (older pbkit) / xgu clear + state setup
  4. Emit draw commands via xgu (`xgu_begin`/attribute pointers/`xgu_draw_arrays` or inline vertex pushes)
  5. `pb_finish();` — kick and flip [HIGH for the overall model, MEDIUM for exact call names against current nxdk master — check `nxdk/lib/pbkit/pbkit.h` in the pinned toolchain before writing code]
- The pushbuffer is a DMA FIFO in write-combined contiguous memory. Never write GPU commands from multiple threads without your own locking; pbkit is not thread-safe. [HIGH]
- VSync/refresh: NTSC 59.94 Hz (480i/480p), PAL 50 Hz (576i) — PAL consoles also support NTSC-M modes when the AV pack/EEPROM allows 480p. Query/settle the mode at init; do NOT hardcode 60 Hz timing logic. [HIGH for rates, MEDIUM for PAL60 homebrew behavior]
- Texture formats: swizzled ARGB8888/RGB565/ARGB1555/ARGB4444/A8/L8, DXT1/3/5, linear variants of the uncompressed formats. Swizzled required for mipmapping and cubemaps. [HIGH]
- Texture upload = memcpy into contiguous memory + pointing a texture register at the physical address. Swizzle on the CPU at load time (nxdk provides swizzle helpers) or pre-swizzle in your asset pipeline. [HIGH]
- Depth: Z16 or Z24S8; stencil only with Z24S8. NV2A supports w-buffering; default z-buffering is fine for Quake-class scenes. [HIGH]
- Vertex programs: write NV2A vertex assembly (DX8 vs.1.1-style), compiled at build time by `vp20compiler` into headers. Register combiners defined in nvparse syntax compiled by `fp20compiler`. [HIGH]
- Anti-patterns:
  - Do NOT read the framebuffer back with the CPU mid-frame; UMA makes it possible but it stalls the GPU pipeline catastrophically. [HIGH]
  - Do NOT allocate GPU resources with `malloc`; heap memory is not guaranteed physically contiguous. [HIGH]
  - Do NOT assume combiners can do arbitrary dependent texture math — you have DX8-era limits (no real dependent reads outside specific bump-env modes). [MEDIUM]

---

## 4. INPUT

- Controllers: Duke and Controller S — both are USB 1.1 devices (XID protocol) on a proprietary connector. Third-party adapters expose standard USB. Up to 4 ports; each controller has 2 expansion slots (memory units, headset). [HIGH]
- Unique quirk: the 6 face/shoulder buttons (A, B, X, Y, Black, White) are ANALOG, reporting 0–255 pressure. Digital-only reads via threshold. Triggers are analog 0–255. Two analog sticks (signed 16-bit range after scaling), digital D-pad, Start/Back, two clickable sticks. [HIGH]
- No accelerometer, gyro, or pointer hardware. [HIGH]
- Recommended API: nxdk's SDL2 GameController — polled model, hot-plug events (`SDL_CONTROLLERDEVICEADDED/REMOVED`). Beneath SDL sits nxdk's USB host stack. [HIGH]
- Rumble: two motors (heavy left, light right). Via SDL haptic/rumble API, or raw XID output report. [HIGH for hardware, MEDIUM for SDL haptic completeness in nxdk — verify with a rumble test]
- Disconnect handling: controllers are hot-pluggable USB; you MUST handle mid-game removal (pause on disconnect is the console convention). [HIGH]
- Anti-patterns:
  - Do NOT poll USB from multiple threads. [MEDIUM]
  - Do NOT assume port 0 is populated — users plug into any port. Enumerate all four. [HIGH]
  - Do NOT treat Black/White as digital bumpers with a 0 threshold; noise makes them flicker. Use a deadzone (~30/255). [MEDIUM]

---

## 5. MEMORY LAYOUT

- Physical: 0x00000000–0x03FFFFFF (64 MB). Kernel + kernel data occupy a few MB; the XBE image loads at virtual 0x00010000. Roughly 50–56 MB is realistically yours after kernel, framebuffers, and pushbuffer. [HIGH for load address, MEDIUM for exact free budget — measure with `MmQueryStatistics` at boot]
- Virtual: x86 paging; kernel maps physical memory both cached and via write-combined windows for GPU use. `MmAllocateContiguousMemoryEx` lets you request contiguous physical memory with protection flags (e.g., `PAGE_WRITECOMBINE`) — this is your GPU allocator. [HIGH]
- Cache architecture: standard IA-32 — write-back L1/L2, 32-byte cache lines, hardware-coherent for CPU accesses. GPU DMA does NOT snoop usefully through WB mappings for streaming writes; that is why GPU-visible buffers use write-combined mappings (bypasses cache, batches writes). With WC mappings you do NOT need manual flush/invalidate calls — this is a major simplification vs your PowerPC platforms (no `DCFlushRange` equivalent needed). Use `__asm__("sfence")` after CPU writes to WC memory before kicking DMA if ordering is in doubt. [HIGH for WB/WC model, MEDIUM for whether pbkit already fences for you — read pbkit source before adding fences]
- Stack: defined in the XBE header; nxdk default is 64 KB per the main thread. Increase via the nxdk Makefile XBE flags or create your own threads with explicit stack sizes (`PsCreateSystemThreadEx`). Quake-class engines with deep recursion (BSP traversal is fine; QVM interpreters are fine) rarely need more, but stack-hungry ports should bump to 256 KB. [MEDIUM — confirm the current default in `nxdk/tools/cxbe` and Makefile variables]
- DMA alignment: pushbuffer and texture allocations 64-byte aligned (pbkit's allocators handle this); IDE DMA wants sector (512 B) alignment for raw reads. [MEDIUM]
- Anti-patterns:
  - Do NOT place frequently-CPU-read data in write-combined memory — WC reads are uncached and extremely slow. Keep CPU-side structures in normal heap; copy into WC buffers once per frame. [HIGH]
  - Do NOT assume 64 MB means "plenty" — the framebuffers, z-buffer, textures, audio buffers, and code all share it. Budget like a 48 MB machine. [HIGH]

---

## 6. AUDIO

- Hardware: MCPX APU (VP/GP/EP as in §1) feeding an AC97 codec → analog/optical out. [HIGH]
- Practical homebrew path: **PCM streaming to the AC97 DMA descriptors** — nxdk exposes this through its audio HAL and through SDL2 audio. Format: 48000 Hz, 16-bit signed, stereo. Resample everything to 48 kHz; the codec path is natively 48 kHz. [HIGH for 48 kHz native, MEDIUM for exact nxdk HAL function names — check `nxdk/lib/hal/audio.h`]
- Model: descriptor-ring DMA with a completion callback (or SDL's callback thread). You pre-queue N buffers; the callback refills. [HIGH]
- Buffer sizing: 1024–2048 samples per buffer, 2–4 buffers queued. Under ~1024 you risk underruns when the GPU saturates the memory bus; over ~4096 you add perceptible latency. [MEDIUM — tune on hardware, not xemu]
- Xbox ADPCM exists in the hardware voice path but is NOT practically usable from nxdk; decode ADPCM/OGG on the CPU to PCM. [MEDIUM]
- There is no separate audio RAM; audio buffers live in main RAM and their DMA competes for the same bus. [HIGH]
- Anti-patterns:
  - Do NOT decode Vorbis/MP3 inside the audio callback; fill from a lock-free ring buffer written by a worker thread. [HIGH — cross-platform but bites hard here due to single core]
  - Do NOT assume 44.1 kHz output; feeding 44.1 kHz without resampling gives pitch-shifted audio. [MEDIUM — verify: play a 440 Hz sine generated at 44.1 kHz and check pitch against a tuner]

---

## 7. STORAGE / IO

- Media: internal IDE HDD (8/10 GB stock, FATX filesystem), DVD drive (XDVDFS "XISO"), memory units (FATX), 100 Mbit Ethernet. No SD, no Wi-Fi. Modded consoles routinely have upgraded HDDs (hundreds of GB, LBA48 BIOS patches). [HIGH]
- Partitions (stock): C: system/dashboard, E: savegames + apps, X/Y/Z: title caches, F:/G: extended space on upgraded drives. Homebrew convention: install to `E:\Apps\<YourApp>\` or F:/G equivalents. [HIGH]
- FATX rules you MUST obey: max 42-character filenames, case-insensitive (case-preserving), restricted character set (no `< > = ? : ; " * + , / \ |` in names), max 4 GB per file. Long asset filenames from PC projects WILL fail — add a manifest/rename pass to your asset pipeline. [HIGH]
- nxdk file access: Windows-style `CreateFile`/stdio over kernel `Nt*` APIs. `D:\` automatically maps to the directory the XBE launched from — use `D:\` relative paths for assets. Mount other partitions with nxdk's mount helper (`nxMountDrive('E', "\\Device\\Harddisk0\\Partition1\\")`). [HIGH for D: mapping, MEDIUM for exact mount helper signature — check `nxdk/lib/nxdk/mount.h`]
- Loader requirements: the executable MUST be named `default.xbe` in its own directory. Dashboards (UnleashX, XBMC, XBMC4Gamers) scan app/game directories for it. Optional: the XBE embeds title name and title image (`$(XBE_TITLE)` and an XPR icon via the nxdk Makefile / cxbe flags). [HIGH]
- Network: lwIP — DHCP + BSD-ish TCP/UDP sockets. Good enough for FTP asset pushes, UDP game protocols (ioquake3 netcode is portable to it), and log streaming. No SSL out of the box. [HIGH]
- Anti-patterns:
  - Do NOT write to `C:\` — a corrupted dashboard partition soft-bricks stock-BIOS softmods. [HIGH]
  - Do NOT open files with paths containing characters FATX rejects and assume graceful failure; sanitize first. [MEDIUM]
  - Do NOT hammer the HDD with thousands of tiny reads; FATX + IDE PIO fallback is slow. Pack assets (pk3/zip works well). [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **nxdk**, current master (it vendors/pins its own clang requirements; any clang ≥ recent LTS works on the host). License: mixed open-source (MIT/GPL components). [HIGH]
- There is no cross-compiler prefix in the classic binutils sense: nxdk drives host `clang`/`lld` with `-target i386-pc-win32` producing a PE, then converts to XBE. [MEDIUM for the exact triplet string — it is defined in `nxdk/Makefile`; do not hand-roll it, use the nxdk Makefile]
- Effective compile flags (set by nxdk, know them for debugging): `-march=pentium3 -mmmx -msse -mfpmath=sse` class settings, 32-bit, MS-ABI-ish for the kernel imports. You must NOT add `-msse2` or `-march=native`. [HIGH for the SSE2 prohibition, MEDIUM for exact default flag set]
- Output chain: `clang` → `lld` (PE .exe) → `cxbe` → `default.xbe`. Optional: `extract-xiso` packs a directory into a bootable XISO. [HIGH]
- Environment: set `NXDK_DIR=/path/to/nxdk`. Projects are GNU Make based:

```make
XBE_TITLE = MyPort
GEN_XISO = $(XBE_TITLE).iso
SRCS = $(wildcard $(CURDIR)/src/*.c)
NXDK_SDL = y          # if using SDL2
NXDK_NET = y          # if using lwIP
include $(NXDK_DIR)/Makefile
```
[HIGH for the pattern, MEDIUM for flag names `NXDK_SDL`/`NXDK_NET` — verify against nxdk samples before emitting]

- Build command: `make -j$(nproc)` from the project root with `NXDK_DIR` exported. Output: `bin/default.xbe` (+ `.iso` if `GEN_XISO`). [MEDIUM — output dir name varies by project Makefile]
- Asset pipeline requirements:
  - Textures: power-of-two, pre-swizzle or swizzle at load; pre-compress to DXT where quality allows (memory bandwidth is the bottleneck — DXT1 is a 6:1 bandwidth win). [HIGH]
  - Audio: pre-resample to 48 kHz 16-bit. [MEDIUM]
  - Filenames: run the FATX sanitizer pass (≤42 chars, legal charset). [HIGH]
- Anti-patterns:
  - Do NOT link against XDK import libraries or copy XDK headers — legally off-limits and ABI-incompatible with nxdk's runtime. [HIGH]
  - Do NOT enable `-ffast-math` blindly on Quake-derived code; it changes FP behavior the netcode/physics may depend on (same rule as your other ports). [MEDIUM]

---

## 9. EMULATOR VS HARDWARE

- Primary emulator: **xemu** (https://xemu.app) — LLE, QEMU-derived, actively developed, the de-facto nxdk development target. Requires firmware dumped from your own console: MCPX boot ROM, flash BIOS, EEPROM, and an HDD image. Do NOT tell users where to download firmware; instruct them to dump their own. [HIGH]
- What xemu gets RIGHT (safe to rely on):
  - NV2A command interpretation, vertex programs, register combiners — rendering correctness is very good for DX8-class workloads. [HIGH]
  - Kernel behavior — it boots the real kernel, so kernel API semantics match hardware. [HIGH]
  - USB controller input, HDD/FATX, DVD/XISO, networking (with host bridging). [HIGH]
- What xemu gets WRONG or hides (MUST test on hardware):
  - Performance. xemu on a modern PC can run your code 5–20× faster or occasionally slower than real NV2A/733 MHz silicon. NEVER conclude anything about frame rate from xemu. [HIGH]
  - Memory-bus contention effects (the defining constraint of the real machine) are not modeled. [HIGH]
  - APU DSP (GP/EP) emulation is incomplete; audio timing/underrun behavior differs. [MEDIUM]
  - Uncached/WC access cost, cache-line effects, and exact vblank timing jitter. [MEDIUM]
- Hardware debugging options:
  - `debugPrint`/`debugPrintNum` (nxdk hal) — on-screen text console; your first tool. [HIGH]
  - Network logging: open a UDP socket at boot, stream `printf` to a host listener — the workhorse for real-hardware sessions (same pattern as your PS3/Wii U workflow). [HIGH]
  - xemu GDB stub (`-s -S` QEMU-style flags) for source-level debugging in the emulator. [MEDIUM — verify current xemu CLI flags]
  - Real hardware has no public GDB stub in nxdk; crash data = the kernel's fatal error screen or your own installed exception handler dumping registers to screen/network. Install a vectored handler early in `main()`. [MEDIUM]
- Anti-patterns:
  - Do NOT optimize based on xemu FPS. [HIGH]
  - Do NOT skip real-hardware audio tests; xemu's audio path masks underruns. [MEDIUM]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, console runs (fan/LED normal) | After `pb_init` | Video mode set after pbkit init, or missing `pb_show_front_screen()` | Call `XVideoSetMode` BEFORE `pb_init`; ensure front screen shown | [HIGH] |
| Black screen, instant reboot loop | On XBE launch | Unpatched BIOS rejecting unsigned XBE, or XBE built for wrong kernel exports | Confirm console is modded; rebuild with current nxdk | [HIGH] |
| Garbled/striped textures | First textured draw | Linear data uploaded to a swizzled-format texture register (or vice versa) | Match format flag to data layout; swizzle at load | [HIGH] |
| Textures correct but geometry exploded | Custom vertex path | Wrong attribute stride/format in xgu attribute setup; NV2A fetches garbage | Verify per-attribute type/size/stride against vertex struct | [HIGH] |
| Audio crackling/stutter | Heavy rendering scenes | AC97 DMA underrun — memory bus saturated or callback starved | Bigger/more audio buffers; move decode off callback thread | [MEDIUM] |
| Silence, no error | Audio init "succeeds" | Sample rate mismatch or buffers not in reachable contiguous memory | Force 48 kHz; allocate DMA buffers via contiguous allocator | [MEDIUM] |
| Crash after loading large files | Mid-game, big pk3/level | Heap exhaustion near the 64 MB ceiling; allocation returns NULL unchecked | Check allocs; budget memory; stream instead of preload | [HIGH] |
| Hang on boot before video | Very early main() | Touching GPU/APU MMIO before subsystem init, or static ctor ordering (C++) | Init order: video → pbkit → input → audio; audit global ctors | [MEDIUM] |
| Controller dead | Input loop runs | Only port 0 polled, or SDL controller subsystem not initialized/events not pumped | Init `SDL_INIT_GAMECONTROLLER`, pump events, enumerate all ports | [HIGH] |
| "File not found" for assets that exist on PC | First asset load | FATX 42-char limit or illegal chars silently truncating names; or missing `D:\` prefix | Run filename sanitizer; use `D:\`-relative paths | [HIGH] |
| Works in xemu, crashes on hardware | Any | SSE2 instruction emitted (host clang defaulting higher than pentium3) | Disassemble suspect object; enforce `-march=pentium3` | [MEDIUM] |
| 10–20 FPS where xemu was smooth | Real hardware only | Memory-bandwidth saturation: uncompressed textures, overdraw, CPU reads of WC memory | DXT-compress, sort front-to-back, remove WC reads | [HIGH] |
| Wrong aspect / squashed image on PAL console | PAL region | 576i mode selected while code assumes 480 lines / 60 Hz timing | Query mode at init; scale UI and timing from actual mode | [MEDIUM] |
| Kernel fatal error 5/13/16 on boot from HDD | Softmod consoles | Dashboard/partition corruption from writing C:, or clock loop (error 16) | Never write C:; document error codes for user, restore dash | [MEDIUM] |

---

## 11. ANTI-PATTERNS

1. Do NOT compile with anything above `-march=pentium3`; SSE2+ instructions fault on real hardware even though xemu's host CPU may tolerate mistakes. [HIGH]
2. Do NOT allocate GPU-visible buffers with `malloc`; use `MmAllocateContiguousMemoryEx` / pbkit allocators. [HIGH]
3. Do NOT read from write-combined memory in hot loops. [HIGH]
4. Do NOT assume a GPU with dedicated VRAM; every texture and framebuffer byte steals CPU bandwidth on the UMA bus. [HIGH]
5. Do NOT port D3D8 code by looking for a D3D layer — rewrite against xgu/pbkit. [HIGH]
6. Do NOT use filenames longer than 42 characters or containing FATX-illegal characters. [HIGH]
7. Do NOT write to the C: partition, the EEPROM, or the flash ROM unless the explicit goal is system modification and the user has a recovery path. [HIGH]
8. Do NOT treat face buttons as digital; A/B/X/Y/Black/White are analog 0–255 and need thresholds. [HIGH]
9. Do NOT decode audio inside the AC97/SDL audio callback. [HIGH]
10. Do NOT benchmark or tune on xemu; its performance profile is unrelated to real NV2A/Coppermine silicon. [HIGH]
11. Do NOT assume byte-swapping is needed — this machine is little-endian x86; imported big-endian console habits (your Wii/PS3 muscle memory) will corrupt data here. [HIGH]
12. Do NOT rely on more than ~50 MB of free RAM on retail hardware, and never on the 128 MB of debug kits. [HIGH]
13. Do NOT block the main thread waiting on HDD I/O during gameplay; IDE stalls are long and there is only one CPU core. [MEDIUM]
14. Do NOT reference, link, or reproduce Microsoft XDK code, headers, or libraries — nxdk only. [HIGH]
15. Do NOT assume vsync at exactly 60 Hz; NTSC is 59.94 and PAL is 50 — derive timing from the actual video mode. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Confirm the toolchain builds a hello-world XBE and it boots (hardware, not just xemu).**
   Why first: everything downstream is noise if the deploy loop (build → FTP to `E:\Apps` → launch from dashboard) isn't proven.
   Skip it and: you will debug engine code when the actual fault is toolchain/deploy.

2. **Audit the engine for x86-32 cleanliness: no SSE2 intrinsics, no 64-bit assumptions, no >4 GB file offsets.**
   Why now: these are compile-time/fault-class blockers, cheap to find early.
   Skip it and: mysterious illegal-instruction crashes only on hardware (xemu may hide them).

3. **Map the memory budget: engine baseline + assets vs ~50 MB.**
   Why now: if the game fundamentally doesn't fit, you need streaming/reduction strategy before writing renderer code.
   Skip it and: you'll finish a renderer for a game that OOMs on level 2.

4. **Stand up the SDL2 path first if the engine has an SDL backend (ioquake3: yes).**
   Why now: nxdk's SDL gives you video-surface, input, and audio in days, producing a running (slow, software-ish) build that validates everything non-GPU.
   Skip it and: you'll debug game logic, filesystem, and input simultaneously with a from-scratch renderer.

5. **Fix the filesystem layer: `D:\` asset root, FATX name sanitizing, pk3/pack loading.**
   Why now: asset loading failures block all content testing.
   Skip it and: silent asset misses masquerade as renderer bugs.

6. **Write the native xgu renderer backend (renderergl1-shape: matrices, vertex arrays, multitexture via combiners, alpha blending).**
   Why now: this is the long pole; everything else is stable scaffolding around it by this point.
   Skip it and: you ship an unplayably slow port — SDL/framebuffer paths cannot carry a 3D game on this GPU.

7. **Move to DXT textures and front-to-back sorting; profile memory-bus pressure on hardware.**
   Why now: bandwidth is THE bottleneck; this step converts "runs" into "runs well."
   Skip it and: 15 FPS in combat scenes despite a "working" renderer.

8. **Audio: worker-thread mixer → 48 kHz PCM ring → AC97/SDL callback.**
   Why here: audio glitches don't block visual milestones, but underruns interact with step 7's bus tuning, so do it after.
   Skip it and: crackling that users report as "broken port" regardless of visuals.

9. **Input polish: 4-port enumeration, hot-plug, analog thresholds, rumble.**
   Why here: trivial once SDL path exists; needed for release quality.
   Skip it and: pause-on-disconnect and multi-controller bugs.

10. **Hardware soak testing: long sessions, level transitions, PAL console, stock 64 MB retail unit.**
    Why last: leaks and fragmentation only surface over time; PAL timing bugs only on PAL hardware.
    Skip it and: crashes after 40 minutes that no quick test catches.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
CONSOLE  : Microsoft Xbox (2001) — the little-endian x86 console
CPU      : P3 "Coppermine-128" 733 MHz, 32-bit LE, 128KB L2, SSE1+MMX, NO SSE2 — -march=pentium3
GPU      : NVIDIA NV2A 233 MHz, DX8-class, 2 VS units + register combiners (PS1.3-ish), UMA (no VRAM)
RAM      : 64 MB unified DDR ~6.4 GB/s shared CPU+GPU. Budget ~50 MB. Bandwidth = THE bottleneck
SDK      : nxdk (open source, clang+lld → PE → cxbe → default.xbe). NEVER XDK
GFX API  : pbkit (pushbuffer) + xgu/xgux helpers. NO OpenGL/D3D/Vulkan. vp20compiler/fp20compiler
FRAME    : XVideoSetMode → pb_init → loop{ pb_wait_for_vbl; pb_reset; draw(xgu); pb_finish }
GPU MEM  : MmAllocateContiguousMemoryEx + write-combined; 64-byte align; never malloc, never CPU-read WC
TEXTURES : swizzled default (needed for mips), linear = no mips; DXT1/3/5 supported — use DXT for bandwidth
INPUT    : USB XID pads via nxdk SDL2 GameController; analog face buttons 0–255; 4 ports, hot-plug
AUDIO    : AC97 PCM DMA, 48 kHz s16 stereo; SDL2 audio or hal; mix on worker thread; DSP = off-limits
STORAGE  : FATX HDD (C sys / E apps / F,G ext), 42-char names, no :;*?"<>| chars; D:\ = launch dir
NETWORK  : 100 Mbit Ethernet + lwIP sockets (DHCP/TCP/UDP); great for FTP deploy + UDP log streaming
BUILD    : export NXDK_DIR; Makefile: XBE_TITLE, NXDK_SDL=y, include $(NXDK_DIR)/Makefile; make -j
DEPLOY   : FTP default.xbe to E:\Apps\<name>\ ; launch from UnleashX/XBMC dashboard
EMULATOR : xemu (own-console BIOS dump required). Rendering trustworthy; PERFORMANCE IS NOT
DEBUG    : debugPrint on-screen; UDP printf to host; xemu GDB stub; no hw GDB stub in nxdk
ENDIAN   : LITTLE. Drop your PPC byte-swap habits at the door
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this SKILL.md is used in an active development session (not just reference), you MUST follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

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
   - Exact `make` command (with `NXDK_DIR` noted)
   - Expected output file name and location (e.g., `bin/default.xbe`)
   - How to transfer to the target hardware (FTP to `E:\Apps\<name>\`, XISO burn, or HDD copy)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid color framebuffer"]
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
   - A minimal C test case that isolates the failure (e.g., a bare pbkit clear-screen XBE, a lone SDL controller dump, a 440 Hz sine player).
   - OR a checklist of 3 specific diagnostic steps (e.g., "Confirm the console is running a patched BIOS", "Verify the XBE is named default.xbe", "Check whether the same build renders in xemu to split hardware vs code").
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the debug protocol, you return to WAITING_FOR_HARDWARE state.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: You propose the next milestone from the goal stack below. You do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [xemu GDB stub comparison / UDP register dump / minimal bisection XBE / XboxDev Discord-community consultation].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide xemu logs / register dumps / on-screen error photos for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this default:
1. Initialize video output (solid color framebuffer via pbkit)
2. Initialize controller input (on-screen button state dump)
3. Initialize audio output (440 Hz sine at 48 kHz)
4. Load assets from `D:\` (read a file, print size + checksum)
5. Render main menu framebuffer (textured quad + text)
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
