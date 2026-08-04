# Penguin Fish Collectible Rework — Design

**Issue:** [game-splashdown#5](https://github.com/freaxnx01/game-splashdown/issues/5)
**Date:** 2026-08-04

## Problem

Two related pieces of tester feedback:

1. The regular herring/fish collectible model (`makeHerring()`, `index.html:184-191` —
   a plain sphere body + single cone tail) looks awkward.
2. The rare golden collectible should be a "golden fish" for the penguin character,
   instead of the universal golden acorn.

Point 2 revisits an explicit prior decision from issue #2, which stated as a non-goal
that the golden acorn ("Goldene Eichel") stays "same shape, same universal availability
regardless of character." That decision is being deliberately revisited here — the golden
item becomes themed per character.

## Scope decision: golden item themed for all three characters

- **Squirrel:** golden acorn — unchanged, reuses existing `makeAcorn()`.
- **Penguin:** golden fish — new, reuses the reworked herring shape (below) in gold colors.
- **Axolotl:** golden strawberry — new, reuses the existing strawberry's body+cap structure
  (`makeStrawberry()`, `index.html:192-197`) in gold colors, matching the same
  "golden version of the character's regular collectible" pattern as the fish.

Score (+50) and spawn ratio (8% golden / 92% common) are unchanged for all three.

## Popup text

Per-character German flavor text, replacing the single hardcoded string at
`index.html:754`:

```js
const GOLDEN_POPUP_BY_CHAR = {
  squirrel: 'Goldene Eichel +50',
  penguin: 'Goldener Fisch +50',
  axolotl: 'Goldene Erdbeere +50',
};
```

## Herring redesign

Improved proportions + forked tail (no new fins/eye — keeps the mesh count the same:
body + 2 tail-cones, same as today's body + 1 tail-cone plus one extra cone).

- **Body:** slimmer, more tapered than today's `scale.set(1.6, 0.75, 0.75)` — tighten to
  a more torpedo-like proportion.
- **Tail:** forked, built from two cones angled outward (roughly ±20°) instead of one
  straight cone.

This shape is shared by both the regular herring (existing gray/blue colors,
`0x9fb8c8` body / `0x7d97a8` tail) and the new golden fish (gold colors, matching the
acorn's palette: `0xe8b64c` body / `0xc08b3f` tail), so the two read as clearly
related — a fish and a golden fish — rather than unrelated shapes.

## Implementation approach

**New per-character tables** (`index.html`, near `COMMON_ITEM_BY_CHAR` at line 226):

```js
const GOLDEN_ITEM_BY_CHAR = {squirrel:'acorn', penguin:'goldenFish', axolotl:'goldenStrawberry'};
const GOLDEN_POPUP_BY_CHAR = {squirrel:'Goldene Eichel +50', penguin:'Goldener Fisch +50', axolotl:'Goldene Erdbeere +50'};
const GOLDEN_ITEM_TYPES = new Set(['acorn', 'goldenFish', 'goldenStrawberry']);
```

**New builder functions**, alongside the existing `makeAcorn()` etc. (`index.html:198-203`):

- `makeHerring()` — reworked in place: slimmer body scale, forked tail (two cones).
- `makeGoldenFish()` — same shape as the reworked `makeHerring()`, gold colors.
- `makeGoldenStrawberry()` — same shape as `makeStrawberry()`, gold colors.

**`makers` map** (`index.html:225`) gains two entries:

```js
const makers = {hazelnut:makeHazelnut, herring:makeHerring, strawberry:makeStrawberry, acorn:makeAcorn, goldenFish:makeGoldenFish, goldenStrawberry:makeGoldenStrawberry, rock:makeRock, log:makeLog, ramp:makeRamp, boost:makeBoost};
```

**Spawn site** (`populateSegment()`, `index.html:303`) — golden branch becomes
character-aware, same 8%/92% split:

```js
const it = getItem(Math.random() < 0.08 ? GOLDEN_ITEM_BY_CHAR[charKind] : COMMON_ITEM_BY_CHAR[charKind]);
```

**Pickup/scoring** (`index.html:754`) — widen the type check from a single string to
the new Set, and use the per-character popup text:

```js
else if(GOLDEN_ITEM_TYPES.has(it.type) && !P.air){ it.taken = true; it.mesh.visible = false; score += 50; popup(GOLDEN_POPUP_BY_CHAR[charKind], 'gold'); sfx.acorn(); }
```

**Idle animation** (`index.html:828`) — widen the same way:

```js
if(COMMON_ITEM_TYPES.has(it.type) || GOLDEN_ITEM_TYPES.has(it.type)){ it.mesh.rotation.y += dt*2; it.mesh.position.y = surfaceY(it.z, it.l)+1.0 + Math.sin(animT*3 + it.z)*0.15; }
```

## Non-goals

- No new sound effects — every golden pickup reuses `sfx.acorn()`, regardless of
  character (consistent with issue #2's "common items all reuse `sfx.berry()`"
  precedent).
- No change to score (+50) or spawn ratio (8% golden / 92% common).
- No change to squirrel's golden acorn shape or colors — `makeAcorn()` is untouched.
- No change to `RAMP_COLOR_BY_CHAR`, `JUMP_SPLASH_COLOR_BY_CHAR`, or any ramp-color
  work from issue #4.
- No new trail/streak effects or new collectible categories beyond the golden variants
  described here.

## Testing

No automated test suite exists in this repo (buildless single-file game). Verify with:

- `node --check index.html` (syntax).
- Manual playthrough as each of the three characters:
  - Squirrel: confirm golden acorn pickup unchanged (shape, popup text, sound, +50).
  - Penguin: confirm regular herring shows the reworked slimmer/forked-tail shape;
    confirm the rare golden item is a golden fish with the same shape, showing
    "Goldener Fisch +50" and +50 score on pickup.
  - Axolotl: confirm the rare golden item is a golden strawberry, showing
    "Goldene Erdbeere +50" and +50 score on pickup.
  - For all three: confirm the idle bob/rotate animation still applies to golden items,
    and that spawn frequency of golden vs. common items still feels like ~8%/92%
    over a few minutes of play (no exact measurement expected, just a sanity check).
