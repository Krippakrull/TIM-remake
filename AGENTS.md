# AGENTS.md — Contraption Lab

Instructions for AI coding agents working on this codebase. Read this fully before editing.

## What this is

A browser-playable Rube Goldberg puzzle game (tribute to The Incredible Machine). The player places parts (planks, trampolines, conveyors, fans, bumpers, cannons, magnets) on a blueprint canvas to guide a ball into a goal bucket. 13 levels, single ball, custom 2D physics.

## Project structure

```
contraption-lab.html    # THE ENTIRE GAME. Single file: CSS + HTML + JS, no build step.
AGENTS.md               # This file.
ROADMAP.md              # Planned features, in priority order.
```

There is deliberately **no build step, no bundler, no framework, no external dependencies**. Keep it that way unless ROADMAP.md says otherwise. The file must remain openable by double-clicking it.

## Hard rules

1. **Never ship a level you haven't machine-verified as solvable.** Use the headless solver harness (below). This is non-negotiable — two real bugs (floating buckets, an unsolvable level 2) were only caught this way.
2. **The ball must enter the bucket from above.** Bucket walls reach the floor by design; side entry is impossible. Every level's intended solution must end with a drop or arc into the mouth. If you move a bucket, its bottom (`bucket.y + 56`) must rest on a surface (floor top edge is y=541; for shelves, align with the shelf's collision surface).
3. **No `localStorage` / `sessionStorage` / browser storage.** This file runs inside claude.ai artifacts where those APIs fail. All state is in-memory.
4. **Run `node --check` on the extracted `<script>` contents after every edit.** Extraction one-liner:
   ```bash
   python3 -c "import re; open('game.js','w').write(re.search(r'<script>(.*)</script>', open('contraption-lab.html').read(), re.S).group(1))" && node --check game.js
   ```
5. **Original art and audio only.** No copyrighted assets, sprites, or sound clips. Everything is drawn with canvas primitives and synthesized with Web Audio.
6. **Physics constants are load-bearing.** Don't tune `GRAV`, friction, restitution, fan/magnet force, or cannon fire speed without re-verifying *all* levels — a friction change once flipped several levels between solvable and unsolvable.

## Architecture (all inside the `<script>` tag)

| Section | What it does |
|---|---|
| `PARTS` | Part catalogue: name, description, segment length (0 for point parts). |
| `LEVELS` | Level definitions: `name`, `hint` (HTML), `ball` spawn, `bucket` position, `statics` (fixed planks via `pl(x,y,len,angleDeg,thick)`), `parts` (crate counts). |
| State block | `levelIdx`, `crate`, `placed[]`, `selected`, `carrying`, `mode` ('edit'/'run'), `ball`, timers. |
| `blip()` | Tiny Web Audio synth. Always wrap in try/catch (already done). |
| Geometry | `segEnds`, `closestOnSeg`, `allSegments()` (statics + placed surface parts + 3 bucket wall segments). |
| `physStep(dt)` | The physics core. Fixed timestep 1/240s. Returns `null` (continue), `'win'`, `'fall'`, `'stuck'`, or `'timeout'`. |
| Drawing | `drawGrid`, `drawSeg` (one big switch per part kind), `drawBucket`, `drawBall`, `render`. |
| Loop | `frame()` with accumulator; `endRun(result)` handles win/fail UX. |
| UI | `loadLevel`, `renderDots`, `renderToolbox` (uses `PART_ICONS` inline SVG), pointer/touch/keyboard handlers. |

### Physics cheat sheet

- Canvas 900×560, `BALL_R=12`, `GRAV=920` px/s², speed cap 1150, fixed step `h=1/240`.
- Surface parts are line segments. Collision: closest-point push-out + normal reflection.
  - Plank restitution 0.35; rolling friction `vt *= 0.999` per contact step.
  - Trampoline: outgoing normal speed `clamp(|vn|*1.02, 470, 980)`.
  - Conveyor: tangential velocity pulled 50% per contact step toward belt speed 260 (sign follows part rotation).
- Fan: directional zone 10–220 px long, ±58 px wide, force `900*(1-along/220)`.
- Magnet: radial pull within 200 px, force `700*(1-d/200)`; solid core r=15 with weak bounce. Drawn field circle is exactly the physics radius — keep them in sync.
- Bumper: solid circle r=22, reflects + adds 260 px/s radial kick.
- Cannon: captures ball within 26 px when `cool<=0`; holds 0.5 s (`loadT`), fires at **700 px/s** along its angle, then `cool=0.9`. Max projectile apex ≈ `700²/(2·920) ≈ 266` px above muzzle — remember this when placing elevated buckets.
- Win: ball center within ±24 px of bucket x and `bucket.y+8 < y < bucket.y+52` for 0.25 s dwell.
- Fail: off-screen (`fall`), speed <16 for 2.2 s (`stuck`), or 30 s elapsed (`timeout`).

### Part runtime state

Per-placed-part mutable fields (`flash`, `cool`, `loadT`) are reset in `resetBall()`. If you add a part with runtime state, reset it there.

## The headless solver harness (verification workflow)

The game runs in Node with ~25 lines of DOM stubs. Use this to (a) prove every level solvable, (b) regression-test engine changes against known-good solutions, (c) gauge difficulty by random hit rate.

```js
// harness boilerplate — stubs DOM, loads the game, exposes the physics API
const el = () => ({ classList:{add(){},remove(){},toggle(){}}, style:{}, set innerHTML(v){}, get innerHTML(){return ''},
  appendChild(){}, addEventListener(){}, getBoundingClientRect(){return{left:0,top:0,width:900,height:560}},
  set textContent(v){}, set onclick(f){this._oc=f;}, get onclick(){return this._oc||(()=>{});}, set disabled(v){} });
const stash = {};
global.document = { getElementById:(id)=>{ stash[id]=stash[id]||el(); return stash[id]; }, createElement:()=>el() };
stash.cv = Object.assign(el(), { width:900, height:560, getContext:()=>new Proxy({},{get:()=> ()=>{}, set:()=>true}) });
global.window = { addEventListener(){} };
global.requestAnimationFrame = ()=>{}; global.setTimeout=()=>0; global.clearTimeout=()=>{};
const fs = require('fs');
fs.writeFileSync('game.cjs', fs.readFileSync('game.js','utf8').replace('"use strict";','')
 + '\nmodule.exports={loadLevelX:loadLevel,physStepX:physStep,resetBallX:resetBall,place:(p)=>placed.push(p),clearPlaced:()=>{placed.length=0;},getBall:()=>ball,LEVELS};');
const G = require('./game.cjs');

function attempt(levelIdx, parts){              // parts: [{type,x,y,angle}] — angle in radians
  G.loadLevelX(levelIdx); G.clearPlaced();
  parts.forEach(p=>G.place(Object.assign({id:1},p)));
  G.resetBallX();
  const h=1/240; let t=0, res=null;
  while(t<22 && !res){ res=G.physStepX(h); t+=h; }
  return res==='win';
}
```

Verification recipe for any level change:

1. **Random search**: sample placements uniformly (`x∈[60,840]`, `y∈[80,530]`, any angle), ~5000 tries. Hits ⇒ solvable *and* forgiving.
2. **Guided search**: if random fails, constrain ranges/angles to the intended solution shape (e.g. shallow downhill angles for plank roads). 0 hits in a guided 8000 ⇒ treat as unsolvable and redesign.
3. **Regression set**: keep known-good solutions for at least L2 (3-plank staircase) and L8 (elevated conveyor line) and replay them after any engine change. Current known-good:
   - L2: planks at (147,187,22°), (393,375,20°), (543,457,9°)
   - L8: plank (150,230,16°), conveyors (330,290,2°), (520,300,0°)
4. **Difficulty heuristic**: random hit rate >1% = tutorial-easy; 0.05–1% = normal; <0.05% = finale-hard. Intro levels for a new part should be on the easy end.

## Conventions

- Coordinates: y grows downward; angles in radians internally, degrees in level defs via the `pl()` helper and in solver notes.
- Hints are HTML strings; new parts get a `New part!` hint with a `<b>bold</b>` name in their intro level.
- New part checklist: `PARTS` entry → physics in `physStep` → `drawSeg` branch → `PART_ICONS` SVG → `hitPart` (point parts go in the radius-30 branch) → runtime-state reset in `resetBall` → an intro level → solver verification → regression run.
- Keep the toast/fail messages playful but instructive (they double as hints).
- Sounds: short `blip()` calls only; keep volumes ≤0.12.
