# Just Gonna Send It — maintainer's guide

An Angry-Birds-style 2D physics game: drag back to launch a snowmobile (ridden by
**Larry Enticer** — denim jacket, mullet, mustache, orange aviators, beer in hand) down a
runway, off a kicker ramp, into towers of beer cans. Land the arc on the elevated cans to
knock them all down and clear the level. 10 levels, targets spread near-to-far across the field.

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

0. **Camera & zoom (`camZoom`, `wideFrame`, `stepCamera`).** The world layer renders through a
   zoom centred on the viewport (`ctx.translate(VW/2)`→`scale(camZoom)`→`translate(-(cam.x+VW/2))`);
   `toWorld` inverts the same, so input maps correctly at any zoom. `computeWideFrame()` (run in
   `startIntroPan` per `buildLevel`) builds the **WIDE frame** = a zoomed-out view that fits the
   WHOLE fort field (full tower heights) on screen — `{x,y}` are view-centre minus `VW/2,VH/2`, `z`
   the zoom (<1 = out). Camera targets by state: **aim/celebrate/gameover** → `z=1` close on Larry;
   **in flight & not yet `landed`** → `z=1` **close-follow the sled** (run-up, launch & arc never go
   off-screen — vertical follow too); **once `landed`** (first touchdown after the flight = impact) →
   ease out to the **wide zoom** (`tz=wideFrame.z`, follow sled w/ look-ahead) to watch the forts
   topple, even two at once. So the dramatic zoom-out fires AFTER impact, not on launch (it felt
   jarring on launch, esp. mobile). `camZoom` lerps ×0.07. (Dead ends: zoom-out on launch = jarring;
   snapping to a fixed field centre = sled flew in from off-screen; z=1 follow-only = toppling forts
   fell off-screen. The `landed`-gated close→wide follow is the fix.)
   **Level intro:** opens held on the WIDE frame (whole field, tops visible — judge the pull), then
   over `INTRO_PAN` ms slowly eases the **zoom IN and pans LEFT to Larry** (`wideFrame`→`introTo`).
   Skippable: any pointer-down during `intro` ends it and the aim camera eases in (`startDrag`).
1. **Aim:** `state === 'aim'`, sled is `static` at `ANCHOR`. Mouse/touch drag sets
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
5. **Air control:** while `airborne` **AND not yet `landed`**, ◄►/A,D nudge angular velocity
   (`AIR=0.004`, capped at `SPIN_CAP=0.26`) so the player levels out to nose into standing cans.
   **`airborne` requires BOTH** `y < GROUND_Y-55` **AND** `Query.collides(...)` empty —
   contact flickers on a resting sled, so "not touching" alone would let it pump endless
   ground cartwheels. **`landed`** latches true on the first touchdown after the flight; from then
   on **controls are dead and the ground brake applies through little bounces too** (`velocity.x*=
   0.86`, `angVel*=0.80`). This is the fix for the "hold ◄/► to *tumble* a short jump forward into
   the first tower" cheat — same decisive stopping drag in front of the towers as behind them.
   `landed` resets on launch (`endDrag`) and `buildLevel`.
6. **Scoring (`checkCans`):** a can counts as knocked when it clearly TIPS OVER `angleOff(angle) >
   0.7` rad (~40°) OR is knocked `> 46` px off its spawn (`can.ox/oy`) — a graze that just wobbles it
   no longer pops (raise/lower these to taste; re-check coverage so none get stranded). All cans down
   ⇒ **arm `winPending`** (do
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
| `LEVEL_X[]` | per-level | `canX()`; **vestigial** — targets sit at absolute `SLOT` depths and the gameplay camera pulls out to the `wideFrame` (whole field) during flight, so this is only the ignored `X` arg to the level builders. Don't rely on it for framing. |
| `SLOT` | N1900 M12250 M22620 F12900 F23260 | **absolute target depths**, one per power band of the launch arc. Measured kill-powers (drag): N≈270, M1≈300, M2≈330, F1≈360, F2≈390. A shot tuned for one slot arcs 500px+ **over** the nearer ones — this is what makes each cluster its own aimed shot and kills the one-shot sweep. |
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
Levels are **DYNAMIC stacked-plank structures that topple & scatter when hit**, beers (the
"pigs") perched on roofs / boxed inside cells / lined up on bridges. You **work the level over
several shots** — the whole level must NOT clear in one hit (that read as a "stupid game").

**⭐ The anti-domino principle (2026-06-12 rewrite — the load-bearing idea):** the launch arc is
a fixed function of power, so *where each power lands low* is measurable and stable. Targets are
spread across **distinct DEPTH SLOTS** (`SLOT.N/M1/M2/F1/F2` = x 1900/2250/2620/2900/3260), each
owned by a different power band (drag ≈270/300/330/360/390). Because a shot tuned for one slot
arcs **500px+ over** the nearer ones, **one power can only clear one cluster — a single shot
physically cannot sweep the level.** This replaced the old "row of towers all at x≈2650 ±360"
design, which let one drag-330 shot bowl straight through the whole row (the user could one-shot
L8–L10). **Do NOT cluster targets back into one x-band**, and don't try to fix difficulty by
making towers heavier — spread them across depth instead. Keep slot structures **short** (beers
~70–280 px up) so neighbouring powers pass cleanly above them. Mix **`tower`** (vertical) and
**`bridge`** (horizontal girder span) shapes for variety so levels look distinct, not just
"taller each time." Counts ~3–14 beers (L1→L10).
**By design a few beers (later levels) can only be reached by air-control STEERING, not a
straight shot — the user wants that; it's the skill, don't "fix" it.** Heights ~38–57 ft (a
tower past ~8 floors self-collapses on spawn — keep them moderate). Sled mass 0.006 so it topples
one structure rather than bulldozing. If you retune, **re-verify** (see Testing): stable at
spawn (0 self-knock), **no single naive power clears a majority** (target <~60%), and every level
is winnable. The ramp is an **X-Games drop-in**: a long gentle straight approach (~3°) eases into
the arc and the ~42°/~15 ft kick, so the sled stays **planted** (no abrupt base that bounces/
flips the nose) and launches predictably. **Don't exceed ~45°** (steeper = launches straight up).
Ramp angle just splits launch speed up-vs-forward; place targets by `SLOT` depth (and tune reach with
horsepower `MAX_DRAG`×`POWER`), not by going vertical.
⚠️ **A ramp "throttle"** (auto-accelerating the sled up the ramp) was tried and *removed* — it made
launches erratic/non-monotonic and broke power-aiming. The gentle geometry alone keeps it planted.

> ⚠️ **History (don't repeat these dead ends):** STATIC scaffolding → beers nested inside get
> shielded and become unreachable. Linked/tippy dynamic towers → cascade-clear in one hit
> (trivial) or self-collapse on spawn past ~8 tiers. **Clustering all targets in one x-band → one
> power sweeps the lot (the user's "one-shot domino" complaint); spread across depth SLOTS
> instead.** Surreal tall leaning "noodle" spires were a long detour — the user landed on
> plain-ish Angry-Birds toppling. Several builder helpers are now **dead code** (`abTower`,
> `wideTower`, `crown`, `span`, `seussTower`, `bulb`, `spire`, `link`, `deck`, `stonePost`) — the
> live builders are `ziggurat` (big tiered forts), `tower` (short fillers), and `bridge`.

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

Builder helpers take `B=addBlock(x,y,w,h,ang,opts)` and `C=addCan(x,baseY)`. `addBlock` makes a
block STATIC when `FIXED_MATS.has(opts.mat)` (the base mats: stone/ice/brick/snow/timber) **or**
`opts.fixed`; everything else (wood, red `metal` girders) is dynamic.

Targets sit at absolute `SLOT` depths — the level lambdas **ignore their `X` arg** and call
builders with `SLOT.N/M1/M2/F1/F2` directly. `const S = SLOT` is aliased above `LEVELS` for brevity.

**Immovable BASES (anti-cheat footing, but NOT cookie-cutter):** `base(B,X,baseY,hw)` builds the
un-bulldozable footing under a fort and returns the y where the wood starts. It picks a **material**
(stone/ice/brick/snow/timber) and one of **five geometries** (plinth / tall stilts / step-pyramid /
chunky legs / arch-gateway) **deterministically from `X`** via `rnd(seed)` (a seeded sine hash) — so
each fort looks different but a level is identical every play (fair & verifiable). `BASE_MATS` +
`FIXED_MATS` list the mats; each renders distinctly in `drawBlock` (ice=glassy blue, brick=terracotta
running-bond, snow=rounded white, timber=vertical logs, stone=grey courses). ⚠️ **Base tops must stay
≲110 px** — taller lifts the lowest beers above the shot's kill-window and strands them (verify!).

**Active builders (`ziggurat`, `tower`, `bridge`):**
- **`ziggurat(B,C,X,G,tiers,metalBase=0)`** — the main BIG fort: a wedding-cake stack of tiers that
  **step in** as they rise (`tiers=[[floors,hw],…]` bottom→top, widest first → low CoG, stays up).
  A `base()` footing underneath; beers boxed inside floors (a **pair** side-by-side on wide tiers
  `hw≥70`, one centred on narrow ones) + a crown beer. `metalBase` = lowest floors with red metal
  walls. Presets: `Z_S/Z_M/Z_L/Z_XL` (4→7 floors, ~480→800 px). ⚠️ **Every tier `hw ≥ 38`** — the
  42-wide beers overlap narrower posts and the whole thing **explodes on spawn** (drift ~640).
- **`tower(B,C,X,G,floors,hw,metalWalls=0,ped=0)`** — the older plain vertical tower (a `base()` +
  stacked cells). Used for the few SHORT towers (e.g. an M1 filler that mustn't clip a neighbour's
  arc). `ped` is now vestigial (base height comes from `base()`).
- **`bridge(B,C,x1,x2,G,ped=70,n=3)`** — two piers (varied material, but **fixed height** = reach-
  critical) spanned by a red `METAL` girder deck with `n` beers in a row. Horizontal silhouette for
  variety; `ped` sets deck height.
- `deck` / `stonePost` still exist but are **unused** (bases go through `base()` now).
- `LEVELS[]` — big BUSY forts across **different SLOT depths + different shapes**. Tall forts go in
  **non-adjacent slots (N/M2/F2)** so the deep shots arc 650–850 px clear over the near ones (no
  one-shot sweep); M1/F1 hold low bridges or short towers. Counts ≈ 6,11,10,17,16,17,20,21,21,21.
- **Dead code (safe to delete):** `abTower`, `wideTower`, `crown`, `span`, `seussTower`, `bulb`,
  `spire`, `link`, `FX`, `deck`, `stonePost`.

**Verify after any structure change** (use the `__dbg` harness — see Testing): (1) **stable at
spawn** — `settle` ~600 idle steps, drift <~13 px (a big number like ~640 = a tower exploded, usually
a beer/post overlap or a too-aggressive tier step); (2) **no one-shot sweep** — `sweep` drag 150→390
on a *fresh build per shot*, best single naive shot <~60% (L1 warmup may be 100%); (3) **winnable** —
`coverage` (union across all powers ±air steer) == total; a few beers needing a steer is intended.
Losing 5 launches = GAME OVER → restart at level 1, so never strand a beer.

**Hard rules (each came from a real failed build — keep them):**

1. **Targets must be elevated, never on the ground.** Ground cans can't be toppled ⇒ unwinnable.
2. **The base is an immovable (static) material.** Anti-cheat: destructible legs would let a
   ground-scrape topple everything and auto-win. The `FIXED_MATS` (stone/ice/brick/snow/timber) are
   all static. Keep base tops ≲110 px so the low cans stay reachable.
3. **Every tier `hw ≥ 38` and base tops ≲110 px.** Overlapping bodies explode on spawn; tall bases
   strand low beers. Both are caught by the `settle`/`coverage` checks.

Block physics: `friction 0.95`, `frictionStatic 6`, `restitution 0`. Cans: `density 0.0011`,
`friction 0.9`. Thin leaning "house of cards" pieces collapse on spawn — keep supports
vertical and beams resting flat.

## Rendering notes

- **Art style = "modern/hip" procedural (2026-06-12).** All art is hand-drawn canvas vector (no
  image assets — we evaluated Kenney.nl CC0 sprites but they read dated/generic and clash with the
  custom Larry, so we went procedural). The look: a **dusk/synthwave palette** — `sky()` is an
  indigo→violet→rose→coral→peach gradient with a low sunset sun + bloom; `mountains()` are dusk-
  mauve/violet silhouettes with peach-lit snowcaps; `ground()` is dusk snow (warm crest → cool
  shadow). Forts use modern flat material colors + a **soft drop shadow** in `drawBlock` for depth;
  girders are coral-red. HUD is a **glassmorphism** card (`backdrop-filter: blur`, gradient title)
  with pill buttons; its font sizes/padding use `clamp(..,vh,..)` so it scales with viewport HEIGHT
  (the tight dimension in mobile landscape — keeps it a small corner chip on phones, full-size on
  desktop); letterbox/gate are deep indigo `#161033`. A light **screen-shake** (`shakeT`,
  bumped in `checkCans`, applied to the world-layer translate, decays ×0.82/frame) adds impact juice.
- **Juice (all procedural):** knocking a can fires `burst()` = dots + spinning **sparkle-stars** +
  an expanding **pop ring** (particles carry a `kind`: `dot`/`star`/`ring`, handled in `stepParticles`
  /`drawParticles`); the can gets a **squash-and-stretch** (`can.pop`, set on knock, decayed in
  `drawCan`); and LIVE (un-knocked) cans carry a soft **golden glow** so the targets pop. Tune these
  in `burst`/`drawParticles`/`drawCan` — purely cosmetic, no gameplay effect.
- **Letterbox, never stretch.** `resize()` computes a uniform `view.scale` + centered
  `offX/offY` so geometry never distorts. `toWorld()` is the exact inverse (letterbox + camera +
  **`camZoom`**) so drags map correctly at any zoom — if you change the world transform, change
  `toWorld` to match (both scale around the viewport centre).
- **Camera:** see the Camera & zoom note in the launch sequence. `cam.x/cam.y` lerp toward a
  state-based target and `camZoom` toward a target zoom (1 = close on Larry to aim, `wideFrame.z`
  < 1 = pulled out to the whole field while flying). The main `loop()` draws screen-space
  sky/mountains/snow, then the zoomed world layer (ground, trees, ramp, blocks, cans, Larry, FX).
- **Larry art:** `drawSnowmobile`, `drawRider`/`drawLarryHead` (seated), `drawStandingLarry`
  (celebration). `drawDabHead` exists but is unused (dab was rejected).
- **Crowd art:** `drawKenny` (one orange-parka hooded spectator, eyes-only, mittens raised,
  jumping) and `drawCrowd(centerX, groundY, t)` (a row flanking Larry, gap in the middle).
  Drawn in the world layer *before* `drawStandingLarry` so Larry is in front.

## Testing — the headless `__dbg` harness (copy-paste this)

Use the Playwright MCP against the local server (`python3 -m http.server 8731`). Two gotchas:
⚠️ **the game is a plain `<script>` (not a module), so its globals are NOT on `window`** — you
can't poke `sled`/`cans`/`buildLevel` from `browser_evaluate` directly; and the dev server
**caches `index.html`**, so always reload with a **cache-buster** (`index.html?v=2`, bump it every
edit) or your change won't load.

**The harness:** paste this block at the very end of the `<script>` (right after the final
`requestAnimationFrame(loop);`). It closes over the live globals. **Remove it before shipping** —
this is the canonical copy so future agents can paste it back when retuning. It drives the sim with
`physicsStep()` (the real fixed-timestep step, incl. the launch backflip boost), NOT the RAF loop.

```js
// ---- TEMP debug harness for headless tuning (remove before ship) ----
window.__dbg = {
  // self-collapse: build a level, let it sit, report the worst can drift (≳13px ⇒ a tower exploded)
  settle(lvl, steps=600) {
    level = lvl; buildLevel(); intro=false;
    for (let i=0;i<steps;i++) Engine.update(engine, STEP);
    let max=0; for (const c of cans) max=Math.max(max, Math.hypot(c.position.x-c.ox, c.position.y-c.oy));
    return { lvl, selfKnockDrift: Math.round(max), totalCans: cans.length };
  },
  // fire ONE shot at a drag distance (air = 'L'|'R'|null to hold a steer); returns cans knocked
  shot(lvl, dragDist, air=null, maxSteps=2200) {
    level = lvl; buildLevel(); intro=false;
    const speed = Math.min(dragDist, MAX_DRAG) * POWER;
    Body.setStatic(sled, false); Body.setVelocity(sled,{x:speed,y:0}); Body.setAngularVelocity(sled,0);
    state='fly'; shots=1; restT=0; launchBoosted=false; landed=false;
    keys.left=air==='L'; keys.right=air==='R';
    let i=0; for (; i<maxSteps; i++){ physicsStep(); if (state==='aim'||celebrating||gameOver) break; }
    keys.left=keys.right=false;
    return { knocked: score, total: cans.length, kk: cans.map(c=>c.knocked) };
  },
  // sweep powers (fresh build each), report the BEST single naive shot ("can I one-shot it?")
  sweep(lvl, from=120, to=390, step=15) {
    const rows=[]; for (let d=from; d<=to; d+=step){ const r=this.shot(lvl,d); rows.push({d,k:r.knocked,t:r.total}); }
    const best = rows.reduce((a,b)=> b.k>a.k?b:a, rows[0]);
    return { lvl, total: rows[0].t, best, rows: rows.map(r=>`${r.d}:${r.k}`) };
  },
  // winnability: union of cans knockable across ALL powers and ±air steer; covered<total ⇒ stranded
  coverage(lvl) {
    const got=new Set(); let total=0;
    for (const air of [null,'L','R']) for (let d=240; d<=390; d+=15){
      const r=this.shot(lvl,d,air); total=r.total; r.kk.forEach((v,i)=>{ if(v) got.add(i); });
    }
    return { lvl, total, covered: got.size, missing: total-got.size };
  }
};
```

**The three checks to run for every level (the pass bar):**
1. `settle(lvl)` → `selfKnockDrift` **<~13 px** (big ⇒ a fort exploded: beer/post overlap `hw<38`,
   too-aggressive tier step, or a base block interpenetrating the wood).
2. `sweep(lvl)` → `best.k / total` **<~60%** (L1 warmup may be 100%) — no one-shot sweep.
3. `coverage(lvl)` → `covered === total` — every beer reachable by some power (±steer is fine).

Plus an **anti-cheat / tumble check** when touching the brake or air-control: fire SHORT drags
(e.g. 180–240) with `air:'R'` held and confirm the sled stops short of the near tower (`SLOT.N`)
and knocks ~0 — a short jump must not be steerable into the towers.

**Framed screenshots:** park the *static* sled at the framing spot and let the follow-camera settle
there: `level=L; buildLevel(); intro=false; state='aim'; Body.setStatic(sled,true);
Body.setPosition(sled,{x: center, y: camY + VH*0.62});` (the `+VH*0.62` makes the aim-camera settle
at world-y `camY`, so a negative `camY` raises the view to show tall fort-tops). Then screenshot.

> ⚠️ **`physicsStep()` in a loop ≈ the real flight** (it includes the liftoff backflip boost), but a
> *constantly* held `keys.left/right` spins the sled into the ground and under-reports — for the
> "is it winnable" union, OR across `null`/`L`/`R` rather than holding one the whole time, and treat
> "a few beers only reachable with a steer" as intended, not a bug.

## Things that were tried and don't work (don't redo them)

- **Loop-the-loop launcher:** impossible in 2D — entry/exit share one track, so the sled
  rams the lower lip or gets trapped spinning. Replaced by kicker + air control.
- **Dab celebration:** the rotated face always looks bad; rejected repeatedly. Use the
  arms-up cheer.
- **Continuous oscillating engine audio:** rejected; a one-shot rev on launch is what's used.
- **Ground-level cans / destructible base:** unwinnable / trivially won respectively (see
  the hard rules above).
- **Clustering all targets at one depth (`LEVEL_X`±360 row):** one drag-330 shot bowled through
  the whole row (the user's "one-shot domino"). Fixed by absolute depth `SLOT`s + non-adjacent tall
  forts. Don't reintroduce a single-depth row.
- **Stepped/`ziggurat` tiers that narrow too hard, or beers wider than the posts (`hw<38`):** bodies
  spawn overlapping and *explode* (drift ~640). Keep tiers `hw≥38` and steps gentle.
- **Tall pedestals/bases to "raise beers into the arc":** lifts the low beers ABOVE the shot's
  kill-window and strands them. Bases top out ≲110 px; reach comes from depth, not height.
- **"Tumble the sled forward on the ground" (hold ◄/► after a short landing):** the player could
  pump a short jump into the first tower. Fixed with the `landed` latch (controls die + brake holds
  through bounces after first touchdown). Don't let air control re-engage after landing.
