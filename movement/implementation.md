> Synced from [R5Rust/docs/movement.md](https://github.com/R5Rust/R5Rust/blob/main/docs/movement.md), the working copy. Paths are relative to the R5Rust repo.

# Movement: Apex Legends (S21 / R5) ported

`crates/shared/src/sim/` is a port of the player movement in `r5apex_ds.exe` (Season 21, the R5Flowstate
dedicated server). The source is the set of bit-exact specs in `C:\Code\apex\R5Regenis\docs\engine\*.md`,
which were read off the disassembly. A few functions those specs leave as calls were disassembled for this port
(GetAccelerate, GetMaxSpeed, UpdateSprint/CanSprint/AllowSprint, LandingSpeedScale), from
`%USERPROFILE%\Games\R5Flowstate\r5apex_ds.exe`. The two builds are the same: the offsets and constants match.

The code follows the specs' order of operations and their MIN/MAX operand order. It is not bit-exact: NaN paths
and networking notifies were dropped. The surface normal (pl+0x662c) is world up except while climbing a wall, and
UPRIGHT is the engine's double-precision test (`sim::upright`).

## Run the checker

```powershell
cargo run --release -p shared --example movecheck
```

It runs 130 checks on a synthetic test world: a floor, a 22u step, a 23u step, a 60u ledge, a 100u cliff, a 300u
wall and a 150u wall.
If the assets are present it also runs them on the real Firing Range, and it re-reads the legend's settings file
to confirm the consts below still match it. It exits with code 1 if any check fails.

Last run: 130/130 passed. A tick on the real 5.4M-triangle map costs **~50 µs** on average and ~0.3 ms at worst
(sprint, jump, slide and mantle mixed, at 60 Hz).

| Check | Got |
|---|---|
| walk / sprint / crouch-walk | 173.5 / 260.0 / 80.0 u/s |
| sprint with speed scale 1.15 (Apex's holstered numbers) | 299.0 |
| 0 to 200 u/s / 0 to 259 u/s while sprinting | 0.38 s / 0.97 s |
| sprint to a full stop | 0.22 s |
| jump takeoff vz / apex height | 271.1 (`sqrt(2*750*56)` minus three half-gravities) / 53.6 u |
| jump again right after landing (anti multi-jump) / 1 s later | 15.5 u / 53.6 u |
| bunny hop within 1 s of landing at 500 u/s | 400 (loses `skip_speed_reduce` 100, never below 260) |
| slide start from a sprint | 396 (260 + 150, capped at 400, minus the first tick of friction) |
| slide speed after 0.5 s / 1 s; the slide ends at | 283 / 189 u/s; 1.38 s |
| slide again 1.5 s later / after 2 s | no boost / boosted |
| slide jump within 0.4 s | speed minus the 140 boost; vz 257 |
| slide jump at 300..350 u/s | snaps to 350 |
| 22u step / 23u step | walks up / blocked |
| mantle onto the 60u ledge | 0.53 s; ends standing on top |
| tap-strafe 0.1 s after a jump | turns the velocity 45° (change capped at 0.7 × 260 per press) |
| lurch D at 0.35 s / 0.45 s after the jump | bends 27.9° / 7.5°, speed kept (strength 0.7 fading to 0 at 0.5 s) |
| lurch S at 0.1 s | speed drops by the cap (182), direction kept |
| null input (A held, D pressed) / wheel spam 30/s while turning | no change (still a tap) / 14 lurches, no decay at default cvars |

### Mantle exits

| Exit | Got |
|---|---|
| Finish while holding W | velocity = input × walk speed: 173.5 along +X |
| Finish while holding W, sprint latched | 260 |
| Mantle jump past the corner (boost off) | 290 along the input plus a 283.6 jump (see the note on the +30) |
| Superglide (boost off): jump-off, then crouch on the next tick | 400 horizontal (290 + slide boost, capped). Crouching one tick later gives no superglide |
| Jump while on the vertical leg with W held | ignored (spec B6.1) |
| Hop-back: jump below the corner without W | 188 away from the wall and 240 up, 0.1 s after the press |
| Cancel: let go of W, or pull back with S | the traversal ends and you drop. Past the rest fraction the mantle finishes on its own |
| Mantle boost (default): jump in the sweet spot | 338 horizontal (260 × 1.3) along the input, vz 367 (`sqrt(2*750*90)`), slide granted. Tap-strafes are blocked until you land |
| Jump outside the sweet spot (state 3) | 260 and a 56u jump (vz 289.8), no slide |

## What is ported, by spec section

| File | Functions | Spec |
|---|---|---|
| `sim/mod.rs` | PlayerMove 0x140ba8fc0 (the per-tick order), SetupMove's buttons and input vector, UpdateSprint 0x140bedf60, CanSprint 0x140bed930, AllowSprint 0x140bed5a0, GetMaxSpeed 0x140b890b0 | playermove.md §3, disassembly |
| `sim/walk.rs` | FullWalkMove 0x140b9ead0, ApplyHalfGravity 0x140b8c2a0, CheckJumpButton 0x140b8d490, WalkMove 0x140b9cdb0 (ground path), Friction 0x140b9c7c0, Accelerate 0x140b9c610, GetAccelerate 0x140b8c5d0, AirMove 0x140b9b740, AirAccelerate 0x140b9b510, CheckVelocity 0x140ba3b60; the lurch is the **S21 client's** AirMove_TapStrafe 0x140a48a30 (r5apex_dx12.exe, see "Lurch") | fullwalkmove.md, walkmove.md, airmove.md, R5Regenis `native/r5r_engine/src/movement.cpp`, decompiled for this port |
| `sim/ground.rs` | CategorizePosition 0x140ba4f30 (ledge re-probe included), SetGroundEntity 0x140ba46d0 (landing energy, landing slowdown, slide-on-land), StayOnGround / GroundSnapTrace, StepSmooth / DecayStepSmooth, CheckStuck, LandingSpeedScale 0x140bf5200 | playermove.md §4-8 |
| `sim/duck.rs` | Duck 0x140ba7500 (4 states, integer-ms timer, smoothstep eye height), CanUnduck, FinishUnDuck, SetDuckViewOffset, StartSliding 0x140ba7ed0, StopSliding | playermove.md §7, slide.md |
| `sim/step.rs` | StepSlide 0x140b9ad70 (all 4 variants), TryPlayerMoveStep 0x140b9a970, StepMove 0x140bb0610 | stepslide.md |
| `sim/traversal.rs` | TraversalMove 0x140b981b0 (progress, rest, jump-off, hop-back, END and completion), TraversalPathPos, Start/Finish/EndTraversal, CheckAutoMantle (rebuilt), the mantle boost | movemodes.md part B |
| `sim/wall.rs` + `walk.rs` (`wall_walk`) | Wall climb: SurfaceTransition 0x140ba1990 and its classifier 0x140ba0a80, WallProbe 0x140ba1690, MantleIntent 0x140b92e80, FacingWall 0x140bf0a40, CanWallClimb 0x140bf0a90, WallTop 0x140b9f680, UpWallBoostScale 0x140ba2aa0, WallSpeedFrac 0x140b8b6b0, WallRunUpdate 0x140b9f6e0, EndWallRun 0x140bf0ba0, WallSurfaceChange 0x140ba2680, WallJump 0x140ba2c90, RememberWall 0x140bacb10; WalkMove's wall branch 7W | walkmove.md 7W, decompiled for this port |
| `collision.rs` | `trace_hull`: an axis-aligned box swept against the triangle BVH | (Source TraceHull semantics) |

### How it feels, briefly

- **Sprint** is a toggle-latch. One press of Shift requests a sprint for 3 s, and holding Shift keeps extending
  that by 0.1 s. Once you sprint, it stays on while you keep moving forward (within ~45°) and are not crouched.
  Stopping, walking backwards or crouching ends it. Holding Shift without forward input does nothing.
- **Acceleration is linear**, in u/s²: lowAcceleration 2500 below 120 u/s, then acceleration 450 up to walk speed,
  then sprintAcceleration 100 up to 260. There is no Source-style `accel × wishspeed`.
- **Friction** is a linear deceleration of 1250 u/s². It applies only to velocity beyond your wish: velocity along
  the input direction, up to the wish speed, is never slowed.
- **Gravity** is applied in halves at the start and end of each tick, plus a third half on a jump tick.
- **Jumping** has no buffer and no auto-hop, as in the spec: a press counts only on the tick it happens. After
  leaving the ground, the coyote grace is 0.2 s.
- **Crouch** takes 400 ms (200 ms when sliding) before the hull shrinks and crouch speed applies. Only the eye
  height eases (smoothstep). Uncrouching needs the standing hull to fit.

## Wall climb

Stock legends have `climbEnabled` and no `wallrun`, so every wall they attach to is climbed. The R5Regenis specs
leave this whole chain as calls, so it was decompiled for this port from r5apex_ds.exe (Ghidra headless, with the
asm checked where the C was unclear): SurfaceTransition and its classifier, WallRunUpdate, WallJump, EndWallRun and
their helpers live in `sim/wall.rs`. The motion on the wall is WalkMove's wall branch (walkmove.md 7W) in
`walk.rs` (`wall_walk`).

While climbing, the wall is the ground entity and `Player::surface_normal` (pl+0x662c) is the wall's normal, so
FullWalkMove runs WalkMove, CategorizePosition probes along -n, and every UPRIGHT test fails. `Player::on_wall()`
reports it; the HUD state is `climb`.

### How it plays

1. **Attach.** You must be in the air: not on the ground and not near it within 0.5 s (a jump resets that clock).
   When AirMove is blocked by a wall, SurfaceTransition's classifier needs: a wall (normal z between -0.5 and 0.7),
   the view yaw facing it within ~45.6° (`cos² > 0.49`), velocity into it, rising no faster than
   climbSpeedStart/2 = 100 u/s, feet no higher than ground + 147 (verticalGainCutoff 175 - climbFinalJumpUpHeight 28),
   and two box probes (feet and mid-body) finding the wall within 17u. If you are pushing and looking into it like
   a mantle, a head probe must find the wall too; if it doesn't, the ledge is in reach and the mantle takes it.
   A probe that misses the wall asks for a retry 0.07 s later (AirMove's deferred transition).
2. **Climb.** Holding W pushes up the wall (the surface basis turns view-forward into "up the wall"). Gravity is 0
   while you push into the wall. The target up-speed falls linearly from climbSpeedStart 200 to climbSpeedEnd 50 over
   climbHeight 100 above WallTop (the attach height, capped at ground + 47), reached at climbAccel 500 u/s².
   Sideways input moves along the wall at climbHorzSpeed 100.
3. **End.** WallRunUpdate ends it when: you reach WallTop + 100 (then a **final jump up**: `vz += sqrt(2g * (28 +
   min(top - z, 10)))`), you look away (facing test fails), you rise slower than 10 u/s after 1 s, you crouch, you
   pull away for 0.05 s, or the wall is no longer within 17u of a probe 16u above the feet. Falling off pushes you
   70 u/s off the wall (wallrun_fallAwaySpeed) and lowers the reference height by 128
   (wallclimb_vertical_gain_reduction), so you cannot climb again until you stand on the floor. The same wall cannot
   be re-attached at or above the last attach height.
4. **Jump off (WallJump).** Out from the wall at `188 * (1 - 0.8 * (n·view) * (n·input))` (both terms only count
   when they face the wall), plus `sqrt(2g * 28)` = 205 up scaled by UpWallBoostScale. A climb jump lowers the
   reference height by 128 first, which puts you above the 175 cutoff, so in practice the up boost is 0 and you keep
   the climb's vertical speed. The same tick, WallRunUpdate sees the lowered WallTop and ends the climb, adding the
   70 fall-away push.
5. **Let go of W.** The no-climb table applies (wallrun speeds, slip ramp, no-input slip 0.7), gravity ramps back
   in over 1 s, and you drop off when the 10 u/s rule fires.

### Checker results (300u and 150u walls)

| Check | Got |
|---|---|
| run + jump into the wall: attaches at | z 48.9, vz 89.6 (the tick vz fell under 100) |
| climb rise above WallTop / duration | 100.2 u / 0.97 s |
| up-speed at the top | 51.0 (climbSpeedEnd 50) |
| final jump up at the top | vz 249.0 = 51.0 + sqrt(1500 * 27.8) - one half-gravity |
| pushed off at the top | -61.7 = -70 plus one tick of air control back toward the wall |
| peak after the climb / then | feet z 192.8, slides down the wall to the floor, no re-climb |
| jump on the wall, W held, facing it | out 99.3 (= 188 * 0.2 + 70 - one tick of air control), vz unchanged |
| jump on the wall, no input | out 258 (188 + 70), vz unchanged |
| let go of W | off the wall 0.67 s later |
| approach at 44° / 50° to the wall | climbs / doesn't |
| 150u wall | climbs 11 ticks, then the mantle takes the ledge, ends standing on top |
| 60u ledge from a running jump | mantles, never attaches |

### Approximated, or a judgement call

- **Animation-driven parts are skipped.** WallClimbUpdate 0x140bf0e90 only plays the `wall_climb` animation and
  its own distance trace turns that animation off; the view punches on attach / time-out and the sounds are not
  ported either.
- **LedgeAssist 0x140baca70** (the jump-assist target search) is not ported, so WallRunUpdate's fall-off never
  finds an assist target and always applies the 70 u/s push, and WallJump never gets its assist.
- **WallJump + WallRunUpdate on the same tick.** The literal order (CheckJumpButton, then WallRunUpdate, with the
  surface normal still the wall's) makes a climb jump also trigger the climb-height fall-off, so a climb jump leaves
  at 188·k + 70 and keeps the climb's vertical speed. That is what the decompile says; it has not been compared
  with the live S21 client.
- **TryPlayerMoveCore's trace record.** The rebuilt Source TryPlayerMove now hands back its first bump's trace
  (Source's `pFirstTrace`) instead of its last one. AirMove's contact plane and SurfaceTransition need the wall
  hit; with the last trace they got an empty normal after a slide. All older checks are unchanged by it.
- **Wall hang** needs wallrunAdsType 1 (stock is 2), so WallHangSlipScale is 1 and the hang branch is not ported.
  The wallrun-legend branches (`wallrun` true: the airborne wallrun start in WallRunUpdate, the up-wall boost at
  attach) are written out where they were short, but no stock legend reaches them.
- **Unknown fields assumed default:** pl+0x6674 (wallrun B/C multiplier) = 1.0 (its constructor value), pl+0x5af0 =
  1, pl+0x6258 time ramp off, the ADS move-speed scale = 1, `*(pl+0x6578) & 3` = 0 (selects the plain wall probe in
  WallRunUpdate), slide state pl+0x5aac never 2/3, `mv+0xa8` up axis = world up, `mv+0x3c` = the view forward.
- **Crouched in the air**, the classifier moves its probes by CanUnduck's hull delta (the same pl+0x65fc..0x6628
  fields); which of the two hulls each field holds is inferred from CanUnduck.
- **Inline sines.** The engine's SSE sine polynomial (cos as sin(a + pi/2)) is replaced by `f32::sin`; the
  thresholds (cos 45°, 50°, 90°) can differ in the last bit.
- **Build doubt.** The ds binary is older than the S21 client; R5Regenis uses it as the movement reference, and so
  does this port. The settings values come from the S21 legend file and match the names the ds binary reads.


## Settings and cvars used

The player settings are from `pilot_survival_firesupport.json` (Bangalore, 72u tall). 20 of the 28 legend files
have the same movement values. The checker asserts the consts match the file.

| Setting | Value | Setting | Value |
|---|---|---|---|
| stand hull / view | 32×32×72 / 60 | crouch hull / view | 32×32×47 / 32 |
| speed / sprintspeed / lowSpeed | 173.5 / 260 / 120 | crouch speed | 80 |
| acceleration / lowAcceleration / sprintAcceleration | 450 / 2500 / 100 | crouch acceleration | 2500 |
| deceleration / sprintDeceleration | 1250 / 1250 | crouch decel | (-1 → 0.6 × 2500) |
| stepHeight | 22 | speedScaleSide / Back | 1 / 1 |
| airSpeed / airAcceleration / airFriction / airDrag | 60 / 500 / 0 / 0 | gravityScale | 1 |
| jumpHeight / slideJumpHeight | 56 / 50 | antiMultiJump frac / tmin / tmax | 0.3 / 0.15 / 0.75 |
| landSlowdown frac / dur / power / hmin / hmax / noSprint | 0 / 1 / 2 / 300 / 800 / 0.7 | canJumpWhileCrouched | false |
| slideRequiredStartSpeed (Air) | 200 (90) | slideSpeedBoost / Cap / Cooldown | 150 / 400 / 2 |
| slideDecel / WantToStopDecel / VelocityDecay / Accel | 100 / 400 / 0.7 / 50 | slideStop / MaxStop / MaxJump speed | 125 / 350 / 350 |
| mantleHeight | 80 | mantleDuration below / level / above / high | 0.7 / 0.6 / 0.5 / 0.35 |
| wallrunJumpOutwardSpeed / UpSpeed (hop-back) | 188 / 240 | wallrunJumpInputDirSpeed | 75 |
| climbEnabled / wallrun / wallrunDuckCausesFallOff | true / false / true | climbHeight / climbFinalJumpUpHeight | 100 / 28 |
| climbSpeedStart / End / Accel / Decel | 200 / 50 / 500 / 300 | climbHorzSpeed / Accel / Decel | 100 / 100 / 500 |
| verticalGainCutoff_wallrun | 175 | wallrunAllowedWallDistance / CeilingLimit | 17 / 0.5 |
| wallrunMaxSpeedVertical / Horizontal / Backward | 112 / 225 / 50 | wallrunAccel/DecelerateVertical | 450 / 300 |
| wallrunAccel/DecelerateHorizontal | 200 / 500 | wallrunGravityRampUpTime / wallrun_timeLimit | 1 / 2 |
| mantleAngleScale | 1 | wallrunAdsType (1 = hang on ADS) | 2 |

Every cvar default below was read from the exe's static ConVar objects (the default string sits at ConVar+0x50):

| Cvar | Default | Cvar | Default |
|---|---|---|---|
| sv_gravity | 750 | sv_maxvelocity | 34000 |
| AirAccelerate's air-control cvar (0x1423705d8; name not resolved) | 2.0 | sv_optimizedmovement | 1 |
| jump_graceperiod | 0.2 | skip_time / _jump_height_speed / _fraction | 1.0 / 450 / 1.0 |
| skip_speed_retain / skip_speed_reduce | -1 / 100 | slide_max_angle_dot | 0.6 |
| slide_whileInAir / slide_auto_stand | 0 / 0 | slide_step_velocity_reduction | 10 |
| traversal_enable / traversal_cooldown | 1 / 0.5 | automantle_enable / _cooldown | 1 / .25 |
| automantle_searchdist / _mindist | 5 / 18 | automantle_height_level / above / below | 10 / 30 / -10 |
| automantle_jumpoff_duration / _anim_maxfrac | 0.1 / 0.5 | automantle_rest_frac / _below / backoff_anim_maxfrac | 0.4 / 0.3 / 0.7 |
| wallrun_enable / wallrun_retry_interval | 1 / 0.07 | wallrun_sameWallDot / Dist / Slope | 0.9 / 100 / 0 |
| wallrun_upwardAutoPush | 0.65 | wallrun_angleChangeMinCos / rotateMaxRate | 0.8 / 3 |
| wallrun_pushAwayFallOffTime / fallAwaySpeed | 0.05 / 70 | wallclimb_vertical_gain_reduction | 128 |
| wallrun_slipstarttime / slipduration / noInputSlipFrac | 1.5 / 1.0 / 0.7 | wallrun_minAngle_air | 180 |
| wallrun_strengthLossStart / End | 1000 / 1000 | wallrun_repelEnable | 1 |
| automantle_maxangle_push / _view / automantle_wallrun_maxangle_view | 50 / 50 / 45 | | |

The wall-climb cvar names were resolved from their registration code (`lea rax, "name"; mov [obj+0x18], rax`): the
ConVar objects only receive their name pointer at runtime, so obj+0x18 is empty in the file.

The lurch cvars (`jump_grace_*`) are the S21 client's; their table is in "Lurch" below.

The mantle boost and its tap-strafe gate come from the R5Flowstate SDK (`s21-r5sdk/src/game/{client,server}/
mantle_boost.cpp`), not from either exe:

| Cvar | Default |
|---|---|
| mantle_boost_enabled | 1 |
| mantle_boost_disables_tap_strafes | 1 |
| mantle_boost_min_valid_traversal_frac | 0.5 |
| bridge_mantle_boost_sprint_mult | 1.3 |
| bridge_mantle_boost_jump_height | 90 |
| mantle_boost_input_setting | 1 (jump) |

All of these live in `crates/shared/src/settings.rs` (`STOCK`, `CVARS`). Each `Player` carries a copy
(`settings`, `cvars`, `speed_scale`), so the server or a test can change them per player.

## Lurch (jump grace / tap-strafe): the S21 client's

The lurch is the one piece of movement taken from the S21 client `r5apex_dx12.exe` instead of the 2019-era
`r5apex_ds.exe`, because that is what an R5Flowstate player feels (the client predicts it every frame). It was
decompiled for this port (Ghidra headless on `reverse-engineering-projects\ghidra\R5Apex.gpr`).

**Functions ported**

| Address (dx12) | What | Where |
|---|---|---|
| 0x140a48a30 | AirMove_TapStrafe: the lurch itself | `walk.rs` `jump_grace` |
| 0x140a48810 | velocity-based multi-tap decay (5 easings; `fminf` 0x1412a82b8) | `walk.rs` `grace_velocity_decay` |
| 0x140a48f90 | the S21 AirMove: calls the lurch right after AirAccelerate 0x140a485e0, as the 2019 one did | (order unchanged) |
| 0x140b09c00 | cap base = pl+0x3278 speed scale × `sprintspeed` of stance 0 (settings field `sprintspeed`, lookup 0x141af7d98) | `jump_grace` |
| 0x140a3b550 (tail) | the jump: pl+0x3670 = now, strength multiplier 1, tap count 0 | `check_jump_button` |
| 0x140b12970 | post-think: on the ground pl+0x366c = now, multiplier 1, tap count 0 | `player_move` post-think |

How each cvar was found: their registration functions (listed in the table) write the ConVar object; the only
other reference to each object is ConVar+0x40 (the parent pointer) read by 0x140a48a30 or 0x140a48810, which then
reads the float at +0x60 / the int at +0x64. The defaults are the strings each registration function points at.

**The algorithm** (per air tick, after AirAccelerate):

1. R5Flowstate's client SDK hook skips the whole function while a full mantle boost is in the air (state 4) until
   landing. That gate is not in the S21 binary; it is in the reference game, so it stays.
2. With `jump_grace_require_tap` a move key must be newly pressed this tick (mv+0x2c & 0x618). Without it the
   analog stick (mv+0x30/0x34) drives it every tick.
3. Within `jump_graceperiodmax` of the jump (`jump_grace_jump 1`; 0 = of leaving the ground), more than
   `jump_grace_min_tap_period` after the last attempt, and under `jump_grace_max_tap_count` lurches.
4. Horizontal speed at or under `jump_grace_cutoff_speed` (1100), else nothing (not even the attempt time).
5. The push = held keys (W−S, D−A) on the flat camera basis, normalised. Opposite keys cancel (a "null input":
   the push is 0). With `jump_grace_limit_to_90` its component against the velocity is removed first.
6. The multiplier shrinks by the velocity decay; `r = ramp × multiplier × jump_grace_strength`, where the ramp is
   1 until `jump_graceperiodmin` and falls linearly to 0 at `jump_graceperiodmax`.
7. `n = lerp(velocity direction, push, r)`. If `|n| <= 0.01` nothing happens. Otherwise it counts a tap
   (from `jump_grace_decay_start_tap_count` on, the multiplier × `jump_grace_multi_tap_decay`), and the velocity
   moves to `|v| × n / |n|`, with the change capped at `jump_grace_max × sprint speed` (182 at scale 1, 209 at
   1.15). Vertical velocity is untouched.

**What changed from the 2019 keyboard grace (the previous port, `jump_keyboardgrace*`).** At the default cvars the
numbers are the same for every ordinary lurch: same window (0.5 / 0.2), strength 0.7, cap 0.7 × sprint speed, and
the 1100 cutoff the R5Regenis bridge had already added. The differences:

- the cutoff is native (and the attempt time is only recorded under it);
- tap count, min tap period, multi-tap decay and velocity decay exist (all off at their defaults);
- when the blended direction is shorter than 0.01 the S21 does nothing; the 2019 code still applied the
  unscaled blend, which pulled the velocity toward zero (a near-exact back press at r = 0.5);
- `jump_grace_limit_to_90` (default 0) can drop the backward part of the push;
- `require_tap 0` would lurch every tick from the analog stick (controllers);
- the window can run from leaving the ground (`jump_grace_jump 0`).

Nothing else in the S21 AirMove changes a number for a stock pilot: AirAccelerate 0x140a485e0 is the same
(including the air-control cvar 2.0 in its turning branch), the air drag uses airDrag/airFriction (0 for pilots;
jetpackPostEffectDrag during a jetpack), and the extra blocks (pl+0x4960 launch fall limiter, grapple, jetpack
pitch, slip restriction) are not stock-pilot paths.

**Cvars** (`settings.rs`, defaults from r5apex_dx12.exe):

| Cvar | Default | Registration |
|---|---|---|
| jump_graceperiodmax | 0.5 | 0x14019fd60 |
| jump_graceperiodmin | 0.2 | 0x14019fb90 |
| jump_grace_strength | 0.7 | 0x14019ff30 |
| jump_grace_max | 0.7 | 0x1401a0100 |
| jump_grace_cutoff_speed | 1100 | 0x1401a0f80 |
| jump_grace_require_tap | 1 | 0x1401a02d0 |
| jump_grace_jump | 1 | 0x1401a0db0 |
| jump_grace_min_tap_period | 0.0 | 0x1401a04a0 |
| jump_grace_max_tap_count | 1000 | 0x1401a0a10 |
| jump_grace_decay_start_tap_count | 1000 | 0x1401a0670 |
| jump_grace_multi_tap_decay | 1.0 | 0x1401a0840 |
| jump_grace_multi_tap_decay_velocity_based_max_decay | 0.0 | 0x1401a1150 |
| jump_grace_multi_tap_decay_velocity_based_max_speed | 900 | 0x1401a1320 |
| jump_grace_multi_tap_decay_velocity_based_easing_type | 1 (0 cubic, 1 sine, 2 circ, 3 linear, 4 step) | 0x1401a14f0 |
| jump_grace_limit_to_90 | 0 | 0x1401a0be0 |
| jump_grace_controller_disable_duration | -1.0 | 0x1401a16c0 |

Per-player state: `Player::grace` (`JumpGrace`: last attempt time pl+0x3674, multiplier pl+0x3678, tap count
pl+0x367c). `Input::null_side` / `null_fwd` carry "both keys of an axis held", so a null input sets both buttons
like Apex does (the client sets them from the binds).

**Approximated or left out:** the controller gate (pl+0x46e0, last controller input) is dropped since there is
no controller path and the default -1 always passes; `limit_to_90`'s rsqrtss + Newton step is a plain 1/sqrt; the
sine easing uses `f32::sin` instead of the binary's polynomial; the jump assist after the lurch (0x140a58ec0 ->
0x140a58560) is not ported, as before. These are live client cvars: R5Flowstate could run other values than the
static defaults. `r5f-probe/plan.py` now dumps all of them at the start of a movement run.

**Against R5Flowstate (`probecmp`, horizontal speed at t 1.75, ours / real):**

| Run | Before (2019 grace) | After (S21) |
|---|---|---|
| lurch_d_150 / lurch_a_150 | 256 / 264, 256 / 263 | 256 / 264, 256 / 263 |
| lurch_s_150 | 48 / 299 | 299 / 299 |
| lurch_sd_150 | 174 / 263 | 256 / 263 |
| forward_ras_ref_0 | 86..192 / 316 | no valid real data |

The S, SD and RAS "real" numbers were probe bugs, not game behaviour:

- **The S21 client has no `+back` command**: it is `+backward` (the exe holds `+backward`/`-backward` and no
  `+back`). Every `+back` in the plans pressed nothing, so lurch_s_150 is really "release W in the air" and
  lurch_sd_150 is really a D lurch. `rp_main.gnut` now sends `+backward` and prints `[RPROBE] backward-fix`;
  `probecmp` replays `+back` as a no-op for logs without that marker, which is where the "after" numbers come
  from. A real S press does cut speed by the cap, in the game and here.
- **forward_ras_ref_0 never ran**: the lurch-trainer plan's events are bare `JUMP` / `TAP` with the notch count
  in a separate field, and the probe script died at the first one ("Index 2 is beyond array size") at 1.5 s. The
  "316 held" was `probecmp` repeating the last sample. `plan.py` now keeps the count (`JUMP:8`), the probe takes a
  bare `JUMP` as one press, and `probecmp` flags bare events. (The lurch trainer's own `r5test` plans use `+back`
  too.)

Still open: the real D/A lurch at 150 ms changes the velocity by ~236 u/s (|Δv| from the logged samples), against
the formula's cap of 209 (0.7 × 260 × 1.15), so ours ends at 256 instead of 264. The direction of the change fits
r ≈ 0.68-0.7. Either the live `jump_grace_max` is higher than the static 0.7 (≈0.79 would fit) or the speed
scale the cap uses is not the holstered 1.15. Re-run the probe (it now dumps the live `jump_grace_*` values and has
the fixed `+backward`, `lurch_null_150` and `tap_spam_d` runs): `python r5f-probe\plan.py` (this rewrites
`rp_plan.gnut` for movement runs), then `r5f-probe\run.ps1`.

## Collision

Apex moves an axis-aligned box. `Collision::trace_hull(start, end, mins, maxs)` sweeps that box against every
solid triangle in the BVH. The triangles are the game's own collision (`collision.bin`, the BSP's BVH4 plus
static prop model trees, filtered by `TRACE_MASK_PLAYERSOLID`). See `docs/collision.md`. It uses an exact separating-axis test on 13 axes: the 3 box axes, the triangle normal
and 9 edge cross products.

- It returns `fraction`, `endpos`, the hit `normal`, `startsolid` and `allsolid`.
- Like Source's `DIST_EPSILON` (1/32), the box stops 1/32 short of the surface along the hit normal. A box resting
  on a floor sits 1/32 above it, and sliding along that floor does not count as a hit.
- When triangles tie on fraction, the trace keeps the one the move goes into hardest (most negative `d·n`), then
  the one whose normal points most upward (floors built from many triangles, ledge edges). Reporting the most
  upward one first was wrong in a crease: a steep slope against a wall a fraction of a degree off-axis hit together
  at fraction 0, the trace always named the slope, and TryPlayerMove never learned about the wall.
- **Shell slack.** A box that starts inside the 1/32 shell of a triangle (where every hit and slide leaves it, give
  or take float rounding) may move closer, down to 1/64 (`SHELL_SLACK`), without the move counting as a hit. The gap
  stays positive, so it still never touches the triangle. Without it, a move clipped exactly parallel to a plane still
  approached it by up to ~0.002 units, because `end = start + v*dt` is rounded to f32 at |x| ~ 10000 (ulp 1/1024),
  and every such move was blocked at fraction 0. That was the "floating in mid-air against walls / on the grate ramp"
  bug: velocity collapsed to (0, 0, -6.25) every tick next to steep (unwalkable) slopes.
- **Triangles, not brushes.** The map is a triangle soup, so a box already overlapping a triangle at the start
  does not block (Quake's "startsolid, but you can get out" rule). The one exception is a move that goes deeper
  into that triangle's plane: it is blocked at fraction 0 and flagged `allsolid`, so an embedded box can walk out
  but cannot fall through a floor. A move within about 3 degrees of the plane (`EMBED_GRAZE`, |d·n| < 0.05 |d|)
  only grazes it and is not blocked, so a box embedded a hair into a near-vertical wall (that leans back by 0.2
  degrees, say) can still fall down along it.
- The capsule functions (`move_capsule`, `sweep_down`, `contacts`) are still in the file. Nothing in the sim uses
  them now.

## Approximated or skipped, and why

- **TryPlayerMoveCore 0x140baeba0 and StepDownSlide 0x140baf900** have no spec yet; the docs only describe them as
  calls. They are rebuilt as Source 2013's TryPlayerMove: 4 bumps, 5 clip planes, overbounce 1, plus Quake 3's
  PM_SlideMove nudge: when a bump hits a plane already in the list, the velocity is pushed out to at least 1 unit/s
  away from every plane touched this move and the bump retraces (Q3 nudges along the re-hit plane only, which in a
  crease just re-hits the other plane until the 4 bumps run out). StepDownSlide is
  a slide of up to 8 iterations that stops on the first upward face. StepMove's `Q` ground-plane argument is
  ignored.
- **CheckAutoMantle 0x140b95a40** is only summarised from the decompile. Its gates are ported: the enable cvars,
  the 0.25 s cooldown, and jump held when on the ground or within 0.25 s of it. The ledge search is ours: a wall
  within `searchdist` along the view yaw, head room, room for the hull over the lip, and a walkable top no higher
  than mantleHeight. It then builds the spec's path (start below the ledge, corner, end) and
  `C = mH / (h + mH)`. One extra rule is ours too: you must be pressing forward. Hang/dangle (`pl+0x6718`) is never
  enabled.
- **TraversalMove runs on its `traversal_anim 0` path**, because there are no first-person animations to drive the
  origin. The origin follows the two-segment path at the stored progress. The animation state is still chosen
  (below/level/above/high from eye height over the ledge), because it sets the climb rate.
- **The mantle boost sweet spot.** The R5Regenis reconstruction measures `|traversal camera pitch - eye pitch| <
  1.5°` on the mantle animation, falling toward zero, with progress above 0.5. Its own note says this "opens about
  the last 0.10-0.13 s of a mantle". With no animation, the window here is the last **0.115 s** of the climb at the
  current rate, with progress above 0.5. `Player::mantle_boost.sweet_spot_frac` exposes where the window opens, for
  a future hint UI.
- **Lurch.** Ported from the S21 client, not the ds binary; see "Lurch" below for what is approximated.
- **The mantle boost makes the native jump-off unreachable.** With `mantle_boost_enabled` on (the default, as in
  S21), every jump press during a mantle latches the boost FSM. TraversalMove never sees that press, so the
  native jump-off, the hop-back and the native superglide cannot happen. They are ported and checked with the
  boost turned off (`p.cvars.mantle_boost_enabled = false`).
- **Slide super jump and the Nitro Gate launch** (`slide_super_jump.cpp`, `slide_gate_launch.cpp`) are not ported.
  The first is the `boosted_slide_jump` mod (`player_slideSuperJumpEnabled` is false in stock settings). The second
  is a map gadget.
- **Not in scope:** wallrun proper (wallrun legends; stock pilots only climb), wall hang, double jump, dodge, zipline, grapple, skydive, jetpack,
  slip triggers, launch pads, gravity fields, base velocity, moving ground, standing on players/NPCs, doors, water,
  fall damage, sounds, view punch and LedgeAssist (the jump-assist target search).
- **CheckStuck.** The 54 nudges are Source's table, which Apex fills the same way at runtime. EA skips the whole
  tick while a nudge keeps failing; here the tick still runs, because our trace lets an embedded hull walk out.
- **StartSprint / StopSprint** drive only view and animation timers (sprintStartDelay and friends), so they are
  not ported. The speed itself switches straight from walk to sprint (GetMaxSpeed), and the ramp-up is entirely
  sprintAcceleration.
- **Constants not resolved:** pl+0x6134 speed scale is 1.0 (weapon out; 1.15 reproduces Apex's holstered numbers).
  pl+0x5af0, pl+0x6678 and the time ramp pl+0x6258 are 1. The air-slow-mo time scale is 1. Surface friction is 1
  (SurfaceFromTrace with default surfaces).

## Where this disagrees with Apex-Movement-Lab (the spec wins)

- **Superglide.** The Lab's figure is 540 u/s, from the movement wiki for current Apex. Here the native superglide
  (boost off) comes out at 400: 260 (sprint) + 30 (the stale contact-plane push) at jump-off, then the slide boost,
  which is capped at 400. The S21 mantle boost gives 338 horizontal and a 90u jump.
- **The +30 contact-plane push.** CheckJumpButton pushes you off the plane you were leaning into. The contact plane
  set when the mantle began (pressing into the wall) is still set at a jump-off, so a mantle jump leaves at
  260 + 30. This is spec behaviour, not a bug in the port.
- **Slide decay.** The Lab fits the wiki's "450 → 350 in 0.21 s" with `exp(-1.2 t)`. The spec is slideDecel 100
  plus `0.7^t`, roughly 240 u/s² at 400 u/s: 396 → 283 in 0.5 s.
- **Mantle needs forward.** The Lab says Apex has no auto-mantle. The S21 binary does: in the air, a ledge is
  mantled with no jump held, as long as you push into it. On the ground you must hold jump.
- **Jump height.** The Lab uses 54.2u (285 u/s). The spec gives a jumpHeight-56 impulse, minus the gravity
  half-steps, for a 53.6u apex.
- **Step height.** The Lab uses 18; Apex's settings say 22.

## Tick rate

The client stays at a fixed 60 Hz. Apex simulates one usercmd per client frame, so the superglide (crouch exactly
one tick after the jump-off) and the mantle-boost window are measured in ticks. At 60 Hz that is a 16.7 ms window,
which is easier than at 144 fps but follows the same rule. The checker covers both.

Taps shorter than a tick still count: the client latches every key press, and each mouse-wheel notch, until the
next tick. The mouse wheel is a second jump bind (`Input::jump_pressed`).
