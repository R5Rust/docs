> Synced from [R5Rust/docs/movement.md](https://github.com/R5Rust/R5Rust/blob/main/docs/movement.md), the working copy. Paths are relative to the R5Rust repo.

# Movement: Apex Legends S21 client, ported 1:1

`crates/shared/src/sim/` is a port of the player movement of the **S21 client `r5apex_dx12.exe`** (what R5Flowstate
players run and predict with every frame). It started as a port of R5Regenis's bit-exact specs of the 2019 (S3)
server `r5apex_ds.exe` (`C:\Code\apex\R5Regenis\docs\engine\*.md`). On 2026-11-17 every function was mapped to its
S21 dx12 address (table below), decompiled with GhidraMCP, diffed against the port, and the differences were
ported. Where S21 and the spec agree, the spec text below still describes the code; ds addresses in the sections
below are the spec's, the S21 ones are in the table.

The code follows S21's order of operations and MIN/MAX operand order, and in the places where float noise decides an
outcome (the slide wish strip, the solver's clip and push-out) its exact operand order. It is not bit-exact
everywhere: the engine's SSE sin/cos polynomial is replaced by `f32` sin/cos, NaN paths and networking notifies were
dropped. The surface normal (S21 pl+0x2ab0, ds pl+0x662c) is world up except while climbing a wall, and UPRIGHT is
the engine's double-precision test (`sim::upright`).

## S21 address map (r5apex_dx12.exe, image base 0x140000000; renamed `S21_*` in the Ghidra project)

| Function | S21 dx12 | ds (spec) | Rust | S21 vs the old port |
|---|---|---|---|---|
| PlayerMove | 0x140a54e80 | 0x140ba8fc0 | `mod.rs` `player_move` | accel/friction stance picked before Duck (`Ctx::move_ducked`); traversal gate adds pl+0x350 / jetpack / pl+0x326a (not stock); no CheckStuck in the client (kept: the sim is also the server) |
| SetupMove (buttons, input, speed scale pl+0x3278) | 0x140a47030 | | `simulate` | speed scale = LandingSpeedScale x weapon `move_speed_modifier` (melee 1.15) x status, per tick |
| usercmd auto-sprint / its option check | 0x140905d80 / 0x140902f10 | | `sim::auto_sprint` | new: sprint is pressed from the previous cmd's move axes (>= 0.94, 0.8 while sprinting, within 45 deg, not crouched) |
| UpdateSprint / CanSprint / AllowSprint / StopSprint | 0x140b033f0 / 0x140b02ff0 / 0x140b02ca0 / 0x140b03270 | 0x140bedf60 / 0x140bed930 / 0x140bed5a0 | `update_sprint` etc. | AllowSprint refuses walls and needs last tick's velocity along the view |
| GetMaxSpeed / SpeedCap / SprintSpeedCap / ADS scale | 0x140a32be0 / 0x140b09ba0 / 0x140b09c00 / 0x140a32b20 | 0x140b890b0 / 0x140bf5350 / 0x140bf53c0 | `Ctx::speed_cap` | identical |
| LandingSpeedScale | 0x140b09a60 | 0x140bf5200 | `ground::landing_speed_scale` | identical (now also scales the speed scale) |
| post-think / reference height | 0x140b12970 / 0x140b12850 | 0x140bfc660 | `player_move` tail | identical |
| FullWalkMove | 0x140a4c810 | 0x140b9ead0 | `walk::full_walk_move` | same order; new steps (launch pads 0x140a37390, pull fields via 0x140aca1a0, path mover 0x140a37560, scripted launch 0x140a3b310, trigger steer 0x140a4c4e0) unreachable for a stock pilot |
| ApplyHalfGravity / CheckVelocity | 0x140a36130 / 0x140a50da0 | 0x140b8c2a0 / 0x140ba3b60 | `walk.rs` | identical; CheckVelocity's cache test in double |
| CheckJumpButton | 0x140a3b550 | 0x140b8d490 | `walk::check_jump_button` | identical for a stock pilot (slide super jump, path-mover refusals are mod paths) |
| WalkMove | 0x140a4a7c0 | 0x140b9cdb0 | `walk::walk_move` / `ground_walk` | PlaneFromNormal 0x140a4a720 before StepSlide; slide wish strip in S21 operand order |
| GetAccelerate / Friction / its decel getter / Accelerate | 0x140a36450 / 0x140a4a280 / 0x140a36580 / 0x140a4a0d0 | 0x140b8c5d0 / 0x140b9c7c0 / inline / 0x140b9c610 | `walk.rs` | stale stance (above); otherwise identical |
| AirMove / AirAccelerate / lurch / its decay | 0x140a48f90 / 0x140a485e0 / 0x140a48a30 / 0x140a48810 | 0x140b9b740 / 0x140b9b510 | `walk::air_move` etc. | drag and air-control multiply order; the steep-slope stall caches its trace for CategorizePosition; air control = cvar `player_extraairaccelleration` 2.0 |
| Duck / FinishUnDuck / SetDuckViewOffset / CanUnduck | 0x140a54080 / 0x140a53bf0 / 0x140a53f30 / 0x140b135d0 | 0x140ba7500 / 0x140ba6fa0 / 0x140ba7330 / 0x140bfd500 | `duck.rs` | identical; new `gamemovement_duck_debounce_threshold` (0 = off) |
| StartSliding / StopSliding / GetDuckFraction | 0x140a54890 / 0x140a54be0 / 0x140b09540 | 0x140ba7ed0 / 0x140ba8250 | `duck.rs` | new `slide_max_duck_frac_to_start` (1.0: never refuses) |
| CategorizePosition / SetGroundEntity / StayOnGround / GroundSnapTrace | 0x140a51bc0 / 0x140a51650 / 0x140a348a0 / 0x140a34670 | 0x140ba4f30 / 0x140ba46d0 / 0x140b8b9f0 / 0x140b8b7c0 | `ground.rs` | CategorizePosition reuses this tick's StayOnGround / AirMove-stall trace (ctx+0x4a0) |
| StepSmooth / DecayStepSmooth / CheckFalling / ImpactCheck | 0x140a47d70 / 0x140afb2f0 / 0x140a53990 / 0x140a4d040 | 0x140b9ac40 / 0x140be80c0 | `ground.rs` | identical |
| StepSlide / TryPlayerMoveStep / StepMove | 0x140a47e30 / 0x140a47a40 / 0x140a5c640 | 0x140b9ad70 / 0x140b9a970 / 0x140bb0610 | `step.rs` | StepMove gets the ground plane; S21 accepts a raised move in the air that clears the obstacle (air step-up) |
| TryPlayerMove / TryPlayerMoveCore / StepDownSlide | 0x140a50cd0 / 0x140a5aba0 / 0x140a5b900 | 0x140ba3a90 / 0x140baeba0 / 0x140baf900 | `step.rs` `core` | ported from the decompile, replacing the Source/Quake rebuild: 8 bumps, a 2-plane list reset after each unit of travel, clip `v + n(1/128 - v.n)`, every bump's end pushed 1/128 off the listed planes, crease velocity plus a 1/128 bisector push, a third plane matching neither stops the move, on the ground a 0 < n.z < 0.5 hit becomes a wall |
| CheckAutoMantle / ledge probe / MantleIntent / StartTraversal | 0x140a43060 / 0x140a40c90 / 0x140a40710 / 0x140a41af0 | 0x140b95a40 / - / 0x140b92e80 / 0x140b940a0 | `traversal.rs` | the real ledge search (was a rebuild): per-usercmd probe heights (`command_number % 7`), crouched-hull probe, the real path, hang |
| TraversalMove / FinishTraversal / TraversalPathPos | 0x140a45380 / 0x140a46e40 / 0x140b054a0 | 0x140b981b0 / 0x140b99b60 / 0x140bf15d0 | `traversal.rs` | `traversal_anim 1`: the animation cycle pl+0x2234 is simulated and ends the traversal |
| SurfaceTransition / its classifier / WallProbe | 0x140a4f120 / 0x140a4e220 / 0x140a4ee20 | 0x140ba1990 / 0x140ba0a80 / 0x140ba1690 | `wall.rs` | crouch held never attaches (new in S21) |
| WallRunUpdate / EndWallRun / WallJump / WallSurfaceChange | 0x140a4d4e0 / 0x140b04df0 / 0x140a50050 / 0x140a4fb00 | 0x140b9f6e0 / 0x140bf0ba0 / 0x140ba2c90 / 0x140ba2680 | `wall.rs` | wall probes on a `command_number % 4` cycle (phase 2: a 32u-up probe and `wallrun_avoid_wall_top_decel` 3000) |
| FacingWall / CanWallClimb / WallTop / UpWallBoostScale / WallSpeedFrac / RememberWall | 0x140b04bb0 / 0x140b04c00 / 0x140a4d470 / 0x140a4fed0 / 0x140a34560 / 0x140a58f60 | 0x140bf0a40 / 0x140bf0a90 / 0x140b9f680 / 0x140ba2aa0 / 0x140b8b6b0 / 0x140bacb10 | `wall.rs` | identical |
| LedgeAssist | 0x140a58ec0 -> 0x140a58560 | 0x140baca70 | - | searches map window-hint entities only; none on our maps, so not ported |
| mantle boost | not in the binary | | `traversal.rs` | only in the R5Flowstate SDK hook (`mantle_boost.cpp`), ported from that source |

## Run the checker

```powershell
cargo run --release -p shared --example movecheck
```

It runs 132 checks on a synthetic test world: a floor, a 22u step, a 23u step, a 60u ledge, a 100u cliff, a 300u
wall and a 150u wall.
If the assets are present it also runs them on the real Firing Range, and it re-reads the legend's settings file
to confirm the consts below still match it. It exits with code 1 if any check fails.

Last run: 132/132 passed. A tick on the real 5.4M-triangle map costs **~50 µs** on average and ~0.3 ms at worst
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
| Let go of W on a ledge with wall below it ("can rest") | climbs to automantle_rest_frac 0.4 and hangs (state 11); W climbs on, S or crouch drops you |
| Let go of W on a low ledge (no hang), or pull back with S | the traversal ends and you drop. Past the rest fraction the mantle finishes on its own |
| Mantle boost (`mantle_boost_input_setting 1`; live R5F has 0 = off): jump in the sweet spot | 338 horizontal (260 × 1.3) along the input, vz 367 (`sqrt(2*750*90)`), slide granted. Tap-strafes are blocked until you land |
| Jump outside the sweet spot (state 3) | 260 and a 56u jump (vz 289.8), no slide |

## What is ported, by spec section

| File | Functions | Spec |
|---|---|---|
| `sim/mod.rs` | PlayerMove 0x140ba8fc0 (the per-tick order), SetupMove's buttons and input vector, UpdateSprint 0x140bedf60, CanSprint 0x140bed930, AllowSprint 0x140bed5a0, GetMaxSpeed 0x140b890b0 | playermove.md §3, disassembly |
| `sim/walk.rs` | FullWalkMove 0x140b9ead0, ApplyHalfGravity 0x140b8c2a0, CheckJumpButton 0x140b8d490, WalkMove 0x140b9cdb0 (ground path), Friction 0x140b9c7c0, Accelerate 0x140b9c610, GetAccelerate 0x140b8c5d0, AirMove 0x140b9b740, AirAccelerate 0x140b9b510, CheckVelocity 0x140ba3b60; the lurch is the **S21 client's** AirMove_TapStrafe 0x140a48a30 (r5apex_dx12.exe, see "Lurch") | fullwalkmove.md, walkmove.md, airmove.md, R5Regenis `native/r5r_engine/src/movement.cpp`, decompiled for this port |
| `sim/ground.rs` | CategorizePosition 0x140ba4f30 (ledge re-probe included), SetGroundEntity 0x140ba46d0 (landing energy, landing slowdown, slide-on-land), StayOnGround / GroundSnapTrace, StepSmooth / DecayStepSmooth, CheckStuck, LandingSpeedScale 0x140bf5200 | playermove.md §4-8 |
| `sim/duck.rs` | Duck 0x140ba7500 (4 states, integer-ms timer, smoothstep eye height), CanUnduck, FinishUnDuck, SetDuckViewOffset, StartSliding 0x140ba7ed0, StopSliding | playermove.md §7, slide.md |
| `sim/step.rs` | StepSlide 0x140b9ad70 (all 4 variants), TryPlayerMoveStep 0x140b9a970, StepMove 0x140bb0610 | stepslide.md |
| `sim/traversal.rs` | The **S21 client's** TraversalMove 0x140a45380 (ds 0x140b981b0), TraversalPathPos 0x140b054a0, StartTraversal 0x140a41af0, FinishTraversal 0x140a46e40, CheckAutoMantle 0x140a43060 with its ledge probe 0x140a40c90 and angle test 0x140a40710, the mantle boost (SDK hook) | movemodes.md part B, decompiled for this port |
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
   A probe that misses the wall asks for a retry 0.07 s later (AirMove's deferred transition). Holding crouch
   never attaches (S21 only; the ds build had no such test).
2. **Climb.** Holding W pushes up the wall (the surface basis turns view-forward into "up the wall"). Gravity is 0
   while you push into the wall. The target up-speed falls linearly from climbSpeedStart 200 to climbSpeedEnd 50 over
   climbHeight 100 above WallTop (the attach height, capped at ground + 47), reached at climbAccel 500 u/s².
   Sideways input moves along the wall at climbHorzSpeed 100.
3. **End.** WallRunUpdate ends it when: you reach WallTop + 100 (then a **final jump up**: `vz += sqrt(2g * (28 +
   min(top - z, 10)))`), you look away (facing test fails), you rise slower than 10 u/s after 1 s, you crouch, you
   pull away for 0.05 s, or the wall is no longer within 17u of a probe 16u above the feet. The wall probes run on
   the usercmd's command_number % 4: phase 0 is that 16u probe; phase 2, while rising, probes 32u up and, if that
   misses (the wall's top is passing the chest), cuts vz by 3·ft·wallrun_avoid_wall_top_decel (3000), floored at
   0; phases 1 and 3 do not probe. Falling off pushes you
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
- **LedgeAssist 0x140baca70** (S21 0x140a58ec0 -> 0x140a58560 -> 0x140668c20) is not ported: it searches a
  spatial list of map-placed "window hint" entities (`GetWindowHintForPath`), not world geometry, and bends the
  velocity toward one. Without those entities it never finds a target, so WallRunUpdate's fall-off always applies
  the 70 u/s push and WallJump never gets an assist. S21 still calls it on both paths.
- **WallJump + WallRunUpdate on the same tick.** The literal order (CheckJumpButton, then WallRunUpdate, with the
  surface normal still the wall's) makes a climb jump also trigger the climb-height fall-off, so a climb jump leaves
  at 188·k + 70 and keeps the climb's vertical speed. The S21 client has the same order (CheckJumpButton
  0x140a3b550 calls SetGroundEntity(NULL), which leaves the surface normal alone, then WallJump; FullWalkMove then
  runs WallRunUpdate 0x140a4d4e0), so both apply on the same tick there too. Not yet compared with recorded
  real-game data (the round-1 climb runs never reached the wall).
- **TryPlayerMoveCore's trace record.** The rebuilt Source TryPlayerMove now hands back its first bump's trace
  (Source's `pFirstTrace`) instead of its last one. AirMove's contact plane and SurfaceTransition need the wall
  hit; with the last trace they got an empty normal after a slide. All older checks are unchanged by it.
- **Wall hang** needs wallrunAdsType 1 (stock is 2), so WallHangSlipScale is 1 and the hang branch is not ported.
  The wallrun-legend branches (`wallrun` true: the airborne wallrun start in WallRunUpdate, the up-wall boost at
  attach) are written out where they were short, but no stock legend reaches them.
- **Unknown fields assumed default:** pl+0x6674 (wallrun B/C multiplier) = 1.0 (its constructor value), pl+0x5af0 =
  1, pl+0x6258 time ramp off, the ADS move-speed scale = 1, slide state pl+0x5aac never 2/3, `mv+0xa8` up axis =
  world up, `mv+0x3c` = the view forward. (`*(pl+0x6578) & 3`, first assumed 0, is the current usercmd's
  command_number % 4: now ported, see "End" above.)
- **Crouched in the air**, the classifier moves its probes by CanUnduck's hull delta (the same pl+0x65fc..0x6628
  fields); which of the two hulls each field holds is inferred from CanUnduck.
- **Inline sines.** The engine's SSE sine polynomial (cos as sin(a + pi/2)) is replaced by `f32::sin`; the
  thresholds (cos 45°, 50°, 90°) can differ in the last bit.
- **S21 check (dx12).** Every function above was decompiled again from r5apex_dx12.exe (addresses in
  `sim/wall.rs`'s header) and diffed: the math, constants (10, 1.0, 0.49, 0.7, 16, 0.2929, 140, 0.25) and every
  wallrun_*/wallclimb_* cvar default are unchanged. S21 additions a stock pilot can't reach: jetpackEnabled skips
  the rise/cutoff tests, the slide super jump (pl+0x36c5, needs player_slideSuperJumpEnabled) adds
  slideSuperJumpAdditionalMaxClimbHeight 65 / ...VerticalGainCutoff 200, scripted launches (pl+0x1d3c == 2) skip
  the rise and same-wall tests, climbingNoSlide (false) would hold the climber against slipping, and three extra
  entry flags (pl+0x3078 / 0x4734 / 0x2fd8: slip triggers and jetpack/skydive states).


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
| automantle_maxangle_push / _view / automantle_wallrun_maxangle_view | 50 / 50 / 45 | wallrun_avoid_wall_top_decel (S21 0x142800100) | 3000 |

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
| mantle_boost_input_setting | **0** (off; SDK source default 1 = jump, 2 = crouch, 3 = custom mask). The live R5Flowstate client recorded 2026-11-17 has 0, so the boost never latches there and every mantle is native |

The S21 binary has no mantle-boost code (no `mantle_boost` or `superglide` strings). S21-only automantle cvars
(dx12 ConVar pointer, default): `automantle_forwarddist` 0x1427fbd00 "26.f", `automantle_dangle_required_space`
0x1427f8610 "60", `automantle_disable_hang` 0x1428000a0 "0"; `traversal_anim` 0x142802080 is "1".

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

Resolved since: in the 2026-11-17 log (live `jump_grace_*` dumped, all at the static defaults) every lurch run (D, A,
SD, WD, the first taps of tap_spam_d) matches the real client within ~0.5 u/s; the 236-vs-209 gap came from the old
probe's timing, not the cap.

## Against R5Flowstate (probe round 2, 2026-11-17)

`r5f-probe` plays 86 scripted runs in the real S21 client and logs its per-frame prediction, including the
client's game clock (curtime), so `probecmp` steps our sim with each usercmd's real frametime and the real view
yaw. The client ran at 17 to 200 fps during the recording (retail Apex was running next to it), so frametimes vary
per run; the comparison uses them as recorded. Two input facts came out of the log:

- **KeyState.** A key that goes down during a frame counts 0.5 for that frame (S21 0x140900e40; 0.25 if released
  within it, 1 when held), so the first usercmd of a press wishes for 0.75 of the speed after the soft clamp (the
  real first-frame speed is always 149.64 = 0.75 x 199.5). Modelled in `probecmp` and in the client's
  `bind_input`.
- **Auto-sprint** is pressed by the usercmd builder from the previous cmd's move axes (`sim::auto_sprint`).

Worst horizontal-speed error / worst position error over each run (u/s / u), the pre-port sim (commit 1fafa52)
against the ported one, both replayed with the same probecmp on the same log. Target: <= 2 u/s and <= 10 u.
31 of 86 runs are within it (23 before); walk, sprint, crouch, jump, slide, bhop, slope, drop and most lurch runs
match to the frame (0.0 u/s).

| Run | before | after |
|---|---|---|
| walk | 0.0 / 0.1 | 0.0 / 0.1 |
| sprint_hold | 3.4 / 9.2 | 2.5 / 8.2 |
| sprint_tap | 0.0 / 0.1 | 0.0 / 0.1 |
| crouch_walk | 13.9 / 0.8 | 0.0 / 0.1 |
| jump_stand | 0.0 / 0.0 | 0.0 / 0.0 |
| sprint_jump | 0.0 / 0.1 | 0.0 / 0.1 |
| lurch_d_50 | 51.5 / 16.6 | 51.5 / 14.3 |
| lurch_d_150 | 2.4 / 15.4 | 2.5 / 13.9 |
| lurch_d_250 | 118.3 / 14.3 | 81.8 / 15.2 |
| lurch_d_350 | 4.4 / 1.8 | 4.2 / 1.8 |
| lurch_d_450 | 0.2 / 0.1 | 0.2 / 0.1 |
| lurch_a_150 | 0.0 / 0.1 | 0.0 / 0.1 |
| lurch_s_150 | 0.0 / 0.1 | 0.0 / 0.1 |
| lurch_sd_150 | 29.5 / 11.6 | 29.5 / 11.6 |
| lurch_null_150 | 0.0 / 0.1 | 0.0 / 0.0 |
| tap_spam_d | 75.8 / 38.5 | 75.8 / 37.4 |
| air_strafe_d | 127.4 / 73.8 | 141.6 / 75.5 |
| slide | 9.4 / 5.9 | 0.0 / 0.1 |
| slide_jump | 3.1 / 4.1 | 0.0 / 3.1 |
| bhop | 0.0 / 4.4 | 0.0 / 4.4 |
| walk_back | 0.0 / 0.0 | 0.0 / 0.0 |
| walk_side | 0.0 / 4.3 | 0.0 / 4.3 |
| walk_diag | 23.1 / 12.4 | 10.3 / 7.3 |
| crouch_stand | 62.6 / 38.0 | 0.0 / 0.1 |
| crouch_tap | 3.3 / 0.4 | 0.0 / 0.0 |
| crouch_jump | 0.0 / 0.0 | 0.0 / 0.0 |
| jump_back | 163.8 / 62.5 | 163.8 / 62.5 |
| sprint_jump_duck | 3.7 / 0.4 | 0.0 / 0.2 |
| sprint_jump_stop | 0.0 / 0.1 | 0.0 / 0.0 |
| air_strafe_turn | 85.3 / 47.7 | 85.3 / 47.7 |
| tap_spam_turn | 0.5 / 1.6 | 0.5 / 1.6 |
| lurch_d_turn | 103.8 / 44.1 | 103.8 / 44.1 |
| lurch_wd_150 | 254.0 / 125.9 | 254.0 / 136.0 |
| slide_long | 6.7 / 6.2 | 0.4 / 0.6 |
| slide_no_w | 2.0 / 3.4 | 2.0 / 3.4 |
| slide_side | 4.1 / 52.7 | 4.1 / 52.7 |
| slide_jump_late | 10.2 / 14.8 | 0.4 / 3.1 |
| slide_cooldown | 93.1 / 51.7 | 0.4 / 4.4 |
| jump_land_slide | 368.0 / 937.9 | 368.0 / 937.8 |
| bhop_slide | 379.0 / 522.3 | 379.0 / 522.3 |
| bhop_fast | 0.0 / 4.4 | 60.8 / 51.7 |
| drop_215 | 299.0 / 519.2 | 299.0 / 519.2 |
| drop_71 | 0.0 / 3.0 | 0.0 / 3.0 |
| drop_71_jump | 0.0 / 6.7 | 0.0 / 6.7 |
| step_13 | 50.9 / 12.5 | 46.7 / 10.4 |
| slope_up | 0.0 / 7.4 | 0.0 / 7.4 |
| slope_down | 0.0 / 7.5 | 0.0 / 7.5 |
| slope_slide | 458.4 / 739.9 | 379.5 / 692.2 |
| slope_jump_up | 202.8 / 1205.2 | 106.7 / 1178.5 |
| mantle_hi | 0.0 / 0.9 | 0.0 / 0.9 |
| mantle_hi_late | 65.3 / 33.8 | 299.0 / 43.8 |
| mantle_hi_stand | 0.0 / 27.6 | 0.0 / 26.1 |
| mantle_lo | 1.4 / 21.3 | 1.4 / 21.3 |
| mantle_lo_late | 63.8 / 24.8 | 3.3 / 26.1 |
| mantle_lo_stand | 0.0 / 0.0 | 0.0 / 0.0 |
| mantle_boost_650 | 0.0 / 2.7 | 0.0 / 2.7 |
| superglide_650 | 299.0 / 73.1 | 199.5 / 20.7 |
| mantle_boost_700 | 0.2 / 56.0 | 0.0 / 27.1 |
| superglide_700 | 299.1 / 87.4 | 199.5 / 33.0 |
| mantle_boost_750 | 0.3 / 56.6 | 0.0 / 23.5 |
| superglide_750 | 299.1 / 92.6 | 199.5 / 33.7 |
| mantle_boost_800 | 299.1 / 99.2 | 201.3 / 63.6 |
| superglide_800 | 459.2 / 83.5 | 258.2 / 105.5 |
| mantle_boost_850 | 299.0 / 106.1 | 205.0 / 61.8 |
| superglide_850 | 90.1 / 107.7 | 211.5 / 62.0 |
| mantle_boost_900 | 299.0 / 21.2 | 211.5 / 61.5 |
| superglide_900 | 299.0 / 22.4 | 211.5 / 61.4 |
| mantle_tall | 299.0 / 119.4 | 299.0 / 131.7 |
| mantle_tall_stand | 299.0 / 87.1 | 299.0 / 94.1 |
| mantle_tall_release | 135.1 / 122.9 | 7.0 / 29.7 |
| mantle_tall_jump_700 | 69.2 / 89.0 | 299.0 / 91.5 |
| superglide_tall_700 | 84.3 / 86.8 | 299.0 / 99.4 |
| mantle_tall_jump_800 | 299.0 / 88.7 | 299.0 / 90.0 |
| superglide_tall_800 | 188.0 / 100.3 | 257.8 / 114.0 |
| mantle_tall_jump_900 | 202.0 / 88.0 | 295.2 / 133.6 |
| superglide_tall_900 | 243.3 / 104.8 | 249.1 / 84.1 |
| boost_on_tall_850 | 91.1 / 61.6 | 84.1 / 59.6 |
| boost_on_tall_900 | 14.6 / 87.8 | 14.6 / 86.7 |
| boost_on_tall_950 | 378.6 / 216.2 | 378.6 / 200.1 |
| boost_on_tall_1000 | 357.6 / 207.1 | 354.6 / 190.9 |
| climb | 189.9 / 36.5 | 185.7 / 41.6 |
| climb_walljump | 187.6 / 22.2 | 186.1 / 21.3 |
| climb_walljump_back | 94.9 / 107.6 | 94.9 / 107.2 |
| climb_release | 209.0 / 75.8 | 209.0 / 78.9 |
| forward_ras_ref_0 | 72.8 / 54.8 | 71.6 / 56.2 |
| forward_ras_no_ts_0 | 63.3 / 137.8 | 63.3 / 136.9 |

What is left, and why:

- **Static-prop collision differs from the game's** (most of the remaining large errors). Every run that touches a
  static prop's sloped or bevelled surface diverges where the real player does not collide with that surface the
  way our `collision.bin` (each prop's detailed tree 0) says: the ramp under the 77.5u ledge spot is ~7u higher in
  ours (real stands at z 1503.25 at (-10416,668), ours at 1509.9: mantle_hi_stand, mantle_lo*, the mantle_boost and
  superglide runs there), the 50-degree chamfer at the base of the hall wall (y 757-787, z 1602-1638) deflects our
  climb jump upward while the real player meets a vertical face and attaches (climb*), the slanted-top block
  behind the hall spawn (jump_back) and the hall-side prop the air-strafe runs reach at y ~-47 (air_strafe_d,
  air_strafe_turn, lurch_d_turn, lurch_wd_150, tap_spam_d). This points at the player trace using a different
  prop tree than ours, most likely the physics replica (tree 1); see docs/collision.md "Not verified against the
  engine binary". It needs `export-map` (`split_trees`) and a re-export, which this pass did not touch.
- **Teleports that did not take.** 13 runs started where the previous run ended (the server ignored `rp tp`
  four times in a row); probecmp starts ours from the real first sample, so those runs still compare like for
  like, but on other floors than planned (jump_land_slide, bhop_slide, drop_215, slope_slide and slope_jump_up run
  long paths over the hall's props and pick up the collision difference above).
- **Jump into a wall (jump_back).** With the prop as it is in our collision, a jump while pressing into it loses
  its vertical motion every other tick: StepMove accepts the raised move B (it gets 1/128 further than move A,
  which the solver pushes back off the wall) and then restores the original velocity. The S21 StepMove and
  TryPlayerMoveCore read the same as ours line by line; the real client keeps move A here, so the difference is in
  what the traces return at that prop.

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

- **TryPlayerMoveCore / StepDownSlide** are ported from S21 (table above). The old Quake-3-style nudge is gone: S21's
  1/128 clip and end-point push-out keep a hull resting on a plane from re-hitting it, which is what the nudge was
  for (hangscan: the same 1 hang in 1944 runs before and after). On a flat floor the push-out makes the feet cycle
  z+1/32, +1/128, +1/128 like the real client's log does.
- **CheckAutoMantle (S21 0x140a43060)** is ported from the S21 decompile, traces included. Gates: (no ground, on
  a wall, or jump held) and (0.25 s since pl+0x362c, or jump held), automantle_cooldown since the last start, and
  (forwardmove >= 0.3, on a wall, or moving along the view yaw). A hull trace along the yaw (searchdist) must hit a
  wall; 0x140a40710 checks it against the input (maxangle_push, no input passes) and the view (maxangle_view x
  mantleAngleScale). The ledge probe 0x140a40c90 sweeps the **crouched** hull into the wall at one height per
  usercmd, picked by `command_number % 7`: mantleHeight + 1/32 (0 and 4), x 66.5/80, 26.5/80, 16.5/80, 34.5/80,
  and 6 = a window-hint entity search (none here, so that tick never mantles). It must get `automantle_mindist` past
  the wall, then traces down onto the ledge, checks head room, refuses a step-height rise walked into on flat
  ground (and stairs/slopes), and casts the hang ray. Without forward you still mantle if the wall continues
  `dangle_required_space - 5` below the lip (you can hang). The path: corner straight above the origin at the lip
  height (found by walking the ledge plane back), start mantleHeight + 1/32 below it, end = the hull centre over
  the wall face (`min(reach, dist - mindist) + halfwidth` in), `C = mH / (h + mH)`, a minimum target from the
  floor under the start, and "can rest" (hang). Not modelled: 0x140a44120's ray refinement of the ledge point
  (taken as found, can rest assumed true when the hang ray hits), window traversal, the pl+0x1d3c == 2 branch,
  grapple, movers, the weapon-lowering prediction. `Player::cmd_number` stands in for the usercmd number.
- **TraversalMove (S21 0x140a45380) runs with `traversal_anim 1`** like the game, minus the animation data: the
  state choice and the smoothed cycle pl+0x2234 (`Traversal::cycle`) are simulated, and the END test reads that
  cycle. The origin, which S21 takes from the animation's root motion, follows the two-segment path at the stored
  progress instead. Jump-off and END still put you on the path (S21 does too).
- **The mantle boost sweet spot.** The SDK measures `|traversal camera pitch - eye pitch| < 1.5°` on the mantle
  animation (falling toward zero) and requires the animation cycle pl+0x2234 above 0.5. Its note says this "opens
  about the last 0.10-0.13 s of a mantle". With no animation, the window here is the last **0.115 s** of the climb
  at the current rate, with the simulated cycle above 0.5. `Player::mantle_boost.sweet_spot_frac` exposes where
  the window opens, for a future hint UI.
- **Lurch.** Ported from the S21 client, not the ds binary; see "Lurch" below for what is approximated.
- **The mantle boost hides the native jump-off when it is armed.** With `mantle_boost_input_setting 1`, every jump
  press during a mantle latches the boost FSM and TraversalMove never sees it, so the native jump-off, hop-back and
  native superglide cannot happen. With the live R5F value 0 the hook never latches and they are the game's behaviour.
  R5Rust folds that "off" into `mantle_boost_enabled = false` (the default), so the client's Mantle Boost toggle
  turns the SDK's jump-input boost on.
- **Slide super jump and the Nitro Gate launch** (`slide_super_jump.cpp`, `slide_gate_launch.cpp`) are not ported.
  The first is the `boosted_slide_jump` mod (`player_slideSuperJumpEnabled` is false in stock settings). The second
  is a map gadget.
- **Not in scope:** wallrun proper (wallrun legends; stock pilots only climb), wall hang, double jump, dodge, zipline, grapple, skydive, jetpack,
  slip triggers, launch pads, gravity fields, base velocity, moving ground, standing on players/NPCs, doors, water,
  fall damage, sounds, view punch and LedgeAssist (the jump-assist target search).
- **CheckStuck** does not exist in the S21 client; it is kept for the server side. The 54 nudges are Source's table, which Apex fills the same way at runtime. EA skips the whole
  tick while a nudge keeps failing; here the tick still runs, because our trace lets an embedded hull walk out.
- **StartSprint / StopSprint** drive only view and animation timers (sprintStartDelay and friends), so they are
  not ported. The speed itself switches straight from walk to sprint (GetMaxSpeed), and the ramp-up is entirely
  sprintAcceleration.
- **Speed scale** (S21 pl+0x3278) = LandingSpeedScale x the active weapon's `move_speed_modifier` x status effects.
  Unarmed in R5 means the melee weapon: 1.15. `Player::speed_scale` holds the weapon part (1.0 by default, so the
  checker's numbers are weapon-out; the client and probecmp set 1.15).
- **Constants not resolved:** S21 pl+0x350 in the traversal gate (`traversal_cooldown + pl+0x350 > now`).
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
- **Auto-mantle.** The Lab says Apex has no auto-mantle. The S21 binary does: in the air (after a jump, or 0.25 s
  after walking off), a ledge you face is mantled with no jump held, if you push forward or it is a ledge you
  can hang from. On the ground you must hold jump.
- **Jump height.** The Lab uses 54.2u (285 u/s). The spec gives a jumpHeight-56 impulse, minus the gravity
  half-steps, for a 53.6u apex.
- **Step height.** The Lab uses 18; Apex's settings say 22.

## Tick rate

The client stays at a fixed 60 Hz. Apex simulates one usercmd per client frame, so the superglide (crouch exactly
one tick after the jump-off) and the mantle-boost window are measured in ticks. At 60 Hz that is a 16.7 ms window,
which is easier than at 144 fps but follows the same rule. The checker covers both.

Taps shorter than a tick still count: the client latches every key press, and each mouse-wheel notch, until the
next tick. The mouse wheel is a second jump bind (`Input::jump_pressed`).
