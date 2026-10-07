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
pinned to toolkit `main` at `bcccf8c`. The toolkit history worth knowing:

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

Checked 7 Oct 2026 with scripted input (`RECOMP_INPUT_SEQ`):

- Reaches the first level (the alley, HUD showing the Beretta) about 20 s after
  launch. No unresolved indirect calls, no crash.
- 60 fps: every frame on one vblank, about 6 ms of work a frame in the level.
- Quick save writes `UDATA/54540003/<id>/SaveData.Mps` and costs one hitch
  (one frame of 4+ vblanks).
- One driven run ended in a clean `ExitProcess(0)` about 35 s in. It is not yet
  known whether a scripted press chose a quit option or the game left on its
  own. Check this first if the game closes during play.

Not checked: moving, shooting, bullet time, loading a save, later levels,
sound by ear.

## Running it unattended

The toolkit's switches apply (`xboxrecomp/CLAUDE.md`). Two specific to here:

- **Saves are not isolated by `RECOMP_SAVE_DIR`.** The game saves through
  `\Device\Harddisk0\Partition1\UDATA`, which the toolkit maps into the game
  folder. Move `game/UDATA/54540003` (the player's saves) aside before a
  scripted run and put it back afterwards.
- A script that reaches the level: `5000:start,10000:a,15000:start,20000:a`.
  The press at 15 s opens the pause menu, and A at 20 s quick-saves and returns
  to play. Start pressed in the level pauses, and the pause menu's first item is
  Quick Save.
