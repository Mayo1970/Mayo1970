---
name: ps2
description: PlayStation 2 homebrew development — Emotion Engine/VU/GS architecture, gsKit graphics, input, memory, audio, build system, debugging, and hardware validation gates.
---

# SKILLS_PS2.md  PlayStation 2 Homebrew Expertise Document

You are assisting with homebrew development for the Sony PlayStation 2. Follow every rule in this document. When this document conflicts with your general training data about PC/mobile development, this document wins. All code must compile with the open-source **ps2sdk** toolchain from the ps2dev organization. You must NOT reference, assume, or emit code for any leaked Sony SDK (SCE libs, `libgraph`, `sce*` headers, `.cpp2` samples, etc.).

---

## 1. HARDWARE ARCHITECTURE

### CPU  Emotion Engine (EE)
- Core: custom MIPS R5900, MIPS III ISA plus 107 proprietary 128-bit MultiMedia Instructions (MMI), running at **294.912 MHz**. [HIGH]
- **Little-endian**, 64-bit GPRs extended to 128-bit for MMI operations (`lq`/`sq` load/store quadwords). [HIGH]
- Caches: **16 KB 2-way instruction cache, 8 KB 2-way data cache** (write-back), 64-byte cache lines. [HIGH]
- **16 KB on-die scratchpad RAM (SPR)** mapped at `0x70000000`, single-cycle access, DMA-able to/from main RAM via the SPR DMA channels. [HIGH]
- FPU (COP1): single-precision only, **NOT IEEE 754 compliant**  no NaN/Infinity, division by zero returns saturated max instead of Inf, non-standard rounding. [HIGH]
- **`double` is emulated in software** by the compiler and is catastrophically slow. Audit all ported code for accidental doubles (unsuffixed literals like `1.0`, `math.h` functions like `sin()` instead of `sinf()`). [HIGH]
- `lq`/`sq` (128-bit) require 16-byte alignment; unaligned access raises an address-error exception. [HIGH]
- Quirk: R5900 has known errata around short loops and cache behavior; GCC's `-mfix-r5900` workaround is enabled by default in the ps2dev toolchain. Do not disable it. [MEDIUM]

### Vector Units (co-processors, see §Co-processors)
- **VU0**: 4 KB micro memory + 4 KB data memory. Usable in *macro mode* as COP2 (inline instructions from EE code) or *micro mode* (independent programs). [HIGH]
- **VU1**: 16 KB micro memory + 16 KB data memory. Micro mode only. Has a direct path to the GS (`XGKICK`  GIF PATH1). This is the platform's "vertex shader": transform/lighting/clipping on VU1 in parallel with the EE. [HIGH]

### GPU  Graphics Synthesizer (GS)
- Clock: **147.456 MHz**, 16 pixel pipelines, embedded **4 MB eDRAM** with ~48 GB/s internal bandwidth. [HIGH]
- **Entirely fixed-function. No shaders of any kind. No shader model.** Per-pixel work is limited to: Gouraud shading, one texture stage per primitive, alpha blending, alpha test, depth test, destination-alpha test, fogging, dithering, scissor. [HIGH]
- Multi-texturing does not exist; multi-pass rendering with blending is the substitute (fill rate is enormous, so multi-pass is idiomatic). [HIGH]
- Framebuffer, depth buffer, textures, and CLUTs **all share the same 4 MB VRAM**. A 640Ã448Ã32bpp double-buffered setup with a 24-bit Z buffer consumes ~3.4 MB, leaving only a few hundred KB for resident textures  texture streaming per frame over the GIF is the standard technique. [HIGH]
- Max texture size: **1024Ã1024**; texture dimensions are specified as power-of-two log2 values in the `TEX0` register. [HIGH]
- Texture formats: PSMCT32/PSMCT24/PSMCT16/PSMCT16S (direct color), PSMT8/PSMT4 (palettized with CLUT), PSMT8H/4HL/4HH (packed into upper bits of 32-bit pages). **No hardware texture compression (no S3TC/DXT).** Palettized 8-bit + CLUT is the PS2's "compression". [HIGH]
- Depth formats: PSMZ32/24/16/16S. Stencil: **no dedicated stencil buffer**  destination-alpha tricks and the alpha-test unit are used as a 1-bit stencil substitute. [HIGH]
- Output resolutions: practical maximum for 3D is 640Ã448 (NTSC) / 640Ã512 (PAL) interlaced, or 640Ã448 progressive (480p) via component. Higher CRTC modes (1080i) exist but are impractical for 3D due to VRAM. [HIGH]
- Bilinear filtering, mipmapping (up to 6 LODs, addresses set manually in `MIPTBP1/2`), and trilinear are supported. Automatic mipmap generation does not exist  you upload each level. [HIGH]

### RAM
- **32 MB main RDRAM** (two 16 MB channels, peak ~3.2 GB/s), physical range `0x000000000x01FFFFFF`. [HIGH]
- **2 MB IOP RAM** (separate, owned by the I/O processor). [HIGH]
- **2 MB SPU2 sound RAM** (separate, holds ADPCM sample data). [HIGH]
- **4 MB GS VRAM** (separate, not CPU-addressable  data gets there only via GIF DMA transfers). [HIGH]
- No demand-paged virtual memory in practice: homebrew runs with a flat mapping; the TLB exists but ps2sdk sets up identity-style mapping for you. [HIGH]
- Alignment: DMA source/dest addresses and sizes must be **16-byte (qword) aligned**; buffers touched by DMA should be **64-byte aligned** (cache line) to make flush/invalidate safe. Use `__attribute__((aligned(64)))` or `memalign(64, ¦)`. [HIGH]

### Bus topology
- EE  GS: 64-bit dedicated bus at 147 MHz (~1.2 GB/s)  all texture/geometry uploads flow through the **GIF** over this. [HIGH]
- EE  IOP: the **SIF** (Subsystem Interface)  a mailbox/DMA bridge. All I/O (pads, memory cards, USB, disc, network, audio commands) is performed by IOP-side driver modules (IRX) and reached from the EE via **SIF RPC calls**. SIF RPCs have high latency (measured in hundreds of microseconds to milliseconds); never place a blocking RPC in your render loop. [HIGH]
- DMAC: 10 EE DMA channels  VIF0, VIF1, GIF, IPU(from), IPU(to), SIF0, SIF1, SIF2, SPR(from), SPR(to). Supports *normal*, *chain* (linked DMAtags in memory), and *interleave* modes. Chain mode is the idiomatic way to build a frame's worth of GS packets and fire them with one kick. [HIGH]
- GIF has three input paths: **PATH1** (VU1 `XGKICK`, highest priority), **PATH2** (VIF1 DIRECT), **PATH3** (GIF DMA channel, simplest  use this first). [HIGH]

### Co-processors
- **IOP**: a MIPS R3000A-class CPU at 36.864 MHz with 2 MB RAM; it is effectively an embedded PS1 running its own kernel. Drivers are relocatable **IRX modules** loaded at runtime (`SifLoadModule`). Pads, memory cards, USB, HDD, network, CD/DVD, and SPU2 control all live here. [HIGH]
- **SPU2**: 48 hardware ADPCM voices, 2 MB sample RAM, hardware reverb, 48 kHz stereo output. Controlled from the IOP; the EE talks to it via RPC (typically through `audsrv`). [HIGH]
- **IPU**: Image Processing Unit  an MPEG-2 macroblock decoder (used for FMV) that can also do CSC and VQ decompression. Optional for most homebrew. [HIGH]
- Late "slim" revisions (SCPH-750xx onward, "Deckard") replace the physical IOP with a PowerPC chip emulating it; behavior is nearly identical but IOP timing-sensitive hacks can differ on those units. [MEDIUM]

### Security / DRM
- Boot chain: boot ROM verifies discs via wobble/mechacon; there is no hypervisor and no runtime code signing once execution is achieved. [HIGH]
- Standard homebrew entry points: **Free MC Boot (FMCB)** on fat/early-slim consoles (exploits memory card boot), **FreeDVDBoot** (DVD player exploit), and **MechaPwn** for later units. Once booted into a launcher (uLaunchELF/wLaunchELF), unsigned ELFs run with full hardware access  homebrew can touch every register described in this document. [HIGH]
- The only thing homebrew cannot do is decrypt/execute official encrypted boot files (KELF) without console-specific keys  irrelevant for original homebrew. [MEDIUM]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official SDK: Sony's SCE PS2 SDK ("TOOL" devkits, `libgraph`/`libdma`/`libpad` etc.). You must NOT use, cite, or assume access to it. [HIGH]
- Homebrew SDK: **ps2sdk** (github.com/ps2dev/ps2sdk), built on **ps2toolchain** (GCC 14.x-era cross compilers as of 2025, Binutils with R5900 + DVP support), licensed AFL/BSD-style. Companion libraries: **gsKit** (graphics), **ps2sdk-ports** (zlib, libpng, freetype, SDL, SDL2, etc.). [HIGH]
- What ps2sdk CAN do: full EE kernel services (threads, semaphores, alarms), complete DMAC/GIF/VIF/GS register access, VU0/VU1 programming (DVP assembler `dvp-as` ships in binutils), IOP module development, pads, multitap, memory cards, USB mass storage, HDD (APA/PFS), Ethernet (lwIP), audsrv audio, CDVD access, host: filesystem over ps2link. [HIGH]
- What it CANNOT do (or does worse than the official SDK): no official-grade VU vector-code compiler pipeline (Sony's VCL macro preprocessor has an open reimplementation, **openvcl**, of varying maturity  hand-written `.vsm` via `dvp-as` is the reliable path) [MEDIUM]; sparse high-level documentation  samples in `ps2sdk/samples` are the de-facto docs [HIGH]; IPU/MPEG playback support is minimal [MEDIUM].
- Critical decision impact: because there is no shader compiler and no mature "GL on PS2", plan your renderer around **gsKit or raw GIF packets**, not around a GL translation layer. `ps2gl` (a GL 1.x subset) exists in ps2gl repos but is only partially maintained  treat it as a prototyping aid, not a production target. [MEDIUM]

---

## 3. GRAPHICS PIPELINE

- API to use: **gsKit** for 2D and straightforward 3D (it manages VRAM allocation, display environment, and GIF packet construction), or raw GIF packets via `packet2` from ps2sdk for full control. Escalate to VU1 microprograms only when EE-side transform becomes the bottleneck. [HIGH]
- Initialization (gsKit):
  ```c
  GSGLOBAL *gs = gsKit_init_global();          // auto-detects NTSC/PAL from ROM region
  gs->PSM  = GS_PSM_CT24;                      // 24-bit color saves VRAM vs CT32
  gs->PSMZ = GS_PSMZ_16S;                      // 16-bit Z saves VRAM
  gs->DoubleBuffering = GS_SETTING_ON;
  gs->ZBuffering      = GS_SETTING_ON;
  dmaKit_init(D_CTRL_RELE_OFF, D_CTRL_MFD_OFF, D_CTRL_STS_UNSPEC, D_CTRL_STD_OFF, D_CTRL_RCYC_8, 1 << DMA_CHANNEL_GIF);
  dmaKit_chan_init(DMA_CHANNEL_GIF);
  gsKit_init_screen(gs);
  gsKit_mode_switch(gs, GS_ONESHOT);
  ```
  [HIGH]
- Framebuffer flow: you allocate two color buffers + one Z buffer in GS VRAM; draw into the back buffer via GIF packets; at vsync, the CRTC's `DISPFB` register is pointed at the finished buffer (`gsKit_sync_flip(gs)` does wait-vsync + flip + queue swap). Rendering is to VRAM only  the EE never has a CPU-visible framebuffer pointer. [HIGH]
- VSync/refresh: NTSC = 59.94 Hz field rate, 640Ã448 interlaced (field-alternating); PAL = 50 Hz, 640Ã512 interlaced; progressive 240p/288p ("non-interlaced") and 480p (`GS_MODE_DTV_480P`, requires component cables) are available via `SetGsCrt`. Design the game loop for both 60 Hz and 50 Hz or force NTSC timing with PAL60 output. [HIGH]
- Texture upload: textures live in main RAM and are DMA'd to VRAM through the GIF in IMAGE mode (`gsKit_texture_upload`). VRAM addresses are word (256-byte-page-structured) offsets managed by `gsKit_vram_alloc`. Texture base pointers and buffer widths are in units of 64-pixel blocks/pages  let gsKit compute them (`gsKit_texture_size`, `gsKit_setup_tbw`). [HIGH]
- Formats & size: prefer PSMT8 + 256-color CLUT for large textures (4Ã smaller than CT32); max 1024Ã1024; no compression. CLUTs for PSMT8 must be uploaded in the GS's swizzled CSM1 layout  gsKit handles this if you use its texture structs. [HIGH]
- Depth/blend: 24-bit or 16-bit Z (greater-or-equal test convention  GS Z is inverted relative to GL defaults; gsKit configures `ZTST` for you); alpha blending is programmed via the `ALPHA` register equation `(A-B)*C>>7 + D` with selectable sources  standard src-alpha blending is `gsKit_set_primalpha(gs, GS_SETREG_ALPHA(0,1,0,1,0), 0)`. [HIGH]
- Fixed-function stages, in order: vertex kick (XYZ2/XYZF2 register writes complete a primitive)  scissor  Z test  texture sample (1 stage)  fog  alpha blend  alpha/dest-alpha test  dither  write. You configure them by writing GS registers inside GIF packets (`PRIM`, `TEX0`, `TEX1`, `CLAMP`, `TEST`, `ALPHA`, `FRAME`, `ZBUF`, `SCISSOR`, `XYOFFSET`). [HIGH]
- Coordinate quirk: GS primitive coordinates are unsigned 12.4 fixed point within a 4096Ã4096 window; the visible area is positioned with `XYOFFSET` (conventionally 2048-centered to allow guard-band clipping). Convert floats with `(int)(x * 16.0f) + (2048 << 4)` style math or use gsKit's helpers. [HIGH]
- Anti-patterns:
  - Do NOT attempt to use OpenGL/Vulkan-style abstractions as the primary renderer; there is no GPU-side transform  all vertex math happens on the EE or VU1. [HIGH]
  - Do NOT keep all textures resident in VRAM like on PC; budget VRAM per frame and stream. [HIGH]
  - Do NOT write GS *general* registers (TEX0, PRIM, ¦) via the privileged `0x12000000` MMIO range  general registers are only reachable through GIF packets; only the privileged set (PMODE, DISPFB, CSR, ¦) is memory-mapped. [HIGH]
  - Do NOT forget `FlushCache(0)` (or build packets in uncached/UCAB memory) before kicking a GIF DMA  the DMAC reads physical RAM, not your D-cache. [HIGH]

---

## 4. INPUT

- Supported controller hardware: DualShock 2 (analog sticks + pressure-sensitive buttons + rumble), original PS1 digital/DualShock pads, multitap (up to 2 taps Ã 4 ports = 8 pads), PS1/PS2 mice, GunCon2 light guns, dance mats, USB peripherals (keyboard/mouse via `ps2kbd`/HID modules). [HIGH]
- Model: **polled**, not event-driven. The EE-side `libpad` talks over SIF RPC to IOP modules.
- Initialization sequence (order matters):
  ```c
  SifLoadModule("rom0:SIO2MAN", 0, NULL);   // SIO2 bus arbiter  must be first
  SifLoadModule("rom0:PADMAN",  0, NULL);   // pad driver
  padInit(0);
  static char padBuf[256] __attribute__((aligned(64)));  // 256-byte DMA buffer, 64-byte aligned
  padPortOpen(0, 0, padBuf);                // port 0, slot 0
  ```
  Then wait until `padGetState(0,0)` returns `PAD_STATE_STABLE` (or `FINDCTP1`) before reading; enter analog mode with `padSetMainMode(0, 0, PAD_MMODE_DUALSHOCK, PAD_MMODE_LOCK)`. [HIGH]
- Reading:
  ```c
  struct padButtonStatus pad;
  padRead(0, 0, &pad);
  u32 buttons = 0xFFFF ^ pad.btns;   // ACTIVE-LOW: raw bits are 0 when pressed
  if (buttons & PAD_CROSS) { ... }
  // analog: pad.ljoy_h/ljoy_v/rjoy_h/rjoy_v, 0..255, center ~128
  // pressures: pad.cross_p etc. when in DualShock2 pressure mode
  ```
  The **inverted button mask** is the single most common input bug on this platform. [HIGH]
- Rumble: `padSetActAlign` once, then `padSetActDirect(port, slot, act)` with `act[0]` = small motor (0/1) and `act[1]` = big motor intensity (0255). [HIGH]
- Expansion: memory cards use separate modules (`MCMAN`/`MCSERV`, see §7); multitap needs `PADMAN` + `MTAPMAN` (`mtapPortOpen`). [HIGH]
- Disconnect handling: `padGetState` returns `PAD_STATE_DISCONN`  poll it every frame and re-run the stabilize/analog-mode sequence on reconnect; do not cache "pad is DualShock 2" forever. [HIGH]
- Anti-patterns: Do NOT assume the pad is in analog mode at boot (PS1 pads and fresh connects start digital); do NOT read `pad.btns` without inverting; do NOT allocate the `padPortOpen` buffer on the stack. [HIGH]

---

## 5. MEMORY LAYOUT

### EE address map (physical / KUSEG view used by homebrew)
| Range | What | Access |
|---|---|---|
| `0x000000000x01FFFFFF` | 32 MB main RAM | cached (default) [HIGH] |
| `0x200000000x21FFFFFF` | same RAM, **uncached** mirror | slow, coherent [HIGH] |
| `0x300000000x31FFFFFF` | same RAM, **uncached accelerated (UCAB)**  write-buffered | ideal for building DMA packets [HIGH] |
| `0x700000000x70003FFF` | 16 KB scratchpad (SPR) | single-cycle, not DMA-visible as normal RAM (use SPR DMA channels) [HIGH] |
| `0x100000000x10001FFF` | EE timers T0T3 | MMIO [HIGH] |
| `0x10003000 / 0x10003800 / 0x10003C00` | GIF / VIF0 / VIF1 registers | MMIO [HIGH] |
| `0x100080000x1000EFFF` | DMAC channel registers (`Dn_CHCR/MADR/QWC/TADR¦`) | MMIO [HIGH] |
| `0x1000F000` | INTC (INTC_STAT/INTC_MASK) | MMIO [HIGH] |
| `0x110000000x1100FFFF` | VU0/VU1 micro + data memory | MMIO [HIGH] |
| `0x120000000x12001FFF` | GS **privileged** registers (PMODE, SMODE2, DISPFB1/2, DISPLAY1/2, CSR, IMR) | MMIO [HIGH] |
| `0x1C000000` | IOP RAM window from EE | debug/interop [MEDIUM] |
| `0x1FC00000` | 4 MB boot ROM | read-only [HIGH] |

- The linked ELF loads at `0x00100000` by default (ps2sdk linkfile); everything below is kernel-reserved. Heap grows from `_end`; the default crt0 places the stack near the top of the 32 MB. You can reserve the stack/heap split with `-Wl,--defsym,_stack_size=¦` style symbols or by overriding the linkfile  check `$(PS2SDK)/ee/startup/linkfile` for the current mechanism before advising exact symbols. [MEDIUM]
- Bank speeds: main RAM « IOP RAM (never bounce hot data through IOP); scratchpad is the fastest CPU-visible memory  use it for hot working sets (matrix palettes, particle scratch, vertex staging) with SPR DMA to stream in/out. [HIGH]
- Allocation: `malloc`/`memalign` from newlib on the EE heap; `gsKit_vram_alloc` for GS VRAM; IOP-side allocations happen inside IRX modules only. [HIGH]

### Cache rules (memorize these)
- D-cache is **write-back**. DMA engines read/write physical RAM directly. Therefore:
  - Before any DMA **from** a buffer the CPU wrote: `FlushCache(0)` (writeback+invalidate everything) or `SyncDCache(start, end)` for a range. [HIGH]
  - After any DMA **into** a buffer the CPU will read: `InvalidateDCache(start, end)` (or have used an uncached pointer). [HIGH]
  - After writing code to RAM (loaders, self-modifying): `FlushCache(0)` then `FlushCache(2)` (I-cache invalidate). [HIGH]
- Cache line = 64 bytes: never let a DMA-target buffer share a cache line with unrelated CPU data  align and pad to 64. [HIGH]
- DMA requirements: addresses and lengths in whole qwords (16 bytes); a single DMA tag moves at most 65,535 qwords (~1 MB)  chain tags for more. [HIGH]
- Anti-patterns: Do NOT build GIF packets in cached memory and forget the flush (classic "works in emulator, garbage on hardware"); do NOT put DMA buffers on the stack; do NOT use the uncached mirror for read-heavy CPU work (it bypasses the cache entirely and is very slow). [HIGH]

---

## 6. AUDIO

- Hardware: **SPU2**  48 ADPCM voices, 2 MB dedicated sound RAM, hardware reverb/effects, fixed 48 kHz stereo output mix. Only the IOP can program it directly. [HIGH]
- Recommended API: **audsrv** (ships with ps2sdk). Load `rom0:LIBSD` (or ps2sdk's `freesd.irx`) then `audsrv.irx`, call `audsrv_init()`. [HIGH]
- Two modes:
  1. **Streaming**: `audsrv_set_format(&fmt)` with PCM (e.g., 44100 or 48000 Hz, 16-bit, 2ch), then push interleaved PCM with `audsrv_play_audio(buf, len)`; `audsrv_wait_audio(len)` blocks until the ring has room. This is how ported engines (SDL audio backends, Quake-style mixers) feed the PS2  mix on the EE, ship PCM to the IOP. [HIGH]
  2. **Voices**: upload VAG/ADPCM samples to SPU2 RAM (`audsrv_load_adpcm`) and trigger with `audsrv_ch_play_adpcm` for low-latency SFX using hardware voices. [HIGH]
- Formats: SPU2 natively plays only its 4-bit ADPCM (VAG); PCM streaming works because audsrv/IOP feeds the SPU2 "core" input. Convert source WAVs to VAG offline (open tools: `vgmstream`-adjacent encoders, ps2sdk-ports `wav2vag`-style utilities). [MEDIUM]
- Buffering: audsrv is double-buffer/ring based on the IOP; keep ¥ 24 KB of PCM queued. Because the transport is SIF RPC, do NOT call `audsrv_play_audio` at fine granularity from the render loop  feed it in chunks of one video frame's worth of audio (e.g., 48000/60 Ã 2ch Ã 2B = 3200 bytes) or run the feeder in its own EE thread. [HIGH]
- Underruns manifest as buzzing/looped stutter (SPU2 keeps replaying its buffer). Fix by increasing queued chunk size, not by calling more often. [MEDIUM]
- Anti-patterns: Do NOT mix audio inside a callback expecting PC-style low-latency callbacks  there is no EE audio interrupt callback in audsrv; it's a push model. Do NOT store streaming PCM in SPU2 RAM manually while audsrv owns it. Do NOT assume 44.1 kHz output  the final mix is 48 kHz; resample or set format accordingly. [MEDIUM]

---

## 7. STORAGE / IO

- Accessible media: memory cards (`mc0:`/`mc1:`), USB mass storage (`mass0:`), internal HDD on fat consoles (`hdd0:` APA partitions with PFS, exposed as `pfs0:`), optical disc (`cdrom0:`/`cdfs:`), **MX4SIO** (SD card in memory-card slot, via BDM), network filesystem `host:` (ps2link over Ethernet), and MMCE devices (SD2PSX/MemCard Pro 2) on current ps2sdk. [HIGH]
- Everything is an IOP IRX module you must load in the right order. Modern ps2sdk uses the **BDM (Block Device Manager)** stack for USB/SD:
  ```c
  SifInitRpc(0);
  // Embed IRX in the ELF (bin2c / EMBED_IRX) or load from the boot device:
  SifExecModuleBuffer(bdm_irx, size_bdm_irx, 0, NULL, NULL);          // block device manager
  SifExecModuleBuffer(bdmfs_fatfs_irx, size_bdmfs_fatfs_irx, 0, ...); // FAT/exFAT filesystem
  SifExecModuleBuffer(usbd_irx, size_usbd_irx, 0, ...);               // USB stack
  SifExecModuleBuffer(usbmass_bd_irx, size_usbmass_bd_irx, 0, ...);   // USB mass storage  BDM
  // wait ~a few hundred ms or poll open("mass0:", ...) until the drive enumerates
  ```
  FAT32 is the safe format; exFAT is supported by the FatFs-based driver in current ps2sdk. [HIGH for FAT32, MEDIUM for exFAT  verify by opening a >4 GB volume on hardware]
- Memory cards: load `rom0:MCMAN` + `rom0:MCSERV` (or ps2sdk's `mcman.irx`/`mcserv.irx`), then use `mcInit(MC_TYPE_XMC)` + `mcGetInfo`/`mcSync`, or simply POSIX `fopen("mc0:/DIR/file", ...)` through fileio. Card filesystem paths are case-sensitive in practice  always use consistent casing. [MEDIUM  verify with a mixed-case open test on hardware]
- Path conventions: device-prefixed (`mass0:/dir/file.ext`, `mc0:/SAVES/...`, `host:file`  note `host:` traditionally takes no leading slash). FAT via BDM is case-insensitive; `host:` inherits the PC's filesystem semantics. Never hardcode one device  take a base path at init. [HIGH]
- Loader conventions: **uLaunchELF/wLaunchELF** browses any device and launches `.ELF` directly  no metadata file is required. For a nice presentation in FMCB/OPL-style menus, ship `TITLE.ELF` plus an `icon.sys` + `*.icn` only if installing to a memory card. Homebrew "apps" folder standard: `mass0:/APPS/YourApp/yourapp.elf` is common but not enforced. [MEDIUM]
- Disc: `cdrom0:\PATH\FILE.EXT;1` (uppercase 8.3, backslashes, `;1` version suffix) via CDVD  relevant only if you master ISOs. [HIGH]
- Network: fat consoles + Network Adapter (or slim built-in Ethernet): load `NETMAN` + `SMAP` + use `ps2ip` (lwIP port) for TCP/UDP sockets. No Wi-Fi. `ps2link` gives you `host:` file access + remote `printf`  the standard dev loop. [HIGH]
- Anti-patterns: Do NOT load `usbd.irx` after the mass-storage driver (order matters); do NOT assume the drive is mounted the instant modules load (enumeration is async  retry-open with a timeout); do NOT write to memory cards without the real MCMAN/MCSERV pair; do NOT use `rom0:` module names on very old/very new ROMs without a fallback to embedded IRX. [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **ps2toolchain / ps2dev** (install via the ps2dev Docker image `ghcr.io/ps2dev/ps2dev` or the release tarballs  building from source is slow but supported). [HIGH]
- Cross-compiler prefixes:
  - EE: `mips64r5900el-ps2-elf-` (gcc/g++/as/ld), target triplet `mips64r5900el-ps2-elf`. [HIGH]
  - IOP: `mipsel-ps2-irx-` for IRX modules. [HIGH]
  - VU: `dvp-as` (in binutils) assembles `.vsm`/`.vcl`-output microprograms. [MEDIUM]
- Environment variables (mandatory):
  ```sh
  export PS2DEV=/usr/local/ps2dev
  export PS2SDK=$PS2DEV/ps2sdk
  export GSKIT=$PS2DEV/gsKit
  export PATH=$PATH:$PS2DEV/ee/bin:$PS2DEV/iop/bin:$PS2DEV/dvp/bin:$PS2SDK/bin
  ```
  [HIGH]
- Required EE compiler flags: `-D_EE -G0 -O2 -Wall` plus `-I$(PS2SDK)/ee/include -I$(PS2SDK)/common/include`. **`-G0` is not optional** on non-trivial projects  the R5900 small-data section (`gp`-relative) overflows and produces relocation errors otherwise. Add `-fsingle-precision-constant` when porting float-heavy engines to kill accidental doubles. [HIGH]
- Linker: `-T$(PS2SDK)/ee/startup/linkfile -L$(PS2SDK)/ee/lib -L$(GSKIT)/lib`; typical libs: `-lgskit -ldmakit -lpad -laudsrv -lpatches -ldebug -lmc -lc -lkernel` (ps2sdk's Makefile fragments append libc/kernel for you). [HIGH]
- Canonical Makefile (use ps2sdk's fragments; do not hand-roll):
  ```make
  EE_BIN   = myapp.elf
  EE_OBJS  = main.o renderer.o
  EE_CFLAGS  += -DPS2 -fsingle-precision-constant
  EE_LIBS  = -L$(GSKIT)/lib -lgskit -ldmakit -lpad -laudsrv -lpatches
  EE_INCS  += -I$(GSKIT)/include

  all: $(EE_BIN)
  clean:
  	rm -f $(EE_BIN) $(EE_OBJS)

  include $(PS2SDK)/samples/Makefile.pref
  include $(PS2SDK)/samples/Makefile.eeglobal
  ```
  [HIGH]
- Output format: standard **ELF** (32-bit MIPS ELF with R5900 flags). No conversion step  launchers run the ELF directly. Strip with `mips64r5900el-ps2-elf-strip` for release; optionally pack with `ps2-packer` to shrink load time. [HIGH]
- IRX embedding: convert `.irx` files to objects with `bin2c`/`bin2o` (ps2sdk tools) so the ELF is self-contained on any boot device. [HIGH]
- Asset pipeline: convert textures offline to raw GS-format buffers or PNG (decode at load via ps2sdk-ports libpng); quantize large textures to 8-bit palettes offline; convert audio to VAG (SFX) and 16-bit PCM at 48 kHz (streams); keep individual texture uploads ¤ VRAM budget per frame. [MEDIUM]
- Anti-patterns: Do NOT compile with a generic `mips-linux-gnu` toolchain (wrong ABI, no R5900 errata fixes, no 128-bit types); do NOT use `-msoft-float` (EE has an FPU; ps2sdk defaults are correct); do NOT drop `-G0`; do NOT pass `-flto` casually  it has a history of breaking crt0/linkfile assumptions on this toolchain [MEDIUM  verify with current GCC before allowing]. [HIGH overall]

---

## 9. EMULATOR VS HARDWARE

- Primary emulator: **PCSX2** (use a current stable/nightly Qt build). Accuracy is high for retail-game behavior; homebrew hits its edges more often. [HIGH]
- Safe to rely on: GS rendering semantics (software renderer especially), general EE/IOP instruction behavior, pad/memcard emulation, `host:`-style ELF boot (`-elf` CLI flag or drag-and-drop), most DMA chain logic. PCSX2's software renderer is the reference when hardware-renderer output looks wrong. [HIGH]
- What it gets WRONG or hides (must test on real hardware):
  - **Cache behavior**: PCSX2 does not emulate the EE D-cache by default  missing `FlushCache` bugs are invisible in-emulator and fatal on hardware. This is the #1 "works in PCSX2, dies on PS2" cause. [HIGH]
  - DMA/SIF timing: RPC races and module-load timing differ; add real synchronization, not delays tuned in-emulator. [HIGH]
  - Performance: emulator FPS says nothing about hardware FPS in either direction. Profile with EE timers (`T0_COUNT`) on hardware. [HIGH]
  - USB/BDM device enumeration timing and exotic IRX behavior. [MEDIUM]
- Hardware debugging options:
  - **ps2link + ps2client** over Ethernet: load ELFs remotely, `host:` filesystem, and `printf` back to the PC console  the standard iteration loop (seconds per cycle, no SD swapping). [HIGH]
  - ps2sdk's default exception handler dumps EE registers + cause code on screen when the ELF crashes  always read the `EPC` (crash address) and map it with `mips64r5900el-ps2-elf-addr2line -e myapp.elf 0xADDR`. [HIGH]
  - `-ldebug` gives `init_scr()`/`scr_printf()`  on-screen text with zero renderer dependencies; use it for first-boot bring-up. [HIGH]
  - EE SIO (bare UART pads on the motherboard) exists for hard-core cases; treat as last resort. [MEDIUM]
- Anti-patterns: Do NOT debug cache-coherency bugs in PCSX2 (enable its EE cache emulation option or go to hardware); do NOT tune performance in-emulator; do NOT trust that a PCSX2-only black screen means broken code  check the emulator log for unsupported homebrew paths first. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, no video ever | First GS bring-up | `gsKit_init_screen` never called / DMAC channel not initialized / wrong CRTC mode for the TV | Init dmaKit + GIF channel before `gsKit_init_screen`; let gsKit auto-detect region; test with `-ldebug` `scr_printf` first | [HIGH] |
| Renders in PCSX2, black/garbage on console | Any GIF/DMA use | Missing `FlushCache(0)` before DMA kick (PCSX2 skips D-cache) | Flush before every kick, or build packets in UCAB (`0x30000000`) memory | [HIGH] |
| Textures corrupted / shifted / rainbow noise | Texture upload | Wrong TBW (buffer width), non-power-of-two dims in TEX0, or VRAM overlap between texture and framebuffer | Use `gsKit_vram_alloc` for everything; verify TBW = width/64 rounded up; log VRAM map | [HIGH] |
| Textures fine near, sparkle/noise far away | 3D scene | No mipmaps + GS LOD misconfig in `TEX1` | Upload mip chain + set MIPTBP, or clamp LOD (`TEX1.MXL=0`) | [MEDIUM] |
| Audio buzzes/loops a short fragment | audsrv streaming | Ring underrun  feeder thread starved or chunks too small | Queue ¥1 video-frame of PCM per push; dedicate an EE thread; check `audsrv_wait_audio` usage | [HIGH] |
| Silence, `audsrv_init` returns error | Audio init | `LIBSD`/`freesd` not loaded before `audsrv.irx`, or IRX load order wrong | Load LIBSD  audsrv; check every `SifLoadModule` return code | [HIGH] |
| Crash on boot before `main` | Fresh project | Linked without ps2sdk linkfile/crt0, or ELF loaded below `0x00100000` | Use `Makefile.eeglobal` fragments; keep default load address | [HIGH] |
| Exception screen: address error (EPC shown) | After enabling MMI/128-bit or memcpy of structs | Unaligned `lq`/`sq`  128-bit access to non-16-byte-aligned address | `addr2line` the EPC; align the struct/buffer to 16 (64 for DMA) | [HIGH] |
| Crash/hang after loading a large file | Asset loading | Heap exhaustion in 32 MB (malloc returns NULL unchecked) or DMA size > 65,535 qwords in one tag | Check allocations; chain DMA tags; track a memory budget | [HIGH] |
| Pad never responds | Input init | SIO2MAN not loaded before PADMAN, pad buffer misaligned, or reading `btns` without inversion | Load order SIO2MANPADMAN; 64-byte-aligned 256-byte buffer; `0xFFFF ^ btns` | [HIGH] |
| `mass0:` open fails right after module load | USB/BDM | Enumeration is async  device not ready yet | Retry open for up to ~3 s; check IRX load order (bdm  fs  usbd  usbmass) | [HIGH] |
| Everything runs ~17% slow / music pitch-shifted | PAL console | Game loop hardcoded to 60 Hz but console outputs 50 Hz | Detect mode from `gs->Mode`; timestep from vsync rate, or force NTSC/480p output | [HIGH] |
| Sudden 50%+ FPS drop when scene grows | Rendering | GIF PATH3 saturation or per-primitive packet overhead (one DMA kick per triangle) | Batch primitives into chained packets; one kick per frame/region; consider VU1 PATH1 | [MEDIUM] |
| Screen shows image but wrong aspect/offset, or shakes | Display setup | Interlace FIELD vs FRAME mode mismatch, or DISPLAY/XYOFFSET misconfigured | Use gsKit defaults; for shaking interlace, enable gsKit's field-mode handling or render 448i properly | [MEDIUM] |
| Random corruption only after minutes of play | Long sessions | DMA buffer sharing a cache line with live CPU data, or VRAM allocator overlap after dynamic uploads | 64-byte-pad all DMA buffers; audit `gsKit_vram_alloc` lifetime | [MEDIUM] |

---

## 11. ANTI-PATTERNS

1. Do NOT use `double` anywhere in hot code  it is software-emulated; compile with `-fsingle-precision-constant` and use `sinf/cosf/sqrtf`. [HIGH]
2. Do NOT skip `FlushCache(0)` before DMA kicks just because PCSX2 renders fine. [HIGH]
3. Do NOT build with `-G` defaults; you must use `-G0`. [HIGH]
4. Do NOT treat the GS like a GPU with shaders or multitexturing  design for multi-pass fixed-function. [HIGH]
5. Do NOT keep a PC-style resident texture atlas in VRAM; stream textures through the GIF each frame within a budget. [HIGH]
6. Do NOT read pad buttons without inverting the active-low bitmask. [HIGH]
7. Do NOT place DMA buffers on the stack or unaligned; 16-byte minimum, 64-byte preferred. [HIGH]
8. Do NOT make blocking SIF RPC calls (file I/O, audsrv, pad mode changes) inside the render loop. [HIGH]
9. Do NOT assume NTSC/60 Hz  handle PAL 50 Hz or force the mode explicitly. [HIGH]
10. Do NOT load IOP modules in arbitrary order or ignore `SifLoadModule` return values. [HIGH]
11. Do NOT expect IEEE float semantics  no NaN/Inf; ported physics/BSP code relying on NaN checks will misbehave silently. [HIGH]
12. Do NOT write GS general registers through MMIO; they only exist behind GIF packets. [HIGH]
13. Do NOT `memset`/touch a buffer with the CPU while a DMA transfer to/from it is in flight  wait on the channel (`dmaKit_wait_fast` / CHCR poll). [HIGH]
14. Do NOT assume the filesystem is case-insensitive everywhere; `mc0:` and `host:` can be case-sensitive while `mass0:` is not. [MEDIUM]
15. Do NOT profile or performance-tune in PCSX2; use EE timer registers on real hardware. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Boot skeleton first** (`main` + `-ldebug` `scr_printf` + clean exit). *Why first:* proves toolchain, linkfile, and loader path before any subsystem can confuse the picture. *Skip it and:* you will debug renderer code when the real problem is a broken ELF.
2. **Establish the dev loop: ps2link over Ethernet (or emulator + hardware pair).* *Why:* iteration time dominates porting speed; `host:` + remote printf turns 5-minute SD-swap cycles into 10-second cycles. *Skip it and:* every later step costs 10Ã the time.
3. **Filesystem abstraction** (device-prefixed paths, retry-on-mount, embedded IRX). *Why:* every engine loads assets before it renders; get `fopen` working on `host:` and `mass0:` behind one base-path variable. *Skip it and:* asset loading failures masquerade as engine bugs.
4. **Float audit**: strip doubles, add `-fsingle-precision-constant`, replace `libm` doubles with `f` variants, remove NaN/Inf-dependent logic. *Why this early:* it's a global, mechanical change that silently destroys performance and correctness if deferred. *Skip it and:* you'll ship at 15 FPS and blame the GS.
5. **Memory budget** (32 MB total; measure engine heap needs; add allocation logging). *Why:* engines assuming ¥128 MB must have their caches/pools capped now, before the renderer piles VRAM staging on top. *Skip it and:* random OOM crashes late in the port.
6. **Video init + game loop timing** (gsKit double-buffer, vsync flip, 50/60 Hz-aware timestep). *Why:* everything downstream renders through this; timing bugs contaminate physics and audio sync. *Skip it and:* PAL units run slow and interlace shakes.
7. **Renderer bring-up on GIF PATH3 / gsKit immediate mode**  untextured triangles  textured  lightmaps as a second blended pass. *Why:* simplest correct path; establishes VRAM budgeting and texture streaming. *Skip it and:* nothing to skip  this is the port.
8. **Input** (libpad with reconnect handling, analog mode). *Why after video:* you need a picture to test against; pads are quick once the IOP module pattern from step 3 exists.
9. **Audio** (audsrv PCM streaming thread fed by the engine mixer; VAG voices for SFX later). *Why late:* audio is decoupled and the push model is forgiving; earlier it just adds noise to debugging.
10. **Optimization passes, in order of payoff:** batch GIF packets (one kick per frame region)  palettize textures to PSMT8  move hot vertex transform to VU1 microcode  stage hot data in scratchpad via SPR DMA. *Why last:* each requires a working, measurable baseline (EE timer profiling). *Skip the ordering and:* you'll hand-write VU1 code to fix what was actually a texture-upload stall.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PS2 / EE: MIPS R5900 @ 294.912 MHz, little-endian, 128-bit MMI, 16K I$/8K D$ (write-back, 64B lines),
  16 KB scratchpad @ 0x70000000. FPU: float-only, NOT IEEE (no NaN/Inf). double = software = FORBIDDEN.
GS: 147 MHz fixed-function, NO shaders, 4 MB shared VRAM (FB+Z+textures), max tex 1024^2, no compression,
  PSMT8+CLUT = your "compression". General regs only via GIF packets; privileged regs @ 0x12000000.
VU0 (COP2, 4K/4K) + VU1 (16K/16K, XGKICK->GIF PATH1) = the "vertex shaders". PATH3 (GIF DMA) = easy path.
RAM: 32 MB main (0x0-0x2000000) | uncached mirror 0x2000_0000 | UCAB 0x3000_0000 | IOP 2 MB | SPU2 2 MB.
DMA: 16B-aligned qwords, <=65535 qw/tag, chain mode for frames. FlushCache(0) BEFORE kick. Always.
IOP: R3000 @ 36 MHz runs IRX drivers; EE talks via SIF RPC (slow  never in render loop).
  Load order matters: SIO2MAN->PADMAN; LIBSD->audsrv; bdm->bdmfs_fatfs->usbd->usbmass_bd -> mass0:.
Input: libpad, polled, buttons ACTIVE-LOW (0xFFFF ^ btns), analog 0-255 center 128, 256B/64B-aligned buf.
Audio: SPU2 48 voices ADPCM(VAG), 2 MB, 48 kHz out; audsrv PCM push-streaming from an EE thread.
Video: NTSC 640x448i 59.94 Hz / PAL 640x512i 50 Hz / 480p via component. gsKit auto-detects; handle both.
Build: ps2sdk (ps2dev), mips64r5900el-ps2-elf-gcc, flags: -D_EE -G0 -O2 -fsingle-precision-constant,
  link via $(PS2SDK)/samples/Makefile.pref + Makefile.eeglobal. Output = plain ELF (uLaunchELF runs it).
Debug: ps2link+ps2client (host: fs + net printf), -ldebug scr_printf, exception screen EPC -> addr2line.
Emulator: PCSX2  good semantics, NO D-cache emulation by default, useless for perf. Ship = hardware-tested.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this SKILLS.MD is used in an active development session (not just reference), you must follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS**  Examine the codebase, identify PS2-specific blockers (doubles, cache assumptions, memory footprint, renderer coupling), and plan the implementation.
- **STATE: IMPLEMENTATION**  Write/modify code based on this SKILLS.MD and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST**  Output the exact build commands and ask the user to compile and run on real hardware (or PCSX2 only when the goal is explicitly emulator-scoped).
- **STATE: WAITING_FOR_HARDWARE**  STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION**  User reports back results. Classify the result as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL**  On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES

1. **ANALYSIS  IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION  BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change. You must include:
   - Exact build command (typically `make clean && make` with the ps2sdk Makefile fragments)
   - Expected output file name and location (e.g., `myapp.elf` in the project root)
   - How to transfer to the target hardware: `ps2client -h <PS2_IP> execee host:myapp.elf` (preferred), or copy to FAT32 USB / MX4SIO SD and launch from uLaunchELF
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST  WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color framebuffer"]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including any error codes, black screen, crashes]
   === STOP ===
   ```
   After this header, you STOP generating. You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE  VALIDATION**: Triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - If user says "SUCCESS": Move to NEXT_GOAL.
   - If user says "FAILURE": Move to DEBUG_PROTOCOL.
   - If user says anything else (e.g., "it kind of works", "almost"): Ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE)  DEBUG_PROTOCOL**: You must output a **DEBUG BUILD PROTOCOL**:
   - A minimal C test case that isolates the failure (e.g., `-ldebug` `scr_printf` skeleton, a single-triangle GIF packet, a bare `padRead` loop), OR
   - A checklist of exactly 3 specific diagnostic steps (e.g., "Confirm `FlushCache(0)` precedes the GIF kick", "Print the return value of every `SifLoadModule` call", "Note the EPC from the exception screen and run addr2line").
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL  WAITING_FOR_HARDWARE**: After providing the debug protocol, you return to WAITING_FOR_HARDWARE state.
7. **VALIDATION (SUCCESS)  NEXT_GOAL**: You propose the next milestone from the goal stack below. You do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [ps2link remote printf / PCSX2 software-renderer + EE-cache-emulation comparison / EE exception EPC + addr2line trace / ps2dev Discord-forum consultation].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide emulator logs / register dumps / exception-screen photos for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this default:
1. Initialize video output (gsKit solid-color framebuffer, vsync flip)
2. Initialize controller input (SIO2MAN/PADMAN, print button presses via scr_printf)
3. Initialize audio output (audsrv PCM sine wave)
4. Load assets from storage (host: during dev, mass0:/mc0: fallback)
5. Render main menu framebuffer (textured 2D via gsKit)
6. Main menu input loop
7. Transition to game state

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
