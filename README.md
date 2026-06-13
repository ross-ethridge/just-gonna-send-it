# 🛷 Just Gonna Send It

A goofy, physics-based Angry-Birds-style game: drag back and **send it** — launch a snowmobile
(ridden by a denim-jacketed, mullet-sporting, beer-clutching legend) down a runway, off a
kicker ramp, and straight through towering stacks of beer cans. Knock 'em all down to clear
the level. Land it, hop off, and let the crowd go nuts.

## ▶️ Play it now

**https://ross-ethridge.github.io/just-gonna-send-it/**

No install, no sign-up — it runs right in your browser (desktop or mobile).

## 🎮 How to play

1. Each level opens with a quick **camera sweep** over the targets so you can size up the layout
   before you shoot.
2. **Drag back** from the snowmobile and **release** to launch it off the kicker. The farther
   you pull, the more power — but **power is your aim**: targets sit at different distances, so a
   soft pull drops onto a near one while a full send arcs clean *over* it to reach the far ones.
3. A throttle gauge, an **mph readout**, and a dotted **flight-path preview** show where you'll
   land while you aim — match the arc to the fort you want.
4. Once you're airborne, tap the arrows to **flip and level out** so you smash into the cans.
   On the tougher levels a few cans are tucked where only a well-steered mid-air nudge reaches them.
5. Knock down **every can** to clear the level. There are **10 levels** of big, rickety,
   multi-tiered beer forts — perched on ice, brick, stone, snow and timber bases — spread from near
   to far and mixed with horizontal **girder bridges**. Later levels pack a fort in every distance
   band, so you dial in a different shot for each.
5. Clear a level and the rider leaps off to celebrate while a crowd of orange-parka spectators
   cheers him on. 🍻

### Controls

| Action | Keys / Input |
|---|---|
| Aim & launch | **Click/tap, drag back, release** |
| Flip / level out mid-air | **◄ ►** or **A / D** |
| Retry the current level | **R** (or the **Retry level** button) |
| Mute / unmute sound | **M** |
| Restart from level 1 | **Reset game** button |

You get **5 launches per level** — knock down all the cans before you run out or the level fails.

## 🛠️ Tech

- A single self-contained `index.html` — HTML5 Canvas for rendering, the
  [Matter.js](https://brm.io/matter-js/) engine for physics. No build step, no framework.
- All art is **hand-drawn procedural vector** (no image assets) in a modern dusk/synthwave style —
  gradient sunset sky, glassmorphism HUD, soft shadows — so it stays crisp at any resolution.
- Matter.js is vendored locally (`matter.min.js`), so the game has **zero external
  dependencies** and works fully offline.
- All sound effects are synthesized at runtime with the Web Audio API (no audio files).
- Physics runs on a fixed timestep, so gameplay behaves identically regardless of your
  monitor's refresh rate.

## 💻 Run it locally

Clone the repo and serve the folder over HTTP (needed so the browser will load the script and
allow audio):

```bash
git clone https://github.com/ross-ethridge/just-gonna-send-it.git
cd just-gonna-send-it
python3 -m http.server 8731
```

Then open **http://localhost:8731/index.html**.

> Opening `index.html` directly via `file://` mostly works, but serving over HTTP avoids
> browser security quirks (and matches how it's hosted).

## 📦 Repo contents

- `index.html` — the entire game
- `matter.min.js` — vendored physics engine (Matter.js 0.20.0, MIT)
- `AGENTS.md` — developer/maintener notes (physics tuning, level structure, gotchas)
- `README.md` — this file

## 📄 License

Matter.js is © its authors under the MIT License. Do whatever you want with the game itself —
just gonna send it.
