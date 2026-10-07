# Installing

Tested only on a **PS4 Pro, firmware 12.02, GoldHEN**. Other firmwares or a base PS4 may or may not
work (a base PS4 would also be slower).

## 1. The package

1. Download `nx-on-orbis-v0.1.0.pkg` from the release.
2. Copy it to the console with FTP in **binary** mode (FileZilla works; some clients corrupt binary
   files), for example to `/data/pkg/`, and install it with GoldHEN's package installer.
3. It shows up as **NX on Orbis** (title ID `EDPS00001`).

The package contains the emulator, the OpenOrbis sample system modules every OpenOrbis homebrew
ships (`libc.prx`, `libSceFios2.prx`, `right.sprx`), and the Homebrew Menu (nx-hbmenu, ISC license),
which starts when no game is installed and needs no keys. It contains no keys, firmware or games.

## 2. Your files

Everything lives under `/data/edenps4/` (created on first start):

| Path | What |
|---|---|
| `keys/prod.keys` (+ `title.keys`) | Keys dumped **from your own Switch** (Lockpick_RCM) |
| `firmware/*.nca` | Firmware dumped **from your own Switch** (TegraExplorer / NXDumpTool); copied into the emulated NAND on first start |
| `roms/*.nsp`, `roms/*.xci` | Your games; listed in a menu at start |
| `updates/` | Update and DLC NSPs, applied when the game starts |
| `game.txt` | Optional: full path of a game to start directly |
| `nomenu.txt` | Optional: skip the game menu and start the last game played |
| `settings.txt` | Optional overrides, one `key=value` per line (below) |

## 3. Using it

- Game menu: Up/Down to choose, Cross to start.
- Controls (Switch layout by position): Circle = A, Cross = B, Triangle = X, Square = Y, L1/R1 = L/R,
  L2/R2 = ZL/ZR, Options = +, touch pad click = -, L3/R3 = sticks.
- Hold **Options + touch pad for 2 seconds** to quit.
- The first runs of a game stutter while shaders compile; they are cached for later runs.

## 4. settings.txt (optional)

Lines are `key=value`; `#` starts a comment. Each recognized line is logged in `boot.log`.

| Key | Values (default first) | What |
|---|---|---|
| `cpu_accuracy` | `unsafe`, `auto`, `accurate` | dynarmic accuracy (unsafe = faster floating point) |
| `gpu_accuracy` | `low`, `high` | GPU emulation accuracy |
| `reactive_flushing` | `off`, `on` | flush GPU data back when the CPU reads it |
| `async_shaders` | `on`, `off` | compile shaders in the background |
| `astc` | `cpu`, `async`, `gpu` | ASTC texture decoding (async and gpu crashed in tests) |
| `profile` | `on`, `off` | sampling profiler in `boot.log` |
| `dyna_state` | `0`-`3` (2) | Vulkan extended dynamic state level |
| `vertex_input_dynamic` | `on`, `off` | |
| `bgra`, `rg`, `a2b10`, `swizzle` | see `frontend/main.cpp` | color self-tests and workarounds (diagnostics) |
| `env` | `NAME=VALUE` | environment for the GPU driver, e.g. `env=ORBIS_ARENA_MIB=1280` |

## 5. Logs (for bug hunting)

- `/data/edenps4/boot.log`: startup checks, a status line every 10 s (fps, memory), the profiler
  every 30 s, and crash reports (registers + `eboot+0x...` backtrace). The previous run is kept as
  `boot.old.log`.
- `/data/edenps4/mesa.log`: the GPU driver's log (previous run: `mesa.old.log`).
- `/data/edenps4/log/eden_log.txt`: Eden's own log.

Symbolize crash offsets with the ELF from the same release:
`llvm-symbolizer --obj=nx-on-orbis-v0.1.0.elf -C -f 0x<offset>`.

Please do not send reports about this port to the Eden project.
