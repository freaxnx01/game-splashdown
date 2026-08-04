# Per-Character Ramp Color Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the ramp obstacle render in a color matched to the active character (brown/wooden for Mokus, black for Tux, pink for Axolotl) instead of always being pink.

**Architecture:** All changes live in the single inline `<script>` block of `index.html` — no build step, no other JS files. A new `RAMP_COLOR_BY_CHAR` lookup (mirroring the existing `JUMP_SPLASH_COLOR_BY_CHAR`) supplies the per-character main color; `makeRamp()` stashes references to its plank and side meshes on `userData` so the spawn site can recolor a pooled instance on every spawn; the ramp-spawn block in `populateSegment()` sets plank/side colors from `charKind` each time a ramp is placed, the same per-spawn-recolor pattern the boost ring already uses.

**Tech Stack:** Vanilla JS, three.js r160 (loaded via CDN `<script src>`, global `THREE`). No bundler, no test framework.

## Global Constraints

- `RAMP_COLOR_BY_CHAR` values must be exactly `{squirrel:0x6b4423, penguin:0x000000, axolotl:0xf28fc0}` — identical to `JUMP_SPLASH_COLOR_BY_CHAR` (spec: "Colors").
- Side-mesh color is always derived as `new THREE.Color(mainColor).lerp(new THREE.Color(0x808080), 0.18)` — a fixed-percentage blend toward mid-gray, not a directional HSL lightness offset. **Revised 2026-08-04:** the original `offsetHSL(0, 0, -0.12)` formula was shipped in PR #6 and found broken in manual review — it clamps to a no-op on penguin's pure black (`0x000000`, lightness already 0), so the side panels came out identical to the plank. The gray-blend formula produces a visible tonal shift for every plank color, including black (spec: "Side-mesh shading").
- Ramp color must be (re-)applied every time a ramp spawns in `populateSegment()`, not only at mesh creation in `makeRamp()` — required to fix pool-staleness across character restarts (spec: "Implementation approach").
- No change to ramp physics, collision, sizing, or positioning (spec: "Non-goals").
- No change to `JUMP_SPLASH_COLOR_BY_CHAR` or any other existing per-character theming (spec: "Non-goals").
- No automated test suite exists in this repo (buildless single-file game). Each task is verified by a `node --check` syntax check plus a manual browser playthrough (spec: "Testing").

---

### Task 1: Add `RAMP_COLOR_BY_CHAR` and update `makeRamp()` to expose recolor targets

**Files:**
- Modify: `index.html:504` (add constant next to `JUMP_SPLASH_COLOR_BY_CHAR`)
- Modify: `index.html:212-220` (`makeRamp()`)

**Interfaces:**
- Consumes: `toon(color, opts={})` (`index.html:87`, unchanged) — builder for `MeshToonMaterial`.
- Produces: `RAMP_COLOR_BY_CHAR` (object, keyed `'squirrel' | 'penguin' | 'axolotl'` → hex number), consumed by Task 2. `makeRamp()` still returns a `THREE.Group`, now with `grp.userData.plank` (the plank `THREE.Mesh`) and `grp.userData.sides` (array of the two side `THREE.Mesh`es), consumed by Task 2.

- [ ] **Step 1: Add the `RAMP_COLOR_BY_CHAR` constant**

Find `JUMP_SPLASH_COLOR_BY_CHAR` at `index.html:504`:

```js
const JUMP_SPLASH_COLOR_BY_CHAR = {squirrel:0x6b4423, penguin:0x000000, axolotl:0xf28fc0};
```

Immediately after that line, add:

```js
const RAMP_COLOR_BY_CHAR = {squirrel:0x6b4423, penguin:0x000000, axolotl:0xf28fc0};
```

- [ ] **Step 2: Update `makeRamp()` to stash plank/side references**

Replace the existing `makeRamp()` at `index.html:212-220`:

```js
function makeRamp(){
  const grp = new THREE.Group();
  const geo = new THREE.BoxGeometry(5.5, 0.5, 5);
  const m = new THREE.Mesh(geo, toon(0xf7c8d8));
  m.rotation.x = -0.42; m.position.y = 1.0;
  const side1 = new THREE.Mesh(new THREE.BoxGeometry(0.4, 1.4, 5), toon(0xe8a8bf)); side1.position.set(-2.8, 0.8, 0); side1.rotation.x = -0.42;
  const side2 = side1.clone(); side2.position.x = 2.8;
  grp.add(m, side1, side2); return grp;
}
```

with:

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

(The `toon(0xf7c8d8)` / `toon(0xe8a8bf)` colors are now just initial placeholders — Task 2 overwrites them on every spawn, before the mesh is ever visible.)

- [ ] **Step 3: Syntax check**

Run: `node --check index.html`
Expected: no output, exit code 0.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(ramps): add RAMP_COLOR_BY_CHAR and expose ramp mesh refs for recoloring"
```

---

### Task 2: Apply per-character color at the ramp spawn site

**Files:**
- Modify: `index.html:320-328` (ramp block inside `populateSegment()`)

**Interfaces:**
- Consumes: `RAMP_COLOR_BY_CHAR` and `grp.userData.plank` / `grp.userData.sides` from Task 1; `charKind` (module-level string, set in `startGame()`, `index.html:577`, unchanged); `getItem(type)` (`index.html:228-234`, unchanged).
- Produces: no new interface — this is the final consumer in the chain.

- [ ] **Step 1: Update the ramp spawn block**

Replace the existing ramp block at `index.html:320-328`:

```js
  // ramp
  if(Math.random() < 0.45){
    const z = z0 + 15 + Math.random()*(SEG_LEN-30), l = (Math.random()*2-1)*5;
    const it = getItem('ramp');
    it.z = z; it.l = l; it.r = 3.0;
    it.mesh.position.set(centerX(z)+l, baseY(z)+WATER_Y-0.3, z);
    it.mesh.rotation.y = Math.atan2(centerDX(z), 1);
    seg.items.push(it);
  }
```

with:

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

- [ ] **Step 2: Syntax check**

Run: `node --check index.html`
Expected: no output, exit code 0.

- [ ] **Step 3: Manual verification — color per character**

Open `index.html` in a browser (e.g. `python3 -m http.server` from the repo root, then visit `http://localhost:8000/`). For each of the three characters:

1. Start a run as Mokus (squirrel). Play until a ramp spawns. Confirm the plank is brown/wooden (`#6b4423`) and the two side panels are a visibly distinct lighter brown (`#6f4f34`).
2. Return to the character select / restart, start a run as Tux (penguin). Confirm ramps are black, with visibly distinct dark-gray sides (`#171717`) — **specifically check this one**: the previous formula silently failed to show any plank/side contrast for penguin, so this is the regression case to verify closely.
3. Restart as Axolotl. Confirm ramps are pink (`#f28fc0`), matching the jump-splash pink, with visibly darker-pink sides (`#dd8cb4`).

Expected: each character's ramp color is visually distinct and matches the character, in all three runs — including the second and third runs, which reuse pooled ramp meshes from the first run (this is the pool-staleness check: if colors from an earlier character "stick," Task 1's `userData` wiring or Task 2's recolor call has a bug).

- [ ] **Step 4: Manual verification — no regressions**

During the same playthrough, confirm for each character:
- Jumping off a ramp still launches the character (physics/height unaffected).
- Ramp position/rotation on the water still looks correct (no shifted or misaligned ramps).
- No console errors in the browser devtools during ramp spawn or collision.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(ramps): color ramp per active character on spawn"
```
