# Per-character collectibles, jump splash color, and randomized boost ring colors

Design for [issue #2](https://github.com/freaxnx01/game-splashdown/issues/2).

## Background

Splashdown currently has 3 playable characters (Mokus/Hazel the squirrel, Tux
the penguin, Mochi the axolotl), all of whom collect the same shared items:
berries (+10, pink sphere) and a rare golden acorn (+50, "Goldene Eichel").
Boost rings are a single fixed teal color. There's no per-character visual
differentiation for collectibles, jumps, or boost rings.

## Goals

- Give each character its own exclusive common collectible, replacing the
  shared berry item, with unchanged scoring (+10).
- Give each character a themed splash-particle color on jump landings.
- Make boost rings visually varied instead of a single fixed color.

## Non-goals

- No new trail/streak effects while airborne — jump color only changes the
  existing landing splash particles.
- No changes to trick/combo scoring or physics.
- No new sound effects — reuse the existing berry SFX cue for every
  character's collectible.
- The rare golden bonus item ("Goldene Eichel", +50, 8% of spawns) stays
  exactly as it is today — a single universal golden acorn, unrelated to
  which character is playing. Out of scope.

## Design

### Collectibles

Replace the shared `berry` item with per-character item types. Scoring stays
identical to today: +10 per pickup, same 92%/8% split against the unchanged,
still-universal golden acorn.

| Character | Common item |
|---|---|
| Mokus (Hazel) | Haselnuss (hazelnut) |
| Tux | Hering (herring/fish) |
| Axolotl (Mochi) | Erdbeere (strawberry) |

Each item gets its own small custom 3D mesh, following the existing pattern
where every item type (berry, acorn, rock, log, ramp, boost) has its own
`make*()` builder function rather than reusing one generic shape:

- Hazelnut: brown ellipsoid.
- Herring: silver-blue elongated fish shape (its own color is silver-blue —
  "jump color" black, below, is unrelated to this item's color).
- Strawberry: red teardrop shape with a green top, echoing the existing
  acorn's body+cap construction.

`populateSegment()` currently spawns `berry` (92%) / `acorn` (8%) regardless
of character. This changes to pick the common item type from the active
`charKind` (hazelnut/herring/strawberry) while keeping the 92/8 ratio against
the unchanged universal acorn — so run balance and pacing don't change, only
which mesh appears as the common pickup.

Each per-character item type is registered under its own distinct item-pool
key (`hazelnut` / `herring` / `strawberry`, not a shared `berry` key). The
game already pools and reuses inactive meshes per type
(`itemPools[type]`); if all three skins shared the `berry` key, a mesh pooled
while playing as one character could resurface with the wrong skin after a
restart as a different character. Distinct keys avoid that entirely — no
pool-invalidation logic needed.

### Jump splash color

When a character lands a jump (returns from air after a ramp), the existing
splash particle burst (`splash()`) uses a per-character color instead of the
current fixed white-blue:

- Mokus: brown
- Tux: black
- Axolotl: pink (not specified in the original issue; chosen to match real
  axolotl coloring)

`splash()` gains a color parameter, sourced from a small `charKind → color`
map at the jump-landing call site. This only affects the splash particle
color on jump landings — no other visual effect changes (no air trail, no
popup-text color change, no change to the take-off splash or paddling wake
splashes, which keep their current default color).

### Boost ring colors

Boost rings (`makeBoost`) currently render as a `THREE.RingGeometry` with a
single fixed teal (`0x7fe0d0`). Each time a boost ring is placed in
`populateSegment()`, it instead gets a random hue via
`new THREE.Color().setHSL(Math.random(), S, L)`, with saturation/lightness
fixed to match the current teal's brightness — so colors vary freely while
staying consistent with the game's pastel-toon palette. This is applied at
spawn time (not baked into `makeBoost()`), so a pooled ring mesh gets a fresh
random color every time it's reused, independent of which character is
playing.

## Testing

This is a single-file, buildless Three.js game with no existing test suite
(consistent with the rest of the codebase — see `CLAUDE.md`). Verification is
manual: play through all three characters, confirm each collects only its
own themed item with correct scoring (+10), confirm the golden acorn is
unchanged for all three, confirm jump-landing splash color matches the
active character, and confirm boost rings show varied colors across a run.
