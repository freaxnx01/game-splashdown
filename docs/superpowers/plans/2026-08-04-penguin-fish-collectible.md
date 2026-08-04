# Penguin Fish Collectible Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rework the herring/fish collectible model to look less awkward, and make the rare golden collectible themed per character (golden acorn for squirrel, golden fish for penguin, golden strawberry for axolotl) instead of a single universal golden acorn.

**Architecture:** All changes live in the single inline `<script>` block of `index.html` — no build step, no other JS files. `makeHerring()` is reworked in place with a slimmer body and a two-cone forked tail; two new builder functions (`makeGoldenFish()`, `makeGoldenStrawberry()`) reuse that same fish shape and the existing strawberry shape respectively, in gold colors. Three new per-character lookup tables (`GOLDEN_ITEM_BY_CHAR`, `GOLDEN_POPUP_BY_CHAR`, `GOLDEN_ITEM_TYPES`) drive the spawn site, pickup/scoring, and idle-animation logic, replacing the single hardcoded `'acorn'` type checks.

**Tech Stack:** Vanilla JS, three.js r160 (loaded via CDN `<script src>`, global `THREE`). No bundler, no test framework.

## Global Constraints

- Golden item mapping is exactly `{squirrel:'acorn', penguin:'goldenFish', axolotl:'goldenStrawberry'}` (spec: "Scope decision").
- Popup text is exactly `{squirrel:'Goldene Eichel +50', penguin:'Goldener Fisch +50', axolotl:'Goldene Erdbeere +50'}` (spec: "Popup text").
- Score (+50) and spawn ratio (8% golden / 92% common) are unchanged for all three characters (spec: "Non-goals").
- No new sound effects — every golden pickup reuses `sfx.acorn()` regardless of character (spec: "Non-goals").
- `makeAcorn()` and squirrel's golden-item shape/colors are untouched (spec: "Non-goals").
- The reworked herring/golden-fish shape keeps the same mesh count as today (body + 2 tail-cones vs. today's body + 1 tail-cone) — no fins, no eye detail (spec: "Herring redesign").
- No automated test suite exists in this repo (buildless single-file game). Each task is verified by a `node --check` syntax check plus a manual browser playthrough (spec: "Testing").

---

### Task 1: Rework herring shape and add golden-item builders + tables

**Files:**
- Modify: `index.html:184-191` (`makeHerring()`)
- Modify: `index.html:198-203` area — add `makeGoldenFish()` and `makeGoldenStrawberry()` after `makeAcorn()`
- Modify: `index.html:225` (the `makers` map)
- Modify: `index.html:226-227` (add new per-character tables next to `COMMON_ITEM_BY_CHAR` / `COMMON_ITEM_TYPES`)

**Interfaces:**
- Consumes: `toon(color, opts={})` (`index.html:87`, unchanged) — builder for `MeshToonMaterial`.
- Produces: `makeGoldenFish()` and `makeGoldenStrawberry()` — each returns a `THREE.Group`, matching the existing `make*()` builder signature used by `getItem()`. Registered in `makers` under keys `'goldenFish'` and `'goldenStrawberry'`. `GOLDEN_ITEM_BY_CHAR` (object, `'squirrel'|'penguin'|'axolotl'` → type-key string), `GOLDEN_POPUP_BY_CHAR` (object, same keys → popup string), `GOLDEN_ITEM_TYPES` (`Set` of `'acorn'|'goldenFish'|'goldenStrawberry'`) — all consumed by Task 2.

- [ ] **Step 1: Rework `makeHerring()`**

Replace the existing `makeHerring()` at `index.html:184-191`:

```js
function makeHerring(){
  const grp = new THREE.Group();
  const body = new THREE.Mesh(new THREE.SphereGeometry(0.45, 12, 10), toon(0x9fb8c8));
  body.scale.set(1.6, 0.75, 0.75);
  const tail = new THREE.Mesh(new THREE.ConeGeometry(0.4, 0.6, 8), toon(0x7d97a8));
  tail.rotation.z = Math.PI/2; tail.position.x = -0.9;
  grp.add(body, tail); return grp;
}
```

with:

```js
function makeHerring(){
  const grp = new THREE.Group();
  const body = new THREE.Mesh(new THREE.SphereGeometry(0.45, 12, 10), toon(0x9fb8c8));
  body.scale.set(1.9, 0.62, 0.62);
  const tailGeo = new THREE.ConeGeometry(0.32, 0.55, 8);
  const tailMat = toon(0x7d97a8);
  const tail1 = new THREE.Mesh(tailGeo, tailMat);
  tail1.rotation.z = Math.PI/2; tail1.rotation.y = 0.35; tail1.position.set(-0.95, 0, 0.15);
  const tail2 = new THREE.Mesh(tailGeo, tailMat);
  tail2.rotation.z = Math.PI/2; tail2.rotation.y = -0.35; tail2.position.set(-0.95, 0, -0.15);
  grp.add(body, tail1, tail2); return grp;
}
```

- [ ] **Step 2: Add `makeGoldenFish()` and `makeGoldenStrawberry()`**

Immediately after `makeAcorn()` (`index.html:198-203`), add:

```js
function makeGoldenFish(){
  const grp = new THREE.Group();
  const body = new THREE.Mesh(new THREE.SphereGeometry(0.45, 12, 10), toon(0xe8b64c));
  body.scale.set(1.9, 0.62, 0.62);
  const tailGeo = new THREE.ConeGeometry(0.32, 0.55, 8);
  const tailMat = toon(0xc08b3f);
  const tail1 = new THREE.Mesh(tailGeo, tailMat);
  tail1.rotation.z = Math.PI/2; tail1.rotation.y = 0.35; tail1.position.set(-0.95, 0, 0.15);
  const tail2 = new THREE.Mesh(tailGeo, tailMat);
  tail2.rotation.z = Math.PI/2; tail2.rotation.y = -0.35; tail2.position.set(-0.95, 0, -0.15);
  grp.add(body, tail1, tail2); return grp;
}
function makeGoldenStrawberry(){
  const grp = new THREE.Group();
  const body = new THREE.Mesh(new THREE.SphereGeometry(0.5, 12, 10), toon(0xe8b64c)); body.scale.set(1, 1.25, 1);
  const cap = new THREE.Mesh(new THREE.ConeGeometry(0.35, 0.35, 8), toon(0xc08b3f)); cap.position.y = 0.62;
  grp.add(body, cap); return grp;
}
```

- [ ] **Step 3: Update the `makers` map**

Change `index.html:225` from:

```js
const makers = {hazelnut:makeHazelnut, herring:makeHerring, strawberry:makeStrawberry, acorn:makeAcorn, rock:makeRock, log:makeLog, ramp:makeRamp, boost:makeBoost};
```

to:

```js
const makers = {hazelnut:makeHazelnut, herring:makeHerring, strawberry:makeStrawberry, acorn:makeAcorn, goldenFish:makeGoldenFish, goldenStrawberry:makeGoldenStrawberry, rock:makeRock, log:makeLog, ramp:makeRamp, boost:makeBoost};
```

- [ ] **Step 4: Add the golden per-character tables**

Immediately after `COMMON_ITEM_TYPES` (`index.html:227`):

```js
const COMMON_ITEM_TYPES = new Set(['hazelnut', 'herring', 'strawberry']);
```

add:

```js
const GOLDEN_ITEM_BY_CHAR = {squirrel:'acorn', penguin:'goldenFish', axolotl:'goldenStrawberry'};
const GOLDEN_POPUP_BY_CHAR = {squirrel:'Goldene Eichel +50', penguin:'Goldener Fisch +50', axolotl:'Goldene Erdbeere +50'};
const GOLDEN_ITEM_TYPES = new Set(['acorn', 'goldenFish', 'goldenStrawberry']);
```

- [ ] **Step 5: Syntax check**

Run: `node --check index.html`
Expected: no output, exit code 0.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(collectibles): rework herring shape, add golden fish/strawberry builders"
```

---

### Task 2: Wire golden items into spawn, pickup, and idle-animation logic

**Files:**
- Modify: `index.html:303` (spawn site inside `populateSegment()`)
- Modify: `index.html:753-754` (pickup/scoring)
- Modify: `index.html:828` (idle animation)

**Interfaces:**
- Consumes: `GOLDEN_ITEM_BY_CHAR`, `GOLDEN_POPUP_BY_CHAR`, `GOLDEN_ITEM_TYPES` from Task 1; `COMMON_ITEM_BY_CHAR`, `COMMON_ITEM_TYPES` (`index.html:226-227`, unchanged); `charKind` (module-level string, set in `startGame()`, `index.html:577`, unchanged); `getItem(type)` (`index.html:228-234`, unchanged); `sfx.acorn()`, `sfx.berry()` (unchanged); `popup(text, colorClass)` (unchanged).
- Produces: no new interface — this is the final consumer in the chain.

- [ ] **Step 1: Update the spawn site**

Change `index.html:303` from:

```js
      const it = getItem(Math.random() < 0.08 ? 'acorn' : COMMON_ITEM_BY_CHAR[charKind]);
```

to:

```js
      const it = getItem(Math.random() < 0.08 ? GOLDEN_ITEM_BY_CHAR[charKind] : COMMON_ITEM_BY_CHAR[charKind]);
```

- [ ] **Step 2: Update pickup/scoring**

Change `index.html:753-754` from:

```js
      if(COMMON_ITEM_TYPES.has(it.type) && !P.air){ it.taken = true; it.mesh.visible = false; score += 10; popup('+10'); sfx.berry(); }
      else if(it.type === 'acorn' && !P.air){ it.taken = true; it.mesh.visible = false; score += 50; popup('Goldene Eichel +50', 'gold'); sfx.acorn(); }
```

to:

```js
      if(COMMON_ITEM_TYPES.has(it.type) && !P.air){ it.taken = true; it.mesh.visible = false; score += 10; popup('+10'); sfx.berry(); }
      else if(GOLDEN_ITEM_TYPES.has(it.type) && !P.air){ it.taken = true; it.mesh.visible = false; score += 50; popup(GOLDEN_POPUP_BY_CHAR[charKind], 'gold'); sfx.acorn(); }
```

- [ ] **Step 3: Update idle animation**

Change `index.html:828` from:

```js
      if(COMMON_ITEM_TYPES.has(it.type) || it.type === 'acorn'){ it.mesh.rotation.y += dt*2; it.mesh.position.y = surfaceY(it.z, it.l)+1.0 + Math.sin(animT*3 + it.z)*0.15; }
```

to:

```js
      if(COMMON_ITEM_TYPES.has(it.type) || GOLDEN_ITEM_TYPES.has(it.type)){ it.mesh.rotation.y += dt*2; it.mesh.position.y = surfaceY(it.z, it.l)+1.0 + Math.sin(animT*3 + it.z)*0.15; }
```

- [ ] **Step 4: Syntax check**

Run: `node --check index.html`
Expected: no output, exit code 0.

- [ ] **Step 5: Manual verification — shapes and pickups per character**

Open `index.html` in a browser (e.g. `python3 -m http.server` from the repo root, then visit `http://localhost:8000/`).

1. Start a run as Mokus (squirrel). Play until a golden item spawns. Confirm it's still the golden acorn, unchanged in shape/color. Pick it up and confirm the popup reads "Goldene Eichel +50" and score increases by 50.
2. Restart as Tux (penguin). Confirm the regular fish collectible shows the new slimmer body with a visible forked tail (two tail lobes instead of one). Play until a golden item spawns — confirm it's a golden fish (same slim/forked shape, gold colors) and picking it up shows "Goldener Fisch +50" with +50 score.
3. Restart as Axolotl. Play until a golden item spawns — confirm it's a golden strawberry (same shape as the regular strawberry, gold colors) and picking it up shows "Goldene Erdbeere +50" with +50 score.

- [ ] **Step 6: Manual verification — no regressions**

During the same playthrough, confirm for each character:
- Regular (+10) collectible pickups still work and show "+10" with the existing sound.
- Idle bob/rotate animation still applies to both regular and golden items for all three characters.
- Golden vs. common item spawn frequency still feels roughly like the existing 8%/92% split (sanity check, not an exact measurement).
- No console errors in the browser devtools during collectible spawn or pickup.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(collectibles): theme golden collectible per character"
```
