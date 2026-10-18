# payne-recomp

Max Payne (Xbox) as a native Windows program. It is built from **your own copy
of the game**: the game's code is translated to C on your machine and compiled
against the [xboxrecomp](https://github.com/phobos665/xboxrecomp) toolkit. No
game code or data is in this repository.

Status: every level loads and plays (checked on an Apple Silicon Mac; Windows
was last checked on an older toolkit, through the first levels of each part).
It has not been played through from start to finish yet; see [Status](#status).

Proof of gameplay on macOS (Testing):

https://www.youtube.com/watch?v=cY0Gk71TAGI


## What you need

- Windows 10 or 11, 64-bit, **or** a Mac with Apple Silicon
- On Windows: Visual Studio 2022 (or 2019) with the **Desktop development with
  C++** workload
- On macOS: the Xcode command line tools, Homebrew's `cmake` and `ninja`, and
  the [Vulkan SDK](https://vulkan.lunarg.com/) (MoltenVK, the loader and DXC)
- [CMake](https://cmake.org/download/) 3.20 or newer, on PATH
- Python 3.10 or newer, with `capstone`: `py -3 -m pip install capstone`
  (`python3 -m pip install capstone` on macOS)
- Git
- **Max Payne, PAL release**, extracted from your disc. The build checks
  `default.xbe` and warns if it is a different version; see
  [Other versions](#other-versions).

## Build

```bat
git clone --recursive https://github.com/phobos665/payne-recomp.git
cd payne-recomp
```

Copy the extracted disc into a folder called `game`, so that `game\default.xbe`
exists (with `data\`, `english\` and the rest beside it). Then:

```bat
py -3 ./scripts/build.py
```

The first run translates the game (a few minutes) and then compiles it (longer).
Later runs only recompile. When it finishes it prints where the program is:

```text
build\Release\payne_recomp.exe             the game
build\Release\payne_recomp_launcher.exe    settings window, then starts the game
```

On macOS run `python3 scripts/build.py`; it builds with Ninja into
`build/Release/payne_recomp` and `build/Release/payne_recomp_launcher`.

If you cloned without `--recursive`: `git submodule update --init`, then in
`xboxrecomp`, `git submodule update --init third_party/SDL third_party/volk
third_party/Vulkan-Headers third_party/VulkanMemoryAllocator
third_party/libsamplerate third_party/miniupnp`. A fully recursive update
works too, but it also fetches several gigabytes of emulator firmware that the
build never uses.

## Play

Run `build\Release\payne_recomp_launcher.exe` to choose settings and play, or
`payne_recomp.exe` to start straight away. Either can be shortcut from anywhere.

| Key | Does |
| --- | --- |
| F9 | show or hide the frame rate |
| F10 | step the frame cap: adaptive, 60, 30, off |
| F11 | save a screenshot and a capture of the frame, beside the executable, for bug reports |

An Xbox or XInput controller works as itself. On the keyboard: arrows = D-pad,
Enter = Start, Backspace = Back, Z = A, X = B, A = X, S = Y, Q = White,
W = Black, E = left trigger, R = right trigger. Rebind both in the launcher's
Input tab.

**Where things are kept:**

- Saves: `game\UDATA\54540003\`
- Settings: `%APPDATA%\xboxrecomp\titles\54540003.conf` (the launcher writes it)
- Controls: `%APPDATA%\xboxrecomp\input_bindings.json`
- Log: `build\Release\payne_recomp.log`, when started without a console

## Status

Checked so far, with scripted input:

- start-up, the intro movie with its sound, and the main menu
- New Game: the opening cutscene and graphic novel, with narration
- all 67 levels, each loaded through the game's own level select and played
  for a minute: HUD, walking, shooting and taking damage, at 60 fps
- quick save, which writes a save and costs one short hitch

Known problem: about one start in ten stops with a crash before the main
menu (on macOS; not yet looked for on Windows). Starting again gets past it.

Not checked yet: playing a level through to its end, loading a save, and
sound by ear.

## Other versions

`config/seeds.json` and `config/xdk_symbols.json` are addresses inside the PAL
`default.xbe`. An NTSC or other-revision disc will most likely need its own, and
the build warns when the file differs. Please open an issue with the SHA-1 it
printed.

## Updating

```bat
git pull
git submodule update --recursive
py -3 scripts\build.py --relift
```

`--relift` translates the game again. Do it whenever the toolkit or `config/`
changes.

## Problems

- **"no default.xbe in ..."**: the disc contents are not in `game\`.
- **Black window, nothing happens**: read `build\Release\payne_recomp.log`.
- **A rendering bug in one place**: press F11 there and attach the `.bmp` and
  `.d3dcap` it writes to an issue.

## Legal

You need your own copy of Max Payne. Do not share `game\`, `src\recomp\` (it is
translated game code) or anything under `build\`. This repository and
xboxrecomp contain no game code or assets.
