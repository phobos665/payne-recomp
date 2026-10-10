# payne-recomp — working context

Max Payne (Xbox), statically recompiled on the xboxrecomp toolkit, which is the
submodule in `xboxrecomp/`. This repository exists to make **this one game**
work, and then work better than it did on the console. The toolkit exists to
make every game work. Most of the judgement in this repo is deciding which of
the two a change belongs in.

Read `xboxrecomp/CLAUDE.md` too. It holds the toolkit's architecture, its debug
table (`stderr shows` → cause) and its switches. This file does not repeat them.

---

## Toolkit or here?

The rule: **if a change contains no fact about Max Payne, it belongs in
xboxrecomp.** A new D3D8 replacement, a kernel call, a lifter fix, a renderer
feature and a settings option are all toolkit work, even when Max Payne is the
game that found them. Here: Max Payne's overrides, seeds, symbols, per-game
patches and defaults, and tools for its own formats.

**Working on the toolkit from here:** make the change in `xboxrecomp/` on a
branch, commit and push it **there** (`origin` is phobos665/xboxrecomp), then
commit the new submodule pin here. Never leave this repo pinned to a toolkit
commit that exists only on this machine.

---

## The title

| | |
| --- | --- |
| Title ID | `54540003` |
| Release | PAL, certificate region 0x4. `default.xbe` SHA-1 `3250db85c3b51d670436ae240f87d2490fd29509` |
| XDK | 4134, plain D3D8 (not LTCG). Movies are Bink |
| Entry point | `0x000BD2F5` |
| Saves | `game/UDATA/54540003/` |
| Settings file | `%APPDATA%\xboxrecomp\titles\54540003.conf` |

`config/seeds.json` and `config/xdk_symbols.json` are addresses in that exact
XBE. Another region needs its own copies. The NTSC build has the same title ID
and a different XBE: the toolkit's old NTSC seeds landed mid-instruction in
this one.

## Where it came from

Brought up in the toolkit as `titles/maxpayne` and moved here on 7 Oct 2026,
first pinned to toolkit `main` at `bcccf8c`. The toolkit history worth knowing:

- Sep 2026, NTSC disc: four lifter fixes (`sar` signed branches, the backward
  jump-table arm, `repe cmps` carry, cpuid/pushfd/popfd answering as the Xbox
  CPU) and a stop on a missing optional file
  (`camera_shake_rumble_low_6s.kf2`) that the game catches as a C++ exception.
  That stop does not happen on the PAL disc.
- 6 Oct 2026: moved to the PAL disc; seeds rebuilt from its own runs (three
  thread starts, the XPP entries `0x003B140C`/`0x003B1523`/`0x003B1B5A`,
  `0x0002A280`, and Bink's MMX blitters `0x003AAA60`/`0x003AAC20`, reached once
  the MMX probe answers yes).

## State

Relifted on 9-10 Oct 2026 against toolkit `main` of 9 Oct (`ca49951`), with
`src/main.c` and `src/recomp_manual.c` regenerated from the toolkit's current
templates (the older copies do not compile against its register model).
Checked with scripted input (`RECOMP_INPUT_SEQ`), every run muted and in the
background:

- The intro movie plays, then the menu. New Game goes through the difficulty
  screen, the graphic novel, the in-engine "New Jersey, Three Years Ago"
  cutscene and into the Prologue, with its auto save.
- Through the level select (below): the tutorial, the Prologue, Part I
  chapter 1 (Roscoe Street Station) and chapter 6 (the fire escapes), and
  Part II's "The Nightmare" load and play. Walking, turning, firing (ammo
  counts down, then the Lead Pipe), bullet time and dying all respond.
- No unresolved indirect calls in those runs. Two were found on the way and
  are in `config/seeds.json`: `0x00171740` (Parts I and III) and `0x00246970`
  (the Prologue). A skipped indirect call returns 0 and says so only in the
  log (`[ICALL] unresolved call target`), so grep each new level's log.
- 60 fps in the levels.

Toolkit bugs this game found, all fixed on the pinned branch:

- **Movies were black.** The Bink player binds its 640x480 texture once and
  rewrites it every frame; the toolkit looked at a texture's texels only when
  it was bound.
- **The world's textures were the wrong size.** The game leaves an
  environment-map texture matrix on stage 1 with `TEXTURETRANSFORMFLAGS` at
  DISABLE. The toolkit applied it anyway, so every base texture drew at twice
  its size and flipped (wall tiles too large, signs unreadable), and decals
  sampled the wrong lightmap texels (floor litter solid black, a graffiti tag
  at full brightness).
- **Graphic novel pages were taken for the screen.** They are allocated on
  memory that was a frame buffer before a display-mode change. Harmless to the
  picture here; 43 failed texture creations in the opening novel.
- Earlier (7 Oct): novel panels black (`D3DRS_*MATERIALSOURCE`), and in-engine
  cutscenes black (Clear with rectangles; in toolkit `main` since 9 Oct).

Found with a frame capture (`RECOMP_D3D8_CAPTURE`) and the replay tool:
`--list-draws`, then `--no-scene --draws N` to find the draw that paints a
region. The game never clears colour, so without `--no-scene` every partial
replay shows the previous frame and looks complete. Replay at the capture's
own `RECOMP_RES_SCALE`.

Not checked: Part II beyond its first level, Part III, the credits and
attract movies, loading a save, sound by ear. Widescreen is not done: the
game has no 16:9 mode (its `CreateDevice` flags are 0 and it never asks for
the video flags), its world is fixed-function with one viewport per portal,
and its camera's FOV is horizontal (75 degrees, set through `sub_000D5DA0`),
with the vertical extent taken from the 640x480 rectangle.

**Pin:** toolkit branch `feat/maxpayne-relift`: `main` (`ca49951`) plus the
material-source fix and the three fixes above, and `--no-scene` for the replay
tool. They change what every title draws, so they need the toolkit's Windows
regression (TimeSplitters 2, BLiNX) before `main`. Move the pin to `main` once
they are merged there.

## Getting to any level

`T:\EnableDeveloperMode.txt` -- any file of that name in
`game/TDATA/54540003/` -- turns on the game's own developer mode. The main
menu gains a `[developer]` entry, selected when the menu opens: God Mode, Get
Bullet Time, Get All Weapons, Display FPS, and **Preview Levels**, which lists
Intro, Part Ia, Ib, II, IIIa and IIIb and under each every level. Developer
mode also draws a grey bar two pixels wide down the right edge of the screen;
that is the game's, not a rendering fault.

The menus move with the **left stick**, not the d-pad. In a script, leave
1.5 s between presses, and do not press before about 15 s after launch: an
earlier press is lost. To Preview Levels: `4000:start,15000:a`, seven
`lstick_down`, `a`.

## Running it unattended

The toolkit's switches apply (`xboxrecomp/CLAUDE.md`). Specific to here:

- **Always give a run a script** (`RECOMP_INPUT_SEQ`, even one that presses
  nothing for an hour). A pad left connected is read by a background window
  too; one unscripted run walked the menus by itself and cleared the control
  bindings. A script owns the pad.
- **Saves are not isolated by `RECOMP_SAVE_DIR`.** The game saves through
  `\Device\Harddisk0\Partition1\UDATA`, which the toolkit maps into the game
  folder. Run tests against a mirror of the game folder instead
  (`RECOMP_GAME_DIR`): junctions to `data`, `english` and the other language
  folders, copies of the loose files and of `TDATA`, and no `UDATA`. Never
  delete through a junction, and note `game` here is one: `..` from inside it
  is not this repository.
- **Only one copy at a time.** The game opens `english/x_data.ras` with no
  sharing, so a second copy (a test run while someone plays) fails to open it,
  throws `ArchiveNotFound` and faults at start-up in `sub_000E9B80` writing
  0xFFFFFFF8. Check `tasklist` for `payne_recomp.exe` first.
- An exit with `ExitProcess(0)` from `shadow_wndproc` -> `xbox_HostExit` is
  the window being closed, not the game quitting.
- Start pressed in a level pauses, and the pause menu's first item is Quick
  Save, which writes `UDATA/54540003/<id>/SaveData.Mps`.
