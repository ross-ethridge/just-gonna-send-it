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
| `CANS_X` | 2750 | tower center; at full pull the sled plows *through* it at ~27 ft (under the ~47 ft top), not over it |
| `LAUNCH_EFF` | 0.72 | measured ramp speed loss; used for the trajectory preview |
| `POWER` | 0.05 | dragDist → launch speed |
| `MAX_DRAG` | 333 | max pull-back — **capped so a full send tops out ~61 mph; you can't fly over the towers** |
| `STONE_H` | 56 | immovable stone footing height under each tower |
| `PH` | 92 | wood tier height (towers are intentionally big) |
| `engine.gravity.y` | 0.364 | ≈ real g at the chosen scale (set in `buildLevel`) |

**Scale & calibration (don't guess — these were measured):** 11.2 px/ft (sled ≈ 10 ft = 112
px). At gravity.y=1.0 Matter's measured step values are gravity ≈0.275 px/step² and air drag
≈0.99944/step (velocities are px/step at 60fps). 60 mph ≈ 16.4 px/step.

**Design intent (fun > precision):** towers are deliberately **big & exaggerated** (~47–65 ft,
10–33 cans) and launch speed is **capped** so the natural full-power "send it" lands in the
demolition zone — you physically can't overshoot. A full send knocks down most/all of a tower
at once (the crazy backflip adds variance; single-tower levels usually clear in one send,
multi-tower L3/L6 in two — launches are unlimited). Sweet spot ≈ **60–63 mph**; below ~58 mph
it falls short. If you retune `POWER`, `gravity.y`, `RAMP.theta`, `MAX_DRAG`, or `CANS_X`,
**re-verify** (see Testing) — they interact and it's easy to recreate the "flies over every
time" bug. ⚠️ The ramp is ~42°; **don't flatten it to widen the window** — 42° is near the
45° max-range angle, so flattening REDUCES range and the sled falls short of the towers.

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

Builder helpers take `B=addBlock(x,y,w,h,ang,opts)` and `C=addCan(x,baseY)`:

- `post()` — **wooden** stick support (dynamic).
- `stonePost()` — wider **static stone** pillar (`isStatic`, `{mat:'stone'}`).
- `beam()` — horizontal board, returns its top y.
- `abTower(...,tiers,topCans)` — a tower = a short **stone footing** (`STONE_H`, immovable)
  carrying `tiers` destructible **wood** tiers of [two posts + beam]. Cans ride the footing
  beam AND every wood platform, so targets span a tall range. `tiers` is the count of *wood*
  tiers above the footing.
- `span()` — a board bridging two bare tower tops, cans riding on it.
- `LEVELS[]` — one arrow fn per level composing the above; called as
  `LEVELS[level](addBlock, addCan, G, ch, CANS_X)`. Wood-tier counts: L1=4, L2=5, L3=2×4+span,
  L4=6, L5=2×5+span, L6=3×5+span → heights ~47–65 ft, 10–33 cans. Posts are 19 px wide for
  stability at these heights (tall thin stacks self-collapse on spawn). Verify settle <~25 px
  AND that nothing self-knocks during a 600-step idle (light cans on a tall tower can drift).

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
