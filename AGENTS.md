# Just Gonna Send It — maintainer's guide

An Angry-Birds-style 2D physics game: drag back to launch a snowmobile (ridden by
**Larry Enticer** — denim jacket, mullet, mustache, orange aviators, beer in hand) down a
runway, off a kicker ramp, into towers of beer cans. Land the arc on the elevated cans to
knock them all down and clear the level. 6 levels, escalating towers.

## Layout / how to run

- **Everything lives in one file:** `index.html` (HTML + CSS + JS, no build step, no deps
  except the Matter.js CDN tag). Edit it directly.
- **Run it:** `python3 -m http.server 8731` in this directory, then open
  `http://localhost:8731/index.html`.
- **Physics:** Matter.js v0.20.0 (CDN). Modules used: `Engine, Bodies, Body, Composite,
  Vector, Query`.
- **Rendering:** HTML5 canvas 2D, hand-drawn vector art (no image assets). Audio is
  synthesized via Web Audio (no sound files).

## The launch sequence (read this before touching physics)

1. **Aim:** `state === 'aim'`, sled is `static` at `ANCHOR` (x=200). Mouse/touch drag sets
   `dragWorld`; `drawPowerGauge()` shows a throttle bar, an mph readout, and a dotted
   **trajectory preview** arc.
2. **Release (`endDrag`):** launch speed = `min(dragDist, MAX_DRAG) * POWER`. The sled is
   un-static'd and given a purely **horizontal** velocity `{x: speed, y: 0}` — it guns
   straight down the flat runway. `state = 'fly'`, `shots++`, one-shot `sndRev()`.
3. **Up the kicker:** the ramp is built from ~22 overlapping tangent arc segments
   (`rampSurface(phi)`), low friction. Climbing it converts horizontal speed into a launch
   angle and **loses ~28% of speed** — see `LAUNCH_EFF = 0.72` (lip speed ≈ 0.72 × launch).
4. **Liftoff:** the instant the sled clears the lip, a one-time angular kick
   (`±0.11`, `launchBoosted`) starts the crazy backflip.
5. **Air control:** while `airborne`, ◄►/A,D nudge angular velocity (`AIR=0.004`,
   capped at `SPIN_CAP=0.26`) so the player levels out to nose into standing cans.
   **`airborne` requires BOTH** `y < GROUND_Y-55` **AND** `Query.collides(...)` empty —
   contact flickers on a resting sled, so "not touching" alone would let it pump endless
   ground cartwheels. Engine effectively idles in the air; controls stop on touchdown
   (`sndThud()` on landing).
6. **Scoring (`checkCans`):** a can counts as knocked when `angleOff(angle) > 0.5` rad OR it
   moved `> 30` px from its spawn (`can.ox/oy`). All cans down ⇒ **arm `winPending`** (do
   NOT celebrate yet). The loop then counts `winT` and lets the sled finish flying & settle;
   once it's slow/landed (or `winT > 1800` ms, min 700 ms) it calls **`startCelebration()`**
   — without that beat Larry teleported off the sled the instant the last can fell, with no
   transition. `startCelebration()` freezes Larry upright (`drawStandingLarry` arms-up cheer
   — **not** a dab; every dab attempt was rejected), draws a **Kenny-from-South-Park crowd**
   (`drawCrowd`/`drawKenny`, orange parkas, hoods, eyes-only) cheering around him, fires
   confetti + air-horn + `sndCrowd()` roar, then advances after a timeout. `maybeRespawn` is
   suppressed during `winPending` so the sled isn't teleported back to the start.
7. **Respawn (`maybeRespawn`):** off-world or settled ⇒ if `shots >= MAX_TRIES` (5) and the
   level isn't cleared, call **`startGameOver()`** instead of respawning; otherwise remove the
   sled, rebuild a fresh static one at ANCHOR, back to `state='aim'`.
8. **GAME OVER (`startGameOver`)** — the mirror of the celebration. Freezes the sled flipped on
   its back in front of the towers, shows the `#win` overlay with a `.over` class (red wash) +
   "GAME OVER 💀" and a random `FAIL_SUBS` line, plays `sndFail()` (sad trombone), and draws
   `drawWreck` (Larry sprawled, dazed — X-eyes/stars/spilled beer) with `drawCrowd(..., false)`
   (Kennys arms-down, disappointed — `drawKenny`'s `cheer=false` mode). Auto-resets to level 1
   after ~5.2 s. `physicsStep` early-returns while `gameOver` so the scene holds still.

## Key constants (in the `// Tunables` / viewport block near the top)

| Const | Value | Meaning |
|---|---|---|
| `VW, VH` | 1280×720 | virtual render size (letterboxed to the real canvas) |
| `WORLD_W` | 4800 | level width (long run-out for the side-scroll) |
| `GROUND_Y` | VH−70 | ground line |
| `ANCHOR` | x=200 | sled start — long runway before the kicker |
| `RAMP` | x0=1020, R=400, θ=0.73 | kicker arc (~42°, lip ~9 ft up at x≈1287); **big R = long, gentle curve → smooth glide up** (built from `N=30` overlapping segments). Bigger R also lengthens range, so it's paired with the `CANS_X` below. |
| `LEVEL_X[]` | all 3050 | **per-level** ramp→structure distance (`canX()`). All levels share one cluster centre in the reachable band; towers spread ±~225 around it. |
| `LAUNCH_EFF` | 0.72 | measured ramp speed loss; used for the trajectory preview |
| `POWER` | 0.05 | dragDist → launch speed |
| `MAX_DRAG` | 390 | max pull-back — full send ≈ **71 mph** |
| `STONE_H` | 56 | immovable stone **footing** height under each tower (anti-cheat: can't win by plowing the base) |
| `PH` | 92 | plank/floor height of a tower cell |
| sled `density` | 0.006 | moderate mass — topples the tower it hits **without bulldozing the whole row** (was 0.010) |
| `engine.gravity.y` | 0.364 | ≈ real g at the chosen scale (set in `buildLevel`) |

**Scale & calibration (don't guess — these were measured):** 11.2 px/ft (sled ≈ 10 ft = 112
px). At gravity.y=1.0 Matter's measured step values are gravity ≈0.275 px/step² and air drag
≈0.99944/step (velocities are px/step at 60fps). 60 mph ≈ 16.4 px/step.

**Design intent — it's ANGRY BIRDS (the user's explicit target, with reference images).**
Levels are **DYNAMIC stacked-plank towers that topple & scatter when hit**, beers (the "pigs")
perched on roofs and boxed inside cells. A shot knocks over the tower it strikes and scatters
*its* beers but leaves the others standing, so you **work the level over ~3–4 shots** — the
whole level must NOT clear in one hit (that read as a "stupid game"). Several towers (varied
heights, some joined by plank `deck`s) spread across each level; counts ~3–14 beers (L1→L6).
**By design a few beers can only be reached by air-control STEERING, not a straight shot — the
user wants that; it's the skill, don't "fix" it.** Heights ~38–57 ft (a tower past ~8 floors
self-collapses on spawn — keep them moderate). Sled mass was lowered to 0.006 so it topples one
tower rather than bulldozing the row. If you retune, **re-verify** (see Testing): stable at
spawn (0 self-knock), one full shot only PARTIALLY clears multi-tower levels, and every level
is winnable. ⚠️ The ramp is ~42°; **don't flatten it** — 42° is near the 45° max-range angle.

> ⚠️ **History (don't repeat these dead ends):** STATIC scaffolding → beers nested inside get
> shielded and become unreachable. Linked/tippy dynamic towers → cascade-clear in one hit
> (trivial) or self-collapse on spawn past ~8 tiers. Surreal tall leaning "noodle" spires were
> a long detour — the user landed on plain-ish Angry-Birds toppling. Several builder helpers are
> now **dead code** (`abTower`, `wideTower`, `crown`, `span`, `seussTower`, `bulb`, `spire`,
> `link`) — only `tower` + `deck` are used.

> 🔴 **CRITICAL — physics is FIXED-TIMESTEP, do not regress this.** Matter velocities and all
> tuned constants are *per step* (60/s). The loop must advance the sim at a constant rate
> regardless of the monitor's refresh, via an accumulator (`STEP=1000/60`, `acc += frame`,
> `while(acc>=STEP) physicsStep()`). The original code did `Engine.update(engine, rawFrameDt)`,
> which made the sled fly ~**4× as far on a 240 Hz display** (and short on a slow one) —
> straight over the towers. NEVER pass raw frame dt to `Engine.update`; always use `STEP`.
> All gameplay logic that runs per physics step (air control, backflip boost, `checkCans`,
> `maybeRespawn`, `winT`) lives in `physicsStep()`; only rendering + time-based cosmetics
> (confetti, `celebrateT`) run per render frame.

## Structures (the `// Level structures` block)

Builder helpers take `B=addBlock(x,y,w,h,ang,opts)` and `C=addCan(x,baseY)`. `addBlock` honors
`opts.mat==='stone'` **or** `opts.fixed` → a STATIC block; `post`/`beam` take an optional opts
arg to pass that through (currently only the stone footing is static).

**Active builders (only these two are used):**
- **`tower(B,C,X,G,floors,hw)`** — a DYNAMIC Angry-Birds tower: a short immovable **stone
  footing** (anti-cheat) + `floors` stacked open "cells" (two dynamic planks at ±`hw` + a board
  on top). A beer is boxed inside every other floor + one on the roof. Returns top y. Topples &
  scatters when hit. Keep `floors ≤ ~6` and `hw ≈ 38–40` (taller/skinnier self-collapses on spawn).
- **`deck(B,C,x1,x2,y,n)`** — a plank platform bridging two tower tops with `n` beers on it.
- `post`/`stonePost`/`beam` are the primitives.
- `LEVELS[]` — `LEVELS[level](addBlock, addCan, G, ch, canX())`. Current: spread towers of
  varied height ± decks — L1 one `tower(3)`; L2 two; L3 two + a `deck`; L4 three (tall middle);
  L5 two decked + a stub; L6 three + two decks. Counts 3,6,8,10,11,14 beers; heights ~38–57 ft.
- **Dead code (unused, safe to delete):** `abTower`, `wideTower`, `crown`, `span`, `seussTower`,
  `bulb`, `spire`, `link`, `boardPath`'s Seuss origins, `FX`.

**Verify after any structure change** (drive `physicsStep()` in a tight loop): (1) **stable at
spawn** — settle ~600 idle steps, 0 self-knock and block drift <~25 px; (2) **not a 1-hit
clear** — a single full send only PARTIALLY clears a multi-tower level; (3) **winnable** — every
beer reachable by some power *or* air-control steer (a few REQUIRING a steer is intended). Note:
losing 5 launches = GAME OVER → restart at level 1, so don't make a level unwinnable.

**Two hard rules (each came from a real failed build — keep them):**

1. **Targets must be elevated, never on the ground.** Ground cans can't be toppled ⇒ the
   level is unwinnable. Cans only ever sit on beams.
2. **The footing is static stone.** Anti-cheat: if the legs were destructible wood, a
   ground-scrape would topple everything and auto-win. Stone forces the player to hit the
   cans up the tower. Renders as grey coursed-block columns (wider than sticks) in
   `drawBlock` via `b.mat === 'stone'`. Keep it short (`STONE_H`) so it's just a footing, not
   most of the tower — otherwise the low cans get unreachable.

Block physics: `friction 0.95`, `frictionStatic 6`, `restitution 0`. Cans: `density 0.0011`,
`friction 0.9`. Thin leaning "house of cards" pieces collapse on spawn — keep supports
vertical and beams resting flat.

## Rendering notes

- **Letterbox, never stretch.** `resize()` computes a uniform `view.scale` + centered
  `offX/offY` so geometry never distorts. `toWorld()` is the exact inverse (letterbox +
  camera) so drags map correctly — if you change the transform, change `toWorld` to match.
- **Camera:** `cam.x/cam.y` lerp-follow the sled; aim state allows negative `cam.x` for
  pull-back room. The main `loop()` draws screen-space sky/mountains/snow, then a
  camera-translated world layer (ground, trees, ramp, blocks, cans, sled/Larry, particles).
- **Larry art:** `drawSnowmobile`, `drawRider`/`drawLarryHead` (seated), `drawStandingLarry`
  (celebration). `drawDabHead` exists but is unused (dab was rejected).
- **Crowd art:** `drawKenny` (one orange-parka hooded spectator, eyes-only, mittens raised,
  jumping) and `drawCrowd(centerX, groundY, t)` (a row flanking Larry, gap in the middle).
  Drawn in the world layer *before* `drawStandingLarry` so Larry is in front.

## Testing (how the physics was tuned — reproduce before shipping physics changes)

Use the Playwright MCP against the local server. Game globals (`engine, sled, cans, blocks,
buildLevel, checkCans, LEVELS, level`, constants) are accessible from the page. Run
**deterministic** loops — `Engine.update(engine, 1000/60)` in a `for` loop, NOT the RAF loop:

- **Winnability / cheat scan:** `buildLevel()`, set the sled's launch velocity for a power
  level, step ~800 updates calling `checkCans()` each step, read `score`. Sweep power across
  the range and confirm a hard send does big damage AND that a low/flat ground-scrape scores 0.
- **Stability:** build a level, settle ~400 steps, confirm blocks moved < ~25 px (no
  spontaneous collapse).
- **Framed screenshots:** teleport/park the sled near what you want to see so the
  follow-camera pans there, wait a beat, screenshot.

> ⚠️ **The deterministic ballistic sweep is NOT the real flight.** The actual `loop()` adds
> a one-time **backflip boost** (angular ±0.11) at liftoff and the player applies **air
> control**, so the real sled tumbles, flies *farther*, and its knock-count has run-to-run
> variance that a clean `setVelocity`+`Engine.update` sweep won't show. Use the ballistic
> sweep for coarse "does a band exist / can the base be cheated" checks, but **confirm the
> real feel through the live RAF loop**: `buildLevel()`, set the launch velocity, set
> `state='fly'`, `launchBoosted=false`, wait a few real seconds, read `score`. Don't force
> `keys.left/right` to fake steering — a constant hold spins the sled into the ground and
> reports false misses (this bit me; a *free* backflip scored 7/8 where forced spin scored 0).

## Things that were tried and don't work (don't redo them)

- **Loop-the-loop launcher:** impossible in 2D — entry/exit share one track, so the sled
  rams the lower lip or gets trapped spinning. Replaced by kicker + air control.
- **Dab celebration:** the rotated face always looks bad; rejected repeatedly. Use the
  arms-up cheer.
- **Continuous oscillating engine audio:** rejected; a one-shot rev on launch is what's used.
- **Ground-level cans / destructible base:** unwinnable / trivially won respectively (see
  the two hard rules above).
