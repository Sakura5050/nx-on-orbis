# Status, expectations and projected performance

Written when the project was closed (2026-10-07). Everything measured comes from `boot.log` /
`mesa.log` captured on one PS4 Pro (firmware 12.02, GoldHEN), with Mario Kart 8 Deluxe (base game,
NSP v0) as the main test title.

## What the project set out to do

1. Get Eden (Switch emulator) to build and run as a native PS4 application, reusing what the PS5
   port ProsperoEden had already solved.
2. Boot a commercial game. Mario Kart 8 Deluxe was the target.
3. Get into a race. ("I need the next build to run the race, even if the color is wrong.")
4. Then speed first, colors second.

The hope was "playable". The honest expectation, given the CPU (see below), was always that a
heavy 3D title like MK8D would run well below full speed.

## What was reached

| Milestone | Test | Result |
|---|---|---|
| Probes: package layout, memory, JIT, GPU | probes 1-3 | Found the package layout that gets 4.4 GiB of direct memory and 2.1 GHz CPU clocks; RWX JIT memory works; double mapping works |
| Eden runs | 2 | Homebrew Menu runs at full CPU speed, GPU draws at 26-29 fps |
| First commercial game | 2-6 | MK8D title screen; menus at 40-60 fps after fixing memory use (fiber stacks, JIT caches) |
| Own game picker | 8-9 | Up/Down + Cross menu on the video output |
| GPU hangs on race load | 12-13 | Driver flattens each submission into a 2 MiB buffer: submit every 256 draws |
| Texture cache corruption | 14-20 | Root cause: the SDK's `wmemchr` searches 16-bit units while the compiler's `wchar_t` is 32-bit; libc++ routes `std::find` over 4-byte types to it. Fixed by 4-byte replacements in the executable |
| GPU out of memory on race load | 20-21 | GPU arena 1280 MiB, texture/buffer cache budgets sized to it, mid-frame collection |
| **Races play** | **21** | Two laps, ~15 minutes without crashing, 10-27 fps, sound perfect |
| Speed work | 22-24 | Jaguar-tuned build (`-march=btver2`), Android-like GPU defaults, sampling profiler, `cpu_accuracy=unsafe`, guest memory placed to limit fragmentation: races at 13-16 fps |
| Fastmem | 25-29 | Works (experimental branch) but races no longer load: out of direct memory |

Open: colors (red/blue swapped on 3D models and videos), speed, memory headroom, and one
heap-corruption crash seen once (test 28).

## Where the time goes (race, v0.1.0)

From the built-in sampling profiler (every 4 ms per thread, reported every 30 s) and the driver's
own budget lines:

- Frame time 60-70 ms (13-16 fps). The game itself reports 100% speed (it drops frames, it does not
  slow down).
- Emulated CPU core 0: **76-84% busy**, and **70-87% of that is JIT-generated code** (the game's
  own ARM code, translated). Core 1: 50-60%, core 2: 40-50%, core 3: idle.
- GPU thread ~30% busy; Vulkan worker ~10-14%.
- The driver reports **0 ms waited for the GPU** in every 5-second window: the GPU is not the limit.
- 3.2-3.9 of the ~7 usable cores are busy on average.
- Note: MK8D is a 32-bit (AArch32) game; the A32 translator is what runs.

In short: the game's main thread, translated from ARM to x86 and running on a 2.1 GHz Jaguar core,
cannot produce frames faster than this.

## Projected maximum performance

These are estimates, not measurements.

- **Why the ceiling is low.** The Switch runs this code natively on four Cortex-A57 cores at
  1.02 GHz. Here every guest instruction goes through a dynamic recompiler, whose output typically
  costs several host instructions per guest instruction, on a Jaguar core (2013, low-power, two-wide
  decode) at 2.13 GHz. The PS4 Pro has roughly a third of the per-core speed of the PS5's Zen 2,
  which is what ProsperoEden targets.
- **What is left to gain**, roughly:
  - Fastmem (guest memory access in one instruction instead of a page-table walk): +10-30% on the
    JIT-bound threads, if the redirect storm is fixed (see below).
  - Fewer stutters (shader compilation is already asynchronous and cached on disk; first runs
    stutter more).
  - Smaller items (JIT cache sizing, thread placement): a few percent each.
- **Projection for MK8D races on a PS4 Pro: about 20-25 fps at best**, with stutters. A stable
  30 fps is unlikely; 60 fps is not reachable on this CPU. A base PS4 (1.6 GHz) would be roughly a
  quarter slower again.
- **Lighter games** (2D, indie, games that are not CPU-heavy on the Switch) are where this port has
  a realistic chance of running at full speed. That was never tested systematically.

## Memory budget (race)

The process gets 4608 MiB of direct memory. In a race (v0.1.0): guest RAM committed on demand
~1.5 GiB, GPU driver arena 1280 MiB, C heap 768-896 MiB (128 MiB carve-outs), JIT code caches
4 x 32 MiB, and the driver's GARLIC allocations for images, which live **outside** the arena and
grow by several hundred MiB while a race loads. About 100 MiB stay free. See `TECHNICAL.md`.

## The experimental branch (tests 25-29)

- **Fastmem works**: an 8 GiB view (512 GiB was refused, 0x8002000c; a 256 GiB one starved the
  address space so the guest page table could not be reserved) whose 16 KiB pages alias the
  lazily committed guest RAM and are mapped on first touch; JIT faults go to dynarmic's slow path
  through a PS4 exception handler. A startup self-test (aliasing, `mprotect`, rip/rsp redirect)
  gates it; `fastmem=off` in `settings.txt` disables it.
- **But**: ~370,000 slow-path redirects per session, almost all "unmapped" (a 16 KiB host page
  whose four 4 KiB guest pages are not all mapped to consecutive backing). Each one recompiles a
  block. Understanding why (guest physical memory fragmented at 4 KiB? mappings that never reach
  the view?) is the first thing to do there.
- **And** fastmem's bookkeeping costs ~128 MiB more C heap, which was enough to run direct memory
  out while loading a race (tests 27 and 29), even with the GPU arena reduced to 1152 MiB.
- Ideas not tried: Eden's stream buffer from 256 to 128 MiB on the PS4 (it is a 256 MiB GARLIC
  allocation outside the arena); lower texture-cache budgets; `env=ORBIS_VRAM_GARLIC=0` (keeps
  images inside the arena, which removes the double cost, but the driver warns it costs GPU speed).
- Test 28 crashed in `getenv` inside the driver because the heap block holding the environment had
  been overwritten with log text. The branch moves the environment to static storage and logs
  `!! HEAP CORRUPTION` if the old block changes. The writer was not found.

## Colors: what was ruled out

Red and blue are swapped on characters, karts and the game's videos (decoded YUV converted by the
game's own shader into A2B10G10R10 render targets); the UI is correct. Tested and correct on the
console, so not the cause: B8G8R8A8 sampling and blits, all 24 component-mapping permutations
(compute self-test), R8G8 sampling, A2B10G10R10 sampling and rendering (both orders), ASTC decode
on the CPU, the swapchain. The VIC (video decoder) output frame was dumped and is correct on a PC.
Still open: Eden's shader translation for this GPU's feature set (no float16, etc.) or something in
how the game's YUV->RGB shader is translated.

## If you want to continue

1. Read `TECHNICAL.md` first, then the Spanish log (`dev-log-es.md`) for the details of each test.
2. Logs: `/data/edenps4/boot.log` (startup checks, status every 10 s, profiler every 30 s, crash
   reports with registers and an offset backtrace), `/data/edenps4/mesa.log` (the driver),
   `/data/edenps4/log/eden_log.txt` (Eden). Symbolize `eboot+0x...` offsets with the ELF from the
   release: `llvm-symbolizer --obj=nx-on-orbis-v0.1.0.elf -C -f 0x<offset>`.
3. Most promising for speed: the fastmem redirect storm, then memory headroom so races load with
   fastmem on.
4. Most promising for colors: compare Eden's SPIR-V for MK8D's 3D and video shaders between a PC
   and this GPU profile.
