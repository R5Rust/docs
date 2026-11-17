# Movement

How player movement works in Apex Legends S21 and how R5Rust reproduces it. Ziplines, zip towers, grapple, jump pads
and legend abilities are not covered yet.

> **Status (2026-11-17):** the movement is being re-ported function by function from the S21 client
> (`r5apex_dx12.exe`). The lurch is already S21; the rest was first ported from a 2019 server build and is being
> checked against the S21 code and real R5Flowstate recordings. Numbers below get updated as each part lands.

## The numbers

Values for a stock legend. Apex scales move speed by what you hold: **1.0 with a weapon out, ~1.15 holstered**
(melee / heirloom). R5Rust has no weapons yet, so it plays holstered.

| | Weapon out | Holstered (R5Rust) |
|---|---|---|
| Walk | 173.5 u/s | 199.5 u/s |
| Sprint | 260 u/s | 299 u/s |
| Crouch walk | 80 u/s | 92 u/s |
| Slide start (from a sprint) | +150 boost, capped at 400 | same cap |

| | Value |
|---|---|
| Gravity | 750 u/s² (applied in halves at the start and end of each tick) |
| Jump height | 56 u setting; about 53.6 u actual apex (takeoff ~271 u/s) |
| Slide-jump height | 50 u |
| Step height | 22 u (a 23 u step blocks you) |
| Player hull | 32 × 32 × 72 standing, 32 × 32 × 47 crouched; eyes at 60 / 32 |
| Mantle height | up to 80 u above the feet |

## Mechanics

### Sprint
- A sprint press asks to sprint for 3 s (holding it keeps extending that). Once you're sprinting it **latches**: it
  stays on while you keep moving forward (within ~45°) and aren't crouched. Stopping, going backwards or crouching
  ends it.
- **Auto Sprint** (an Apex setting, on by default in R5Rust) sprints whenever you push forward, no key needed.
- Sprint only engages on the ground (or within 0.25 s of it) and while moving mostly forward.

### Acceleration and friction
- Acceleration is **linear**, not Source's `accel × wishspeed`: 2500 u/s² below 120 u/s, then 450 up to walk speed,
  then 100 up to sprint speed. So 0 to 200 u/s takes about 0.38 s and reaching full sprint about 1 s.
- Friction is a flat 1250 u/s² deceleration, and only acts on velocity **beyond** what you're asking for: speed
  along your input direction, up to your wish speed, is never slowed. A full sprint stops in about 0.22 s.

### Jumping
- A jump press counts only on the tick it happens (no buffer, no auto-hop).
- **Coyote time:** you can still jump for 0.2 s after walking off a ledge.
- **Anti multi-jump:** jumping again right after landing gives a much smaller jump (15.5 u if immediate, full height
  after about 0.75 s).
- **Bunny hop speed loss:** jumping within 1 s of landing above sprint speed bleeds 100 u/s off the excess, never
  below sprint speed.

### Crouch and slide
- Crouching takes 400 ms (200 ms into a slide) before the hull shrinks. Standing up needs room for the standing hull.
- **Slide:** crouch while moving at least 200 u/s on the ground. You get a +150 boost (capped at 400) if you haven't
  slid in the last 2 s. The slide then slows by friction plus a speed decay (roughly 396 → 283 u/s over 0.5 s) and
  ends below about 125 u/s.
- **Slide jump:** jumping within 0.4 s of the slide boost removes the boost; at 300–350 u/s it snaps to 350.

### Air strafing
- Air control is weak: 60 u/s wish speed with 500 u/s² acceleration, so you can bend your path but not gain speed
  the Source way.

### Lurch / tap-strafe (S21 jump grace)
- For **0.5 s after a jump**, pressing a movement key **newly** (a tap, not a held key) redirects your velocity toward
  that key's direction. Strength is full (0.7) for the first 0.2 s, then fades to 0 at 0.5 s.
- Each lurch can change your velocity by at most 0.7 × sprint speed (209 u/s holstered). Turning further than that
  costs speed.
- **Pressing back (S)** cuts speed by that cap: it brakes rather than reverses.
- **Null inputs:** A+D or W+S together cancel, so pressing the second key of a pair does nothing.
- **Scroll-wheel tap-strafing** (each wheel notch is a forward tap) works, because every notch is a new press.
- No lurches above 1100 u/s, and none while a mantle boost is in the air.

### Mantle and mantle boost
- **Mantle:** push into a ledge up to 80 u above your feet. On the ground you must hold jump; in the air, pushing
  into it is enough. It takes 0.35–0.7 s depending on how high the ledge is.
- **Mantle exits:** finishing while holding W leaves at walk speed (sprint if latched). Jumping off past the corner
  gives a jump plus your input speed. Jumping below the corner without W hops you back off the wall.
- **Superglide** (mantle boost off): jump off at the end of a mantle, then crouch on the very next tick, for about
  400 u/s.
- **Mantle boost** (on by default, the universal S21 mechanic): jump in the **sweet spot** at the very end of a mantle
  (about the last 0.115 s) to launch at sprint × 1.3 (338 u/s) with a 90 u jump and a free slide. Tap-strafes are
  blocked until you land. Jumping outside the sweet spot gives a normal 260 u/s, 56 u jump.

### Wall climb and wall jump
- **Attach:** jump into a wall while looking at it (within ~45°), moving into it and rising slower than 100 u/s, with
  your feet no more than about 147 u above the ground. If a ledge is within mantle reach, the mantle wins instead.
- **Climb:** hold W. Gravity is off while you push into the wall; you rise at 200 u/s slowing to 50 u/s over 100 u,
  then get a small final hop up (about 28 u).
- **Ends** when you reach the top, look away, slow under 10 u/s after 1 s, crouch, pull away, or lose the wall. You
  can't climb again until you've stood on the floor.
- **Wall jump (bounce):** jump while climbing to kick off the wall: 188 u/s outward (less if you're still facing and
  pushing into the wall) plus a 70 u/s push-off.

## How close R5Rust is to the real game

Measured by recording the same inputs in real R5Flowstate (Octane, 60 fps, holstered) and replaying them through
R5Rust's movement code (`r5f-probe` + `probecmp`). Horizontal speed, R5Rust / real:

| Run | Result |
|---|---|
| Walk / sprint top speed | 299 / 299 |
| Crouch walk | 92 / 92 |
| Standing jump (vertical speed at 0.5 s) | 146 / 143 |
| D lurch 150 ms after a jump | 256 / 263 |
| S+D lurch | 174 / 168, then 184 / 180 |
| S (back) lurch | 48 / 30, then 69 / 69 |
| Forward RAS (LiTTle's reference timing) | 468, 491, 354, 327 / 470, 496, 363, 330 |

Being re-verified as the S21 port lands: acceleration curves, slides on slopes, bhop chains, superglide, step-ups,
mantles, mantle boost, wall climb and wall jump.

## Sources

- The S21 client `r5apex_dx12.exe` (decompiled), the game's legend settings files, and live cvar values read from a
  running R5Flowstate.
- R5Regenis's bit-exact specs of the older server build (`r5apex_ds.exe`) for the parts not yet re-ported.
- The mantle boost from R5Flowstate's SDK.
- Full function list, addresses, every setting and cvar: [implementation.md](implementation.md).
