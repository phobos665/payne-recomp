# Mouse look: why it is not 1:1 (parked)

**Status, 2 Nov 2026: investigated, parked.** Everything below comes from
reading the lifted code (`src/recomp/gen`, lifted on toolkit `d7ee74e`).
None of it has been run or measured yet. The fix designed at the end is not
implemented.

**Symptom:** with the toolkit's `mouse_look` on, a mouse movement does not
turn the camera by a proportional amount. Slow movements barely turn the
camera, fast ones stop turning it any faster, and near enemies the mouse
seems to stick.

**Cause:** the mouse reaches the game as a right-stick deflection. That
makes it a turning *speed*, which the game then shapes the way it shapes a
thumbstick. The uncommitted dead-zone override in `src/recomp_manual.c`
(`sub_002954B0`, see CLAUDE.md "Mouse look") removes only one of the five
stages below.

## From mouse count to camera, stage by stage

### Toolkit (`xboxrecomp/src/input/input_bindings.c`)

1. **`recomp_mouse_move`** adds up the host's relative counts, in sixteenths of
   a count. The counts come from SDL relative mode on macOS and from
   `WM_INPUT` on Windows.
2. **`look_axes`** turns them into a speed in counts per second. It does this
   at most every 12 ms, and **averages each speed with the previous one**.
   While the response is linear that changes nothing in total. Because stage 5
   below is cubic, it loses part of the turn whenever the mouse's speed
   varies.
3. **`look_shape`** maps the speed to a stick deflection. Full deflection is
   `100000 / mouse_sensitivity` counts a second, which is 1250 at the default
   of 80. **Anything faster is clamped** to full deflection.
4. **`add_look`** adds the deflection to controller 1's right stick and clamps
   it to ±32767.

### Game

5. **Stick shaping.**
   - `sub_000F9050` reads the stick as `short / 32767`, with no shaping of its
     own.
   - `sub_002954B0` (`recomp_0060.c`) applies the invert sign (controller
     +0x1D) and the per-axis dead zones (+0x85 X, +0x89 Y), then the curve:

     ```
     rate = (1.5·s + 0.5) · (0.8·x)³
     ```

     `s` is the axis's sensitivity. The constants are at 0x3B9048, 0x3B8D80
     and 0x3D459C. The result is stored at controller +0x26 (X) and +0x2A (Y).
     At half deflection the turn is one eighth of full.
6. **Routing to the look actions.**
   - `sub_00293C60` picks the active controller and `sub_00294090` the axis.
   - The binding preset `sub_000257C0` puts axis 1 on action 0x4E71A4 (yaw)
     and axis 2 on 0x4E7177 (pitch).
   - Two getters on the player's input object (player+0x298, vtable 0x3BA5A4)
     read them: `sub_000273B0` for X (vtable +0x4C) and `sub_00027310` for Y
     (+0x48).
   - While action 0x4E71D1 (preset code 7, not identified) is held, the value
     is doubled. When the horizontal sensitivity is 0.5 or more it is then
     divided by the second sensitivity (+0x3E/+0x42) through `sub_00295040`.
7. **This frame's turn.** `sub_0002E6D0` is the input object's update
   (vtable +0x80, thiscall, one argument, `recomp_0004.c`). It writes:

   ```
   +0x25 = LookX · π · dt      (radians)
   +0x29 = LookY · π · dt
   ```

   `dt` is the float at +0x10 of the time object it is passed. The getters
   `sub_0002DE60` (+0x5C) and `sub_0002DE70` (+0x60) hand the two values on.
8. **Auto-aim** is in the same function. The game ignores the look input and
   computes +0x25/+0x29 itself (`sub_00024C30`, `sub_00024DF0`) when all of
   these hold:
   - there is a target (+0x39);
   - auto-aim is on (0x4D0CCA, line 3 of `GameSettings.txt`);
   - `dt` is not zero;
   - both shaped look values are below the threshold from `sub_00024C10`;
   - either the flag at 0x4E6F5C is set or the player is moving.

   The threshold is 0.6, or 0.1 when the flag at 0x4E6F5C is set. That flag is
   set by player code and not yet identified.

   At sensitivity 0.9167 the shaped value is `0.96·x³`. So **any mouse input
   below about 85% of full deflection is discarded while a target is near.**

### What this adds up to at the default settings

The fastest turn is `0.96·π` rad/s, about 173°/s. It is reached at 1250
counts a second and stays there however fast the mouse moves. Below that, the
angle turned per count grows with the square of the speed (about `0.138·x²`
degrees per count).

**Not found:** where the game finally stores the camera's yaw and pitch. No
consumer of the vtable +0x5C/+0x60 getters turned up statically. The fix below
does not need it, because the game's own consumer and pitch limit still apply
after it. One way to find it: in a scratch build, wrap `sub_0002DE60` to log
its return address (`MEM32(g_esp)`) once, then `RECOMP_WATCH_WRITE` the field
that caller writes.

## Proposed fix (not implemented)

Stop sending the mouse through the stick, and add its movement to the
frame's turn as an angle.

**Toolkit, generic** (and documented in `docs/technical/input-binding.md`,
"The mouse"):
- `int recomp_mouse_take(float *dx, float *dy)` returns the counts since the
  last call and zeroes them.
- `void recomp_mouse_look_direct(int on)` lets a title claim the mouse. While
  it is claimed, `look_axes`/`add_look` do not run, so the counts are neither
  turned into stick nor drained twice.
- A degrees-per-count scale is derived from `mouse_sensitivity` (for example
  80 → about 0.055°/count, to be tuned by playing).
- Counts older than about 100 ms are dropped, so a menu or pause does not end
  in a jump.

**Max Payne, in `src/recomp_manual.c`:**
1. Wrap `sub_0002E6D0`: call `sub_0002E6D0_gen`, then take the mouse counts.
2. If the mouse moved, set:

   ```
   +0x25 = LookX·π·dt + dx·k
   +0x29 = LookY·π·dt + dy·k·invert
   ```

   - LookX and LookY come from the vtable +0x4C/+0x48 getters.
   - `invert` is the controller's +0x1D sign.
   - This keeps the pad's own contribution, and overrides auto-aim only on
     frames where the mouse moved, which is the game's own rule: assist only
     while the player is not looking.
3. If the mouse did not move, leave the game's values as they are.
4. The mouse term is a distance, not scaled by `dt`, so frame rate does not
   change how far a movement turns. Bullet time should leave mouse aim at full
   speed, as PC Max Payne is thought to (check against the PC game).
5. Apply this only to controller 1's input object (vtable 0x3BA5A4).
6. Revert the dead-zone override: with the mouse off the stick it fixes
   nothing, and the pad should keep its console feel.

## To check when this is picked up

- The direction of each axis, and whether invert should apply to the mouse.
- Whether bullet time scales `dt` (time object +0x10).
- That one full turn takes the same mouse distance at 30 fps, 60 fps and
  uncapped.
- The auto-aim override in a firefight, with `GameSettings.txt` line 3 at 1.
- What action 0x4E71D1 (code 7) is, since it doubles the look value.
- The final consumer of +0x25/+0x29 (the logging run above).

## Risks

- **A mid-entry that skips the wrapper.** `sub_0002E6D0` has a separately
  lifted mid-entry, `sub_0002E812`. Anything that jumps there skips a wrapper
  on `sub_0002E6D0`.
- **Other camera modes.** Shootdodge and cutscenes may read the look actions
  directly rather than these per-frame values.
- **The 0x4E6F5C flag.** It changes the auto-aim threshold and is not
  identified.
