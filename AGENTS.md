# Just Gonna Send It — maintainer's guide

An Angry-Birds-style 2D physics game: drag back to launch a snowmobile (ridden by
**Larry Enticer** — denim jacket, mullet, mustache, orange aviators, beer in hand) down a
runway, off a kicker ramp, into towers of beer cans. Land the arc on the elevated cans to
knock them all down and clear the level. 10 levels, escalating towers.

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
| `RAMP` | x0=720, A=0.05, La=150, R=600, θ=0.74 | **X-Games drop-in**: a long **gentle straight approach** (`La`≈13 ft at `A`≈3°) so the sled stays *planted*, then an eased circular arc up to the **~42° / ~15 ft lip**. The surface is a sampled polyline (`RAMP.pts`); segments are laid along it (no abrupt curved base to bounce/flip the nose). **Keep θ ≤ ~0.8 (~45°)** — steeper launches go mostly *up*. Reach ~2060–3160 ⇒ `LEVEL_X≈2650`. |
| `LEVEL_X[]` | all 2650 | `canX()`; the **centre of the reachable band** (giant ramp ⇒ reach ~2030–3160). Towers spread ±~360 so a near tower needs a soft pull and a far one a full send — each tower is its own aimed shot. |
| `LAUNCH_EFF` | 0.72 | measured ramp speed loss; used for the trajectory preview |
| `POWER` | 0.05 | dragDist → launch speed |
| `MAX_DRAG` | 390 | max pull-back — full send ≈ **71 mph** (reaches the far tower; near ones use less) |
| **launch power = horsepower** | `MAX_DRAG` × `POWER` | ⭐ **This is THE lever for launch power/reach.** To make up speed lost to a longer runway (or to shift reach), just turn up the sled's "horsepower" here — do NOT fiddle with ground friction / "ice" / momentum hacks to compensate. (User's call: horsepower affects power; don't compensate with friction.) |
| `STONE_H` | 56 | immovable stone **footing** height under each tower (anti-cheat: can't win by plowing the base) |
| `PH` | 92 | plank/floor height of a tower cell |
| sled `density` | 0.006 | moderate mass — topples the tower it hits **without bulldozing the row** |
| block `density` | 0.0018 wood / **0.0028 metal** | metal beams are heavier so they crash down on the beers below |
| **ground brake** | — | in `physicsStep`, once `launchBoosted` and the sled is back near the ground, `velocity.x *= 0.86` / `angularVelocity *= 0.85` so it doesn't skate forever (gated on `launchBoosted` so it NEVER brakes the runway launch). Sled `restitution: 0` (no bounce). |
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
is winnable. The ramp is an **X-Games drop-in**: a long gentle straight approach (~3°) eases into
the arc and the ~42°/~15 ft kick, so the sled stays **planted** (no abrupt base that bounces/
flips the nose) and launches predictably. **Don't exceed ~45°** (steeper = launches straight up).
Ramp angle just splits launch speed up-vs-forward; tune range with `LEVEL_X`, not by going vertical.
⚠️ **A ramp "throttle"** (auto-accelerating the sled up the ramp) was tried and *removed* — it made
launches erratic/non-monotonic and broke power-aiming. The gentle geometry alone keeps it planted.

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

**Active builder (`tower`):**
- **`tower(B,C,X,G,floors,hw,metalWalls=0)`** — a DYNAMIC Angry-Birds tower: a short immovable
  **stone footing** (anti-cheat) + `floors` stacked open "cells" (two dynamic planks at ±`hw` +
  a board). A beer is boxed inside every other floor + one on the roof; **every other ceiling is
  a heavy red `METAL` beam** (`{mat:'metal'}` → riveted red girder, falls & crushes the beer
  below). **`metalWalls`** = how many of the LOWEST floors have **metal posts (red walls)** → a
  reinforced, heavier, harder-to-topple tower (difficulty knob; later levels use 1–3). Keep
  `floors ≤ ~6`, `hw ≈ 38–40`. Verify metal towers stay topple-able (every beer still reachable).
- `deck(B,C,...)` exists (a metal girder bridging two towers) but is **currently unused** —
  decks connect towers so a hit cascades both, which fights the multi-shot design; avoid.
- `LEVELS[]` — **SEPARATE towers spread ±~360 across the reachable band** (no decks): L1 one
  tower; L2/L3 two (near + far); L4/L5/L6 three across the band; L7–L10 four towers (L7 intro,
  L8 taller, L9 heavy metal throughout, L10 max height + max reinforcement — the gauntlet).
  Counts 3–16 cans; heights ~38–57 ft.
  Spreading is what makes it multi-shot — a near tower needs a soft pull, a far one a full send,
  so a single shot only topples ~one tower (clustering them lets one shot sweep the lot).
- **Dead code (safe to delete):** `abTower`, `wideTower`, `crown`, `span`, `seussTower`, `bulb`,
  `spire`, `link`, `FX`, and `deck` (unused).

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
