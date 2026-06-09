# 🛷 Just Gonna Send It

A goofy, physics-based Angry-Birds-style game: drag back and **send it** — launch a snowmobile
(ridden by a denim-jacketed, mullet-sporting, beer-clutching legend) down a runway, off a
kicker ramp, and straight through towering stacks of beer cans. Knock 'em all down to clear
the level. Land it, hop off, and let the crowd go nuts.

## ▶️ Play it now

**https://ross-ethridge.github.io/just-gonna-send-it/**

No install, no sign-up — it runs right in your browser (desktop or mobile).

## 🎮 How to play

1. **Drag back** from the snowmobile and **release** to launch it off the kicker. The farther
   you pull, the more power — pull back hard to *send it*. (You can't over-power it past the
   towers, so when in doubt, send it.)
2. A throttle gauge, an **mph readout**, and a dotted **flight-path preview** show where you'll
   land while you aim.
3. Once you're airborne, tap the arrows to **flip and level out** so you smash into the cans.
4. Knock down **every can** to clear the level. There are **6 levels**, each a bigger, taller,
   more ridiculous tower than the last.
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

Launches are unlimited — if you come up short, just send it again.

## 🛠️ Tech

- A single self-contained `index.html` — HTML5 Canvas for rendering, the
  [Matter.js](https://brm.io/matter-js/) engine for physics. No build step, no framework.
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
