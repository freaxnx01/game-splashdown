# Per-character collectibles, jump splash color, and randomized boost ring colors

Design for [issue #2](https://github.com/freaxnx01/game-splashdown/issues/2).

## Background

Splashdown currently has 3 playable characters (Mokus/Hazel the squirrel, Tux the
penguin, Mochi the axolotl), all of whom collect the same shared items: berries
(+10, pink sphere) and rare golden acorns (+50). Boost rings are a single fixed
teal color. There's no per-character visual differentiation for collectibles,
jumps, or boost rings.

## Goals

- Give each character its own exclusive collectible (common + golden rare
  variant), replacing the shared berry/acorn system, with unchanged scoring.
- Give each character a themed splash-particle color when landing a jump.
- Make boost rings visually varied instead of a single fixed color.

## Non-goals

- No new trail/streak effects while airborne.
- No changes to trick/combo scoring or physics.
- No new sound effects — reuse existing berry/acorn SFX cues per item type.

## Design

### Collectibles

Replace the shared `berry` / `acorn` item types with per-character item types.
Scoring stays the same as today: common item = +10, golden rare variant = +50.

| Character | Common item | Rare (golden) item |
|---|---|---|
| Mokus (Hazel) | Haselnuss (hazelnut) | Golden hazelnut |
| Tux | Hering (herring) | Golden herring |
| Axolotl (Mochi) | Erdbeere (strawberry) | Golden strawberry |

Each item gets its own small custom 3D mesh, following the existing pattern
where every item type (berry, acorn, rock, log, ramp, boost) has its own
`make*()` builder function rather than reusing one generic shape:

- Hazelnut: brown ellipsoid
- Herring: silver-blue elongated fish shape
- Strawberry: red teardrop shape with a green top

The golden rare variant of each item reuses the same geometry as its common
counterpart, with a gold material swap — mirroring how the existing golden
acorn is a gold-colored variant of the same shape family as the berry system
today.

Item spawn logic currently spawns `berry` (92%) / `acorn` (8%) regardless of
character. This changes to switch on the active character to choose which
common/rare item pair to place, keeping the same 92/8 common/rare spawn ratio
and the same scoring, so run balance and pacing don't change — only which
mesh and item name appear.

### Jump splash color

When a character lands a jump (returns from air after a ramp), the existing
splash particle burst (`splash()`) uses a per-character color instead of the
current fixed color:

- Mokus: brown
- Tux: black
- Axolotl: pink (not specified in the original notes; chosen to fit the
  axolotl's real-world coloring so all three characters are visually
  consistent)

This only affects the splash particle color on jump landings — it does not
add any new visual effect (no air trail, no other particle changes).

### Boost ring colors

Boost rings (`makeBoost`) currently render as `THREE.RingGeometry` with a
single fixed teal (`0x7fe0d0`). Each spawned boost ring instance instead picks
a random solid color from a small fixed palette, independent of which
character is playing — purely visual variety on the course, not tied to
character or run.

## Testing

This is a single-file, buildless Three.js game with no existing test suite
(consistent with the rest of the codebase — see `CLAUDE.md`). Verification is
manual: play through all three characters, confirm each collects only its own
themed item pair with correct scoring (+10 / +50), confirm jump-landing splash
color matches the active character, and confirm boost rings show varied
colors across a run.
