# Per-Character Ramp Color — Design

**Issue:** [game-splashdown#4](https://github.com/freaxnx01/game-splashdown/issues/4)
**Date:** 2026-08-04

## Problem

Ramps always render pink (`makeRamp()`, `index.html:212-220`) regardless of which
character is playing. Tester feedback wants the ramp themed per character: brown/wooden
for Mokus (squirrel), black for Tux (penguin).

## Goal

Ramp color matches the active character, consistent with the per-character jump-splash
theming already shipped in issue #2.

## Colors

Reuse the existing `JUMP_SPLASH_COLOR_BY_CHAR` values (`index.html:504`) as a new
`RAMP_COLOR_BY_CHAR` table, rather than inventing new hex values:

```js
const RAMP_COLOR_BY_CHAR = {squirrel:0x6b4423, penguin:0x000000, axolotl:0xf28fc0};
```

- Mokus (squirrel): `0x6b4423` — brown/wooden.
- Tux (penguin): `0x000000` — black.
- Axolotl: `0xf28fc0` — same pink used for its jump splash (not the current default
  ramp pink `0xf7c8d8`, which was an arbitrary placeholder, not a themed choice).

Rationale: one shared table means any future character only needs a single color entry
to cover both splash and ramp theming, instead of maintaining two parallel tables that
can drift out of sync.

## Side-mesh shading

**Revised 2026-08-04, post-review of PR #6:** the original `offsetHSL(0, 0, -0.12)`
approach is broken for penguin's black (`0x000000`). `THREE.Color.setHSL` clamps
lightness to `[0, 1]`, so `0 - 0.12` clamps straight back to `0` — the side panels come
out identical to the plank, silently failing to darken. This was caught in manual review
of the shipped PR, not before — the fix below replaces the lightness-offset approach
entirely, since no fixed *directional* offset (darker or lighter) can work at both ends
of the lightness range.

The ramp is a `THREE.Group` of 3 meshes: a main plank and two side meshes. Instead of a
directional HSL offset, blend the plank color a fixed 18% toward mid-gray
(`0x808080`). This gives a visible tonal shift for every plank color, including black
(which lightens toward dark gray) and light colors (which darken toward gray) — the
old "always darker" framing doesn't hold at the extremes, but "visibly distinct from
the plank" does, and that's the actual requirement:

```js
const sideColor = new THREE.Color(mainColor).lerp(new THREE.Color(0x808080), 0.18);
```

Verified output per character: squirrel `0x6b4423` → `0x6f4f34` (subtly lighter),
penguin `0x000000` → `0x171717` (dark gray, clearly distinct from pure black),
axolotl `0xf28fc0` → `0xdd8cb4` (darker pink) — all three now show visible plank/side
contrast, unlike the old formula's silent no-op on penguin.

## Implementation approach

**`makeRamp()` (`index.html:212-220`):** keep building the group exactly as today, but
stash references on `userData` so the spawn site can recolor it later without querying
`grp.children` by index:

```js
function makeRamp(){
  const grp = new THREE.Group();
  const geo = new THREE.BoxGeometry(5.5, 0.5, 5);
  const m = new THREE.Mesh(geo, toon(0xf7c8d8));
  m.rotation.x = -0.42; m.position.y = 1.0;
  const side1 = new THREE.Mesh(new THREE.BoxGeometry(0.4, 1.4, 5), toon(0xe8a8bf)); side1.position.set(-2.8, 0.8, 0); side1.rotation.x = -0.42;
  const side2 = side1.clone(); side2.position.x = 2.8;
  grp.add(m, side1, side2);
  grp.userData.plank = m;
  grp.userData.sides = [side1, side2];
  return grp;
}
```

The initial `toon(0xf7c8d8)` / `toon(0xe8a8bf)` colors baked in here are placeholders —
they're always overwritten at spawn time, before the mesh is ever visible.

**Ramp spawn site (`populateSegment()`, `index.html:320-328`):** apply the
character-specific color every time a ramp is spawned, the same pattern the boost ring
already uses for its per-spawn random color (`index.html:333`):

```js
// ramp
if(Math.random() < 0.45){
  const z = z0 + 15 + Math.random()*(SEG_LEN-30), l = (Math.random()*2-1)*5;
  const it = getItem('ramp');
  const mainColor = RAMP_COLOR_BY_CHAR[charKind];
  const sideColor = new THREE.Color(mainColor).lerp(new THREE.Color(0x808080), 0.18);
  it.mesh.userData.plank.material.color.set(mainColor);
  it.mesh.userData.sides.forEach(s => s.material.color.set(sideColor));
  it.z = z; it.l = l; it.r = 3.0;
  it.mesh.position.set(centerX(z)+l, baseY(z)+WATER_Y-0.3, z);
  it.mesh.rotation.y = Math.atan2(centerDX(z), 1);
  seg.items.push(it);
}
```

Applying color on every spawn (rather than once at mesh creation) fixes the
pool-staleness problem: `itemPools` (`index.html:179`) is never reset across
`startGame()` restarts, so a ramp mesh created during a squirrel run and reused after
restarting as penguin would otherwise keep showing the squirrel's brown. Re-applying
color at spawn time means the pooled mesh always reflects the currently active
character.

## Non-goals

- No change to ramp physics, collision detection, sizing, or positioning — this is a
  material-color-only change.
- No change to `JUMP_SPLASH_COLOR_BY_CHAR` or any other existing per-character theming.
- No new characters or new item types.

## Testing

No automated test suite exists in this repo (buildless single-file game). Verify with:
- `node --check index.html` (syntax).
- Manual playthrough as each of the three characters, confirming ramp plank + side
  colors match the character, and that switching characters between runs (page not
  reloaded) shows the new character's color, not a stale pooled color.
