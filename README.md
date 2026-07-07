# Splashdown! 🐿️🐧🌸

An endless waterslide racer inspired by the classic Tux Racer — race Hazel the squirrel, Tux the penguin, or Mochi the axolotl down a procedurally generated mountain river.

**▶ Play it here: `https://github.freaxnx01.ch/game-splashdown/`**

![Toon-shaded 3D, cute & cozy pastel style](https://img.shields.io/badge/style-toon--shaded%203D-f6aebf) ![Single file](https://img.shields.io/badge/build-none%20needed-8fd8e8) ![License](https://img.shields.io/badge/license-MIT-bfe3b2)

## Features

- **3 playable characters** on inner tubes: Hazel (Eichhörnchen), Tux (Pinguin), Mochi (Axolotl)
- **Endless procedural course** — winding river, banks, trees, ramps, boost rings, rocks & logs
- **Waterfalls** — the river plunges over 9 m drops; ride the lip to catch air
- **Air tricks** — 360° spins, front/back flips for combo bonus points; botch the landing and you wipe out
- **Collectibles** — berries (+10) and rare golden acorns (+50)
- **Toon-shaded 3D** with splash particles, chase cam with speed-reactive FOV
- **Synthesized sound effects** (Web Audio, no audio files)
- High score saved locally

## Controls

| Input | Action |
|---|---|
| A / D or ← → | Steer (spin tricks in air) |
| W / ↑ | Paddle (front flip in air) |
| S / ↓ | Brake (back flip in air) |
| Mouse | Steer |
| Space / P | Pause |
| Esc | Back to menu |
| M | Mute |

UI is in German.

## Running locally

Open `index.html` in any modern browser — that's it. (Three.js loads from CDN, so you need an internet connection.)

## Tech

Single HTML file, ~800 lines. Three.js (r160) toon materials, procedural terrain segments recycled in a ring buffer, custom physics (slope acceleration, drag, stun), Web Audio API for SFX. No build tools, no framework.

## License

MIT — see [LICENSE](LICENSE). Tux appears as an affectionate homage to the Linux mascot; this is an original game, not derived from Tux Racer code or assets.
