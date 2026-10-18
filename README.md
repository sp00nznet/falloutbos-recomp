# falloutbos-recomp

Static recompilation of **Fallout: Brotherhood of Steel (Xbox, 2004)** to a
native Windows executable, using [xboxrecomp](https://github.com/sp00nznet/xboxrecomp).

No emulation. The x86 in `default.xbe` is translated to C, compiled, and linked
against the xboxrecomp runtime (kernel replacement, NV2A pushbuffer executor,
MCPX APU, USB). The game shipped on Xbox and PS2 only; there was never a PC
version.

This repo follows the xboxrecomp house style (layout, `regen.sh`, netlab
recipe) shared with [hl2-recomp](https://github.com/sp00nznet/hl2-recomp) and
the other titles. **It contains no game code or data.** The generated C is
derived from your own disc and is never distributed; you produce it locally.

## Status

**v0.1.0-dev, alpha.** It boots with no arguments and nothing forced, and the
modelled Xbox controller drives it: through the main menu, character and
difficulty select, chapter 1's intro, and into the first level, Carbon, with
Nadia on screen, the HUD up and the world textured. 0 faults and 0 unresolved
indirect calls in a 270 s scripted run.

Not yet good: the software rasteriser runs the level at about 5 fps, the Bink
videos show scattered block artifacts, and no one has played past the opening
dialogue yet.

![Fallout: Brotherhood of Steel (Xbox) recompiled, in Carbon](docs/images/carbon.png)

![The main menu](docs/images/main-menu.png) ![Difficulty select](docs/images/difficulty.png)

*`fbos.exe` with `RECOMP_PB_EXEC=1`, captured from the surface the title's own
pushbuffer drew into. The level shot was reached headless with a scripted pad.*

| Step | State |
|---|---|
| XBE parsed | done: entry `0x0009547C`, 28 sections, 122 kernel imports, XDK 5558 |
| Disassembled | done: 3,881 functions (no RTTI in this image) |
| Codegen | done: 3,888 functions, 543k lines of C, 1 unresolved target (`sgdt`, never reached) |
| Kernel bridge | done: 122/122 imports resolved |
| D3D device + display mode | done: 640x480 X8R8G8B8 |
| DirectSound / APU | done: the title starts the APU itself, streams create |
| Async file streaming | done: needs deferred I/O APCs (see [docs/boot.md](docs/boot.md)) |
| Bink FMV | plays: Interplay logo, intro; block artifacts in motion |
| Main menu | done |
| Controller | done: the title's own USB stack enumerates the modelled pad |
| Menus to gameplay | done: character select, difficulty, chapter intro, Carbon |
| Gameplay | renders at ~5 fps; not played past the opening |

What it took, and why, is in [docs/boot.md](docs/boot.md).

## Getting Started

> A one-click `Setup.cmd` (REPO_RULES section 5's Quick start) is not written
> yet; see [ROADMAP.md](ROADMAP.md). Until then this is the technical route.

### Prerequisites

- Windows 10/11, x64
- Visual Studio 2022 with the C/C++ desktop workload, or clang-cl
- CMake 3.20+
- Python 3.10+ with `capstone` (`py -3 -m pip install capstone`). Use `py -3`,
  not `python`: the Microsoft Store alias for `python` is not a real install.
- Git
- Your own **Fallout: Brotherhood of Steel (USA)** disc image (`.iso`)

### Step by step

1. Clone this repo and the toolkit side by side, so the toolkit is `../xboxrecomp`:

   ```bash
   git clone https://github.com/sp00nznet/falloutbos-recomp falloutbos
   git clone https://github.com/sp00nznet/xboxrecomp xboxrecomp
   ```

   This port currently needs three toolkit changes that are not merged yet
   (see [Toolkit changes](#toolkit-changes)). Check out a toolkit branch that
   has them.

2. Unpack your disc into `game/`:

   ```bash
   cd xboxrecomp
   py -3 -m tools.xiso unpack "/path/to/Fallout - Brotherhood of Steel (USA).iso" -o ../falloutbos/game
   ```

   Expected: `extracted 491 files to ../falloutbos/game`.

3. Generate the C (about 3 minutes, most of it the disassembler):

   ```bash
   cd ../falloutbos
   ./regen.sh --disasm
   ```

   Expected, at the end:

   ```
   3888/3888 functions (0 failed), 543045 lines of C, 8 source files
   1 unresolved call targets stubbed (addresses called but not detected as functions)
   ```

4. Build:

   ```bash
   cmake -S . -B build-msvc -A x64
   cmake --build build-msvc --config Release
   ```

   The executable lands in `bin/fbos.exe`.

5. Run from the repo root (it reads `game/default.xbe`):

   ```bash
   RECOMP_PB_EXEC=1 RECOMP_FB_WINDOW=1 ./bin/fbos.exe
   ```

   An XInput controller (Xbox 360/One pad) plugged into the PC is player 1.
   Without one, add `RECOMP_KEYBOARD=1`: WASD left stick, IJKL right stick,
   Z/X/C/V = A/B/X/Y, Enter = Start, Backspace = Back, 1/3 = triggers. Keys are
   read from the game window while it has focus.

## Usage

| Variable | Effect |
|---|---|
| `RECOMP_PB_EXEC=1` | execute the title's NV2A pushbuffer, so it actually draws |
| `RECOMP_FB_WINDOW=1` | show the framebuffer in a window. Off by default: runs are headless |
| `RECOMP_FB_DUMP=prefix` | write frames as `prefix00000.bmp` ... |
| `RECOMP_FB_DUMP_FLIPS=<file>` | dump every flip while `<file>` exists |
| `RECOMP_WATCHDOG_SECS=N` | after N seconds, report where the guest is |
| `RECOMP_KERNEL_LOG_BUDGET=N` | log the first N kernel calls (default 200) |
| `RECOMP_KEYBOARD=1` | keyboard as player 1 (see Getting Started) |
| `RECOMP_PAD_SCRIPT="ms:button,..."` | scripted presses for headless runs, e.g. `25000:start,35000:a` |
| `RECOMP_USB_STATS=1` | controller transfer counts every few seconds |

`RECOMP_AC97_READY` and `RECOMP_USB` are set by `main.c` itself: the title
cannot initialise DirectSound without the first, or see a controller without
the second.

From power-on to Carbon, headless:

```bash
S="25000:start,35000:a,50000:a,65000:a"
for t in $(seq 80000 5000 250000); do S="$S,$t:a"; done
RECOMP_PAD_SCRIPT="$S" RECOMP_PB_EXEC=1 RECOMP_FB_DUMP=shot ./bin/fbos.exe
```

Headless capture, no window, works over RDP:

```bash
RECOMP_PB_EXEC=1 RECOMP_FB_DUMP=shot ./bin/fbos.exe
```

## Building from source

`./regen.sh` re-runs func_id, ABI analysis and codegen from the existing
disassembly; `./regen.sh --disasm` starts from the XBE. `RECOMP=<path>` points
it at a toolkit checkout other than `../xboxrecomp`. Entry tracing for
bring-up: list addresses in `config/trace_functions.json` and run
`TRACE_FUNCS=config/trace_functions.json ./regen.sh`.

Day-to-day builds go through [recomp-netlab](https://github.com/sp00nznet/recomp-netlab):
`netlab build falloutbos-recomp` (recipe `projects/falloutbos-recomp.env`).
The farm never sends `game/` or the disc image.

### Toolkit changes

Found by this port, each on its own branch in xboxrecomp:

| Branch | What |
|---|---|
| `fix/disasm-tables-in-code-sections` | function-pointer tables inside XDK code sections (D3D's render-state handlers) |
| `fix/io-apc-delivery` ([#226](https://github.com/sp00nznet/xboxrecomp/pull/226)) | file I/O APC: restore esp after delivery; opt-in delivery at the next alertable wait |
| community PR #186 | `pushfd`/`popfd`/`cpuid` lifted as the hardware does them. Until it lands, `recomp_manual.c` overrides the one function that needs it |

## License

MIT, see [LICENSE](LICENSE). That covers the code in this repo only.
Fallout: Brotherhood of Steel and its data belong to their owners; nothing from
the game is in this repo, and the generated C is never committed.
