---
name: wii
description: Nintendo Wii homebrew development — architecture, graphics, input, memory, audio, build system, debugging, and hardware validation gates.
---

# SKILLS_Wii.md – Nintendo Wii Homebrew Expertise Document

You are operating as a Nintendo Wii homebrew developer. Follow every rule in this document. Prefer libogc/devkitPPC semantics over anything you know from PC, mobile, or other consoles. When this document conflicts with your general training data, this document wins.

---

## 1. HARDWARE ARCHITECTURE

### CPU – "Broadway"
- IBM PowerPC 750CL derivative ("Broadway"), 729 MHz, 32-bit, **big-endian**. [HIGH]
- Superscalar in-order-ish (750-class: limited OoO completion), short pipeline – branch-heavy code runs comparatively well; deep unrolling helps less than on modern CPUs. [HIGH]
- Caches: 32 KB L1 instruction + 32 KB L1 data (8-way), 256 KB L2. L1 D-cache line size is **32 bytes**. [HIGH]
- FPU: full hardware double/single-precision FPU. You must build hard-float. [HIGH]
- SIMD: **paired singles** – two packed 32-bit floats per FPR, with dedicated `ps_*` instructions and quantized load/store (GQR registers). Enabled via `-mpaired`-style intrinsics or hand asm; GCC support is partial, so hot paths usually use inline asm or the libogc `ps` helpers. [HIGH]
- No AltiVec. Do NOT emit AltiVec/VMX code. [HIGH]
- Quirk: paired-single quantized loads (`psq_l`) require GQR setup; misconfigured GQRs silently corrupt float data. [MEDIUM]
- Quirk: unaligned floating-point loads/stores can raise alignment exceptions in some cases; keep float data naturally aligned. [MEDIUM]

### GPU – "Hollywood" package / GX
- The Hollywood package contains the **GX** graphics core (an overclocked GameCube "Flipper", ATI/ArtX design) at 243 MHz, plus the Starlet I/O processor (see co-processors). [HIGH]
- **Fixed-function only. There are no shaders, no shader model, no GLSL/HLSL.** Programmability comes from the **TEV** (Texture EnVironment) unit: up to **16 TEV stages**, each a configurable color/alpha combiner over up to 8 textures and 2 rasterized colors. [HIGH]
- Embedded memory: **~2 MB EFB (embedded framebuffer)** + **1 MB TMEM (texture cache)**. You render into the EFB, then **copy** to an XFB (external framebuffer) in main RAM for display. [HIGH]
- EFB maximum size: 640×528 pixels. Practical modes: 640×480 (NTSC), 640×528 (PAL). No true HD output – component cable gives 480p at best. [HIGH]
- EFB pixel formats: RGB8 + 24-bit Z, or RGBA6 + 24-bit Z (destination alpha requires the RGBA6 mode), or RGB565 + 16-bit Z ("pixel format Z16"). You cannot have full RGBA8 + destination alpha simultaneously. [HIGH]
- Texture formats: I4, I8, IA4, IA8, RGB565, RGB5A3, RGBA8, CI4/CI8/CI14 (palettized via TLUT), and **CMPR** (4-bit S3TC/DXT1-style block compression). Max texture size 1024×1024. [HIGH]
- All textures are stored **tiled/swizzled in 4×4 texel blocks** (32-byte tiles), not linear. You must swizzle on upload or preprocess offline. RGBA8 textures are stored as two AR/GB planes per tile. [HIGH]
- Depth: 24-bit Z in EFB, with Z-compression modes (ZC_LINEAR, ZC_NEAR/MID/FAR). Full Z test/write control via `GX_SetZMode`. [HIGH]
- Stencil: **no stencil buffer.** Emulate with destination alpha or multi-pass tricks. [HIGH]
- Blending: standard src/dst factor blending, plus logic ops and subtract mode, via `GX_SetBlendMode`. Destination alpha blending only in RGBA6 EFB mode. [HIGH]
- Indirect texturing: 4 indirect stages enable EMBM/distortion effects. [HIGH]
- Command submission: CPU writes GX commands into a **FIFO** (write-gather pipe at 0xCC008000); libogc's GX API wraps this. Display lists are pre-recorded command buffers replayed by the GP. [HIGH]

### RAM
- **MEM1: 24 MB of 1T-SRAM** – low latency (~10 ns class), the "fast" pool. Physical 0x00000000–0x017FFFFF. [HIGH]
- **MEM2: 64 MB GDDR3** – higher latency, higher capacity. Physical 0x10000000–0x13FFFFFF. GX can texture directly from both. [HIGH]
- Virtual memory: the MMU is used only for **static BAT-style mapping**, not paging. There is **no swap and no demand paging**. Effective addresses:
  - MEM1 cached: `0x80000000`, MEM1 uncached: `0xC0000000` [HIGH]
  - MEM2 cached: `0x90000000`, MEM2 uncached: `0xD0000000` [HIGH]
  - Hardware registers: `0xCC000000` region (VI, PI, DSP, EXI, AI, GX FIFO), Hollywood/IOS registers at `0xCD000000`. [HIGH]
- Alignment: **32-byte alignment is the platform's magic number** – GX FIFO, display lists, texture data, XFB, DMA buffers, and IOS IPC buffers all require 32-byte alignment (use `memalign(32, ...)` and pad sizes to multiples of 32). [HIGH]
- A chunk of MEM2 (top ~12–16 MB depending on IOS) is **reserved for IOS**; libogc's arena reflects this. Do not assume all 64 MB is yours. [HIGH]

### Bus topology
- Broadway – Hollywood over a 64-bit bus (~243 MHz); GX, VI, AI, EXI, and the memory controller live in Hollywood. All CPU access to RAM and I/O goes through Hollywood. [HIGH]
- GX reads vertex/texture data from main RAM via DMA; the CPU–GP write-gather FIFO is the primary command path. [HIGH]
- EFB–XFB copy is a dedicated hardware blit (`GX_CopyDisp`) that can also downsample (AA) and Y-scale. [HIGH]
- Bottlenecks: CPU-side vertex submission (immediate mode `GX_Position3f32` calls are function-call-bound), and MEM2 latency for CPU random access. Prefer display lists / indexed arrays in MEM1 for hot geometry. [HIGH]

### Co-processors
- **Starlet (ARM926EJ-S)** inside Hollywood runs **IOS**, the I/O operating system. It owns SD, USB, Wi-Fi, Bluetooth, NAND, and crypto hardware. The PPC talks to it via an **IPC mailbox** interface (`/dev/...` resource nodes, `IOS_Open/IOS_Ioctl(v)`). libogc wraps this. [HIGH]
- **DSP**: a Macronix 16-bit DSP for audio mixing/decoding. It runs uploaded microcode ("ucode"); libogc ships a mixer ucode used by ASND/AESND. CPU–DSP via mailbox registers + ARAM-less DMA (Wii has no GameCube ARAM; the DSP DMAs from main RAM). [HIGH]
- **AI (Audio Interface)**: streams a 16-bit stereo PCM buffer from main RAM to the DAC at 32 or 48 kHz via DMA. [HIGH]

### Security / DRM
- Boot chain: boot0 (mask ROM) – boot1 – boot2 – IOS – System Menu. RSA signature checks at each stage; the famous strncmp/fakesign bug is patched on later boot1s. [HIGH]
- Homebrew entry: Homebrew Channel (HBC) installed via exploits; it launches DOLs with **AHBPROT** left disabled, giving the PPC full hardware access (direct SD/USB/NAND registers) if you keep the rights. [HIGH]
- With AHBPROT disabled you can access nearly all hardware from PPC. Without it, you must go through IOS IPC. Reloading IOS (`IOS_ReloadIOS`) **loses AHBPROT** unless patched. [HIGH]
- Homebrew CANNOT: run unsigned code on Starlet without an exploit (bootmii/mini), or use official Nintendo network services. NAND writes are possible but are the #1 way to brick consoles – treat NAND as read-only. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official: Nintendo **Revolution SDK (RVL SDK)** with CodeWarrior/GX libraries. You must NOT use, reference, or assume access to it. All guidance here targets the community stack. [HIGH]
- Homebrew: **devkitPPC** (GCC cross-toolchain) + **libogc** (OS, VIDEO, GX, WPAD/PAD, AESND, IOS IPC) + satellite libs: **libfat** (FAT filesystems), **wiiuse/libbte** (Wiimote Bluetooth), **asndlib/aesndlib** (audio), **libwiikeyboard**, **GRRLIB** (2D helper). All under permissive/zlib-style licenses; install via devkitPro pacman. [HIGH]
- libogc CAN: full GX access (equivalent in capability to official GX – same hardware, near-identical API naming), threading (LWP), IOS IPC, SD/USB/network, Wiimote with MotionPlus. [HIGH]
- libogc CANNOT / gaps:
  - No official-quality documentation – the header files and existing repos ARE the documentation. [HIGH]
  - No NAND-title/WAD tooling in-core (community tools exist, high brick risk). [HIGH]
  - DSP: only the shipped mixer ucode; writing custom DSP ucode is expert territory with sparse docs. [MEDIUM]
  - Wi-Fi networking works (libogc `net_*`) but is slower and flakier than official stacks; no WPA3. [MEDIUM]
- Decision impact: because GX has no shaders, **any engine assuming programmable shaders needs its material system re-expressed as TEV stage configurations** – this is the single biggest porting cost on Wii. [HIGH]

---

## 3. GRAPHICS PIPELINE

- API: **libogc GX** (`<ogc/gx.h>`). Do NOT use OpenGL. OpenGX exists as a GL1.x translation layer but costs ~30–50% performance versus native GX in real ports; use it only for bootstrapping, then migrate hot paths to native GX. [HIGH]

### Initialization (canonical sequence)
```c
VIDEO_Init();
GXRModeObj *rmode = VIDEO_GetPreferredMode(NULL);
void *xfb[2] = {
    MEM_K0_TO_K1(SYS_AllocateFramebuffer(rmode)),
    MEM_K0_TO_K1(SYS_AllocateFramebuffer(rmode)),
};
VIDEO_Configure(rmode);
VIDEO_SetNextFramebuffer(xfb[0]);
VIDEO_SetBlack(FALSE);
VIDEO_Flush();
VIDEO_WaitVSync();
if (rmode->viTVMode & VI_NON_INTERLACE) VIDEO_WaitVSync();

void *gp_fifo = memalign(32, 256*1024);   // 32-byte aligned, >=64 KB
memset(gp_fifo, 0, 256*1024);
GX_Init(gp_fifo, 256*1024);
GX_SetCopyClear((GXColor){0,0,0,255}, GX_MAX_Z24);
GX_SetViewport(0,0,rmode->fbWidth,rmode->efbHeight,0,1);
GX_SetScissor(0,0,rmode->fbWidth,rmode->efbHeight);
GX_SetDispCopySrc(0,0,rmode->fbWidth,rmode->efbHeight);
f32 yscale = GX_GetYScaleFactor(rmode->efbHeight, rmode->xfbHeight);
u32 xfbHeight = GX_SetDispCopyYScale(yscale);   // required when efbHeight != xfbHeight
GX_SetDispCopyDst(rmode->fbWidth, xfbHeight);
GX_SetCopyFilter(rmode->aa, rmode->sample_pattern, GX_TRUE, rmode->vfilter);
GX_SetFieldMode(rmode->field_rendering,
                (rmode->viHeight == 2 * rmode->xfbHeight) ? GX_ENABLE : GX_DISABLE);
GX_SetPixelFmt(GX_PF_RGB8_Z24, GX_ZC_LINEAR);
```
[HIGH – this sequence is HW-proven in ioQuake3-wii for default, 240p Ds and 264p Ds modes]
- Check `SYS_AllocateFramebuffer()` for NULL **before** `MEM_K0_TO_K1` – the macro destroys the NULL. [HIGH]

### Frame flow
- Render into **EFB** – `GX_CopyDisp(xfb[cur], GX_TRUE)` copies (and clears) EFB–XFB – `GX_DrawDone()` – `VIDEO_SetNextFramebuffer(xfb[cur]); VIDEO_Flush();` – flip index. Double-buffer XFBs; the EFB itself is single. [HIGH]
- `VIDEO_WaitVSync()` per frame is **optional and has a cost**: it locks the frame rate to 60 Hz multiples, so a 17 ms frame stalls to 33 ms (FPS halves). ioQuake3-wii removed it; `GX_DrawDone()` before `VIDEO_SetNextFramebuffer`, two XFBs, and an engine frame cap give correct pacing. [HIGH – HW-confirmed]
- Do NOT add a `VIDEO_GetCurrentFramebuffer()` "XFB still in scanout" guard to a non-vsync loop without first logging its trip rate and pointer values – on real hardware it made frame pacing unplayable (cause not diagnosed). [HIGH – HW-confirmed]
- The XFB is **YUV (YCbCr 4:2:2)**, written by the copy engine – you never write RGB directly to XFB except via libogc's `CON_*` console or careful YUV code. [HIGH]

### Refresh / regions
- NTSC: 640×480, 60 Hz (interlaced or 240p). PAL: 640×528, 50 Hz, plus **PAL60/EuRGB60** 480i@60. Progressive 480p only with component cable and `VIDEO_HaveComponentCable()`. Always use `VIDEO_GetPreferredMode` instead of hardcoding. [HIGH]
- Widescreen: console setting readable via `CONF_GetAspectRatio()`; the hardware does NOT render wider – you render anamorphic 640-wide and the TV stretches. Adjust your projection matrix. [HIGH]
- 240p/264p: `TVNtsc240Ds` / `TVPal264Ds` work on hardware (CRTs, RetroTINK/OSSC; many flat panels reject them). If you substitute a Ds mode for a preferred mode, match the broadcast format (`viTVMode >> 2`: NTSC/PAL/MPAL/EURGB60). A EURGB60 console forced to `TVNtsc240Ds` gave a stretched black-and-white picture. [HIGH – HW-confirmed]
- Overscan: inset the viewport and scissor render-side. Do NOT edit `rmode` fields (`viXOrigin`/`viWidth`/...) or `VIDEO_Configure` for overscan – that is a flicker/FPS trap. Top-line crop on true interlaced modes over composite happens in the TV's deinterlacer and no render-side inset reaches it. [HIGH – HW-confirmed]

### Textures
- Upload: `GX_InitTexObj` + `GX_LoadTexObj`. Data must be tiled (see §1), 32-byte aligned, and **flushed from D-cache (`DCFlushRange`) before GX reads it**. [HIGH]
- Prefer **CMPR** for diffuse maps (4bpp, huge TMEM/bandwidth win); RGB5A3 for UI with alpha; RGBA8 only when quality demands (it costs 2 TMEM lookups). [HIGH]
- CMPR caveat: runtime CMPR encoding through OpenGX's `GX_TF_CMPR` path **crashed on real hardware** in ioQuake3-wii. The HW-proven runtime path there is RGB565 (opaque) / RGB5A3 (alpha) swizzled from the engine's already-downscaled buffer, with a CPU-built mip chain. Full-size RGBA8 uploads exhausted the heap. RGB5A3 translucent texels keep only 3-bit alpha. [HIGH – HW-confirmed]
- Texture buffers that the engine deletes (map change, `vid_restart`) need `memalign(32)`/`free`. Do not serve them from a one-way bump allocator. [HIGH – HW-confirmed]
- Mipmaps: supported and strongly recommended; without them GX thrashes TMEM on minified textures – a known real-world slowdown cause. [HIGH]

### Geometry
- Three submission paths: immediate mode (`GX_Begin`/`GX_Position3f32`.../`GX_End`) – slowest; **indexed arrays** (`GX_SetArray` + index streams) – good; **display lists** (`GX_BeginDispList`/`GX_CallDispList`) – fastest for static geometry. Display lists must be 32-byte aligned, padded to 32-byte multiples, and `DCFlushRange`d after recording. [HIGH]
- Vertex formats are described via VAT (`GX_SetVtxAttrFmt`) – supports compressed attributes (s8/s16 fixed-point positions, u8 colors). Use compressed formats; bandwidth matters more than ALU. [HIGH]
- Indexed arrays: call `GX_InvVtxCache()` before an indexed draw whose arrays were rewritten at the same address, and after any `GX_DIRECT`↔`GX_INDEX*` descriptor switch. Without it: garbled text and malformed models on hardware. If the CPU rewrites the arrays right after submit, copy them to a staging buffer or call `GX_DrawDone()` before returning. [HIGH – HW-confirmed]

### TEV in one paragraph
- Each of up to 16 stages computes `output = (d op ((1-c)*a + c*b)) * scale + bias` independently for color and alpha, with inputs selectable from texture, rasterized color, konst registers, and previous stage. Typical Q3-style diffuse×lightmap = 2 textures, 2 stages (or 1 stage with 2 texcoords collapsed). Multi-pass PC blending often collapses into fewer TEV stages – always try to collapse before adding passes. [HIGH]

### Anti-patterns (GPU)
- Do NOT use OpenGL/OpenGX for hot render paths in production. [HIGH]
- Do NOT forget `DCFlushRange` on textures, vertex arrays, and display lists – Dolphin will hide this, hardware will show garbage. [HIGH]
- Do NOT read the EFB per-pixel from CPU in a loop (`GX_PeekARGB`) for effects – it is extremely slow. [HIGH]
- Do NOT exceed 640×528 render size or assume HD. [HIGH]

---

## 4. INPUT

- Controller types: **Wii Remote** (buttons, 3-axis accelerometer, IR pointer, speaker, rumble) + expansions: **Nunchuk** (stick + accel), **Classic Controller (Pro)**, **MotionPlus** (gyro); **GameCube controllers** on the 4 GC ports (original Wii only, RVL-101 lacks them); USB keyboards; Balance Board. [HIGH]
- Two separate libogc APIs, both **polled**:
  - `PAD_Init(); PAD_ScanPads(); PAD_ButtonsDown(0); PAD_StickX(0);` – GameCube pads. [HIGH]
  - `WPAD_Init(); WPAD_ScanPads(); WPAD_ButtonsDown(WPAD_CHAN_0);` – Wiimotes. [HIGH]
- Call `WPAD_ScanPads()`/`PAD_ScanPads()` **exactly once per frame**, then use `ButtonsDown` (edge), `ButtonsHeld` (level), `ButtonsUp`. [HIGH]
- Rich data: `WPAD_SetDataFormat(chan, WPAD_FMT_BTNS_ACC_IR)` then `WPAD_Data(chan)` – `WPADData*` with `.ir` (pointer x/y, valid flag), `.accel`, `.orient`, `.gforce`, `.exp` (expansion union: `.exp.nunchuk`, `.exp.classic`). Set `WPAD_SetVRes(chan, 640, 480)` for screen-space IR. [HIGH]
- MotionPlus: `WPAD_SetMotionPlus(chan, 1)`; gyro data in `exp.mp`. Calibration drift is your problem. [MEDIUM]
- Expansion hot-plug: check `data->exp.type` **every frame** – nunchuks are plugged/unplugged at runtime. `WPAD_Probe(chan, &type)` reports connection status; handle `WPAD_ERR_NO_CONTROLLER`. [HIGH]
- Rumble: `WPAD_Rumble(chan, 1/0)` and `PAD_ControlMotor(chan, PAD_MOTOR_RUMBLE/STOP)`. No force curves – it's on/off; PWM it yourself for intensity. [HIGH]
- Wiimote speaker: exists, ADPCM, poorly supported in libogc – avoid. [MEDIUM]
- Classic Controller init race: a freshly attached CC often reports "no expansion" until you call `WPAD_SetDataFormat(chan, WPAD_FMT_BTNS_ACC_IR)` **again** after first detection. Re-send it once per attach (reset the flag on detach). Without it, CC input reads nothing for several seconds. [HIGH – HW-confirmed]
- USB HID gamepads (raw `ogc/usb.h`), HW-confirmed rules from ioQuake3-wii (PS3/PS4/DualSense work):
  - Do NOT call `USB_SetConfiguration()` – every real pad rejected it; IOS has already configured the device. [HIGH]
  - Every IPC buffer comes from `memalign(32, …)` heap or a `static` aligned array, never a stack local (even with an alignment attribute). Size the interrupt-IN buffer for the endpoint's max packet (a 32-byte buffer crashed on a 64-byte DS4 report). [HIGH]
  - No blocking IOS call and no file I/O inside an IOS async callback – set a `volatile` flag and act on the main thread. [HIGH]
  - Hotplug: poll `USB_GetDeviceList()` about once per second. `USB_DeviceChangeNotifyAsync()` froze the console on attach. [HIGH]
  - Vendor-class (0xFF) pads (Xbox 360/One/Series) crash the console on their first input report – a libogc/IOS limit, not app code. [HIGH]
  - First `USB_Initialize()`/raw USB use before network init hung the console in this port; init it after `net_init`/`if_config`. [MEDIUM]
- Anti-patterns:
  - Do NOT assume a Wiimote is connected at boot – pairing may complete seconds after your main loop starts. [HIGH]
  - Do NOT assume only one controller layout; support Wiimote-sideways, Wiimote+Nunchuk, Classic, and GC pad if feasible. [HIGH]
  - Do NOT poll WPAD from multiple threads. [MEDIUM]

---

## 5. MEMORY LAYOUT

| Region | Effective addr | Size | Notes |
|---|---|---|---|
| MEM1 cached | 0x80000000 | 24 MB | 1T-SRAM, fast; code+hot data here [HIGH] |
| MEM1 uncached | 0xC0000000 | 24 MB | alias; XFB access, MMIO-ish buffers [HIGH] |
| MEM2 cached | 0x90000000 | 64 MB | GDDR3; bulk assets, textures [HIGH] |
| MEM2 uncached | 0xD0000000 | 64 MB | alias [HIGH] |
| HW registers | 0xCC000000 | – | VI/PI/DSP/AI/EXI/GX FIFO [HIGH] |
| Hollywood/IPC | 0xCD000000 | – | IOS IPC, GPIO, AHBPROT [HIGH] |

- Allocation: `malloc`/`memalign` draw from **MEM1 first, then spill into MEM2 arena2** when arena1 is full (libogc `_sbrk_r` with the default `MALLOC_MEM2 == 0`; verified by disassembling libogc 3.0 `sbrk.o`). MEM2 via libogc arena: `SYS_GetArena2Lo()/SYS_GetArena2Hi()` or the `AllocateMEM2`-style wrappers many ports write (grab arena2, run your own allocator). If you carve a private MEM2 region, call `SYS_SetArena2Hi()` below it, or malloc's MEM2 spill collides with it. Top of MEM2 belongs to IOS – respect the arena bounds. [HIGH]
- The malloc heap and a private MEM2 region form a **static wall**: neither side can borrow from the other at runtime. Growing one shrinks the other (zero-sum on hardware). A floor for the malloc side is mandatory – a MEM2 reservation equal to all of arena2 crashed boot. [HIGH – HW-confirmed]
- Heap diagnostics: on this newlib target only `mallinfo().fordblks` is trustworthy; `arena`/`uordblks` read back implausibly large. [HIGH – HW-observed]
- vWii (Wii U) exposes the same 88 MB (24 MEM1 + 64 MEM2) and a fixed CPU clock. There is no extra RAM to unlock from a vWii app. [HIGH]
- Rule of thumb: code, stack, vertex/display lists, and per-frame data in MEM1; textures, audio assets, file caches in MEM2. GX textures work fine from MEM2. [HIGH]
- Stack: the main-thread default is small (current devkitPPC `libogc_common.ld` defines no stack-size symbol; ioQuake3-wii records the default as 16 KB – verify on your toolchain). HW-proven override on current devkitPPC/libogc: define the crt0 hook symbol at file scope –
  `static unsigned char s_mainStack[512*1024] __attribute__((aligned(8)));`
  `void *__ppc_main_sp __attribute__((section(".sdata"))) = &s_mainStack[sizeof(s_mainStack)];`
  LWP threads take an explicit stack pointer/size at `LWP_CreateThread`. Deep-recursion engines (Q3 QVM, parsers) should get 512 KB+. [MEDIUM – override HW-confirmed; default size unverified]
- Cache: L1D is **write-back**, 32-byte lines, **no hardware coherency with GX/DMA/IOS**. Therefore:
  - `DCFlushRange(ptr, len)` – after CPU writes, before GX/DSP/IOS reads. [HIGH]
  - `DCInvalidateRange(ptr, len)` – before CPU reads data written by DMA/IOS (buffers must be 32-byte aligned AND 32-byte-multiple sized, or invalidation eats neighboring data). [HIGH]
  - `ICInvalidateRange(ptr, len)` – after writing code (loaders, JIT). A PPC JIT must DCFlush + ICInvalidate every emitted block. [HIGH]
- IOS IPC buffers (file I/O via `/dev/fs`, USB, etc.): 32-byte aligned, 32-multiple sized. libfat handles this internally but your custom IOS calls must too. [HIGH]
- Anti-patterns:
  - Do NOT `memcpy` into a display list or texture without flushing after. [HIGH]
  - Do NOT put multi-MB assets in MEM1 "because it's fast" – you'll starve the allocator; MEM1 speed matters for CPU-touched data, not GX-streamed textures. [HIGH]
  - Do NOT pass stack buffers to DMA/IOS – stack lines share cache lines with live locals. [HIGH]

---

## 6. AUDIO

- Hardware: AI streams 16-bit big-endian stereo PCM from main RAM at **32000 or 48000 Hz** via DMA; the DSP (with libogc's mixer ucode) mixes voices ahead of it. [HIGH]
- Recommended API: **AESND** (`<aesndlib.h>`) – modern, voice-based, low overhead; or **ASND** (`<asndlib.h>`) – older, simpler, used by many ports. SDL-Wii and most engine ports sit on ASND/AESND. [HIGH]
- Formats: signed 16-bit PCM (mono/stereo) is the safe path; 8-bit supported by ASND voices. Compressed formats (Ogg/MP3) are decoded on CPU (libtremor/libmad ports exist). [HIGH]
- Model: **callback-driven double buffering.** AESND calls your voice callback when a buffer drains; you refill and `DCFlushRange` it. Typical buffer: 2–4 KB per callback at 48 kHz. [HIGH]
- ASND caveat for engines with their own DMA-style mixer (Quake-family `snd_dma`): `ASND_SetVoice` + an IRQ callback that raced a main-thread `DCFlushRange` gave one-speaker-only stereo on hardware. HW-proven alternative: one looping `ASND_SetInfiniteVoice()` over a ring buffer, `ASND_GetTickCounterVoice()` (scaled from 48 kHz to the voice rate) as the read position, and `DCFlushRange` on the main thread after each mix. ASND voices resample, so a 22050 Hz voice plays correctly. [HIGH – HW-confirmed]- No separate audio RAM on Wii (unlike GameCube's ARAM) – everything streams from MEM1/MEM2. [HIGH]
- Glitch avoidance:
  - Keep the callback allocation-free and lock-free; feed it from a ring buffer filled on the main/worker thread. [HIGH]
  - Flush every buffer you hand to the DSP/AI (`DCFlushRange`) – stale cache = crackle/static. [HIGH]
  - Match your mixer rate to 32/48 kHz; resample offline, not per-frame. (ASND voices at other rates, e.g. 22050 Hz, are resampled by ASND and work – see the caveat above.) [HIGH]
- Anti-patterns: Do NOT decode Ogg inside the audio callback; do NOT assume little-endian WAV data plays correctly – byteswap PCM to big-endian. [HIGH]

---

## 7. STORAGE / IO

- Media: **front SD slot** (SD/SDHC; SDXC unreliable), **USB mass storage** (USB2 via IOS58), NAND (via IOS `/dev/fs` – treat read-only), DVD drive (unreliable for homebrew without cIOS on later drives – avoid as a requirement), network. [HIGH]
- **Wii Mini (RVL-201)** has no SD slot and no USB2-capable IOS: USB storage is the only data path and it is slow (USB 1.1 throughput). Probe `sd:/` then `usb:/`. Do not attempt an IOS reload for USB2 on a Mini. [HIGH – HW-confirmed booting from USB]
- Filesystem: **libfat API** – `fatInitDefault()` mounts `sd:/` and `usb:/` and wires stdio; then plain `fopen("sd:/apps/mygame/data.pak","rb")` works. FAT32 recommended; FAT is case-insensitive but **case-preserving** – do not rely on case to distinguish files, and keep asset paths lowercase for sanity. [HIGH]
- Current devkitPro ships the **dvm** backend behind `<fat.h>` (`<dvm.h>`). The convenience inits (`fatInitDefault()`, `dvmInit()`) mount **every** device found, and each mounted disc gets its own FAT cache. On a memory-tight port a USB drive plugged in for no reason ate enough heap to crash the app ("Not enough memory"). Mount per device instead: `dvmProbeMountDiscIface("sd", &__io_wiisd, pages, sectors_per_page)` (`<sdcard/wiisd_io.h>`), and only if needed `dvmProbeMountDiscIface("usb", &__io_usbstorage, …)` (`<ogc/usbstorage.h>`) in a retry loop – USB enumerates slower than SD. `dvmUnmountVolume()` a mount you will not use. Default cache is 4 pages × 64 sectors = 128 KB. [HIGH – HW-confirmed]
- Newer alternative: libogc2/its `fatfs` variants exist. [MEDIUM]
- HBC app layout (required):
```
sd:/apps/yourapp/boot.dol      (or boot.elf)
sd:/apps/yourapp/meta.xml      (name, version, coder, short/long description)
sd:/apps/yourapp/icon.png      (128x48 PNG)
```
[HIGH]
- Network: libogc `net_init()` then BSD-ish sockets (`net_socket`, `net_connect`, ...). Wi-Fi b/g via IOS; Ethernet via USB-LAN adapter. Throughput is modest (~1–2 MB/s realistic). `wiiload` (TCP port 4299 to HBC) is your fast deploy path. [HIGH]
- Network gotchas (HW-confirmed in ioQuake3-wii): `if_config()` with many DHCP retries before video is up black-screens boot on a console with no router – keep retries low (3) and treat the result as best-effort. `net_select()` with a zero timeout or an empty fd set can block/hang – skip it when there are no sockets. `net_socket()` needs protocol `0`. [HIGH]
- Anti-patterns:
  - Do NOT write to NAND unless you are writing a NAND tool on purpose; a bad write bricks consoles. [HIGH]
  - Do NOT assume the SD card is present or mounted – check `fatInitDefault()`'s return and fail gracefully. [HIGH]
  - Do NOT do synchronous file I/O on the render thread mid-game; SD latency spikes cause frame hitches. [HIGH]

---

## 8. BUILD SYSTEM

- Toolchain: **devkitPPC** (current release via devkitPro pacman: `pacman -S wii-dev`), GCC 13+ era. Prefix/triplet: **`powerpc-eabi-`** (`powerpc-eabi-gcc`, `powerpc-eabi-g++`). [HIGH]
- Environment (mandatory): `DEVKITPRO=/opt/devkitpro`, `DEVKITPPC=$DEVKITPRO/devkitPPC`. [HIGH]
- Compiler flags (canonical Wii set):
```
-mrvl -mcpu=750 -meabi -mhard-float -DGEKKO
```
plus your `-O2 -ffunction-sections`. `-mrvl` selects the Wii machine config. Do NOT use soft-float. [HIGH]
- Linker: `-mrvl -meabi -mhard-float` again, libs in this order pattern:
```
-lwiiuse -lbte -lfat -laesnd (or -lasnd) -logc -lm
```
(`wiiuse` needs `bte`, everything needs `ogc` last-ish). Library/include paths: `$DEVKITPRO/libogc/lib/wii`, `$DEVKITPRO/libogc/include`. [HIGH]
- Output: link to **ELF**, convert with **`elf2dol`** to `boot.dol`. HBC runs both, but DOL is the standard. [HIGH]
- Makefile: start from `$DEVKITPRO/examples/wii/template/Makefile` – it includes `$(DEVKITPPC)/wii_rules` which defines all of the above. Do not hand-roll unless you must; CMake works via the devkitPro toolchain file (`$DEVKITPRO/cmake/Wii.cmake`). [HIGH]
- Asset pipeline:
  - Textures: pre-convert to GX tiled formats offline where possible (`gxtexconv` ships with devkitPro; TPL files) – runtime swizzling of PNGs costs load time. [HIGH]
  - Audio: pre-convert to 16-bit big-endian PCM at 32/48 kHz. [HIGH]
  - Data files: remember **big-endian** – either byteswap binary formats at load or preprocess. This is the #1 silent porting bug from PC codebases. [HIGH]
- Anti-patterns: Do NOT compile with `-mcpu=powerpc` generic (loses paired-single/750 tuning); do NOT link desktop libm/pthreads assumptions – threading is LWP, not pthreads (a thin pthread wrapper exists but is partial). [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary: **Dolphin**. Accuracy is excellent for retail-game GPU behavior; very good for libogc homebrew. [HIGH]
- Safe to rely on: GX rendering semantics (TEV, formats), general control flow, WPAD emulation basics, DOL loading, virtual SD image mounting, logging (`OSReport`/stderr visible in Dolphin log). [HIGH]
- NOT safe to rely on:
  - **Cache coherency.** Dolphin (default JIT) does not emulate the D-cache; missing `DCFlushRange`/`DCInvalidateRange` bugs are invisible in Dolphin and fatal on hardware. This is the classic "works in Dolphin, garbage on Wii" cause. [HIGH]
  - **Performance.** Dolphin FPS bears no relation to hardware FPS in either direction. Never optimize from Dolphin timings. [HIGH]
  - Exact timing (VI/DSP/IOS latencies), alignment exceptions, real Bluetooth pairing behavior, SD/USB timing, uninitialized-memory contents. [HIGH]
  - EFB peek/poke and some copy edge cases depend on Dolphin accuracy settings. [MEDIUM]
- Hardware debugging:
  - **USB Gecko** (EXI slot B serial adapter): printf over `usbgecko` + a GDB stub in libogc (`_break()`, `DEBUG_Init(GDBSTUB_DEVICE_USB, 1)`). Rare hardware but the gold standard. [HIGH]
  - **wiiload** over Wi-Fi: fast deploy; combine with on-screen console (`CON_Init`/`printf`) for logging. [HIGH]
  - libogc installs a default **exception handler** that dumps a register + stack trace ("code dump") on screen: `DSISR/DAR` for data faults, `SRR0` = faulting PC. Map SRR0 back with `powerpc-eabi-addr2line -e app.elf 0x800xxxxx`. [HIGH]
  - Net-based logging: trivial UDP `printf` sink is 30 lines of code and worth it. [HIGH]
  - SD/USB file logging: keep **one** file handle open for the whole run. Per-call `fopen`/`fclose` from the render path stalled the GX FIFO and corrupted frames. `fflush` alone only reaches the FAT sector cache, so the log tail vanishes after a crash – add `fsync(fileno(f))` on load/error paths (never mid-frame). [HIGH – HW-confirmed]
- Anti-pattern: Do NOT ship anything that has only ever run in Dolphin. [HIGH]

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, console alive (HBC returns on reset) | After VIDEO/GX init changes | Missing `VIDEO_SetBlack(FALSE)` / no `VIDEO_Flush()` / XFB not set or not in uncached alias | Follow canonical init order in §3; XFB via `MEM_K0_TO_K1` | [HIGH] |
| Black screen only on PAL console | Hardcoded NTSC mode | `rmode` hardcoded instead of `VIDEO_GetPreferredMode` | Use preferred mode; handle 528-line PAL EFB height | [HIGH] |
| Textures garbage/checkerboard on hardware, fine in Dolphin | Texture upload path | Missing `DCFlushRange` after writing texture/vertex data | Flush every buffer GX reads; align to 32 | [HIGH] |
| Random polygons / GPU hang mid-frame | Display lists or FIFO | Display list not 32-byte aligned/padded, or FIFO overflow | `memalign(32)`, pad size to 32, flush; enlarge FIFO | [HIGH] |
| Static/crackle in audio | Streaming audio | Unflushed audio buffers or callback underrun | `DCFlushRange` each buffer; bigger ring; no decode in callback | [HIGH] |
| Silence, everything else works | Audio init | Wrong sample rate constant or voice never started; PCM in little-endian | Init AESND at 48k, byteswap samples | [HIGH] |
| Crash on boot before main() output | Startup | .dol conversion of ELF with unsupported sections, or stack overflow from huge statics | Rebuild with template Makefile; move big arrays off stack/bss into heap | [MEDIUM] |
| Crash (DSI, DAR–small value) after loading large file | Asset loading | malloc returned NULL (MEM1 exhausted) and unchecked | Allocate bulk data from MEM2 arena; check returns | [HIGH] |
| Crash with SRR0 in copied/generated code | Loader/JIT | Missing `ICInvalidateRange` after writing instructions | DCFlush + ICInvalidate every emitted range | [HIGH] |
| Wiimote never responds | Input init | Polling before pairing completes, or `WPAD_ScanPads` missing from loop | Poll every frame; handle `WPAD_ERR_NO_CONTROLLER`; press 1+2/red sync | [HIGH] |
| `fatInitDefault()` returns false / files missing | SD access | SDXC/exFAT card, or app launched from USB expecting `sd:/` | FAT32 card; probe both `sd:/` and `usb:/` prefixes | [HIGH] |
| 40 FPS instead of 60, CPU-bound | Rendering | Immediate-mode vertex submission or GL translation layer overhead | Display lists for static geo, indexed arrays, native GX | [HIGH] |
| Squashed/stretched image | Display | Rendering 4:3 projection on 16:9 console setting (anamorphic) | Read `CONF_GetAspectRatio()`, widen projection | [HIGH] |
| Mid-session slowdown scaling with texture count | Rendering | No mipmaps – TMEM thrash | Generate/upload mip chains, set `GX_LIN_MIP_LIN` | [HIGH] |
| FPS halves the moment a frame exceeds ~16.7 ms | Present path | `VIDEO_WaitVSync()` every frame locks to 60 Hz multiples | Drop the per-frame vsync; `GX_DrawDone` + two XFBs + frame cap | [HIGH – HW-confirmed] |
| Permanent black screen / main loop spins forever | Timing | `gettime()` tick count converted to `int` milliseconds overflows `INT_MAX` → negative time | Capture a base tick on first call and subtract it before the cast | [HIGH – HW-confirmed] |
| Black screen at boot only without router/cable | Network init | `if_config()` DHCP retry loop runs before video with a high retry count | Few retries (3), best-effort, or init network later | [HIGH – HW-confirmed] |
| Garbled text / malformed models with indexed arrays | GX vertex cache | Arrays rewritten at the same address or descriptor switched without `GX_InvVtxCache()` | `GX_InvVtxCache()` before the indexed draw | [HIGH – HW-confirmed] |
| One-speaker-only stereo | ASND streaming | Voice IRQ callback races a main-thread `DCFlushRange` | `ASND_SetInfiniteVoice` + `ASND_GetTickCounterVoice` read position | [HIGH – HW-confirmed] |
| Classic Controller reads nothing for seconds after attach | WPAD | CC handshake raced the data-format request | Re-call `WPAD_SetDataFormat` once on first CC detection | [HIGH – HW-confirmed] |
| "Not enough memory" only when a USB drive is plugged in | Storage init | Mount-everything init gives each device its own FAT cache | Per-device `dvmProbeMountDiscIface`; mount USB only if SD has no data | [HIGH – HW-confirmed] |
| Power button returns to HBC instead of powering off | Shutdown | Bluetooth/WPAD still running, STM poweroff fails silently | `WPAD_Shutdown()` before `SYS_ResetSystem(SYS_POWEROFF, 0, 0)`; latch flags in power/reset callbacks, act in main loop | [HIGH – HW-confirmed] |
| Exit lands in the Wii System Menu, not HBC | Shutdown | `SYS_ResetSystem(SYS_RETURNTOMENU, …)` | `exit(0)` returns to the HBC loader | [HIGH – HW-confirmed] |
| Console freezes on USB pad attach / first report | Raw USB HID | Async device-change notify, `USB_SetConfiguration`, stack IPC buffers, blocking call in IOS callback, or vendor-class 0xFF pad | See §4 USB HID rules | [HIGH – HW-confirmed] |

---

## 11. ANTI-PATTERNS

1. Do NOT use OpenGL/GLSL mental models – there are no shaders; think TEV stages. [HIGH]
2. Do NOT skip `DCFlushRange` after CPU-writing anything GX, DSP, AI, or IOS will read. [HIGH]
3. Do NOT skip `DCInvalidateRange` before CPU-reading DMA/IOS-written buffers. [HIGH]
4. Do NOT allocate GX/IOS/DMA buffers without `memalign(32, size_rounded_to_32)`. [HIGH]
5. Do NOT assume little-endian: byteswap every binary file format ported from PC. [HIGH]
6. Do NOT trust Dolphin for cache bugs or performance numbers. [HIGH]
7. Do NOT write to NAND. Ever, unless that is the app's explicit purpose. [HIGH]
8. Do NOT hardcode NTSC 640×480 – query `VIDEO_GetPreferredMode` and `CONF_GetAspectRatio`. [HIGH]
9. Do NOT assume a specific controller: probe Wiimote/Nunchuk/Classic/GC pad every frame. [HIGH]
10. Do NOT malloc multi-MB assets from the default heap without a plan – it fills MEM1 first and then spills into MEM2 arena2; reserve private MEM2 regions explicitly with `SYS_SetArena2Hi()` and a floor for the heap. [HIGH]
11. Do NOT do file I/O or heavy allocation inside the audio callback or render loop. [HIGH]
12. Do NOT use immediate-mode `GX_Position3f32` spam for static geometry – record display lists. [HIGH]
13. Do NOT emit or expect AltiVec/VMX; SIMD means paired singles. [HIGH]
14. Do NOT `IOS_ReloadIOS` casually – you lose AHBPROT and may unmount devices. [HIGH]
15. Do NOT rely on stencil buffers or destination alpha in RGB8 EFB mode – plan materials around it. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Confirm memory budget (24+~52 MB usable).** Priority first because if the engine's working set exceeds ~70 MB with no asset-streaming path, the port is dead before rendering matters. Skip it – weeks of work end in OOM. [HIGH]
2. **Audit endianness.** Grep every `fread` into structs, every binary format, network code. Big-endian breakage is silent data corruption. Skip it – "loads but everything is garbage" bugs that look like ten other bugs. [HIGH]
3. **Stand up the build:** devkitPPC template Makefile, stub main, `boot.dol` in HBC, on-screen console printing. Everything downstream needs a deploy/log loop. Skip it – you debug blind. [HIGH]
4. **Bring up video + input skeleton** (solid color via GX copy, WPAD/PAD polling, clean exit to HBC). Establishes the frame loop contract. Skip it – no validation platform for later stages. [HIGH]
5. **Replace the renderer backend.** Map the engine's material/blend model onto GX+TEV; textures to CMPR/RGB5A3 with mips; static geometry to display lists. This is the long pole – schedule it as such. Skip it (i.e., use a GL shim permanently) – ~30–50% perf ceiling loss. [HIGH]
6. **File I/O to libfat paths** (`sd:/`), async or preloaded; MEM2 allocator for assets. Skip it – hitching and MEM1 OOM. [HIGH]
7. **Audio to AESND** double-buffered ring, offline-resampled assets. Comes after video because audio bugs don't block visual validation. Skip it – shippable-looking build that crackles. [HIGH]
8. **Threading model:** map engine threads to LWP or flatten to single-thread + async I/O. One core – parallelism buys nothing except I/O overlap. Skip it – deadlocks/perf surprises late. [MEDIUM]
9. **Performance pass:** mipmaps, TEV collapsing, paired singles on hot math, MEM1 placement for hot data. Only after correctness. Skip it – fine, but 40 FPS. [HIGH]
10. **Hardware soak + PAL testing + release packaging** (meta.xml, icon.png, both aspect ratios, controller matrix). Skip it – bug reports from half your users. [HIGH]

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Nintendo Wii – PPC750CL "Broadway" @729MHz, 32-bit BIG-ENDIAN, hard-float, paired-single SIMD, no AltiVec
GPU: "GX"/Flipper-class @243MHz – FIXED FUNCTION, 16 TEV stages, NO shaders, NO stencil; 2MB EFB + 1MB TMEM
RENDER: draw→EFB, GX_CopyDisp→XFB(YUV, main RAM), GX_DrawDone, VIDEO_Flush (per-frame WaitVSync halves FPS on slow frames); max 640x528; NTSC 60 / PAL 50 / EuRGB60
RAM: MEM1 24MB 1T-SRAM @0x8000_0000 (fast; heap fills it first) | MEM2 64MB GDDR3 @0x9000_0000 (arena2; heap spills here; IOS owns top)
MAGIC NUMBER: 32 – cache line, alignment for GX/DMA/IOS/display lists (memalign(32), size%32==0)
CACHE: write-back, NOT coherent with GX/DMA/IOS – DCFlushRange after CPU write, DCInvalidateRange before CPU read,
       ICInvalidateRange after writing code (JIT/loader)
SDK: devkitPPC + libogc (+libfat, wiiuse/bte, aesnd/asnd); triplet powerpc-eabi-; flags: -mrvl -mcpu=750 -meabi -mhard-float
BUILD: ELF – elf2dol – sd:/apps/<name>/{boot.dol, meta.xml, icon.png}; deploy fast via wiiload (HBC, TCP 4299)
INPUT: WPAD_ScanPads/PAD_ScanPads once per frame; probe expansion type every frame; IR via WPAD_FMT_BTNS_ACC_IR
AUDIO: AESND/ASND, 48kHz s16 BIG-ENDIAN stereo, callback double-buffer (or ASND infinite voice + tick counter for DMA mixers), DCFlushRange every buffer
STORAGE: fat.h/dvm → sd:/ usb:/ (FAT32); mount per device (each mount has its own cache); Wii Mini = USB only; NAND = read-only, DVD = avoid; net_init() BSD-ish sockets
DEBUG: Dolphin (hides cache bugs & perf!), USB Gecko GDB stub, on-screen code dump: SRR0=crash PC → addr2line
TOP 3 HARDWARE-ONLY BUGS: missing DCFlushRange, unaligned display lists, little-endian data assumptions
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this document is loaded in an active development session (not pure Q&A), you MUST operate the following state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS** – Examine the codebase, identify Wii-specific blockers (endianness, cache, GX, memory), and plan the implementation.
- **STATE: IMPLEMENTATION** – Write/modify code based on this document. No hardware testing occurs here.
- **STATE: BUILD_REQUEST** – Output exact build commands and ask the user to compile and run on real hardware.
- **STATE: WAITING_FOR_HARDWARE** – STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION** – User reports back. Classify as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL** – On SUCCESS, propose the next milestone and ask for confirmation.

### STATE TRANSITION RULES

1. **ANALYSIS – IMPLEMENTATION**: Only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION – BUILD_REQUEST**: Only after providing a complete, compilable change, including:
   - Exact build command (typically `make` in the devkitPPC template project)
   - Expected output file and location (e.g., `build/app.dol`)
   - Transfer method (SD card to `sd:/apps/<name>/boot.dol`, or `wiiload app.dol` over Wi-Fi)
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST – WAITING_FOR_HARDWARE**: You MUST output this exact header, then STOP generating:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color framebuffer"]
   BUILD: [Command]
   DEPLOY: [Method]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including code dumps, black screen, crashes]
   === STOP ===
   ```
   You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE – VALIDATION**: Triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - "SUCCESS" – NEXT_GOAL.
   - "FAILURE" – DEBUG_PROTOCOL.
   - Anything else ("it kind of works", "almost") – ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE) – DEBUG_PROTOCOL**: Output a **DEBUG BUILD PROTOCOL**:
   - A minimal C test case isolating the failure (e.g., solid-color GX copy with the suspect buffer), OR
   - A checklist of 3 specific diagnostics (e.g., "Confirm `DCFlushRange` is called on the display list after recording", "Print `fatInitDefault()` return value on screen", "Report the SRR0 address from the code dump and run addr2line").
   - Ask the user to run it and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL – WAITING_FOR_HARDWARE**: After providing the protocol, return to WAITING_FOR_HARDWARE.
7. **VALIDATION (SUCCESS) – NEXT_GOAL**: Propose the next milestone from the goal stack. Do NOT implement it until the user confirms.

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [USB Gecko GDB stub / on-screen code dump + addr2line / Dolphin-vs-hardware differential / WiiBrew or GBAtemp community thread].`
- Ask the user whether to:
  a) Skip this goal and mark it BLOCKED, or
  b) Provide code dumps / Dolphin logs / register values for further analysis.

### GOAL STACK (default if the user provides none)
1. Initialize video output (solid color framebuffer via GX copy-clear)
2. Initialize controller input (print WPAD/PAD button presses on screen)
3. Initialize audio output (play a sine wave through AESND)
4. Load assets from SD (`fatInitDefault`, read + verify a test file)
5. Render main menu framebuffer (textured quads, correct on NTSC and PAL)
6. Main menu input loop (navigation + selection, expansion hot-plug safe)
7. Transition to game state

Work ONLY on the active goal (top of stack). Do NOT implement future goals speculatively.
