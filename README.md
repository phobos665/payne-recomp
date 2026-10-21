# payne-recomp

Max Payne (Xbox, PAL) as a native program for Windows and Apple Silicon Macs.
You supply your own copy of the game. Its code is translated to C on your
machine and compiled against the
[xboxrecomp](https://github.com/phobos665/xboxrecomp) toolkit. This repository
holds no game code or data.

[Gameplay on macOS (video)](https://www.youtube.com/watch?v=cY0Gk71TAGI)

## Status

**Playable.** Last checked on 21 Oct 2026 against xboxrecomp `main` (`9bf3ec2`)
plus one fix, on an Apple Silicon Mac, with scripted input:

| | |
| --- | --- |
| Start-up, intro movie, main menu | Works, with sound |
| New Game: opening cutscene, graphic novel, narration | Works |
| Levels | All 67 load through the game's level select. Each played for 48 s (walking, shooting, taking hits): 66 without a fault |
| Frame rate | 60 fps |
| Sound | No dropouts measured in any run (muted test runs; not yet re-checked by ear) |
| Start-up | The occasional crash before the menu (about 1 start in 10) is fixed |
| Widescreen | Optional (launcher switch); see [Widescreen](#widescreen) |

Known problems:

- Part Ib, *Rooftops and Alleys* 3 can crash about a minute in (2 of 4
  scripted runs). It predates this toolkit update and is being looked into.

Not checked yet: a level played to its end, loading a save, the credits.
Windows was last checked on an older toolkit.

## What you need

- Windows 10/11 (64-bit) **or** a Mac with Apple Silicon
- Windows: Visual Studio 2022 or 2019 with **Desktop development with C++**
- macOS: Xcode command line tools, Homebrew `cmake` and `ninja`, and the
  [Vulkan SDK](https://vulkan.lunarg.com/)
- CMake 3.20+, Git, and Python 3.10+ with `capstone`
  (`python3 -m pip install capstone`)
- **Max Payne, PAL release**, extracted from your disc

## Build

```sh
git clone --recursive https://github.com/phobos665/payne-recomp.git
cd payne-recomp
```

Copy the extracted disc into a folder called `game`, so `game/default.xbe`
exists with `data/`, `english/` and the rest beside it. Then:

```sh
python3 scripts/build.py      # Windows: py -3 scripts\build.py
```

The first run translates the game (a few minutes), then compiles it. The
program lands in `build/Release/`: `payne_recomp` (the game) and
`payne_recomp_launcher` (settings, then the game). On Windows these are
`.exe` files.

Cloned without `--recursive`? Run `git submodule update --init`, then in
`xboxrecomp/`: `git submodule update --init third_party/SDL third_party/volk
third_party/Vulkan-Headers third_party/VulkanMemoryAllocator
third_party/libsamplerate third_party/miniupnp`. Those are all the build needs.

## Play

Start `payne_recomp_launcher` to pick settings, or `payne_recomp` to play
straight away.

An Xbox or XInput controller works as itself. On the keyboard: arrows = D-pad,
Enter = Start, Backspace = Back, Z = A, X = B, A = X, S = Y, Q = White,
W = Black, E = left trigger, R = right trigger. Rebind in the launcher's Input
tab.

| Key | Does |
| --- | --- |
| F9 | Show or hide the frame rate |
| F10 | Step the frame cap: adaptive, 60, 30, off |
| F11 | Save a screenshot and a frame capture beside the program, for bug reports |

| What | Where |
| --- | --- |
| Saves | `game/UDATA/54540003/` |
| Settings | `%APPDATA%\xboxrecomp\titles\54540003.conf`; on macOS `~/Library/Application Support/xboxrecomp/titles/54540003.conf` |
| Log | `build/Release/payne_recomp.log`, when started without a terminal |

### Widescreen

Turn on **Widescreen** in the launcher. The game has no 16:9 mode of its own,
so the camera is widened instead (Hor+). Menus, loading screens and movies stay
4:3 between bars. The HUD stays in the middle 4:3 area.

## Updating

```sh
git pull
git submodule update --recursive
python3 scripts/build.py --relift
```

`--relift` translates the game again. Do it whenever the toolkit or `config/`
changes.

## Problems

- **"no default.xbe in ..."**: the disc contents are not in `game/`.
- **Black window**: read the log (above).
- **Something draws wrong in one place**: press F11 there and attach the
  `.bmp` and `.d3dcap` it writes to an issue.
- **A different disc**: `config/` holds addresses inside the PAL
  `default.xbe` (SHA-1 `3250db85…`). The build warns if yours differs; other
  versions need their own. Please open an issue with the SHA-1 it prints.

## Legal

You need your own copy of Max Payne. Do not share `game/`, `src/recomp/`
(translated game code) or anything under `build/`. This repository and
xboxrecomp contain no game code or assets.
