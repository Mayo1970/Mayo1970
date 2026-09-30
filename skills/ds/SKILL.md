---
name: ds
description: Nintendo DS / DSi homebrew development expertise (libnds, BlocksDS, devkitARM, ARM9/ARM7 dual-CPU, 2D/3D engines, maxmod, libfat/DLDI, dswifi). Use this skill whenever the user works on DS homebrew, ports an engine or game to the DS, mentions .nds files, ndstool, melonDS, no$gba, flashcarts, DLDI, NitroFS, VRAM banks, the DS 3D engine, ARM7/ARM9 IPC, or asks anything about Nintendo DS hardware, memory, audio, input, Wi-Fi, or build systems — even if they don't say "homebrew" explicitly. Also use it to drive hardware-validated, goal-gated development sessions on DS.
---

# SKILLS_DS.md — Nintendo DS Homebrew Expertise Document

You are assisting with Nintendo DS (and DSi-enhanced) homebrew development. Follow every rule in this document. Prefer open-source community toolchains (BlocksDS, devkitARM/libnds) in all cases. You must NOT reference, assume, or emit code for Nintendo's proprietary SDK (Nitro SDK / TwlSDK). All examples must compile with the community toolchain.

---

## 1. HARDWARE ARCHITECTURE

### CPUs (dual, asymmetric)
- Main CPU: **ARM946E-S ("ARM9") @ 67.028 MHz**, ARMv5TE, little-endian, 32-bit. 8 KB instruction cache, 4 KB data cache, 32 KB ITCM, 16 KB DTCM. [HIGH]
- Coprocessor: **ARM7TDMI ("ARM7") @ 33.514 MHz**, ARMv4T, little-endian, no cache, no TCM (has 64 KB private WRAM). [HIGH]
- **No FPU on either CPU.** All floating point is software-emulated and slow. You must use fixed-point math (libnds provides `int32` f32/v16/t16 types, `divf32`, `sqrtf32`, and hardware divide/sqrt registers on ARM9). [HIGH]
- ARM9 has hardware divide and square-root units via memory-mapped registers (0x04000280 DIVCNT, 0x040002B0 SQRTCNT) — the ARM cores themselves have no divide instruction. [HIGH]
- The ARM7 exclusively owns: touchscreen (SPI), audio hardware, Wi-Fi, RTC, power management, and firmware flash. The ARM9 must ask the ARM7 for these via IPC (FIFO). libnds ships a default ARM7 binary that services all of this; do not write your own ARM7 core unless you must. [HIGH]
- ARMv5TE quirk: unaligned 32-bit loads rotate data rather than fault-or-work — never rely on unaligned access. [HIGH]

### GPU
- Two independent **2D engines** (Engine A "main", Engine B "sub"), each driving one 256×192 LCD. Tile/map backgrounds (text, affine, extended), bitmap modes, 128 sprites per engine (max 64×64), per-engine palettes and OAM. Evolved GBA architecture. [HIGH]
- One **fixed-function 3D engine** (geometry engine + rendering engine), attachable to only ONE 2D engine at a time (normally Engine A, as BG0). [HIGH]
- 3D limits: **6144 vertices and 2048 polygons per frame** (hard hardware RAM limits). Rendering is scanline-based into a ~48-line buffer — there is no 3D framebuffer, and exceeding per-scanline fill limits drops geometry. [HIGH]
- 3D features: perspective-correct texturing, texture matrices, alpha blending, fog, toon/highlight shading, edge marking (outlines), rear-plane, hardware box test, and a coarse anti-aliasing mode. No shaders of any kind. [HIGH]
- Max texture size **1024×1024**. Texture formats: 4/16/256-color paletted, A3I5 and A5I3 (alpha+intensity), 16-bit direct color, and 4×4 texel compressed (COMP — the format you should prefer for large textures). [HIGH]
- Textures and texture palettes are read from dedicated VRAM banks mapped as "texture slots" (max 512 KB textures = banks A–D; palettes in E/F/G). Textures are NOT read from main RAM. [HIGH]
- Display capture unit can copy rendered output into VRAM — this is the mechanism for 3D-on-both-screens (alternating frames; each screen effectively runs at 30 FPS). [HIGH]
- Refresh: **~59.83 Hz on both screens, worldwide — there is no NTSC/PAL split.** [HIGH]

### RAM
- **4 MB main RAM** at 0x02000000, shared by both CPUs, on a slow 16-bit bus. DSi: 16 MB (only in DSi mode). [HIGH]
- **656 KB VRAM** in 9 banks: A/B/C/D = 128 KB each, E = 64 KB, F/G = 16 KB, H = 32 KB, I = 16 KB. Banks are software-mappable to backgrounds, sprites, textures, palettes, ARM7 workram, or plain LCDC memory. Bank mapping is a core design decision of every DS project. [HIGH]
- 32 KB shared WRAM (2×16 KB blocks, assignable to either CPU) at 0x03000000; 64 KB ARM7-private WRAM. [HIGH]
- ARM9 TCMs: 32 KB ITCM (put hot code here), 16 KB DTCM (libnds puts the ARM9 stack here). [HIGH]
- No MMU, no virtual memory. ARM9 has an MPU (protection regions only). Alignment: keep data naturally aligned; DMA and cache ops want 32-byte alignment for safety. [HIGH]

### Bus / DMA
- 4 DMA channels per CPU. DMA bypasses the ARM9 cache and **cannot access ITCM/DTCM**. [HIGH]
- Main RAM's 16-bit bus is the classic bottleneck; VRAM and TCM are much faster from the ARM9. [HIGH]
- GBA cart slot (slot-2) maps at 0x08000000 (NDS only, absent on DSi) — usable for rumble paks, RAM expansion paks (~8 MB, slow), and guitar-grip-style peripherals. [HIGH]

### Co-processors / other silicon
- Audio: 16-channel hardware mixer controlled by the ARM7 (see §6). [HIGH]
- Wi-Fi: 802.11b radio controlled by the ARM7. [HIGH]
- Touchscreen ADC (TSC2046-compatible) on the ARM7's SPI bus. [HIGH]
- DSi adds: cameras, SD slot, NAND, AES engine, faster 134 MHz ARM9 — accessible only to DSi-mode homebrew (Unlaunch/hbmenu ecosystem). [HIGH]

### Security / what homebrew can access
- NDS: no hypervisor, no runtime signature checks — once code runs (flashcart, or DSi-side loaders), homebrew has full hardware access except the cart secure-area boot path. [HIGH]
- DSi: signed boot chain; Unlaunch patches it, after which DSi-mode homebrew gets SD, NAND, cameras, extra RAM/clock. [HIGH]
- Firmware flash is writable via ARM7 SPI. **Never write to it.** Bricking is real. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: Nintendo Nitro SDK / TwlSDK (proprietary). Do NOT use, reference, or assume it. [HIGH]
- Homebrew, two mature options:
  - **BlocksDS** — modern, actively maintained (AntonioND et al.), plain GNU `arm-none-eabi` GCC + picolibc, ships libnds, maxmod, dswifi, libteak (DSi DSP), ndstool, and Make-based templates. Recommended for new projects. Licenses: Zlib/MPL/GPL components; SDK itself FOSS. [HIGH]
  - **devkitPro devkitARM + libnds** — the long-standing classic toolchain; huge body of existing example code targets it. [HIGH]
- What the homebrew stack CAN do: everything the hardware does — both 2D engines, full 3D engine, 16-channel audio (maxmod for MOD/XM/S3M/IT + SFX), touchscreen, mic, Wi-Fi (dswifi), FAT storage (libfat + DLDI), NitroFS embedded assets, DSi mode extras (camera and DSP support exist but are the least-mature corners). [HIGH]
- Gaps vs official: no Nintendo Wi-Fi Connection matchmaking stack, weaker local-wireless (NiFi) support (raw local multiplayer is possible but sparsely documented [MEDIUM]), no official-grade profiling tools. [MEDIUM]
- Feature gap that affects design: **cart secure-area / commercial-cart booting is irrelevant to homebrew** — you ship a `.nds` loaded by a flashcart kernel or TWiLight Menu++.

---

## 3. GRAPHICS PIPELINE

- API: **libnds** — thin wrappers over hardware registers (`videoSetMode`, `vramSetBankX`, `bgInit`, `oamInit`) plus a fixed-function GL-flavored 3D API (`glInit`, `glBegin(GL_TRIANGLES)`, `glTexImage2D`, ...). This "gl" API is NOT OpenGL — it maps 1:1 onto DS geometry-engine commands. Do NOT try to port real OpenGL/GLES code directly. [HIGH]
- Init (typical 3D main screen + 2D console on sub):
  ```c
  videoSetMode(MODE_0_3D);
  videoSetModeSub(MODE_0_2D);
  vramSetBankA(VRAM_A_TEXTURE);        // textures
  vramSetBankE(VRAM_E_TEX_PALETTE);    // texture palettes
  vramSetBankC(VRAM_C_SUB_BG);
  consoleDemoInit();                   // printf on sub screen
  glInit(); glEnable(GL_TEXTURE_2D); glEnable(GL_ANTIALIAS);
  glViewport(0,0,255,191);
  ```
  [HIGH]
- Framebuffer flow: 2D engines composite tiles/sprites per scanline — there is no CPU-visible "the framebuffer" unless you use bitmap BG modes or `MODE_FB0` (LCDC direct VRAM display, e.g. VRAM_A as raw 256×192×16bpp). 3D has no framebuffer at all (scanline pipeline); `glFlush()` at end of frame swaps geometry buffers, synchronized to VBlank. Double-buffering of 3D is implicit; for bitmap modes you double-buffer by swapping which VRAM bank/base is displayed. [HIGH]
- Always call `swiWaitForVBlank()` once per frame loop; update OAM/scroll/matrices during VBlank. [HIGH]
- Depth: 24-bit depth (W or Z selectable in `glFlush` flags); no stencil, but 3D supports per-polygon alpha, wireframe, and shadow-volume polygon type as a stencil-ish trick. [HIGH]
- One alpha-blended 3D polygon layer per pixel per scanline effectively — translucent overdraw is severely limited; sort and minimize translucency. [MEDIUM — verify per-scene by artifact-checking on hardware]
- Anti-patterns:
  - Do NOT write 8-bit values into VRAM — 8-bit VRAM writes are ignored by the hardware. Use 16/32-bit stores (this breaks naive `memcpy`-of-bytes and struct copies). [HIGH]
  - Do NOT exceed 2048 polys / 6144 verts per frame — geometry silently disappears. Use the hardware box test (`BoxTest`) for culling. [HIGH]
  - Do NOT expect to render 3D to both screens for free — it needs the capture-and-alternate technique at 30 FPS/screen. [HIGH]
  - Do NOT stream textures per-frame from main RAM; textures live in mapped VRAM banks. Unmap bank → CPU-write → remap to update. [HIGH]

---

## 4. INPUT

- Built-in: D-pad, A/B/X/Y, L/R, Start/Select, resistive single-touch touchscreen, hinge (lid) switch, microphone. No analog sticks, no gyro/accelerometer in the base unit. [HIGH]
- Model: **polled.** Each frame:
  ```c
  scanKeys();
  u32 down = keysDown(), held = keysHeld(), up = keysUp();
  if (held & KEY_TOUCH) { touchPosition t; touchRead(&t); /* t.px, t.py */ }
  ```
  [HIGH]
- A/B/Start/Select/D-pad/L/R come from REG_KEYINPUT on ARM9; X/Y, touch, and lid come from the ARM7 via IPC — this is invisible when you use libnds' default ARM7. [HIGH]
- Touch is single-point resistive: simultaneous two-point presses read as a midpoint. Do NOT implement multitouch gestures. [HIGH]
- Microphone: recorded via ARM7 (`soundMicRecord` in maxmod/libnds paths), 8/12-bit samples. [HIGH]
- Slot-2 peripherals (NDS/NDS Lite only): Rumble Pak (poked via GBA-slot addresses, libnds `rumble.h`), RAM expansion, Guitar Grip, paddle. Detect before use; absent on DSi/3DS-family. [HIGH]
- No controller disconnect concept — but DO handle the lid: closing the lid should power the backlights down / pause (default ARM7 handles sleep if you let it). [HIGH]
- Anti-pattern: Do NOT read touch coordinates without checking `KEY_TOUCH` is held — released-pen reads are garbage. [HIGH]

---

## 5. MEMORY LAYOUT

| Region | ARM9 address | Size | Notes |
|---|---|---|---|
| ITCM | 0x00000000 (mirr. 0x01000000) | 32 KB | fastest code RAM; DMA can't see it [HIGH] |
| Main RAM | 0x02000000 | 4 MB (16 MB DSi) | cached on ARM9; slow 16-bit bus [HIGH] |
| Shared WRAM | 0x03000000 | 0–32 KB | bank-assigned ARM9/ARM7 [HIGH] |
| I/O registers | 0x04000000 | — | REG_* [HIGH] |
| Palettes | 0x05000000 | 2 KB | 16-bit writes only [HIGH] |
| VRAM (BG/OBJ views) | 0x06000000… | mapped | 16/32-bit writes only [HIGH] |
| OAM | 0x07000000 | 2 KB | update during VBlank [HIGH] |
| GBA slot | 0x08000000 | ≤32 MB | slot-2 peripherals [HIGH] |
| DTCM | set by crt0 (libnds: near end of main RAM region) | 16 KB | ARM9 stack lives here [HIGH] |

- Allocation: `malloc` serves main RAM. Nothing allocates VRAM for you — you budget banks manually with `vramSetBankA..I`. Hot code → ITCM via `ITCM_CODE` attribute; hot data → DTCM via `DTCM_DATA` (mind the stack sharing it). [HIGH]
- **Stack: the ARM9 stack is in 16 KB DTCM.** A 20 KB local array = instant corruption. Heap-allocate anything big. Enlarging the stack means moving it to main RAM via linker-script/crt0 changes — prefer not to. [HIGH]
- Cache: ARM9 data cache is write-back, 32-byte lines, **no hardware coherency** with DMA or the ARM7. You must:
  - `DC_FlushRange(src, size)` before DMA reads a buffer or the ARM7 reads shared data. [HIGH]
  - `DC_InvalidateRange(dst, size)` after DMA writes into main RAM (or FlushRange before, Invalidate after — safest). [HIGH]
  - `IC_InvalidateAll()` after writing code (e.g., loading overlays). [HIGH]
- DMA: `dmaCopy` and friends; source/dest must not be in TCM; keep 4-byte (prefer 32-byte) alignment. For small copies, a CPU loop or `swiCopy` often beats DMA setup cost. [HIGH]
- Anti-patterns:
  - Do NOT `memcpy` byte-wise into VRAM/palettes/OAM (8-bit writes ignored). Use `dmaCopy`/`swiCopy`/16-bit loops. [HIGH]
  - Do NOT DMA from a freshly-written buffer without flushing the data cache first — the classic "corrupted texture only on hardware" bug. [HIGH]
  - Do NOT put big lookup tables in DTCM; you'll silently eat your own stack. [HIGH]

---

## 6. AUDIO

- Hardware: 16 channels mixed in hardware, owned by the **ARM7**. Formats per channel: PCM8, PCM16, IMA-ADPCM; channels 8–13 can be PSG rectangular wave, 14–15 white noise. Per-channel rate/volume/pan; loop points in hardware. [HIGH]
- Final mixer output is fixed-rate (~32.8 kHz path with reduced effective bit depth); per-channel sample rates are arbitrary via timer dividers. [MEDIUM — exact mixer bit-depth details rarely matter; verify against GBATEK if you need mastering-grade precision]
- Recommended library: **maxmod** (ships with both SDKs): plays MOD/XM/S3M/IT modules plus SFX from a `soundbank.bin` generated at build time by `mmutil`. For plain one-shot samples, libnds `soundPlaySample()` suffices. [HIGH]
- Buffering model: hardware channels auto-fetch sample data from main RAM via the ARM7 — no CPU callback needed for one-shots/loops. Maxmod streams module playback with an ARM7-side tick; there is also a maxmod streaming mode (`mmStream`) with a fill callback for custom mixing. [HIGH]
- No dedicated audio RAM — samples live in main RAM. **Flush the data cache after writing/decoding sample data on ARM9**, or the ARM7/DMA fetch reads stale bytes → crackling/garbage. [HIGH]
- Avoiding glitches: keep the ARM9 from starving the bus during heavy DMA bursts; for `mmStream`, keep the fill callback short and the buffer ≥ several video frames. [MEDIUM]
- Anti-patterns:
  - Do NOT mix audio in software on the ARM9 "like on PC" — you have 16 hardware channels; use them. [HIGH]
  - Do NOT forget `DC_FlushRange` on decoded audio buffers before playback. [HIGH]

---

## 7. STORAGE / IO

- Media: flashcart microSD (slot-1) via **libfat + DLDI driver**; DSi SD slot in DSi mode; embedded read-only assets via **NitroFS** (filesystem appended inside the .nds — the best default for game assets). [HIGH]
- Init:
  ```c
  fatInitDefault();      // FAT on flashcart/DSi-SD; returns false on failure — CHECK IT
  nitroFSInit(NULL);     // embedded assets; needs argv support or DLDI fallback
  ```
  [HIGH]
- DLDI: homebrew ships DLDI-agnostic; the loader (or user) patches the correct flashcart driver into the binary. Modern loaders (TWiLight Menu++, hbmenu) auto-patch. FAT paths are case-insensitive-but-preserving (FAT semantics); `fat:/` and `nitro:/` prefixes select the filesystem. [HIGH]
- Loader conventions: `.nds` files anywhere on the card; TWiLight Menu++ / hbmenu read the embedded banner (icon + title) from the ROM header — set it with `ndstool -b icon.bmp "Title;Subtitle;Author"`. Save data for homebrew = ordinary files you create on FAT. [HIGH]
- Network: **dswifi** — 802.11b infrastructure mode with a sockets-like API (TCP/UDP via its lwIP-ish stack). Classic NDS-mode limitation: open or WEP networks only; WPA2 support exists only via DSi-mode work in recent BlocksDS/dswifi. [MEDIUM — verify current dswifi WPA2 status in DSi mode before promising it]
- Local DS-to-DS wireless (NiFi): possible, poorly documented, packet-loss-prone. Treat as experimental. [MEDIUM]
- Anti-patterns:
  - Do NOT assume `fatInitDefault()` succeeded — on emulators without a mounted image, or unpatched DLDI, it fails and every `fopen` returns NULL. [HIGH]
  - Do NOT write to the SPI firmware flash or (DSi) NAND. Ever. [HIGH]
  - Do NOT do many tiny fread()s — FAT over DLDI is slow; read in large blocks into main RAM. [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **BlocksDS** (recommended; install via `wf-pacman` or native packages) or **devkitARM** (install via devkitPro pacman). Both use prefix **`arm-none-eabi-`**. [HIGH]
- Two programs per ROM: an ARM9 ELF and an ARM7 ELF (use the SDK's prebuilt default ARM7). `ndstool` fuses them into the final `.nds`. [HIGH]
- Canonical flags (ARM9):
  - CPU: `-march=armv5te -mtune=arm946e-s`
  - Float: `-mfloat-abi=soft` (there is no FPU — never use hard-float) [HIGH]
  - Interworking/Thumb: `-mthumb -mthumb-interwork` (Thumb for size; `ARM_CODE`/ITCM for hot paths)
  - Defines: `-DARM9`; link with SDK's `ds_arm9.specs`/linker script, `-lnds9` (+ `-lmm9` for maxmod, `-lfat`, `-ldswifi9` as needed) [HIGH]
- ARM7 (only if custom): `-mcpu=arm7tdmi -DARM7`, `-lnds7`. [HIGH]
- Output chain: `arm-none-eabi-gcc → .elf → ndstool -c game.nds -9 arm9.elf -7 arm7.elf -b icon.bmp "Title;Sub;Author" [-d nitrofs_dir/]`. [HIGH]
- Environment: devkitARM needs `DEVKITPRO`/`DEVKITARM`; BlocksDS needs `BLOCKSDS` (or wf default paths). Start every project from the SDK's template Makefile — do not hand-roll. [HIGH]
- Asset pipeline (preprocess at build time, never at runtime):
  - Images → tiles/maps/palettes with **grit** (devkitPro) or BlocksDS converters (`grit`/`squeezer`); 3D textures → DS texel formats offline. [HIGH]
  - Audio → `mmutil` builds `soundbank.bin`/`soundbank.h` from .mod/.xm/.it/.wav. [HIGH]
  - 3D models → convert to display lists / packed vertex commands offline (no runtime OBJ parsing). [HIGH]
- Anti-patterns:
  - Do NOT compile with hard-float or `-march` newer than armv5te for ARM9 code. [HIGH]
  - Do NOT link ARM9-only libs into the ARM7 binary or vice versa (`-lnds9` vs `-lnds7`). [HIGH]
  - Do NOT use `double` casually — it's soft-float poison in inner loops. Fixed-point first. [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary: **melonDS** — the accuracy reference today; includes a **GDB stub** for source-level debugging, plus DLDI/SD image and Wi-Fi emulation. [HIGH]
- Secondary: **no$gba** — superb debugger UI, register/VRAM viewers, and its GBATEK docs are the platform bible. DeSmuME is fine for quick runs but less accurate in 3D edge cases. [HIGH]
- Safe to rely on (melonDS): CPU behavior, 2D engines, most 3D rasterization quirks, timers, IPC/FIFO, audio in broad strokes. [MEDIUM]
- Must test on hardware:
  - **Cache-coherency bugs** — emulators typically don't model the ARM9 data cache, so missing `DC_FlushRange` works in emu and corrupts on hardware. This is the #1 "works in melonDS, broken on DS" cause. [HIGH]
  - Main-RAM bus contention / real performance (emu FPS ≠ hardware FPS). [HIGH]
  - DLDI/flashcart timing and SD driver behavior. [HIGH]
  - Wi-Fi radio realities, mic input quality, exact LCD gamma. [MEDIUM]
- Hardware debugging: `consoleDemoInit()` + printf on the sub-screen; libnds `defaultExceptionHandler()` prints a **Guru Meditation** (register dump + address) on data/prefetch abort — install it in every debug build; melonDS GDB stub for pre-hardware stages; no commodity JTAG. [HIGH]
- Anti-pattern: Do NOT profile or "optimize" based on emulator frame rate; measure with hardware + `cpuStartTiming()/cpuEndTiming()`. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Two white/frozen screens at boot | Any launch | Crash before video init, or ARM9/ARM7 binary mismatch | Install `defaultExceptionHandler()` early; init console first; rebuild both CPUs from same SDK | [HIGH] |
| Guru Meditation (data abort) | During play | NULL/garbage pointer, or >16 KB stack use in DTCM | Read the dumped PC/LR in the .map file; move big locals to heap | [HIGH] |
| Textures fine in melonDS, garbage on hardware | 3D, after loading | Missing `DC_FlushRange` before DMA/GPU consumed the data | Flush data cache after every CPU write that hardware/DMA reads | [HIGH] |
| Graphics half-written / odd bytes zero | Writing VRAM with memcpy | 8-bit writes to VRAM ignored | Use `dmaCopy`/`swiCopy`/u16 loops | [HIGH] |
| 3D objects randomly vanish | Heavy scenes | >2048 polys or >6144 verts submitted | Cull with BoxTest, cut geometry, LOD | [HIGH] |
| Audio crackles/garbage after loading new SFX | Streaming/decoded audio | Stale data cache seen by ARM7 fetch | `DC_FlushRange` sample buffers before play | [HIGH] |
| Silence, everything else works | Audio init | Custom ARM7 without sound service, or maxmod soundbank not loaded/found | Use default ARM7; check `mmInitDefault` path & NitroFS/FAT init order | [HIGH] |
| `fopen` always NULL / "SD not detected" | Flashcart or emu | DLDI not patched, or `fatInitDefault()` failed and was ignored | Launch via auto-patching loader; check init return; in melonDS mount an SD image | [HIGH] |
| Crash when loading a large file | Asset loading | Buffer on DTCM stack, or heap exhaustion in 4 MB RAM | Heap-allocate, stream in chunks, budget memory | [HIGH] |
| Input ignores X/Y/touch but D-pad works | Custom ARM7 builds | ARM7 side not sampling/forwarding via IPC | Use libnds default ARM7 or add input service to yours | [HIGH] |
| Whole game runs at half speed suddenly | Frame loop | Missed VBlank (frame took >16.7 ms) → hard 30 FPS step | Profile with cpuStartTiming; move hot code to ITCM; reduce main-RAM traffic | [HIGH] |
| Bottom-screen UI appears on top screen (or swapped) | Video init | Engine A/B vs physical screen mapping confusion | `lcdMainOnBottom()/lcdMainOnTop()` explicitly | [HIGH] |
| Runs on emu, black screens on DSi via TWiLight | DSi-mode launch | ROM header/DSi flags or unsupported slot-2 access on DSi | Build with current SDK defaults; guard slot-2 code behind hardware detection | [MEDIUM] |
| Sprites flicker/tear | OAM updates | Writing OAM mid-frame | Buffer sprite state; commit OAM during VBlank (`oamUpdate` after `swiWaitForVBlank`) | [HIGH] |

---

## 11. ANTI-PATTERNS

1. Do NOT use `float`/`double` in per-frame code — there is no FPU; use libnds fixed-point (f32/v16) and the hardware divide/sqrt registers. [HIGH]
2. Do NOT write bytes to VRAM, palette RAM, or OAM — 8-bit writes are ignored. [HIGH]
3. Do NOT DMA or hand data to the GPU/ARM7 without `DC_FlushRange` first — emulators will hide this bug from you. [HIGH]
4. Do NOT allocate large arrays on the stack — the ARM9 stack is 16 KB of DTCM. [HIGH]
5. Do NOT treat libnds' `gl*` API as OpenGL — no shaders, no VBOs, no glReadPixels, hardware limits everywhere. [HIGH]
6. Do NOT exceed 2048 polygons/6144 vertices per frame or assume overflow errors — geometry silently drops. [HIGH]
7. Do NOT parse/convert assets at runtime — preconvert with grit/mmutil/offline tools into DS-native formats. [HIGH]
8. Do NOT assume a filesystem exists — check `fatInitDefault()`/`nitroFSInit()` return values and design a failure screen. [HIGH]
9. Do NOT busy-wait the frame; call `swiWaitForVBlank()` (also saves battery). [HIGH]
10. Do NOT touch the SPI firmware flash or DSi NAND from homebrew. [HIGH]
11. Do NOT write your own ARM7 binary until the default one provably blocks you. [HIGH]
12. Do NOT assume slot-2 (GBA slot) exists — DSi and later have none; feature-detect rumble/RAM paks. [HIGH]
13. Do NOT update OAM, scroll registers, or VRAM mid-scanout; batch into VBlank. [HIGH]
14. Do NOT rely on unaligned memory access — ARMv5TE rotates loads instead of faulting. [HIGH]
15. Do NOT ship without `defaultExceptionHandler()` in debug builds — a Guru Meditation dump is your only crash telemetry on hardware. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Memory audit first.** Sum the engine's static + peak heap needs against 4 MB total (≈3.5 MB usable). Why first: nothing else matters if it can't fit; DS ports live or die on this. Skip it and you discover mid-port that assets must be redesigned. Plan asset budgets: VRAM banks for textures (≤512 KB), main RAM for everything else.
2. **Kill floating point.** Convert math cores to fixed-point (or verify soft-float is only in cold paths). Why second: it changes data types everywhere; retrofitting later means touching every file twice. Skipping it yields a port that "works" at 4 FPS.
3. **Stand up the toolchain + hello world on hardware.** BlocksDS template → .nds → real DS via flashcart. Why: validates build, DLDI, and your deploy loop before any engine code exists. Skipping it means debugging engine *and* pipeline simultaneously.
4. **Filesystem layer.** Route the engine's file I/O through NitroFS (read-only assets) + FAT (saves). Convert to large-block reads. Why now: everything downstream (assets, config) depends on it.
5. **Renderer rewrite, not translation.** Map the engine's draw path onto DS reality: 2D → tiles/sprites where possible; 3D → display lists, ≤2048 polys, VRAM-resident textures in DS formats, BoxTest culling. Why: the DS renderer is architecturally different; a shim over a PC-style renderer will blow every limit. Skipping the rethink = missing geometry and single-digit FPS.
6. **Audio via maxmod/hardware channels.** Replace software mixers with hardware channels; convert music to tracker formats or streamed ADPCM. Skipping = ARM9 CPU burned on mixing you get for free.
7. **Input mapping.** Map to D-pad/ABXY/L/R + touch; design for no analog. Consider touch as mouse-look/aim. 
8. **Performance pass on hardware.** ITCM for hot code, DTCM for hot data, minimize main-RAM traffic, batch DMA. Only now — profile-guided, on real hardware.
9. **DSi-mode enhancements (optional).** 16 MB RAM / 134 MHz unlocks bigger ports; gate features on hardware detection.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
NINTENDO DS — ARM946E-S 67MHz (ARMv5TE, 8K/4K cache, 32K ITCM, 16K DTCM=stack)
           + ARM7TDMI 33MHz (owns touch/audio/wifi/RTC; use libnds default ARM7)
NO FPU — fixed-point only. Little-endian. No MMU/VM. HW div/sqrt via registers.
RAM: 4MB @0x02000000 (16-bit bus, slow; DSi=16MB) | VRAM 656KB banks A-I (map manually)
     Shared WRAM 32KB @0x03000000 | I/O @0x04000000 | OAM @0x07000000
GPU: 2x 2D engines (256x192@59.83Hz each, 128 sprites ea) + 1 fixed-function 3D engine
     3D: 2048 polys/6144 verts per frame HARD CAP, scanline renderer (no 3D framebuffer),
     tex ≤1024², DS-native formats only, textures live in VRAM banks, no shaders.
AUDIO: 16 HW channels (PCM8/16, IMA-ADPCM, PSG, noise) on ARM7; use maxmod + mmutil.
STORAGE: libfat+DLDI (flashcart SD), NitroFS (embedded assets), dswifi (802.11b).
BUILD: BlocksDS or devkitARM; arm-none-eabi-; ARM9: -march=armv5te -mfloat-abi=soft;
       elf → ndstool → game.nds; assets via grit/mmutil at build time.
CARDINAL RULES: DC_FlushRange before DMA/ARM7 reads (emu hides this!);
       no 8-bit VRAM writes; stack=16KB DTCM (no big locals); swiWaitForVBlank each frame;
       defaultExceptionHandler() in debug builds (Guru Meditation dump).
EMU: melonDS (accuracy + GDB stub), no$gba (debugger + GBATEK docs). Trust hardware only
     for cache bugs and performance. Docs: GBATEK. Never touch firmware flash/NAND.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this skill is used in an active development session (not just reference), you must follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** — Examine the codebase, identify DS-specific blockers (floats, memory, renderer, I/O), and plan the implementation.
- **STATE: IMPLEMENTATION** — Write/modify code based on this document and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** — Output the exact build commands and ask the user to compile and run on real hardware (flashcart / TWiLight Menu++).
- **STATE: WAITING_FOR_HARDWARE** — STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** — User reports back results. Classify as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** — On SUCCESS, propose the next milestone and ask for confirmation.

### STATE TRANSITION RULES
1. **ANALYSIS → IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION → BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change, including:
   - Exact `make` command (BlocksDS/devkitARM template)
   - Expected output file name and location (e.g., `build/game.nds`)
   - How to transfer to hardware (copy .nds to flashcart microSD / DSi SD; launch via flashcart kernel or TWiLight Menu++)
   - What the user should observe on screens/audio/input
3. **BUILD_REQUEST → WAITING_FOR_HARDWARE**: You MUST output this exact header, then STOP generating (no fixes, no guesses):
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Solid-color framebuffer on main screen"]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including Guru Meditation dumps, white screens, freezes]
   === STOP ===
   ```
4. **WAITING_FOR_HARDWARE → VALIDATION**: Triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - "SUCCESS" → NEXT_GOAL. "FAILURE" → DEBUG_PROTOCOL.
   - Anything else ("kind of works", "almost") → ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) → DEBUG_PROTOCOL**: Output a **DEBUG BUILD PROTOCOL**:
   - A minimal C test case isolating the failure (e.g., bare `videoSetMode(MODE_FB0)` + solid fill), OR
   - A checklist of exactly 3 diagnostic steps (e.g., "Confirm `fatInitDefault()` returns true and print it", "Add `defaultExceptionHandler()` and report the Guru Meditation PC value", "Run the same .nds in melonDS with an SD image and compare").
   - Ask the user to run the diagnostic and report back. Do NOT rewrite the implementation. Do NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL → WAITING_FOR_HARDWARE**: After providing the protocol, return to WAITING_FOR_HARDWARE.
7. **VALIDATION (SUCCESS) → NEXT_GOAL**: Propose the next milestone from the goal stack. Do NOT implement until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [melonDS GDB stub / no$gba trace / Guru Meditation register dump / gbadev Discord-forum].`
- Ask the user whether to (a) mark the goal BLOCKED and skip, or (b) provide emulator logs / register dumps for further analysis.

### GOAL STACK (default if the user provides none)
1. Initialize video output (solid color framebuffer, `MODE_FB0`)
2. Initialize controller input (print button presses via `consoleDemoInit`)
3. Initialize audio output (play a sample/PSG tone via maxmod or `soundPlaySample`)
4. Load assets from SD/NitroFS (verify `fatInitDefault`/`nitroFSInit`)
5. Render main menu (2D background + sprites)
6. Main menu input loop (D-pad + touch)
7. Transition to game state

You must ONLY work on the active goal (top of stack). You must NOT implement future goals speculatively.
