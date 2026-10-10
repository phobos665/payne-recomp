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
- Through the level select (below), at least one level of every part loads
  and plays, each with its graphic novel page and opening cutscene: the
  tutorial, the Prologue, Part I chapters 1 (Roscoe Street) and 6 (the fire
  escapes), Part II's Nightmare, Hotel, Docks, Restaurant and Manor, and the
  first levels of Part IIIa (the Drug Trip) and IIIb. Walking, turning,
  firing (ammo counts down, then the Lead Pipe), bullet time, taking damage,
  killing and dying all respond.
- No unresolved indirect calls in those runs. Three were found on the way
  and are in `config/seeds.json`: `0x00171740` (Parts I and III),
  `0x00246970` (the Prologue) and `0x00072E90` (Part IIIb). A skipped
  indirect call returns 0 and says so only in the log (`[ICALL] unresolved
  call target`), so grep each new level's log; ninety seconds of one level is
  not the whole level.
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
- **Movie sound broke up (issue #3, 10 Oct).** Bink's sound thread polls its
  115,712-byte ring every ~20 ms but stops for 140-300 ms at a time, and the
  toolkit fed the host only when the game called DirectSound and threw away
  any stretch over 100 ms: up to 640 ms lost in every five seconds of the
  intro movie, the rest crackling. A toolkit service thread now feeds buffers
  and streams on its own clock. The game's other sound is DirectSound streams
  (Xbox ADPCM: music, the novel's narration, mono at 44.1 kHz, and a 22,100 Hz
  one), fed by event; in the intro cutscene and opening novel none of them
  lost any sound before or after the change (`0 skipped unsent`).

Found with a frame capture (`RECOMP_D3D8_CAPTURE`) and the replay tool:
`--list-draws`, then `--no-scene --draws N` to find the draw that paints a
region. The game never clears colour, so without `--no-scene` every partial
replay shows the previous frame and looks complete. Replay at the capture's
own `RECOMP_RES_SCALE`.

Not checked: a level played through to its end, the credits and attract
movies, loading a save, sound by ear.

**Pin:** toolkit `main` at `b775a86` (10 Oct 2026), which has everything
above (merged as #56) and the launcher's single Widescreen switch (#57),
except the movie sound fix, which is on toolkit branch
`fix/dsound-ring-buffer-audio` (`afb73c7`) and pinned from this repository's
`fix/issue-3-audio` branch for review. Once it is merged, pin toolkit `main`
again. Keep the pin on a commit of toolkit `main`: a feature branch's commits
go away when it is squash-merged.

## Widescreen

On with the launcher's one Widescreen switch: `widescreen = 1` in the
settings file (`RECOMP_WIDESCREEN=1`). The toolkit works out the rest from
what this project says about the game -- `recomp_title_widescreen(HOR_PLUS)`
in `CMakeLists.txt`, for the launcher, and the claim the camera code makes at
run time -- and hands the camera its 0.75. A number in `hor_plus` still
decides when one is written out: `hor_plus = 0` with widescreen on is the 4:3
picture stretched.

The game has no 16:9 mode: `CreateDevice` is given no widescreen flag and the
game never asks for the video flags. Its world is fixed-function, drawn
through one viewport for each portal, and its camera keeps a horizontal field
of view (75 degrees) and takes the vertical from the 640x480 rectangle it
also draws into. So (`src/recomp_manual.c`, "the widescreen camera"):

- `sub_000D5DA0`, the camera's set-field-of-view, is wrapped: the field of
  view is widened by 4/3 in the tangent. Culling, portals and projections all
  follow. The picture then grows both ways, and the toolkit crops the rows
  that added (`xbox_D3D8SetVerticalZoom`).
- The 2D -- HUD, menus, graphic novel, text -- is quads at the camera's mid
  depth under the world's own projection, issued through the engine's general
  batch flush. `mp_is_2d_layer` tells the renderer which draws those are
  (`xbox_D3D8SetScreenSpaceTest`); otherwise the HUD is cropped with the
  world.
- Frames with nothing in perspective are shown at 4:3 between bars
  (`xbox_D3D8SetWideFramesAuto`): menus, loading screens, movies.

Checked in the Prologue, the tutorial, Roscoe Street and the Docks: people and
rooms in proportion, portals joined, HUD whole, cutscene letterbox bars across
the picture. To look at a run's frame dumps as shown, resize them to 16:9: the
dump is the 4:3 scene before the stretch.

Open: the HUD sits in the centre 4:3 block rather than at the screen's edges;
the graphic novel is shown wide with its panels at 4:3 in the middle (some
draw in it counts as perspective; not looked into); a field of view the game
sets some other way than `sub_000D5DA0` would not be widened -- the sniper
scope and the Drug Trip's distortion have not been looked at in widescreen.

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

That timing is a fast machine's. On a slow laptop the loading screen holds
for about 100 s before the menu (the script's clock starts at the first pad
read, during loading), and presses held for the default 200 ms were missed:
`RECOMP_SKIP_VIDEO=1`, the first press at 115 s and `:400` on every step
reached the opening novel every time. A press that lands too early is spent
silently; the frame dumps' file times say when the menu came up.

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
