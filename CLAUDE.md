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

**21 Oct 2026: relifted against toolkit `main` at `9bf3ec2`** (#70-#76: the
heap lock merged, the XGRA audio work, Turok's D3D sync points, PES5's
narrow multiply). No new seeds; `main.c` and `recomp_manual.c` unchanged.
Regression with a scratch harness (one APFS-clone mirror per run, muted,
background, `RECOMP_INPUT_SEQ`): boot and intro movie, New Game to the
tutorial, and all 67 Preview Levels, 48 s of play each, 4 at a time.

- **The start-up crash below is fixed, and it was never a host-thread race.**
  On `9bf3ec2` it hit every level load. `sub_001DD9F0`, the particle
  factory, creates nothing unless `sub_000F6F70` says more than 3.5 MB is
  free (XAPI `GlobalMemoryStatus` at `0x002DA02C`, i.e. `MmQueryStatistics`).
  The loader stores the NULL and calls through it at `0x001DED16`. The
  toolkit answered with the *host's* free memory, and on macOS
  `GlobalMemoryStatusEx` was a TODO that filled nothing in, so the figure was
  uninitialised stack: #76's diagnostic hook in `kernel_bridge.c` changed
  what was left there, and 1 boot in 10 became 10 in 10. Fixed in toolkit
  `fix/mm-query-statistics` (`1d469e9`): the console reports half its 64 MB
  free, always. Found by bisecting #71-#76 (`7bf9c18` 0/4 crashes, `4f70cc6`
  0/4, `829bba4` 4/4; `829bba4` with only `kernel_bridge.c` reverted 0/4).
- After the fix: 68 of 69 runs clean. Every level loaded its own `.ai`, no
  unresolved indirect calls, no black frames, 58-60 fps with one copy
  (Part IIIb dipped to 31-45 fps only with four running). The host's voices
  never ran dry in any run; the only DirectSound discards are 64-byte buffers
  in five Part II/III levels during those four-copy stalls.
- **Open: Part Ib, Rooftops and Alleys 3 (`p1_l4-3`, L2.06).** About 88 s
  into the scripted run, `sub_002379F0` (a range copy of 28-byte elements,
  called from `sub_00238830`) is handed a garbage `first`/`last` pair and
  reads off the end at `0x74000000`. Same registers every time
  (`eax=0x73F24D09`). Not a regression: the `7bf9c18` build crashes there 3
  of 3, the `9bf3ec2` one 2 of 4. The 18 Oct pass drove that level with
  different input and did not reach it. Next step: `RECOMP_WATCH_WRITE` on
  the container that holds the pair.

**18 Oct 2026: relifted against toolkit `main` at `26c5215` (#69) and built
on macOS (Apple Silicon, Vulkan on MoltenVK).** `src/main.c` regenerated from
the current template; `CMakeLists.txt` and `scripts/build.py` made portable
(Ninja into `build/Release/`, as split2-recomp does). The lift needed no new
seeds. Checked, muted and in the background, with scripted input:

- **Every level in Preview Levels, all 67**, each in a run of its own: it
  loads the intended level (the log names its `Levels\<part>\<level>.ai`),
  then walks, turns, fires and takes hits for 48 s. No unresolved indirect
  calls and no faults in any level. 60 fps in nearly all; Rooftops and
  Alleys-2/-3 and Docks A-2 dipped to 33-46 fps with four copies running at
  once, not rechecked alone.
- **Sound.** The intro movie plays through with sound; New Game's opening
  cutscene and novel narration measure as speech (zero-crossing rate 1-10%,
  peaks -15 to -27 dBFS), not the clipped noise of issue #3, with a narration
  stream recreated stereo at the address a mono one had. No voice ran dry in
  any level (`RECOMP_AUDIO_METER=1`). The "voices reset with sound queued"
  count is the game stopping its own buffers; it is 0 in the menus.
- **Start-up crashes, found and half fixed.** About one start-up in ten died
  before the menu, in two ways:
  - *Fixed, toolkit `fix/heap-alloc-lock` (`e02bcce`).* The toolkit's heap
    had no lock; a host thread's blocks and the main thread's 1 MB
    reservation all got 0x00F80000, and the game opened `D:\language.tx`,
    threw `FileNotFound` and faulted. 5 of 67 runs before, 0 of 30 since.
  - *Open.* `sub_001DEB90`, loading the particle database, calls through a
    null object in a list at `0x001DED16` and faults: 2 of 20 boots with the
    fix, 2 of 67 before. Only the main guest thread exists then, and the log
    up to the fault matches a good run line for line, so something on a
    host thread changes guest state. Restarting gets past it.

On macOS the game opens files without share modes, so test runs can go in
parallel, each against its own mirror of the game folder. The input scripts
of three level runs lost a press in the menus under that load; the `.ai`
line in the log is how to tell a run that never left them.

Before that, relifted on 9-10 Oct 2026 against toolkit `main` of 9 Oct (`ca49951`), with
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
  and streams on its own clock.
- **Graphic novel narration was clipped noise (issue #3, 10 Oct).** The game
  makes a DirectSound stream for each line of narration, in that line's own
  format (`sounds/story/part*`: mostly stereo 44.1 kHz Xbox ADPCM, some mono),
  and the heap puts them at the same few addresses. The toolkit kept the
  first stream's format for an address, so stereo lines were decoded as mono.
  Found by decoding the disc's WAVs with a copy of the toolkit's decoder: the
  captured noise was `P1L0a_001.wav` decoded as mono, block for block. Every
  sound on the disc is Xbox ADPCM (tag 0x69), so that comparison works for
  any of them. Confirmed by ear on New Game.

Found with a frame capture (`RECOMP_D3D8_CAPTURE`) and the replay tool:
`--list-draws`, then `--no-scene --draws N` to find the draw that paints a
region. The game never clears colour, so without `--no-scene` every partial
replay shows the previous frame and looks complete. Replay at the capture's
own `RECOMP_RES_SCALE`.

Not checked: a level played through to its end, the credits and attract
movies, loading a save, sound by ear.

**Pin:** toolkit branch `fix/mm-query-statistics` (`1d469e9`), which is
toolkit `main` at `9bf3ec2` (21 Oct 2026; the allocator lock merged as #70)
plus the free-memory fix above. Once that is merged, pin toolkit `main`
again. Keep the
pin on a commit of toolkit `main`: a feature branch's commits go away when it
is squash-merged.

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

Preview Levels holds 67 levels: Intro 2, Part Ia 14 (Subway, Hotel,
Tenements A), Ib 12 (Tenement Buildings B, Rooftops and Alleys, Nightclub,
Nightclub Showdown), II 14 (Nightmare to Manor), IIIa 14 (Drug Trip to
Garage), IIIb 11 (Asgard Building, Aesir Corp, Nicole Horne's Penthouse).
B backs out of a list with its cursor kept. To load one: Preview Levels,
then `lstick_down` once per part below Intro, `a`, `lstick_down` once per
level, `a`.

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
- **Only one copy at a time on Windows.** (macOS ignores share modes, so
  there each run needs only its own mirror.) The game opens
  `english/x_data.ras` with no sharing, so a second copy (a test run while someone plays) fails to open it,
  throws `ArchiveNotFound` and faults at start-up in `sub_000E9B80` writing
  0xFFFFFFF8. Check `tasklist` for `payne_recomp.exe` first.
- An exit with `ExitProcess(0)` from `shadow_wndproc` -> `xbox_HostExit` is
  the window being closed, not the game quitting.
- Start pressed in a level pauses, and the pause menu's first item is Quick
  Save, which writes `UDATA/54540003/<id>/SaveData.Mps`.
