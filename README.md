# R5Rust docs

How the things R5Rust reimplements actually work in Apex Legends (the S21 build R5Flowstate runs): the mechanics,
the real numbers, where they come from in the game, and how closely R5Rust matches the real game.

Every number here is either read from the game (decompiled from the S21 client `r5apex_dx12.exe`, or from the game's
own settings files) or measured in real R5Flowstate with [r5f-probe](https://github.com/R5Rust/r5f-probe). Anything
inferred or approximated is marked as such.

## Topics

| Topic | What's in it |
|---|---|
| [Movement](movement/README.md) | Sprint, acceleration, jumps, crouch, slides, air strafing, lurch / tap-strafe, bhops, superglides, mantle + mantle boost, wall climb + wall jump. Deep detail: [implementation reference](movement/implementation.md) |

## Adding a topic

One folder per topic (`rendering/`, `collision/`, `map-data/`, ...), each with a `README.md` written for people who
want to understand the mechanic, plus an `implementation.md` for the code-level detail (functions, addresses, order
of operations). Add a row to the table above.

Code lives in the [R5Rust](https://github.com/R5Rust/R5Rust) repo; its `docs/` folder is the working copy the
implementation pages are synced from.
