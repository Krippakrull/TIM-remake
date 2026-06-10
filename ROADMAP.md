# ROADMAP.md — Contraption Lab

Planned features in priority order. Each item notes scope, the engine surface it touches, and its verification burden (see AGENTS.md for the solver workflow — nothing ships unverified).

## Shipped

- **v1** — Core engine, 10 levels, 5 parts (plank, trampoline, conveyor, fan, bumper), headless solver harness, win/fail/stuck detection, touch + keyboard support.
- **v2** — Cannon (capture → aim → fire) and magnet (radial pull field), levels 11–13, magnet tuning pass, regression suite for engine changes.

## Milestone 3 — Cheap part variants (small, high fun-per-line)

Surface/force variants of existing code paths. Each is roughly a `PARTS` entry, one physics branch, one draw branch, one intro level.

1. **Ice plank** — frictionless segment (`vt *= 1.0`), restitution ~0.05. Enables speed-control puzzles. Engine: trivial variant of plank branch.
2. **Sticky pad** — segment that zeroes velocity on contact (ball parks on it). Pairs with fan/cannon for two-stage contraptions. Watch the stuck-detector: a parked ball must not trigger `stuck` if a fan/cannon can still move it — likely needs `stuckT` exemption while ball rests on a sticky pad inside a fan zone, or simply a longer stuck timeout on levels that include sticky pads.
3. **Spinner** — motorized wheel: bumper geometry but tangential impulse instead of radial (flings the ball around its rim). One sign flip from the bumper code plus rotation-direction UI (reuse rotate buttons to set spin direction).
4. **Portal pair** — placed as a linked pair; teleports the ball preserving velocity, short cooldown to prevent oscillation. First part type placed as two objects — placement UX needs a "place A, then B" carrying flow.

Levels 14–17, one new mechanic each. Difficulty target: intro-easy per the hit-rate heuristic.

## Milestone 4 — Multi-object engine (the real TIM step)

The defining feature of The Incredible Machine is causal chains between objects. This is a deliberate refactor milestone — it touches the win condition, solver harness, and level design simultaneously, so it ships alone.

- **Engine**: `ball` → `bodies[]`, each with `radius`, `mass`, `restitution`, `gravityScale`. Circle–circle collision between bodies (impulse exchange by mass ratio). Fans/magnets/cannons act per-body.
- **Bodies**: standard ball; **bowling ball** (heavy, low bounce, crushes through bumper kicks); **balloon** (negative `gravityScale`, rises; popped by a **pin** part — pop removes the body, drops anything resting on it).
- **Seesaw** — plank with a central pivot and simple torque from body weight. First rotating dynamic; keep it kinematic-with-one-degree-of-freedom rather than full rigid-body.
- **Goals**: per-level target body ("get the **red** ball in the bucket"); other bodies are tools. Win check loops over the target body only.
- **Solver**: `attempt()` already isolates `physStep` — extend stubs for `bodies[]`, and extend the regression set with one multi-body level the moment it exists.

Levels 18–22. Expect heavy tuning; budget the largest verification pass of any milestone.

## Milestone 5 — Player-facing polish (no engine changes)

- **Undo** (single-level history stack of `placed[]` snapshots) — cheap, high value.
- **Par + stars** — star rating per level for solving under a parts-used or time par; pure UI + level metadata.
- **Sound toggle** — one button, gates `blip()`.
- **Share codes** — serialize `placed[]` to a compact string in the URL hash so solutions can be shared. Hash, not storage APIs (artifact-safe). Note: outside claude.ai artifacts, `localStorage` progress persistence becomes possible — gate it behind a feature detect, never assume it.
- **Replay ghost** — store the winning trail and redraw it faintly in edit mode.

## Milestone 6 — Level editor (stretch)

- Edit mode that exposes statics, spawn, bucket, and crate counts; exports a `LEVELS`-format JSON snippet.
- Built-in "verify" button that runs a budgeted random + guided search *in the browser* (the solver is just `physStep` in a loop — it already runs client-side, only the harness stubs are Node-specific).
- Import via URL hash, same constraint as share codes.

## Explicitly out of scope

- Frameworks, bundlers, build steps, external assets — the single double-clickable file is a feature.
- Networked features, accounts, leaderboards.
- Copyrighted TIM assets, names, or level recreations — this stays an original tribute.
