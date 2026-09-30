---
name: psp
description: Sony PSP homebrew development — architecture, graphics (sceGu/GE), input, memory, audio, build system, debugging, and hardware validation gates.
---

# SKILLS_PSP.md  Sony PSP Homebrew Expertise Document

> Persistent expertise brief for AI assistants working on Sony PSP homebrew.
> Read this fully before writing a single line of PSP code. Obey Section 14 in active development sessions.

---

## 1. HARDWARE ARCHITECTURE

### CPU
- The main CPU is the Sony **"Allegrex"**, a MIPS32-based core derived from the R4000 lineage, running the MIPS32 ISA with custom extensions. [HIGH]
- **Little-endian, 32-bit.** You will never deal with big-endian byte swapping on this platform (unlike Wii/PS3/GameCube). [HIGH]
- Clock: software-selectable **1333 MHz**. Default for most contexts is 222 MHz; you must call `scePowerSetClockFrequency(333, 333, 166)` to run at full speed (CPU 333, bus 166). On firmware 6.xx this is permitted from user mode. [HIGH]
- Caches: **16 KB instruction cache, 16 KB data cache**, 64-byte cache lines, write-back D-cache. Cache coherency with the GPU and DMA is YOUR responsibility (see Section 5). [HIGH]
- FPU: single-precision hardware FPU (no hardware doubles  `double` is emulated in software and is catastrophically slow; always use `float`). [HIGH]
- SIMD: the **VFPU** (Vector FPU)  a 128-bit SIMD coprocessor with a 4Ã4Ã8 register file (128 single-precision registers addressable as scalars, vectors, and 4Ã4 matrices). It provides single-instruction matrix multiply, dot products, and transcendental approximations. It is accessible only from threads created with `PSP_THREAD_ATTR_VFPU`. [HIGH]
- VFPU loads/stores (`lv.q`, `sv.q`) require **16-byte alignment**; violating this raises an exception on hardware. [HIGH]
- Quirk: GCC does not auto-vectorize for the VFPU. You must use inline assembly, the `pspvfpu` context library, or existing VFPU math libraries (e.g., `libpspmath`, sceGum's VFPU path). [HIGH]

### GPU
- The **Graphics Engine (GE)** is a fixed-function GPU clocked at **166 MHz** (bus-locked, half of a 333 MHz CPU clock). [HIGH]
- No shaders of any kind. No programmable pipeline. Do NOT plan around GLSL/HLSL/Cg  everything is register-configured fixed function. [HIGH]
- It is a display-list processor: the CPU writes command words into memory; the GE DMA-reads and executes them. The `sceGu` library wraps this. [HIGH]
- Hardware T&L: transform, lighting (4 hardware lights), fog, morphing (up to 8 morph weights), and skeletal skinning (up to 8 bones per vertex) are done in GE hardware. [HIGH]
- Embedded DRAM (VRAM) on a wide internal bus; framebuffers and hot textures live here. **The GE aperture is 2 MB on every model, and you should budget for 2 MB.** [HIGH]
- **`sceGeEdramSetSize(4*1024*1024)` is NOT reliably available and must not be assumed. [HIGH — corrected 2026-08-12, measured]** The 4 MB unlock is widely described (DaedalusX64 does it) and this document previously rated it [HIGH] as a working technique. On a **PSP-2000 running ARK-4 with a current pspdev toolchain** it returns **`0x8002013A`** — the unresolved-import/module-not-found error — and `sceGeEdramGetSize()` continues to report **2,097,152 bytes**. The stub is simply not linkable in that configuration. Measured on hardware in a shipping engine port; the boot line read `PSP model 1, sceGeEdramSetSize(4MB) -> 0x8002013A, GE eDRAM 2097152 bytes at 0x4000000`.
  - **What to do:** call it if you like, but **check the return value and then size your allocator from `sceGeEdramGetAddr()`/`sceGeEdramGetSize()` regardless.** Never hardcode 4 MB, and never let a feature (32-bit colour buffers, a bigger texture cache) become load-bearing on the unlock succeeding. Plan the VRAM budget at 2 MB and treat 4 MB as a bonus you detect at runtime.
  - Do not spend a hardware round rediscovering this: a `8002013A` from a `sceGe*` call is a **link/stub** problem, not a logic one (§8, §10).
- Native display: **480Ã272 LCD**. Framebuffer stride is fixed at **512 pixels** regardless of visible width. [HIGH]
- Framebuffer formats: 16-bit (5650 / 5551 / 4444) or 32-bit (8888). Depth buffer: 16-bit. Stencil: implemented via destination-alpha bits of the framebuffer (so 8888 gives you 8 stencil bits, 5551 gives 1, 5650 gives none). [HIGH]
- Texture limits: **max 512Ã512**, dimensions must be powers of two. Up to 8 mipmap levels. [HIGH]
- Texture formats: RGBA8888, RGBA4444, RGBA5551, RGB565, 4-bit and 8-bit paletted (CLUT), and **DXT1/DXT3/DXT5** compressed. [HIGH]
- Textures should be **swizzled** (tiled into 16-byte Ã 8-row blocks) for maximum sampling bandwidth; unswizzled linear textures can halve fill rate. [HIGH]
- One texture unit. Multi-texturing requires multiple passes with blending. [HIGH]

### RAM
- **PSP-1000: 32 MB main DDR RAM. PSP-2000 and later: 64 MB.** The second 32 MB is normally reserved by the OS for UMD caching; homebrew claims it on CFW via **large memory mode**, a `MEMSIZE` key in `PARAM.SFO`. **On firmware above 3.90 the correct value is `MEMSIZE=2`, not `1`.** PSPSDK's own `build.mak:43-55` says so: *"CFW versions after M33 3.90 guard against expanding the user memory partition on PSP-1000, making MEMSIZE obsolete. It is now an opt-out policy with `PSP_LARGE_MEMORY=0`"* — it defaults `EXPAND_MEMORY = 2` for `PSP_FW_VERSION > 390` and emits *"PSP_LARGE_MEMORY flag is not necessary"* if you force 1. `2` is also `CreatePBP.cmake`'s default and what DaedalusX64 ships (it passes no `MEMSIZE` at all). Measured with `MEMSIZE=2` on a PSP-2000/ARK-4: **55,641 KB free**, so the expansion is real. [HIGH]
- Do not confuse the two mechanisms: **`PSP_LARGE_MEMORY` is a `build.mak` variable**, not a PARAM.SFO key. It is translated to a `MEMSIZE` value per firmware by the rules above. Copying `PSP_LARGE_MEMORY = 1` from an old Makefile into a literal `MEMSIZE=1` is wrong on modern firmware. [HIGH]
- **Targeting note:** the Slim and later models (PSP-2000/3000/Go/Street) are the dominant homebrew hardware today; PSP-1000 is the constrained minority. Decide your memory floor explicitly per project: a 64 MB (Slim+) baseline is a legitimate choice for demanding ports/emulators (DaedalusX64, big engine ports) — document the requirement and fail gracefully with a clear message on a Phat. If you promise PSP-1000 support, budget for 24 MB and test on a 32 MB profile. Detect at runtime with `kuKernelGetModel()` (0 = PSP-1000) rather than assuming either way. [HIGH]
- User-mode partition (partition 2) gives homebrew roughly **24 MB** of heap on a 32 MB unit; the kernel keeps the rest. With large memory mode on a Slim+, the user partition grows to roughly **~55-56 MB** usable. [HIGH]
- **Volatile memory (partition 5): ~4 MB extra on ALL models**, normally used by the OS for suspend/UMD caching. Lock it with `sceKernelVolatileMemLock(0, &ptr, &size)` (pair with `scePowerLock(0)` to prevent suspend corrupting it), then allocate from it with `sceKernelAllocPartitionMemory(5, ...)`. DaedalusX64 uses this as an extra heap. Trade-off: suspend/resume is blocked or must be handled manually while locked. [HIGH]
- **16 KB scratchpad SRAM** at `0x00010000`  very fast CPU-local memory. The GE and DMA cannot read it, so use it for CPU working sets only (mixing buffers, particle math), never for display lists or textures. [MEDIUM]  verify by pointing `sceGuStart` at scratchpad and observing a GE hang/garbage.
- **2 MB VRAM (4 MB on Slim+ after `sceGeEdramSetSize`)** at physical `0x04000000` (see memory map in Section 5).
- No demand-paged virtual memory. Addresses are effectively physical with fixed segment mapping; there is a simple MMU used by the kernel for user/kernel protection, but you cannot page or remap. [MEDIUM]
- Alignment rules: 16 bytes for VFPU and GE display lists; 64 bytes for buffers touched by DMA or the GE if you flush by range (cache-line granularity). Use `memalign(16, ¦)` or `memalign(64, ¦)` habitually. [HIGH]

### Bus topology
- CPU, GE, Media Engine, and DMA controllers share the main memory bus (166 MHz at full clock). The GE reads display lists, vertices, and textures directly from main RAM or VRAM via its own DMA. [HIGH]
- VRAM has far higher effective bandwidth for the GE than main RAM; render targets MUST be in VRAM, and hot textures should be. [HIGH]
- A general-purpose DMA controller (`sceDmac*`) can do memory-to-memory copies asynchronously; `sceDmacMemcpy` beats `memcpy` for large uncached transfers. [MEDIUM]

### Co-processors
- **Media Engine (ME):** a second Allegrex core (no VFPU) with private eDRAM — **2 MB on PSP-1000, doubled to 4 MB on PSP-2000 and later** [MEDIUM] — normally reserved by the OS for audio/video codec offload (`sceAudiocodec`, `sceMpeg`, ATRAC3). Direct homebrew code execution on the ME is possible only via kernel-mode tricks and community libraries (e.g., mcidclan's ME work, older `libpspme`); it is fragile and firmware-dependent. Treat the ME as "codec accelerator via official-style APIs" unless you explicitly budget research time. [MEDIUM]
- **VME (Virtual Mobile Engine):** a reconfigurable DSP used internally for codec work. Undocumented and inaccessible to homebrew. Do NOT plan around it. [HIGH]
- No separate audio DSP chip in the usual sense  audio output is a hardware mixer fed by the CPU/ME (Section 6). [HIGH]

### Security / DRM
- Boot chain: pre-IPL ROM  IPL  signed kernel. Homebrew runs via **custom firmware (CFW)**  ARK-4 (current, actively maintained), PRO, ME/LME  or permanently via the **Infinity** exploit on firmware 6.61. Every PSP model, including PSP Go and the PSP Street E-1000, is software-hackable on 6.60/6.61 today. [HIGH]
- Since the 2011 "signed EBOOT" break, homebrew EBOOTs can be signed to run even on **official firmware 6.60/6.61** in user mode  but most of the scene assumes CFW; target CFW user mode as your baseline. [HIGH]
- User mode vs kernel mode: normal homebrew runs user mode. Kernel-mode PRX plugins are possible on CFW and unlock flash access, ME control, and full 64 MB. Do not require kernel mode unless the feature demands it. [HIGH]
- `flash0:/` and `flash1:/` (NAND firmware/settings) are writable from kernel mode on CFW. Writing there can **brick** the console. Never touch flash in an app unless the user explicitly asks and confirms. [HIGH]

---

## 2. OFFICIAL VS HOMEBREW SDK

- Official SDK: Sony's proprietary PSP SDK (leaked copies circulate). You must NOT use, reference, or generate code against it. All examples in this document compile with the community toolchain. [HIGH]
- Homebrew SDK: **PSPSDK** under the **pspdev** umbrella (`github.com/pspdev/pspdev`), open source (BSD-style licensing across components). It bundles: `psp-gcc` (patched GCC targeting Allegrex), binutils, newlib, PSPSDK headers/libs, `psp-cmake`, `pspsh`/`psplink` tooling, and a large ported-library set (SDL2, SDL3, zlib, libpng, vorbis, freetype, curl, etc.) installable via `psp-pacman`. [HIGH]
- What PSPSDK CAN do: full user-mode API surface (GU graphics, ctrl, audio, power, net/inet sockets, ad-hoc, savedata/OSK/netconf utility dialogs, UMD, RTC, USB), kernel-mode module development, PRX plugins, EBOOT packaging, on-hardware debugging via psplink. This is one of the most complete homebrew SDKs of any console. [HIGH]
- What it CANNOT do (or does poorly) vs official:
  - No supported path to **run arbitrary code on the Media Engine**; codec APIs (`sceAudiocodec`, `sceMpeg`, ATRAC3+) are usable but thinly documented  expect reverse-engineered headers. [MEDIUM]
  - No official-quality **profiler/analyzer** tooling; you profile with `sceKernelGetSystemTimeLow` deltas and the GE list sync timings. [HIGH]
  - Some **utility dialogs** (e.g., certain netconf/DRM flows) have incomplete wrappers. [MEDIUM]
  - Documentation is header comments + community wikis (pspdev docs, older ps2dev-era wiki mirrors), not a manual. When headers and wiki disagree, trust headers + working repos. [HIGH]
- Critical decision impact: because PSPSDK is mature, you should port with real OS APIs (threads, semaphores, sockets) rather than reinventing schedulers  the PSP kernel gives you preemptive threads, VBlank callbacks, and alarms out of the box. [HIGH]

---

## 3. GRAPHICS PIPELINE

### API choice
- Use **`sceGu` + `sceGum`** (libgu/libgum from PSPSDK). `sceGu` builds GE display lists with a GL-flavored immediate API; `sceGum` adds a VFPU-accelerated matrix stack. This is the canonical path used by virtually every performant PSP homebrew title. [HIGH]
- `pspgl` (partial OpenGL 1.x) exists but is incomplete and slower; use it only to bootstrap a port that is drowning in GL calls, then migrate hot paths to sceGu. [MEDIUM]
- SDL2/SDL3 for PSP has a GU-backed renderer  acceptable for 2D ports, not for a 3D engine's hot path. [HIGH]
- Do NOT plan around ANGLE, Vulkan, or any shader translator. There are no shaders. [HIGH]

### Initialization (canonical sequence)
```c
static unsigned int __attribute__((aligned(16))) list[262144];

void initGu(void) {
    sceGuInit();
    sceGuStart(GU_DIRECT, list);
    sceGuDrawBuffer(GU_PSM_8888, (void*)0, 512);            // VRAM offset 0
    sceGuDispBuffer(480, 272, (void*)0x88000, 512);         // second buffer
    sceGuDepthBuffer((void*)0x110000, 512);                  // 16-bit Z
    sceGuOffset(2048 - (480/2), 2048 - (272/2));
    sceGuViewport(2048, 2048, 480, 272);
    sceGuDepthRange(65535, 0);                               // inverted Z, PSP convention
    sceGuScissor(0, 0, 480, 272);
    sceGuEnable(GU_SCISSOR_TEST);
    sceGuDepthFunc(GU_GEQUAL);                               // pairs with inverted range
    sceGuEnable(GU_DEPTH_TEST);
    sceGuFrontFace(GU_CW);
    sceGuEnable(GU_CULL_FACE);
    sceGuEnable(GU_TEXTURE_2D);
    sceGuFinish();
    sceGuSync(0, 0);
    sceDisplayWaitVblankStart();
    sceGuDisplay(GU_TRUE);
}
```
- Buffer addresses passed to `sceGuDrawBuffer`/`sceGuDispBuffer`/`sceGuDepthBuffer` are **VRAM offsets**, not CPU pointers. [HIGH]
- The 8888 double-buffered + 16-bit Z layout above consumes 512Ã272Ã4Ã2 + 512Ã272Ã2  1.36 MB of the 2 MB VRAM, leaving ~640 KB for textures. Using 16-bit color buffers roughly doubles free VRAM. On a Slim+ with `sceGeEdramSetSize(4MB)` enabled, the same 8888 layout leaves **~2.6 MB** free for textures/render targets — enough that 32-bit color everywhere becomes the sensible default (DaedalusX64's approach). [HIGH]

### Frame loop / buffering
- Per frame: `sceGuStart(GU_DIRECT, list)`  issue draw calls  `sceGuFinish()`  `sceGuSync(0,0)`  `sceDisplayWaitVblankStart()`  `sceGuSwapBuffers()`. Double buffering is the norm; triple buffering is possible with manual `sceDisplaySetFrameBuf` management but rarely worth the VRAM. [HIGH]
- Display lists live in main RAM, must be **16-byte aligned**, and must not be reused while the GE is still executing them (that's what `sceGuSync` guards). [HIGH]
- Vertex data drawn with `sceGuDrawArray` from main RAM must be visible to the GE: either flush the D-cache (`sceKernelDcacheWritebackRange`) after writing it, write through an uncached pointer (`addr | 0x40000000`), or allocate it inside the display list with `sceGuGetMemory` (the common pattern for dynamic geometry). [HIGH]

### VSync / refresh
- The LCD refreshes at **59.94 Hz** on every PSP in every region. There is NO NTSC/PAL split for the built-in screen. [HIGH]
- Video-out (PSP-2000+ component/composite cables) can output 480i/480p regions of a larger frame; treat it as an edge case, not a target. [MEDIUM]
- `sceDisplayWaitVblankStart()` is your frame pacer; for a 30 FPS target call it twice or track vblank counts. [HIGH]
- **Turn the vblank wait OFF before you profile anything. [HIGH — measured 2026-08-12]** It blocks until the next refresh boundary, so with it on, frame time can only ever be a whole multiple of **16.68 ms**. Below 60 fps that quantises your framerate to a ladder — **20 / 15 / 12 / 10 fps and nothing in between** — and, far worse, it quantises the *median and p95 frame times* a profiler reports, which is where a real A/B result would have shown up. A change worth 3 ms is completely invisible until it happens to push a frame across a refresh boundary. A PSP engine port spent a whole optimisation session unable to read its own measurements for this reason. Below 59.94 fps the wait also buys nothing for tearing: use a `sceDisplayGetVcount()` check so it only fires when the engine is genuinely running ahead of the panel, and keep triple buffering so skipping it is safe. [HIGH]
  - Sanity check for any PSP profiler: if your reported frame times cluster on multiples of 16.68 ms, you are measuring the panel, not your code.

### Textures
- Upload = point the GE at memory: `sceGuTexMode(GU_PSM_8888, 0, 0, swizzled)`, `sceGuTexImage(mip, w, h, bufwidth, ptr)`. There is no "upload" copy unless you do one  the GE samples from wherever `ptr` points (VRAM preferred). [HIGH]
- Swizzle all static textures at asset-build time or load time; keep the swizzle function in your codebase (16-byte-wide, 8-row blocks). [HIGH]
- CLUT (paletted) textures: `sceGuClutMode` + `sceGuClutLoad`; 4-bit CLUT is the cheapest format on the platform and ideal for UI/fonts. [HIGH]
- Max 512Ã512  engine ports must downscale or atlas-split anything larger at asset-pipeline time, not at runtime. [HIGH]

### Blending / depth / stencil
- Standard blend equations via `sceGuBlendFunc(GU_ADD, GU_SRC_ALPHA, GU_ONE_MINUS_SRC_ALPHA, 0, 0)`; fixed-color blend factors supported. [HIGH]
- Depth is conventionally **inverted** (`sceGuDepthRange(65535, 0)` + `GU_GEQUAL`) because the GE's Z precision behaves better that way; keep this convention or you will fight Z-fighting. [MEDIUM]
- Stencil ops exist (`sceGuStencilFunc/Op`) but consume framebuffer alpha bits  you cannot have both destination alpha and stencil in the same 8888 target. [HIGH]

### Anti-patterns (GPU)
- Do NOT emulate shaders with multipass unless profiled  fill rate at 480Ã272 is decent but the GE stalls on state changes; batch by texture/state. [HIGH]
- Do NOT render to main RAM. Render targets must be VRAM. [HIGH]
- Do NOT forget `sceGuTexFlush()` after CPU-writing a texture the GE already sampled  the GE has a small texture cache. [HIGH]
- Do NOT issue per-triangle `sceGuDrawArray` calls; build vertex buffers per material. [HIGH]
- Do NOT build anything load-bearing on **`sceGuTexMapMode(GU_TEXTURE_MATRIX, …)` + `sceGuTexProjMapMode(GU_UV)`** without a known-good example to copy. It is documented in `pspgu.h` and it does *not* behave as that documentation implies with a `GU_TEXTURE_32BITF` (two-float) texture coordinate. **Measured over three hardware rounds, 2026-08-12, PSP-2000/ARK-4:** an engine port folding 2-D affine texture transforms into the GE texture matrix rendered geometry with visibly wrong texturing under **both** candidate translation columns (column 3 assuming the GE supplies q=0, column 2 assuming q=1) **and** with a `sceGuSetMatrix(GU_TEXTURE, identity)` reset in place so no stale matrix could leak. Static analysis had cleared everything checkable: composition order, the affine coefficients, the `GU_TEXTURE_COORDS`/`GU_TEXTURE_MATRIX`/`GU_UV` constants, and `sceGumUpdateMatrix` itself (disassembled — it loops over all four stacks, so it *does* upload `GU_TEXTURE`). The decisive fact was found by grep, not by reasoning: **no PSP project in DaedalusX64, Quake3PSP, or any other reference tree uses `GU_TEXTURE_MATRIX` at all.** If nothing ships it, you are writing against docs, not against working silicon — budget accordingly or do the transform on the CPU/VFPU. [HIGH]
  - Corollary, and the cheap part of that lesson: **before building on any GE feature, grep your reference ports for it.** Zero hits is a risk signal worth more than an hour of design.
  - Note the transform is only half of what such a change usually buys. The other half — pointing the GE's texcoord array at existing per-vertex data instead of copying it into a scratch buffer first — needs no exotic GE state, and it worked on the first try. Ship the halves behind separate cvar values so one hardware round can tell them apart. [HIGH]

---

## 4. INPUT

- Controls are built in: D-pad, face buttons (³Ã¡), L/R shoulders, Start/Select/Home/Vol/Note/Screen, and **one analog nub**. No second stick  camera schemes must map to face buttons or D-pad (the classic "Monster Hunter claw" problem). [HIGH]
- Model: **polled**, not event-driven. Initialize once:
```c
sceCtrlSetSamplingCycle(0);                       // sync sampling to vblank
sceCtrlSetSamplingMode(PSP_CTRL_MODE_ANALOG);     // enable the nub
```
Then per frame:
```c
SceCtrlData pad;
sceCtrlReadBufferPositive(&pad, 1);   // blocks until fresh sample
// or sceCtrlPeekBufferPositive(&pad, 1);  // non-blocking, latest sample
if (pad.Buttons & PSP_CTRL_CROSS) { ... }
```
[HIGH]
- Analog nub: `pad.Lx`, `pad.Ly` in 0255, center  128. Hardware centering is sloppy; apply a **deadzone of ~1520%** or drift is guaranteed. [HIGH]
- Use `ReadBufferPositive` (blocking, vsync-paced) in simple games; use `Peek` if your loop has its own pacing  mixing both patterns causes latency confusion. [MEDIUM]
- The **HOME button is owned by the firmware** in user mode: you must register the standard exit-callback thread (see Section 8 boilerplate) or HOME will show the exit dialog and then hang your app. [HIGH]
- No rumble, no gyro, no accelerometer, no touch. Do NOT invent APIs for them. [HIGH]
- Peripherals: headphone remote (button events via `sceHprm*`), GPS and camera accessories (kernel APIs, niche, [LOW]  verify against ARK-4-era samples before promising support), PSP Go Bluetooth pairing of DualShock 3 is a CFW/OS feature, not something you code against  its input arrives through the same `sceCtrl` interface. [MEDIUM]
- Disconnect handling: not applicable for built-in controls; for the Go's BT pad, input simply stops changing  implement an idle timeout if it matters. [MEDIUM]
- Anti-pattern: do NOT busy-poll `sceCtrlPeekBufferPositive` in a tight loop without yielding; you'll starve the audio thread. Pace with vblank. [HIGH]

---

## 5. MEMORY LAYOUT

### Memory map (physical/kernel view)
| Range | What | Notes |
|---|---|---|
| `0x000100000x00013FFF` | 16 KB scratchpad SRAM | CPU-only, fast; not GE/DMA visible [MEDIUM] |
| `0x040000000x041FFFFF` | 2 MB VRAM (GE eDRAM) | CPU-mappable; GE render targets [HIGH] |
| `0x042000000x043FFFFF` | Upper 2 MB VRAM (Slim+ only) | Invalid until `sceGeEdramSetSize(4*1024*1024)` succeeds — **and it may not**: measured `0x8002013A` on PSP-2000/ARK-4 + current pspdev, size stayed 2 MB. Always trust `sceGeEdramGetSize()` over the model number [HIGH] |
| `0x080000000x087FFFFF` | Kernel RAM (8 MB) | Off-limits in user mode [HIGH] |
| `0x088000000x09FFFFFF` | User RAM (~24 MB) | Your heap/code/data [HIGH] |
| `0x0A000000+` | Extra 32 MB (PSP-2000+) | Only with CFW large-memory mode [MEDIUM] |
| `0x1C0000000x1FFFFFFF` | Hardware I/O registers | Kernel mode only [MEDIUM] |
| `addr \| 0x40000000` | Uncached mirror of any address | Bypasses D-cache [HIGH] |

- User code sees KUSEG-style addresses; ORing `0x40000000` onto a pointer yields an **uncached view** of the same memory. This is the single most important address trick on the platform. [HIGH]

### Allocation
- Standard `malloc`/`memalign` allocate from the user partition via newlib. Set the heap size with `PSP_HEAP_SIZE_KB(n)`. [HIGH]
- Prefer an **explicit positive size** for predictability, but do **not** believe the folk rule that "negative `PSP_HEAP_SIZE_KB` is broken". That claim was retracted 2026-08-10 after it was traced to a different bug. [HIGH]
  - What was originally measured (PSP-2000 / ARK-4 / pspdev GCC 15.2.0, minimal PRX): no directive → crash; `-4096` → crash; `-8192` → crash; `+20480`, `+49152` → OK. Faulting register `t1 = 0x03256508`, inside `malloc_extend_top` (`_mallocr.c:2226`).
  - What that actually was: **an unresolved `sceKernelAllocPartitionMemory` import stub** (§8 link order). With the stub unresolved, `_sbrk` never obtains a heap and hands newlib garbage — which produces exactly a wild-pointer store in `malloc_extend_top`. Switching to a positive value **did not fix anything**; every allocation still failed, the symptom merely moved. The confounded measurements above cannot separate the two causes. [HIGH]
  - `xash3d-fwgs` ships `PSP_HEAP_SIZE_KB(-3 * 1024)` and runs on hardware, as does Crow_bar's Quake3PSP with `-8 * 1024`. Negative values work. [HIGH]
  - **If allocation fails, audit the link before touching this number.** Heap tuning is the most attractive wrong answer available here. [HIGH]
  - Budget the number: total usable minus your module image (`psp-size`: text+data+**bss**) minus headroom for PRX modules and utility dialogs.
- **The heap is claimed lazily, on the first `_sbrk` — not at startup.** `libcglue`'s `_sbrk` (`pspsdk-src/src/libcglue/glue.c:688-720`) runs `sceKernelAllocPartitionMemory` the first time anything calls `malloc`. Consequences you must design around: [HIGH]
  - `sceKernelMaxFreeMemSize()` **before** any allocation reports the partition *including* the space your heap will take; **after**, it reports only what is left **outside** the heap. Sampling it at boot and sizing engine arenas from it is a real and easy mistake.
  - **Two different pools.** `malloc`/`calloc` come from the newlib heap; **`sceKernelLoadModule` (PRX) and `sceKernelAllocPartitionMemory` come from the partition, outside it.** Size game arenas from `PSP_HEAP_SIZE_KB`; size your PRX/module budget from `sceKernelMaxFreeMemSize()` measured *after* the heap exists. A 36 MB heap on a 64 MB unit left **2,962 KB** for PRX modules — not enough for three.
  - On failure `_sbrk` returns `-1` **for every subsequent call**, so `sbrk(0) == 0xffffffff` is a one-line check that the heap never came up.
- **`calloc` may not touch the memory it returns.** newlib skips the `memset` for blocks obtained fresh from `MORECORE`/`sbrk`, assuming sbrk memory is already zero — which is not guaranteed of `sceKernelAllocPartitionMemory` memory. So a bad heap pointer can survive `calloc` and only fault at the caller's first real write, pointing your debugging one function too far downstream. [MEDIUM] — verify against your newlib's `MORECORE_CLEARS` if it matters.
  - `PSP_HEAP_SIZE_MAX()` does **not exist** in current PSPSDK — `pspmoduleinfo.h` defines only `PSP_HEAP_SIZE_KB` and `PSP_HEAP_THRESHOLD_SIZE_KB`. [HIGH]
- VRAM has **no allocator** in the base SDK: you hand-compute offsets past your framebuffers, or vendor a tiny valloc helper (several exist in the community). Track it explicitly. [HIGH]
- Kernel partition allocation (`sceKernelAllocPartitionMemory`) lets you pick partition and placement (low/high)  needed for large-memory mode and for aligning big DMA buffers. [HIGH]
- **Slim+ VRAM unlock — attempt it, do not depend on it.** Call `sceGeEdramSetSize(4*1024*1024)` once at init when `kuKernelGetModel() > 0`, BEFORE sizing your VRAM allocator, then **size the allocator from `sceGeEdramGetAddr()`/`sceGeEdramGetSize()` whatever the call returned.** That way one code path covers both models *and* the case where the unlock is refused — which is what happens on a PSP-2000/ARK-4 with a current pspdev toolchain: `0x8002013A`, and the size stays 2 MB (§1). Never hardcode either constant. [HIGH]
- **Volatile memory:** `sceKernelVolatileMemLock(0, &ptr, &size)` hands you ~4 MB (partition 5) on every model  ideal for caches and streaming buffers. Hold `scePowerLock(0)` while using it, or re-acquire on resume. [HIGH]

### Stack
- The main thread stack defaults to **256 KB**; override with `PSP_MAIN_THREAD_STACK_SIZE_KB(1024)`. Threads you create get whatever you pass to `sceKernelCreateThread`  undersized thread stacks are a classic silent-corruption source. [HIGH]

### Cache discipline (memorize this)
- D-cache is write-back, 64-byte lines. The GE and DMA read **physical memory**, not your cache. Therefore:
  - `sceKernelDcacheWritebackRange(ptr, size)`  after the CPU writes data the GE/DMA will read (vertices, textures, display lists built outside sceGu). [HIGH]
  - `sceKernelDcacheWritebackInvalidateRange(ptr, size)`  before the CPU reads data that DMA/GE wrote (e.g., render-to-texture readback). [HIGH]
  - `sceKernelDcacheWritebackAll()`  sledgehammer; fine at load time, too slow per-frame. [HIGH]
  - `sceKernelIcacheInvalidateAll()`  after writing executable code (self-modifying/JIT/loader). [HIGH]
- Range flushes operate on 64-byte lines: if your buffer shares a cache line with unrelated live data, `WritebackInvalidate` can destroy that neighbor's pending writes. Align DMA/GE buffers to 64 bytes and pad sizes to multiples of 64. [HIGH]
- Alternative discipline: write through uncached pointers (`| 0x40000000`) and skip flushing  correct but slower for scattered small writes; best for write-once streaming buffers. [HIGH]

### Anti-patterns (memory)
- Do NOT put GE-read data in scratchpad. [MEDIUM]
- Do NOT assume 64 MB blindly; gate large-memory features behind `kuKernelGetModel()`/free-memory checks  or declare Slim+ as a hard requirement and fail with a clear on-screen message on a Phat. [HIGH]
- Do NOT `free()` memory a GE list still references before `sceGuSync`. [HIGH]

---

## 6. AUDIO

- Output hardware: a mixer that accepts PCM from up to **8 user channels** (07), each reserved with a fixed sample count per push. Native output is **44100 Hz, 16-bit, stereo**. [HIGH]
- Core API (`sceAudio`):
```c
int ch = sceAudioChReserve(PSP_AUDIO_NEXT_CHANNEL, 1024 /*samples*/, PSP_AUDIO_FORMAT_STEREO);
sceAudioOutputBlocking(ch, PSP_AUDIO_VOLUME_MAX, buffer);   // pushes 1024 stereo frames
```
- Sample counts must be **multiples of 64**, max 65472 per push. [HIGH]
- Non-44100 rates: `sceAudioSRCChReserve`/`sceAudioSRCOutputBlocking` use the hardware sample-rate converter (occupies a special channel). Prefer resampling to 44100 in software/assets and using plain channels. [MEDIUM]
- Recommended structure: **one dedicated audio thread per stream** running `sceAudioOutputBlocking` in a loop  the blocking call is your timing source. This is exactly what `pspaudiolib` (PSPSDK) implements with callbacks; use it or copy its pattern. [HIGH]
- Buffering: double-buffer per channel (fill B while A plays). 1024-sample pushes (~23 ms) are a safe default; smaller lowers latency but demands stricter thread priorities. Give the audio thread **higher priority (lower number, e.g., 0x12)** than the game loop. [HIGH]
- Formats: the mixer eats raw PCM only. Compressed audio (MP3/AAC/ATRAC3+) can be decoded by the ME via `sceAudiocodec`/`sceMp3`  works on real firmware, thinly documented, PPSSPP emulates most of it. Budget verification time. [MEDIUM]
- For a port: decode Ogg/MP3 with stb_vorbis/minimp3 on the CPU at 333 MHz only if you have headroom; otherwise pre-convert music to ATRAC3+ or low-rate PCM at asset time. [MEDIUM]
- There is no separate audio RAM; buffers live in main RAM. [HIGH]
- Anti-patterns: do NOT do decoding or file I/O inside the same loop iteration that pushes to `sceAudioOutputBlocking` without double buffering; do NOT push from the render thread (a long GE sync = audio gap); do NOT reserve channels per-sound-effect (mix in software onto ¤2 channels: one music, one SFX mix). [HIGH]

---

## 7. STORAGE / IO

- Primary storage: **Memory Stick Pro Duo**, mounted as `ms0:/`. PSP Go internal 16 GB storage is `ef0:/`  check both at startup and abstract the prefix. [HIGH]
- Filesystem: FAT16/FAT32. **Case-insensitive but case-preserving.** Do not rely on case sensitivity, and do not create two names differing only by case. [HIGH]
- Standard homebrew layout: `ms0:/PSP/GAME/YourApp/EBOOT.PBP` plus your data files beside it. The XMB reads title/icon metadata from inside the PBP (`PARAM.SFO`, `ICON0.PNG` 144Ã80, optional `PIC1.PNG` 480Ã272 background, `SND0.AT3`). [HIGH]
- **Working-directory trap:** the CWD depends on the launcher. Always build absolute paths from `argv[0]` at startup (strip the executable name, keep the directory) instead of using relative paths. This is the #1 "file not found on hardware but works in PPSSPP" cause. [HIGH]
- File API: `sceIoOpen/Read/Write/Lseek/Close/Dopen/Dread` or plain newlib `fopen` (which wraps sceIo). Memory Stick reads are slow (~18 MB/s depending on stick); load-time streaming with big sequential reads beats many small reads. [HIGH]
- UMD: readable on CFW via `sceUmdActivate` then `disc0:/` (ISO9660). Only relevant if you're building a loader-adjacent tool. [MEDIUM]
- `flash0:/`, `flash1:/`: firmware NAND. Kernel mode required to write; writing carelessly bricks consoles. Do NOT write there. [HIGH]
- Network: **802.11b Wi-Fi** with an Internet (infrastructure) stack and an ad-hoc stack.
  - Init dance: `sceUtilityLoadNetModule(PSP_NET_MODULE_COMMON)` + `(PSP_NET_MODULE_INET)`, then `sceNetInit`, `sceNetInetInit`, `sceNetResolverInit`, `sceNetApctlInit`, then `sceNetApctlConnect` against a saved access-point profile. After that you get **BSD-style sockets** (`sceNetInetSocket/Bind/Sendto/Recvfrom/Select`, with newlib `sys/socket.h` compat wrappers). [HIGH]
  - Hard constraint: the PSP supports **WEP and WPA-PSK (TKIP) only  no WPA2, no WPA3**, and 802.11b only. Modern routers must expose a legacy 2.4 GHz network or connection is impossible. Warn users in docs. [HIGH]
  - **The apctl state constants are an ENUM, NOT A SEQUENCE.** `DISCONNECTED`=0, `SCANNING`=1, `JOINING`=2, `GETTING_IP`=3, `GOT_IP`=4, `EAP_AUTH`=5, `KEY_EXCHANGE`=6. A WPA association runs **2 → 6 → 3 → 4**, i.e. it moves *backwards by number* every single time. Any `if (state < oldstate) fail;` guard therefore kills every WPA connect one step short of an IP and only ever works on an open or WEP AP. Quake3PSP-mirror's `unix/unix_net.c:515` contains exactly that guard  do not copy it. The only real failure is returning to `DISCONNECTED` after having left it, plus a timeout. Budget **~15 s**: it has to cover a key exchange *and* a DHCP lease over 802.11b. [HIGH]
  - **Stored connection profiles are persistent and sparse.** They are 1-based, but deleting and recreating connections leaves gaps  a console's only profile can sit in slot 2 with slot 1 empty. Never hardcode index 1. `sceUtilityCheckNetParam` only reports a *contiguous run* (pspsdk's own `samples/utility/netconf/main.c:82` counts with `while (CheckNetParam(n)==0) n++`), so it must never gate the connect. **`sceNetApctlConnect` is the authority and the cheapest enumerator**: scan slots 1..10 with it: a non-existent slot is refused instantly with `0x80110601` (`PSP_NETPARAM_ERROR_BAD_NETCONF`) and costs nothing, so only a slot the firmware accepts ever gets polled. All three pspsdk `samples/net/*` call `sceNetApctlConnect` with no netparam check at all. [HIGH]
  - **`sceNetApctlDisconnect` is asynchronous.** Issuing a new `sceNetApctlConnect` while teardown is in flight returns `0x80410A80`. Poll `sceNetApctlGetState` back to `DISCONNECTED` before the next attempt, or a scan produces a cascade of bogus refusals. [HIGH]
  - Local IP/netmask come from `sceNetApctlGetInfo(PSP_NET_APCTL_INFO_IP / _SUBNETMASK, ...)`  a union, so one call per field. Do not resolve your own hostname for this. [HIGH]
  - `INADDR_NONE` is **missing** from PSPSDK's `netinet/in.h` (`INADDR_ANY` and `INADDR_BROADCAST` are present); define it yourself. `inet_addr` is `#define`d straight to `sceNetInetInetAddr` in `arpa/inet.h`. Non-blocking is the PSP-specific `SO_NONBLOCK` socket option, not an `fcntl` dance. [MEDIUM]
  - Ad-hoc (`sceNetAdhoc*`) enables local multiplayer; PPSSPP + AdhocServer can emulate it over the internet. [MEDIUM]
- USB: device-mode Memory Stick export (`sceUsb*`) and, for development, **usbhostfs** via psplink gives you `host0:/` mapped to a PC directory  the fastest dev iteration loop on real hardware. [HIGH]
- Anti-patterns: do NOT fopen per-asset in an inner loop (MS latency); do NOT assume `ms0:/` exists on a Go; do NOT ship paths with backslashes. [HIGH]

---

## 8. BUILD SYSTEM

### Toolchain
- Install the **pspdev prebuilt toolchain** from `github.com/pspdev/pspdev` releases (Linux/macOS/WSL; also a Docker image `pspdev/pspdev`). Building from source via `pspdev.sh` is the fallback. [HIGH]
- Compiler prefix: **`psp-`** (`psp-gcc`, `psp-g++`, `psp-ld`, `psp-objcopy`¦). The GCC target is a custom Allegrex-patched MIPS (`psp` target); modern pspdev ships GCC 13/14-era compilers. [MEDIUM]  run `psp-gcc --version` to pin the exact version in your project README.
- Required environment:
```sh
export PSPDEV=/usr/local/pspdev        # or wherever installed
export PATH=$PATH:$PSPDEV/bin
# PSPSDK path used by build.mak: $(PSPDEV)/psp/sdk
```
[HIGH]

### Flags
- **`-G0` is mandatory.** It disables gp-relative small-data sections, which break PRX relocation. Every PSP Makefile in existence carries it; omitting it yields relocation errors or runtime corruption. [HIGH]
- Baseline: `CFLAGS = -O2 -G0 -Wall`. Float is hard single-precision by default on this target  do NOT pass soft-float flags, and audit ports for `double` usage. [HIGH]
- C++: add `-fno-exceptions -fno-rtti` unless you truly need them (binary size, and exception tables interact poorly with tiny stacks). [MEDIUM]

### Program boilerplate (mandatory)
```c
#include <pspkernel.h>
PSP_MODULE_INFO("MyApp", 0, 1, 0);                       // name, attr(0=user), ver
PSP_MAIN_THREAD_ATTR(PSP_THREAD_ATTR_USER | PSP_THREAD_ATTR_VFPU);
PSP_HEAP_SIZE_KB(36864);   // KB. Positive is predictable; negative ("all free
                           // minus N") also works - see §5. If allocation fails,
                           // audit the link (§8) before changing this number.
PSP_MAIN_THREAD_STACK_SIZE_KB(512);  // default is only 256 KB. Engine ports put
                           // multi-KB arrays on the stack (e.g. a directory
                           // lister with char *list[4096] = 16 KB) and recurse.

static int exitRequest = 0;
static int exitCallback(int a, int b, void *c) { exitRequest = 1; return 0; }
static int cbThread(SceSize args, void *argp) {
    int cbid = sceKernelCreateCallback("exit", exitCallback, NULL);
    sceKernelRegisterExitCallback(cbid);
    sceKernelSleepThreadCB();
    return 0;
}
static void setupCallbacks(void) {
    int th = sceKernelCreateThread("cb", cbThread, 0x11, 0xFA0, 0, 0);
    if (th >= 0) sceKernelStartThread(th, 0, 0);
}
```
Missing `PSP_MODULE_INFO` = no boot. Missing the callback thread = HOME button hangs. Missing `PSP_THREAD_ATTR_VFPU` = crash on first VFPU instruction (including inside sceGum). [HIGH]

### Makefile template
```make
TARGET   = myapp
OBJS     = main.o gfx.o
INCDIR   =
CFLAGS   = -O2 -G0 -Wall
CXXFLAGS = $(CFLAGS) -fno-exceptions -fno-rtti
ASFLAGS  = $(CFLAGS)
BUILD_PRX = 1
LIBDIR   =
LDFLAGS  =
LIBS     = -lpspgum -lpspgu -lpspaudio -lpsppower -lm
EXTRA_TARGETS   = EBOOT.PBP
PSP_EBOOT_TITLE = My App
PSP_EBOOT_ICON  = ICON0.PNG
include $(PSPSDK)/lib/build.mak
```
- `build.mak` handles ELF  PRX (`psp-prxgen`)  `EBOOT.PBP` (`pack-pbp` + `mksfoex`). `BUILD_PRX = 1` is the modern default (relocatable, required for CFW niceties). [HIGH]
- CMake alternative: `psp-cmake ..` with the shipped toolchain file; the pspdev docs and sample repos cover it. Use whichever the ported codebase prefers. [HIGH]

### Asset pipeline
- Textures: convert to power-of-two ¤512², pre-swizzle, pick the smallest viable format (4-bit CLUT for UI, 5650/5551 for opaque/cutout world textures, DXT for large surfaces). [HIGH]
- Audio: pre-resample to 44100 Hz 16-bit PCM (or ATRAC3+ if using codec APIs); keep SFX as raw PCM blobs for zero-cost loading. [HIGH]
- Models: pre-interleave vertices into GE vertex formats (e.g., `GU_TEXTURE_32BITF | GU_COLOR_8888 | GU_VERTEX_32BITF`) at build time; runtime conversion wastes the little CPU you have. [HIGH]

### Link order and the spec libraries  read before adding ANY `-lpsp*`

`psp-gcc`'s own spec already appends a fixed tail to **every** link. Check it with
`psp-gcc -dumpspecs` and read the `*lib:` line; on pspdev GCC 15.2.0 it is:

```
-lm --start-group -lpthreadglue -lpthread -lcglue -lc --end-group
-lpsputility -lpsprtc -lpspnet_inet -lpspnet_resolver -lpspsdk -lpspmodinfo -lpspuser
```

- **Never list a library the spec already provides** (`m`, `pspuser`, `psprtc`, `psputility`, `pspnet_inet`, `pspnet_resolver`, `pspsdk`, `pspmodinfo`). Doing so gives the linker **two scan points for one PSP library**, which splits that library's stubs into two runs in `.sceStub.text`. `psp-fixup-imports` requires every stub of a library to be contiguous and otherwise prints `Warning: could not fixup imports, stubs out of order.` and gives up  leaving an import table where the loader patches one run and leaves the other unresolved. The failing import then "returns" with `v0` untouched. **Treat that warning as a hard build failure; the link must be silent.** [HIGH]
- Corollary for helper wrappers: `pspSdkInetInit()` lives in `libpspsdk.a`, which the spec places *after* anything you list, so its undefined `sceNetInit`/`sceNetApctlInit` would need `libpspnet`/`libpspnet_apctl` scanned later still  they are not, and the link fails. Call `sceNetInit`/`sceNetInetInit`/`sceNetResolverInit`/`sceNetApctlInit` directly from your own object (first in link order) instead. Its exact arguments are readable from the archive: `psp-objdump -d pspSdkInetInit.o`  `sceNetInit(0x20000, 32, 4096, 32, 4096)`, `sceNetApctlInit(0x1600, 0x42)`. [HIGH]
- So for infrastructure networking the libraries you actually add are only **`pspnet`, `pspnet_apctl`, `pspwlan`**. [HIGH]
- Verify after any library change: link silent (0 × "stubs out of order"), and `psp-objdump -s -j .rodata.sceResident <elf>` shows nothing named `*ForKernel`. [HIGH]

### Anti-patterns (build)
- Do NOT drop `-G0`. [HIGH]
- Do NOT use `double` anywhere hot. [HIGH]
- Do NOT link `-lpspgu` before `-lpspgum` incorrectly (gum depends on gu; order matters with static libs: gum first, then gu). [MEDIUM]
- Do NOT add a library the `psp-gcc` spec already appends  see the link-order block above. [HIGH]
- Do NOT assume an unresolved-at-load-time net import causes `8002013C`. `sceNetInet` is imported by every pspdev binary via the spec and the net PRXs are made resident later by `sceUtilityLoadNetModule`; that is the normal arrangement. `8002013C` on a user-mode module means **kernel** imports (`-lpspkernel` → `*ForKernel`). [HIGH]

---

## 9. EMULATOR VS HARDWARE

- Primary emulator: **PPSSPP**. It is an HLE emulator with excellent compatibility and the single best GE debugging tool in the homebrew world (frame dump, per-draw state inspection, texture viewer). Develop in PPSSPP, validate on hardware. [HIGH]
- What PPSSPP gets RIGHT (safe to rely on): sceGu/GE rendering semantics for common paths, sceCtrl, sceAudio timing well enough for logic, sceIo on mapped folders, thread scheduling approximately, most utility dialogs, ad-hoc via AdhocServer. [HIGH]
- What PPSSPP gets WRONG or hides (MUST test on hardware):
  - **Cache coherency**: PPSSPP does not emulate the D-cache. Code missing `sceKernelDcacheWritebackRange` renders perfectly in PPSSPP and shows garbage/flicker on hardware. This is the #1 emulator-masked bug class. [HIGH]
  - **Performance**: PPSSPP's JIT on a PC is orders of magnitude faster. Never profile in PPSSPP; a 60 FPS PPSSPP build can be 12 FPS on hardware. [HIGH]
  - **Alignment faults**: PPSSPP tolerates some unaligned/VFPU-alignment sins that except on hardware. [MEDIUM]
  - **Memory limits**: PPSSPP defaults can be laxer about heap exhaustion and kernel/user boundaries. [MEDIUM]
  - **Module load rules**: PPSSPP HLEs every module, so it enforces **neither the user/kernel import split nor module residency**. A user-mode module importing `*ForKernel` stubs, or importing `sceNet*` before `sceUtilityLoadNetModule`, runs fine in PPSSPP and is refused outright by firmware (`8002013C`). This bug class is invisible in the emulator **by construction** — audit the binary instead: `psp-objdump -s -j .rodata.sceResident <elf>`. [HIGH]
  - **Import-table integrity**: a misordered stub table that leaves `sceKernelAllocPartitionMemory` unresolved on hardware — killing every allocation in the process — runs perfectly in PPSSPP, because HLE resolves by NID and never uses the stub table the loader patches. Same construction problem as the module-load rules above. [HIGH]
  - **`sceKernelMaxFreeMemSize()` returns fiction** — commonly a flat 2048 MB. Never size anything from a PPSSPP memory reading. [HIGH]
  - **Memory Stick speed**: instant in PPSSPP, slow on hardware  streaming code must be hardware-tested. [HIGH]
- Hardware debugging: **psplink + usbhostfs + pspsh**  run modules from the PC over USB, get `printf` back on your terminal, `host0:/` file mapping, exception dumps with full register state and EPC, and a GDB stub (`psp-gdb`). This is mandatory equipment; insist the user sets it up **before** starting any hardware-gated work, not after it stalls. [HIGH]
- **psplink setup, the parts that actually block people** (verified on Windows + ARK-4 + PSP-2000): [HIGH]
  - The PSP enumerates as **"PSP Type A"** (and after a driver swap, **"PSP Type B"**) — that *is* the Sony device; the `054C` VID shows in Zadig's USB ID field, not the name. PSPLink running = PID `01C9`; mass-storage mode = `02D2`. Bind **libusb-win32 to both** A and B entries (try WinUSB if `usbhostfs_pc` then sees nothing). Enable *Options → List All Devices* if neither appears.
  - **Serve an ABSOLUTE path**: `usbhostfs_pc.exe "E:\path\to\out"`. A relative path silently mounts an unexpected root and *every* launch fails with `Error invalid file` — which reads exactly like a rejected binary. **Run `ls` at the `pspsh` prompt before believing any "invalid file" message.**
  - **psplink runs PRX modules, not static ELFs.** A normal static ELF is rejected with `Error invalid file`. With CMake, configure `-DBUILD_PRX=ON` — `CreatePBP.cmake` declares exactly this option ("Build a PRX for use with PSPLink").
  - **Build `RelWithDebInfo`, not `Release`.** `CreatePBP.cmake` runs `psp-strip` on Release non-PRX builds, which kills `psp-addr2line` — the main reason you set psplink up. Keep the unstripped ELF next to the PRX and point `addr2line` at the ELF.
- Reading an exception dump: `Cause` bit 31 is the **BD flag** — when set, the faulting instruction is the **branch delay slot at EPC+4**, not the branch at EPC. `Address` is module-relative, `EPC` absolute; either works with `psp-addr2line -f -e module.elf <addr>` provided you use the matching one for how the ELF was linked. [HIGH]
- Crash forensics without psplink: the CFW exception handler shows EPC/registers on screen; `psp-addr2line -e myapp.elf 0x08804xxx` maps EPC to a source line (subtract nothing  user ELFs load at link address `0x08804000` by default; PRX crashes report a module-relative offset in psplink). [MEDIUM]
- Anti-pattern: do NOT conclude "it works" from PPSSPP alone, and do NOT optimize based on PPSSPP FPS. [HIGH]

### Debugging a boot failure when psplink is not available

Every hour lost to a PSP boot bug is lost to *not knowing where it stopped*. If the only channel
is a log on the Memory Stick, that log has to be built to survive and to be unambiguous. Set all
of this up **before** the first failing run, not after the third. [HIGH]

1. **Make the log write-through.** There is no `fsync` in `sceIo`, and a held-open `SceUID` that
   is only closed on clean shutdown produces a **0-byte file** after any crash — the FAT
   directory entry never gets updated. Open/append/close **per line**; reopening is what commits
   the size. ~3-8 ms a line on a Memory Stick, which is nothing next to one wasted hardware round.
   Write the log **before** echoing to `pspDebugScreen`, so a fault in the screen path still
   leaves the line on disk.
2. **Stamp the binary.** Print `__DATE__ " " __TIME__` as the first line. Without it you cannot
   tell a stale EBOOT from a fresh one that died early — and you will accuse the wrong one.
3. **Change the log filename every build** (`q3psp4.log`, `q3psp5.log`, …). A leftover log from a
   previous run is otherwise indistinguishable from a run that produced identical output. If the
   new filename does not exist after a launch, the binary did not run — no interpretation needed.
4. **Bisect with markers, do not reason.** Sprinkle `Sys_Print("mark: x")` between the last known
   line and the next expected one, using the rawest output path available (not the engine's
   `printf` wrapper, which may itself route through uninitialised subsystems).
5. **Print values, not conclusions.** Pointers, error codes, block ids, `sbrk(0)`, free memory
   before *and* after the operation. Cross-check the formatter itself by printing something whose
   value you already know (e.g. `&_end`) — a `%p` that prints garbage invalidates every other
   pointer in the log, and that is worth ruling out in the same run.
6. **When the values make no sense, disassemble.** `psp-objdump -d` the linked function and read
   what actually executes. This is what ends these investigations: source describes intent,
   and the failure is usually in the layers below it (loader, stub patching, `libcglue`, newlib).
   A `jal` whose return value no syscall wrote is visible in ten seconds of disassembly and in no
   amount of source reading.

**Order matters.** Cheap and decisive beats clever: read the build log, then check the binary
statically (`psp-objdump`, `psp-nm`, `psp-readelf`), then instrument, and only then theorise.

---

## 10. COMMON FAILURE MODES

| Error Signature | Platform Context | Likely Cause | Fix | Confidence |
|---|---|---|---|---|
| Black screen, PSP alive (backlight on) | First GU bring-up | `sceGuDisplay(GU_TRUE)` never called, wrong VRAM offsets, or list never `sceGuFinish`/`sceGuSync`'d | Follow the canonical init in §3 exactly; verify buffer offsets 0 / 0x88000 / 0x110000 | [HIGH] |
| Black screen, then instant return to XMB | Boot | Missing `PSP_MODULE_INFO`, or kernel attr (0x1000) on user firmware, or corrupt EBOOT.PBP | Use module attr 0; rebuild with `BUILD_PRX=1`; re-pack PBP | [HIGH] |
| **Every allocation fails — `malloc`/`calloc` return NULL (or a wild pointer) forever.** Symptoms include a black screen with no output, or a fault in `malloc_extend_top` (`_mallocr.c:2226`) storing through a wild pointer. Runs fine in PPSSPP | Boot, any program reaching `malloc` (directly or via C-runtime startup) | **An unresolved import stub for `sceKernelAllocPartitionMemory`, from library link order** (§8). `libcglue`'s `_sbrk` calls it to claim the heap; when the stub is unpatched it returns with `v0` untouched, so the "block id" is whatever junk was in `v0`, `sceKernelGetBlockHeadAddr` on it yields 0, `heap_bottom` stays NULL and every later `_sbrk` returns `-1`. **Not** a `PSP_HEAP_SIZE_KB` problem — the size is never even read | **Read the build log**: `Warning: could not fixup imports, stubs out of order` means exactly this. Remove libraries the `psp-gcc` spec already appends (§8). Three-line runtime check: `sbrk(0)` returns `0xffffffff`; `sceKernelMaxFreeMemSize()` is unchanged across a `malloc`; the libcglue block id is not a plausible UID. Terminal proof: `psp-objdump -d` the linked `_sbrk` and confirm the `jal` to `sceKernelAllocPartitionMemory` returns a register no syscall wrote | [HIGH] |
| Engine memory arena (hunk/zone) collapses to its floor once the heap finally works | Any engine port that sizes arenas at boot | Sizing them from `sceKernelMaxFreeMemSize()`. That is the partition **outside** the newlib heap; arenas are `malloc`'d **from** the heap. While the heap is broken nothing is ever claimed, so the number looks sane and the bug hides | Size arenas from `PSP_HEAP_SIZE_KB`; use `sceKernelMaxFreeMemSize()` *after* the heap exists as your **PRX/module** budget (§5) | [HIGH] |
| Large `.bss` (≥10 MB) suspected of not being zeroed by the loader | Engine ports with big static arrays | Usually a misattribution. **Measured on PSP-2000/ARK-4: 10.2 MB of `.bss`, 57 non-zero words total** — the loader zeroes it correctly. Garbage in a `.bss` global is far more likely a real store from a real code path | Scan `&__bss_start`..`&_end` as the first statement in `main()` and bucket the non-zero words before blaming the loader. Note a **PRX** EBOOT with a very large `.bss` is a separate, real problem — it can be refused at load | [MEDIUM] |
| **"The game could not be started. (8002013C)"** — refuses to launch from the XMB; runs fine in PPSSPP; nothing executes so no log is written | Link / module load | **A user-mode module (`PSP_MODULE_INFO(..., 0, ...)`) importing kernel-only stub libraries.** Linking `-lpspkernel` drags in `IoFileMgrForKernel`, `StdioForKernel`, `ThreadManForKernel`, `SysclibForKernel`, `LoadExecForKernel`, `SysMemForKernel`, `ModuleMgrForKernel`. Firmware enforces the user/kernel split at load time. Watch out for functions present in **both** archives — e.g. `sceKernelMaxFreeMemSize` is in `libpspkernel.a` (`SysMemForKernel`) *and* `libpspuser.a` (`SysMemUserForUser`); link order decides which stub you get | Drop `-lpspkernel` from user-mode links; the `*ForUser` equivalents come from `pspuser`/`pspsdk`. **Audit the built binary:** `psp-objdump -s -j .rodata.sceResident <elf>` — nothing named `*ForKernel` may appear in a user-mode module. The same check catches imports of non-resident modules (`sceNet*` before `sceUtilityLoadNetModule`) | [HIGH] |
| Module will not load, or loads into corruption, only when built as a PRX | PRX packaging | **`--gc-sections` on a PRX link.** A PRX links with `-Wl,-q` (emit relocations) because `psp-prxgen` consumes the relocation table; garbage-collecting sections afterwards leaves dangling relocations | Drop `--gc-sections` and the `-ffunction-sections`/`-fdata-sections` that feed it from PRX builds. Neither DaedalusX64 nor Crow_bar's Quake3PSP uses them. The `.eh_frame` savings are negligible on PSP anyway (measured 104 bytes in a 1.4 MB C engine) | [MEDIUM] |
| Renders in PPSSPP, garbage/flicker on hardware | Any dynamic geometry/texture | Missing D-cache writeback before GE reads | `sceKernelDcacheWritebackRange` after CPU writes, or uncached pointers, or `sceGuGetMemory` | [HIGH] |
| Textures look like diagonal scrambled blocks | Texture path | Swizzle flag mismatch (data linear, `sceGuTexMode` says swizzled, or vice versa) | Match the flag to the data; swizzle offline | [HIGH] |
| Texture is stale/previous frame's content | Render-to-texture, procedural textures | GE texture cache not flushed | `sceGuTexFlush()` after modifying texture memory | [HIGH] |
| Audio clicks/stutters | Gameplay under load | Audio pushes on game thread, or buffer <64-multiple, or audio thread priority too low | Dedicated audio thread, priority ~0x12, 1024-sample double buffer | [HIGH] |
| Crash the moment 3D math runs (exception, EPC in your code) | sceGum / VFPU code | Thread lacks `PSP_THREAD_ATTR_VFPU`, or VFPU load from non-16-byte-aligned address | Add the attr; `memalign(16, ¦)` all VFPU-touched data | [HIGH] |
| Crash after loading a large file | Asset loading | Heap exhaustion (~24 MB user RAM) or fragmentation; malloc returned NULL unchecked | Check allocs; raise the explicit `PSP_HEAP_SIZE_KB` value; stream instead of whole-file loads | [HIGH] |
| Buttons dead, analog dead | Input bring-up | `sceCtrlSetSamplingMode(PSP_CTRL_MODE_ANALOG)` missing (nub), or reading with wrong buffer count | Init per §4; read 1 buffer per frame | [HIGH] |
| Works from psplink, "file not found" from XMB (or vice versa) | File I/O | Relative paths + launcher-dependent CWD | Derive absolute base path from `argv[0]` at startup | [HIGH] |
| `ms0:/` I/O fails entirely on a PSP Go | Storage | Go uses `ef0:/` internal storage | Probe both prefixes at boot | [HIGH] |
| 12 FPS on hardware, 60 in PPSSPP | Performance | CPU at 222 MHz, unswizzled textures, doubles, per-poly draw calls, cache misuse | `scePowerSetClockFrequency(333,333,166)`; swizzle; batch; floats only | [HIGH] |
| Exception screen: BadVAddr = pointer-looking value, EPC in memcpy/your code | Runtime | Unaligned access or wild pointer; often a struct read from disk assuming padding | `psp-addr2line` the EPC; pack/serialize structs explicitly | [HIGH] |
| Image squashed/offset, right 32px garbage | Display | Used 480 as framebuffer stride instead of 512 | Stride/bufwidth = 512 everywhere (draw, disp, texcopy) | [HIGH] |
| Distorted/corrupted screen when allocating VRAM past 2 MB on a Slim | VRAM budget expansion | `sceGeEdramSetSize(4*1024*1024)` never called  the aperture is still 2 MB | Call it at init when `kuKernelGetModel() > 0`, then size the allocator from `sceGeEdramGetSize()` | [HIGH] |
| HOME button freezes the app | Any | No exit-callback thread registered | Add the §8 callback boilerplate; poll `exitRequest` in main loop | [HIGH] |
| `sceGeEdramSetSize(4*1024*1024)` returns `0x8002013A`; `sceGeEdramGetSize()` stays 2 MB on a Slim | VRAM budget | The stub is not linkable in that toolchain/CFW combination — an **import** problem, not a logic one. Measured on PSP-2000/ARK-4 with current pspdev | Do not chase it. Size the VRAM allocator from `sceGeEdramGetAddr()`/`sceGeEdramGetSize()` **after** the attempt and budget for 2 MB; treat 4 MB as a runtime-detected bonus | [HIGH] |
| Frame times only ever land on multiples of 16.68 ms; FPS steps 20 / 15 / 12 / 10 with nothing between; an optimisation shows no measurable effect | Profiling / benchmarking | A blocking `sceDisplayWaitVblankStart()` in the frame loop. It quantises median and p95, which is exactly where an A/B result lives | Gate the wait on `sceDisplayGetVcount()` (only wait when running ahead of the panel) and triple buffer; re-measure. See §3 | [HIGH] |
| A texture transform via `sceGuTexMapMode(GU_TEXTURE_MATRIX, …)` renders wrong, and no matrix layout fixes it | Graphics | The GE texture matrix does not behave as `pspgu.h` implies with a two-float (`GU_TEXTURE_32BITF`) coordinate. No reference PSP project uses it | Do the transform on the CPU/VFPU, or fold it into the source coordinates offline. Do not spend more rounds on layout permutations — see §3 anti-patterns | [HIGH] |
| A setting appears to have no effect no matter what it is set to | Any engine port with a cvar/config system | The name does not exist in the code that was actually compiled — it was copied from a different engine or fork. Config systems create unknown names silently instead of erroring | Grep the compiled source for the exact string before debugging behaviour. In one Quake 3 port an injected `r_dynamic 0` was a no-op for five sessions because that renderer's cvar is `r_dynamiclight` — every performance measurement in the project was taken with dynamic lights unintentionally on | [HIGH] |
| Wi-Fi connect fails on modern router | Networking | PSP is 802.11b + WEP/WPA-TKIP only | Legacy 2.4 GHz SSID with WPA-PSK(TKIP); document for users | [HIGH] |
| Association reaches "getting IP" then your code aborts it. WLAN light comes on, then goes out | apctl connect loop | A `state < oldstate` guard. `KEY_EXCHANGE`=6 precedes `GETTING_IP`=3, so WPA runs 2→6→3→4 and always looks like it went backwards | Delete the monotonic check. Fail only on returning to `DISCONNECTED` after leaving it, plus a ~15 s timeout | [HIGH] |
| `sceNetApctlConnect(1)` returns `0x80110601` on a console whose browser works | Profile lookup | `PSP_NETPARAM_ERROR_BAD_NETCONF`  slot 1 is genuinely empty. Stored profiles are 1-based but **sparse** | Scan slots 1..10 with `sceNetApctlConnect` itself; empty slots are refused instantly. Never gate on `sceUtilityCheckNetParam` (it reports only a contiguous run) | [HIGH] |
| Every profile after the first failed one returns `0x80410A80` | Profile scan | `sceNetApctlDisconnect` is asynchronous; apctl refuses a connect during teardown | Poll `sceNetApctlGetState` back to `DISCONNECTED` before the next attempt | [HIGH] |
| A cvar-driven networking default never takes effect | Any engine port with archived cvars | An archived value from an earlier run overrides the new default. A default is not a mechanism | Make the value a *preference* the code falls through, not a decision; write back what actually worked | [MEDIUM] |
| `INADDR_NONE` undeclared | Networking build | PSPSDK's `netinet/in.h` has `INADDR_ANY`/`INADDR_BROADCAST` but not `INADDR_NONE` | Define it locally as `0xffffffff` | [HIGH] |

---

## 11. ANTI-PATTERNS

1. Do NOT use `double`  the hardware FPU is single-precision only; doubles are software-emulated and ~50100Ã slower. [HIGH]
2. Do NOT omit `-G0` from CFLAGS under any circumstances. [HIGH]
2b. Do NOT reach for `PSP_HEAP_SIZE_KB` when allocation fails. **Audit the link first** (§8, §10). Heap tuning is the most attractive wrong answer on this platform: it is one line, it feels causal, and changing it moves the symptom convincingly enough to be mistaken for a fix. Positive vs negative, 36 MB vs 20 MB — none of it matters when `sceKernelAllocPartitionMemory` is unresolved, because the value is never read. [HIGH]
2c. Do NOT link `-lpspkernel` into a user-mode module, and do NOT use `--gc-sections` on a PRX link. Audit the result with `psp-objdump -s -j .rodata.sceResident`. [HIGH]
2d. Do NOT copy build constants forward from an older port without re-testing them. Values from `build.mak`-era projects (`PSP_HEAP_SIZE_KB(-8*1024)`, `PSP_LARGE_MEMORY=1` → `MEMSIZE=1`) are correct **for their own toolchain and firmware** and can be fatal on a current one. Copy the *shape* of a proven port, verify the *constants*. [HIGH]
3. Do NOT trust PPSSPP for cache-coherency correctness or performance numbers  hardware is the oracle. [HIGH]
4. Do NOT forget `sceKernelDcacheWritebackRange` (or uncached pointers) for anything the GE or DMA reads after the CPU writes it. [HIGH]
5. Do NOT run VFPU code (including sceGum) on threads created without `PSP_THREAD_ATTR_VFPU`. [HIGH]
6. Do NOT silently assume a memory profile  detect it (`kuKernelGetModel()`, free-memory probe) and pick your floor deliberately. Slim+ (64 MB, 4 MB VRAM) is the mainstream homebrew target today and a valid baseline for heavy ports if documented; if you claim PSP-1000 support, actually budget for 24 MB and test it. And never assume a second analog stick  every model has one nub. [HIGH]
7. Do NOT use relative file paths; derive an absolute base from `argv[0]`. [HIGH]
8. Do NOT ship without the exit-callback thread; a hanging HOME button fails scene QA instantly. [HIGH]
9. Do NOT place render targets or GE-read data outside VRAM/main RAM (scratchpad is CPU-only). [MEDIUM]
10. Do NOT use 480 as a framebuffer stride  it is 512, always. [HIGH]
11. Do NOT push audio from the render loop; give audio its own higher-priority thread. [HIGH]
12. Do NOT leave the CPU at 222 MHz in a demanding game  request 333/166 explicitly and handle failure gracefully. [HIGH]
13. Do NOT malloc/free per frame  24 MB fragments fast; pool and arena-allocate. [HIGH]
14. Do NOT write to `flash0:/`/`flash1:/` from application code  brick risk. [HIGH]
15. Do NOT emulate missing GPU features (shaders, >512² textures, two texture units) at runtime  solve them in the asset pipeline or cut them. [HIGH]
16. Do NOT treat `sceNetApctl` states as an ordered sequence, and do NOT hardcode network profile slot 1. Both assumptions look correct on an open/WEP AP with a pristine config and fail on a real WPA router with a reused console. [HIGH]
17. Do NOT re-list a library `psp-gcc`'s spec already appends  it splits that library's import stubs and silently corrupts the import table. [HIGH]
18. Do NOT dismiss a toolchain warning because the program "works". `Warning: could not fixup imports, stubs out of order` was printed on **every build** of a Quake 3 port for two sessions and waved off twice as cosmetic, because the engine reached its milestone anyway. It was the root cause of a total heap failure, and the message named the fix (`Ensure the SDK libraries are linked in last`). On this platform a warning about the *import table* or *relocations* is a hard failure that has not surfaced yet. [HIGH]
19. Do NOT debug a PSP boot failure by reasoning about the source. The layers below you  loader, import stubs, `libcglue`, newlib  fail silently and in ways the source cannot express. Instrument, measure, and disassemble the actual linked binary. See §9's checklist. [HIGH]
18. Do NOT let a reference port's networking code be the authority just because it runs. Quake3PSP-mirror ships **two** net implementations that disagree: `unix/unix_net.c` gates on `sceUtilityCheckNetParam` and carries the monotonic-state bug, while `unix/qunix_net.c` does neither. Cross-check against `$PSPDEV/psp/sdk/samples/net/*` before copying either. [HIGH]

---

## 12. PORTING DECISION TREE

1. **Audit the codebase for hard blockers**  `double` math in hot paths, textures >512², shader dependence, RAM footprint >20 MB, threading model.
   *Why first:* these decide feasibility and effort class before any code is written.
   *Skip it and:* you discover at week 4 that the renderer is shader-dependent and the port dies.
2. **Stand up the toolchain + "hello triangle" skeleton** (Makefile from §8, GU init from §3, exit callback, psplink workflow).
   *Why:* every later step needs a known-good build/deploy/observe loop.
   *Skip it and:* you debug engine code and toolchain problems simultaneously  unbounded.
3. **Replace the platform layer**  file I/O  sceIo with absolute paths, timing  `sceKernelGetSystemTimeLow`/vblank, threads  sceKernel threads, allocator  fixed pools sized for 24 MB.
   *Why:* the engine must run headless-correct before graphics fidelity matters.
   *Skip it and:* every graphics bug is confounded by timing/IO bugs.
4. **Port the renderer to sceGu**  map the engine's draw calls to display lists; interleave vertices into GE formats at load; establish the cache-flush discipline immediately.
   *Why:* this is the biggest single work item and gates everything visual.
   *Skip it (e.g., lean on pspgl long-term) and:* you lock in a permanent ~2Ã performance loss.
5. **Asset pipeline pass**  offline downscale/atlas to ¤512², pre-swizzle, CLUT where possible, resample audio to 44.1 kHz PCM.
   *Why:* runtime conversion burns RAM and CPU you do not have; asset size drives load times on slow Memory Sticks.
   *Skip it and:* you ship 15-second level loads and VRAM thrash.
6. **Memory budget enforcement**  hard caps per subsystem; instrument peak usage; decide the supported floor up front. If you support PSP-1000, test on a 32 MB profile even when developing on large-memory CFW; if the port is Slim+-only (a common, legitimate call today), enforce the ~55-56 MB large-memory budget instead and add a model check + clear error at boot.
   *Why:* the memory wall (24 MB Phat / ~56 MB Slim) is the most common late-stage port killer.
   *Skip it and:* the port works in PPSSPP and OOM-crashes on real hardware.
7. **Input mapping**  design around one analog nub + face buttons; add configurable camera assists (auto-center, lock-on) if the source game needs twin-stick.
   *Why:* deferred input design produces unplayable ports even when technically perfect.
8. **Audio integration**  dedicated mixer thread, double-buffered channels, music streaming decision (CPU decode vs pre-converted PCM/ATRAC3+).
9. **Hardware performance pass**  profile on the real device at 333 MHz: batch draws, swizzle audit, VFPU-ify hot math, move hot textures to VRAM, consider 16-bit framebuffer.
   *Why last among engineering steps:* optimizing before correctness wastes effort; but do NOT skip  PPSSPP numbers are fiction.
10. **Release hardening**  HOME/sleep handling (`scePower` callbacks for suspend/resume  Memory Stick and Wi-Fi handles die across suspend and must be reopened [MEDIUM]), PSP Go `ef0:/` support, PARAM.SFO metadata, icon assets, README with CFW requirements.
    *Skip it and:* the port suspends into a corrupted state the first time a user closes the lid.

---

## 13. QUICK REFERENCE CHEAT SHEET

```
PLATFORM: Sony PSP  MIPS Allegrex @ 333MHz (request via scePowerSetClockFrequency(333,333,166)), little-endian, 32-bit
FPU: single-precision only (NEVER use double). SIMD: VFPU (needs PSP_THREAD_ATTR_VFPU, 16-byte alignment)
RAM: PSP-1000 32MB (24MB user); PSP-2000+ (modern target) 64MB (~55MB user w/ CFW large-mem)
  MEMSIZE=2 in PARAM.SFO on FW>3.90 (NOT 1 - build.mak defaults 2; measured 55,641 KB free)
  +4MB volatile mem (partition 5, ALL models): sceKernelVolatileMemLock(0,&p,&s) + scePowerLock(0)
VRAM: budget 2MB eDRAM. sceGeEdramSetSize(4MB) MAY unlock 4MB on Slim+ but is NOT reliable -
  measured 0x8002013A (unresolved stub) on PSP-2000/ARK-4 + current pspdev, size stayed 2MB.
  Always size the allocator from sceGeEdramGetAddr()/GetSize() AFTER the attempt, never from the model.
SCREEN: 480x272 @59.94Hz, framebuffer STRIDE = 512 ALWAYS. Scratchpad 16KB @0x00010000 (CPU-only)
GPU: "GE" fixed-function display-list processor, NO SHADERS, max tex 512x512 pow2, swizzle textures
GFX API: sceGu/sceGum (PSPSDK). Init: DrawBuffer/DispBuffer/DepthBuffer(VRAM offsets), inverted Z (GEQUAL, range 65535->0)
CACHE: 64B lines, write-back. GE/DMA don't see D-cache: sceKernelDcacheWritebackRange after CPU writes; uncached mirror = ptr|0x40000000
AUDIO: sceAudio, 8 ch, 44100Hz s16 stereo, pushes multiple of 64 samples; dedicated high-prio thread + double buffer
INPUT: polled sceCtrl; sceCtrlSetSamplingMode(PSP_CTRL_MODE_ANALOG); nub 0-255 center 128, deadzone ~15%; ONE stick only
STORAGE: ms0:/ (Go: ef0:/), FAT case-insensitive; app dir ms0:/PSP/GAME/App/EBOOT.PBP; absolute paths from argv[0]
NET: 802.11b, WEP/WPA-TKIP ONLY (no WPA2); LoadNetModule(COMMON,INET) -> sceNetInit -> InetInit -> ResolverInit -> ApctlInit -> ApctlConnect(slot)
  apctl states are an ENUM not a sequence: WPA runs JOINING(2)->KEY_EXCHANGE(6)->GETTING_IP(3)->GOT_IP(4). NEVER "state < oldstate" = fail
  profile slots are 1-based but SPARSE: scan 1..10 with sceNetApctlConnect (empty = instant 0x80110601). CheckNetParam is not a gate
  sceNetApctlDisconnect is async: wait for DISCONNECTED or the next connect gets 0x80410A80
LINK: psp-gcc spec already appends -lpsputility -lpsprtc -lpspnet_inet -lpspnet_resolver -lpspsdk -lpspmodinfo -lpspuser (-lm ... -lcglue -lc)
  NEVER re-list a spec library -> split stubs -> "stubs out of order" -> half-patched imports. Add only pspnet, pspnet_apctl, pspwlan
TOOLCHAIN: pspdev (github.com/pspdev/pspdev), psp-gcc, CFLAGS="-O2 -G0 -Wall" (-G0 MANDATORY), BUILD_PRX=1, build.mak -> EBOOT.PBP
BOILERPLATE: PSP_MODULE_INFO + PSP_MAIN_THREAD_ATTR(USER|VFPU) + PSP_HEAP_SIZE_KB(n) + PSP_MAIN_THREAD_STACK_SIZE_KB(512)
  + exit-callback thread (HOME).  Main stack default is only 256 KB. PSP_HEAP_SIZE_MAX() does not exist.
  HEAP: positive is predictable, negative works too (xash3d ships -3*1024). Claimed LAZILY on first _sbrk, not at startup.
  TWO POOLS: malloc <- newlib heap (PSP_HEAP_SIZE_KB); PRX modules + AllocPartitionMemory <- partition OUTSIDE it.
  So size game arenas from PSP_HEAP_SIZE_KB, and read MaxFreeMemSize AFTER first malloc as the PRX budget.
  ALL allocations failing? sbrk(0)==0xffffffff -> audit the LINK, not the heap size. Never the heap size.
LINK: no -lpspkernel in user-mode modules; no --gc-sections on PRX (-Wl,-q) links
  AUDIT: psp-objdump -s -j .rodata.sceResident <elf>  -> no *ForKernel, no non-resident sceNet*
DEBUG: PPSSPP (GE debugger; ignores caches, lies about perf, HLEs all modules so it cannot catch
  import/residency bugs, and MaxFreeMemSize returns fiction) -> psplink+usbhostfs (printf/GDB/exception dumps)
  psplink: PRX only (-DBUILD_PRX=ON), ABSOLUTE usbhostfs path, RelWithDebInfo (Release strips symbols)
  Exception dump: Cause bit31 = BD -> faulting insn is at EPC+4, not EPC
  NO psplink? build the log to survive: open/append/CLOSE per line (no fsync on PSP; a held fd = 0-byte file after a crash),
  stamp __DATE__ __TIME__, and CHANGE THE LOG FILENAME every build (a stale log reads exactly like an early death).
  Print values not conclusions; sanity-check %p against &_end; then psp-objdump -d the function that misbehaves.
TOP BUGS: unresolved import stub from re-listing a spec library ("stubs out of order" -> every malloc returns NULL);
  missing Dcache writeback (works in emu, garbage on HW); stride 480 vs 512; no VFPU attr; relative paths; doubles;
  kernel imports in a user module (8002013C).  NEVER dismiss a build warning about imports or relocations.
```

---

## 14. GOAL-ORIENTED WORKFLOW WITH HARDWARE VALIDATION GATES

When this SKILLS.MD is used in an active development session (not just reference), you must follow this exact state machine. Do NOT proceed to the next state until explicitly instructed by the user.

### STATE DEFINITIONS
- **STATE: ANALYSIS**  Examine the codebase, identify PSP-specific blockers (doubles, texture sizes, RAM footprint, cache discipline, path handling), and plan the implementation.
- **STATE: IMPLEMENTATION**  Write/modify code based on this SKILLS.MD and training data. No hardware testing occurs here.
- **STATE: BUILD_REQUEST**  Output the exact build commands and ask the user to compile and deploy to real hardware (or PPSSPP for logic-only goals  but graphics/cache/performance goals REQUIRE real hardware).
- **STATE: WAITING_FOR_HARDWARE**  STOP. Output ONLY a hardware test protocol. Do NOT write code, do NOT speculate on fixes, do NOT enter debugging loops.
- **STATE: VALIDATION**  User reports back results. Classify the result as SUCCESS, PARTIAL, or FAILURE.
- **STATE: NEXT_GOAL**  On SUCCESS, propose the next logical milestone and ask for confirmation.

### STATE TRANSITION RULES

1. **ANALYSIS  IMPLEMENTATION**: Allowed only after you have identified the specific files to modify and listed them to the user.
2. **IMPLEMENTATION  BUILD_REQUEST**: Allowed only after you have provided a complete, compilable code change. You must include:
   - Exact `make` (or `psp-cmake` + `make`) command
   - Expected output: `EBOOT.PBP` (XMB deploy) or `myapp.elf`/`myapp.prx` (psplink deploy) and its location
   - Transfer method: copy to `ms0:/PSP/GAME/AppName/` via USB device mode, or `pspsh> ./myapp.elf` via psplink/usbhostfs
   - What the user should observe on screen/audio/controller
3. **BUILD_REQUEST  WAITING_FOR_HARDWARE**: You MUST output the following exact header:
   ```
   === HARDWARE TEST REQUIRED ===
   GOAL: [Current Goal Name, e.g., "Render solid-color framebuffer"]
   BUILD: [Command]
   DEPLOY: [Method: ms0:/PSP/GAME copy | psplink host0:/]
   OBSERVE: [Specific expected behavior]
   REPORT BACK: Please reply with EXACTLY one of:
     - SUCCESS: [Describe what you saw]
     - FAILURE: [Describe what you saw, including any error codes, black screen, crashes, EPC/BadVAddr from the exception screen]
   === STOP ===
   ```
   After this header, you STOP generating. You do not offer fixes. You do not guess.
4. **WAITING_FOR_HARDWARE  VALIDATION**: Triggered ONLY by a user message containing "SUCCESS" or "FAILURE".
   - If user says "SUCCESS": Move to NEXT_GOAL.
   - If user says "FAILURE": Move to DEBUG_PROTOCOL.
   - If user says anything else (e.g., "it kind of works", "almost"): Ask for clarification using the SUCCESS/FAILURE binary. Do NOT proceed.
5. **VALIDATION (FAILURE)  DEBUG_PROTOCOL**: You must output a **DEBUG BUILD PROTOCOL**:
   - A minimal C/assembly test case that isolates the failure (e.g., a 30-line program that only sets the framebuffer via `sceDisplaySetFrameBuf` with an uncached pointer, removing sceGu entirely), OR
   - A checklist of exactly 3 diagnostic steps drawn from this document, e.g.:
     1. "Confirm the vertex buffer pointer is flushed: add `sceKernelDcacheWritebackRange(v, size)` immediately before `sceGuDrawArray`  does the corruption change?"
     2. "Run the same ELF via psplink and paste the exception dump (EPC, BadVAddr, registers)."
     3. "Run `psp-addr2line -e myapp.elf <EPC>` and paste the source line."
   - You must ask the user to run this diagnostic and report back.
   - You must NOT rewrite the entire implementation. You must NOT guess and patch simultaneously.
6. **DEBUG_PROTOCOL  WAITING_FOR_HARDWARE**: After providing the debug protocol, you return to WAITING_FOR_HARDWARE state.
7. **VALIDATION (SUCCESS)  NEXT_GOAL**: You propose the next milestone from the goal stack below. You do NOT implement it until the user confirms.

### DIAGNOSTIC METHODOLOGY (learned the expensive way — read before bisecting)

A real session burned **five hardware rounds** on a boot failure whose cause a single psplink
exception dump named instantly. The engineering mistakes, in order of cost:

1. **Get the diagnostic channel working BEFORE the first hardware-gated goal.** If a session's
   gate is "prints its report over psplink", psplink is a prerequisite, not a parallel task.
   Deferring it converts every failure into a one-bit answer (black screen / not) and every
   fix into a guess. This is the single highest-leverage rule in this document.
2. **A control binary must exercise the subsystem under test.** A minimal program that prints
   text and sleeps proves *nothing* about heap, `.bss`, packaging, or imports — it never calls
   `malloc`. Five such "passing" variants were treated as a control group clearing packaging,
   `MEMSIZE`, `.bss` and PRX-vs-ELF; four rounds of theory rested on them. **Ask what the
   control actually executes**, and make it do the thing you are trying to exonerate.
3. **Diff the artifacts, not the source.** `psp-readelf -l` (segments), `psp-objdump -s -j
   .rodata.sceResident` (imports), `psp-size` (text/data/bss) between a working and a failing
   binary is minutes of work and settles questions that source-reading only generates
   hypotheses about.
4. **When a symptom contradicts a shipping reference, the theory is wrong.** "Large `.bss`
   breaks loading" survived three rounds while Crow_bar's Quake3PSP — with *more* `.bss` —
   boots on the same console. Measure the reference (compile its TUs, `psp-size` the objects)
   rather than reasoning about it.
5. **"No text at all" vs "some text then death" is the single most valuable bit** you can get
   from a black-screen failure: it separates load-time rejection from runtime crash. Ask for
   it first, and instrument so the answer is unambiguous (print *before* each risky call).

### ANTI-LOOP PROTOCOL
If the same goal fails hardware validation more than **2 times**, you MUST:
- Stop attempting fixes.
- **Prefer a build that BISECTS over a build that FIXES.** When a change has two or more
  separable parts and it fails, do not guess which part broke — ship them as separate values of
  one runtime-toggleable setting and let a single hardware round say which. In a 2026-08-12
  session a renderer change with two halves (a safe data-layout change and an unproven GE
  feature) was split into levels of one cvar cycled by a pad chord; **one round isolated the
  fault, and the working half shipped** instead of the whole change being reverted. Design for
  this before the first hardware round, not after the second failure.
- **Declare the stopping rule in advance, and honour it.** "If both candidates fail, keep the
  proven half and close the item" — written down *before* the run — is what converts a third
  round into a decision instead of a fourth round. An unbounded hypothesis list on hardware the
  user has to test by hand is how a session becomes a multi-session bug.
- **Stop theorising about the source and switch to measurement.** Two failed rounds means your
  model of the system is wrong, and more reasoning from the same model produces more wrong
  answers — each costing a hardware round the user pays for. Go to §9's boot-failure checklist:
  re-read the build log for warnings you dismissed, inspect the binary statically
  (`psp-objdump`/`psp-nm`/`psp-readelf`), instrument the failing path to print values, and
  disassemble the function that misbehaves. A real case: five rounds went to `PSP_HEAP_SIZE_KB`,
  `.bss` size and a suspected stale deploy; the answer was one line of disassembly showing a
  `jal` returning a register no syscall had written, and a build warning that had been printed
  every single time.
- **Prefer one build that answers several questions over several builds that each answer one.**
  When multiple hypotheses are live, ship one binary that dumps all the relevant values, or
  several binaries the user can test in a single sitting with distinct log filenames.
- Output: `ESCALATION: This goal requires hardware debugging beyond static analysis. Recommend: [psplink GDB stub / exception dump analysis / PPSSPP GE frame-dump comparison / pspdev Discord-forum consultation].`
- Ask the user if they want to:
  a) Skip this goal and mark it as BLOCKED, or
  b) Provide psplink exception dumps / PPSSPP GE frame dumps / register traces for further analysis.

### GOAL STACK (User-Defined or Default)
At the start of the session, the user may provide a GOAL_STACK. If not provided, use this PSP default:
1. Initialize video output (solid-color framebuffer via sceGu clear, or raw `sceDisplaySetFrameBuf` + uncached fill)
2. Initialize controller input (read button presses, draw state as colors)
3. Initialize audio output (play a sine wave on one sceAudio channel from a dedicated thread)
4. Load assets from `ms0:/` (absolute-path resolution from argv[0], read a test file, display its contents)
5. Render main menu framebuffer (textured quad: swizzled texture, cache-flush discipline proven)
6. Main menu input loop (nub deadzone + button navigation)
7. Transition to game state (state machine + clean teardown, HOME exit verified)

You must ONLY work on the **active goal** (top of stack). You must not implement future goals speculatively.
