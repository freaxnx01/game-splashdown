# Per-Character Collectibles, Jump Splash Color, Random Boost Ring Colors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give each of Splashdown's three characters their own collectible (hazelnut/herring/strawberry), a themed splash-particle color on jump landings, and make boost rings show a random color per spawn instead of a fixed teal.

**Architecture:** All changes live in the single inline `<script>` block of `index.html` (lines 65-828 as of this plan — no build step, no other JS files). Three new item-builder functions replace `makeBerry`; `populateSegment()` picks the active character's item type off a small lookup map instead of a hardcoded `'berry'` string; pickup and idle-animation type-checks are widened from a single string to a shared `Set`; the splash-particle pool is changed from one shared material to one clone per particle so different splashes can show different colors at once; and the boost-ring spawn site randomizes that ring's own material color (not the shared builder).

**Tech Stack:** Vanilla JS, three.js r160 (loaded via CDN `<script src>`, global `THREE`). No bundler, no test framework.

## Global Constraints

- Scoring stays identical: common item pickup = +10 (`score += 10; popup('+10'); sfx.berry();`), golden acorn pickup = +50 unchanged (spec: "Collectibles").
- Golden acorn (`makeAcorn`, type `'acorn'`) is not touched in any task — same shape, same universal 92%/8% spawn ratio regardless of character (spec: "Non-goals").
- No new sound effects — every common collectible reuses `sfx.berry()` (spec: "Non-goals").
- No new trail/streak effects while airborne — jump color only changes the existing landing `splash()` call (spec: "Non-goals").
- `charKind` values are exactly `'squirrel'` (Mokus/Hazel), `'penguin'` (Tux), `'axolotl'` (Mochi/Axolotl) — set once in `startGame(kind)` (index.html:554) and never changes mid-run.
- No automated test suite exists in this repo (buildless single-file game — see `CLAUDE.md`). Each task is verified by a `node --check` syntax check (objective, scriptable) plus a manual browser playthrough (spec: "Testing").

---

### Task 1: Per-character collectible meshes

**Files:**
- Modify: `index.html:179` (delete `makeBerry`)
- Modify: `index.html:207` (the `makers` map)

**Interfaces:**
- Produces: `makeHazelnut()`, `makeHerring()`, `makeStrawberry()` — each returns a `THREE.Object3D` (`Mesh` or `Group`), matching the existing `make*()` builder signature used by `getItem()`. Registered in `makers` under the keys `'hazelnut'`, `'herring'`, `'strawberry'`.
- Consumes: `toon(color, opts={})` (index.html:86, unchanged), used by every existing builder.

- [ ] **Step 1: Delete `makeBerry` and add the three new builders**

Delete line 179:

```js
function makeBerry(){ const m = new THREE.Mesh(new THREE.SphereGeometry(0.55, 12, 10), toon(0xf295ab)); return m; }
```

In its place, add:

```js
function makeHazelnut(){
  const m = new THREE.Mesh(new THREE.SphereGeometry(0.55, 12, 10), toon(0x8a5a34));
  m.scale.set(1, 1.15, 0.9); return m;
}
function makeHerring(){
  const grp = new THREE.Group();
  const body = new THREE.Mesh(new THREE.SphereGeometry(0.45, 12, 10), toon(0x9fb8c8));
  body.scale.set(1.6, 0.75, 0.75);
  const tail = new THREE.Mesh(new THREE.ConeGeometry(0.4, 0.6, 8), toon(0x7d97a8));
  tail.rotation.z = Math.PI/2; tail.position.x = -0.9;
  grp.add(body, tail); return grp;
}
function makeStrawberry(){
  const grp = new THREE.Group();
  const body = new THREE.Mesh(new THREE.SphereGeometry(0.5, 12, 10), toon(0xd9435a)); body.scale.set(1, 1.25, 1);
  const cap = new THREE.Mesh(new THREE.ConeGeometry(0.35, 0.35, 8), toon(0x5a9c4a)); cap.position.y = 0.62;
  grp.add(body, cap); return grp;
}
```

- [ ] **Step 2: Update the `makers` map**

Change line 207 from:

```js
const makers = {berry:makeBerry, acorn:makeAcorn, rock:makeRock, log:makeLog, ramp:makeRamp, boost:makeBoost};
```

to:

```js
const makers = {hazelnut:makeHazelnut, herring:makeHerring, strawberry:makeStrawberry, acorn:makeAcorn, rock:makeRock, log:makeLog, ramp:makeRamp, boost:makeBoost};
```

- [ ] **Step 3: Syntax-check the file**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
sed -n '65,828p' index.html > /tmp/splashdown-main.js
node --check /tmp/splashdown-main.js
```

Expected: no output, exit code 0. (These three builders aren't called anywhere yet — this step only confirms the JS still parses. Visual verification happens once Task 2 wires them into spawning.)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add per-character collectible meshes (hazelnut, herring, strawberry)"
```

---

### Task 2: Wire character-specific spawning, pickup, and idle animation

**Files:**
- Modify: `index.html:207` area (add two lookup constants right after the `makers` map)
- Modify: `index.html:283` (`populateSegment` common-item spawn)
- Modify: `index.html:730` (pickup handling)
- Modify: `index.html:805` (idle animation)

**Interfaces:**
- Consumes: `makers` keys `'hazelnut'`/`'herring'`/`'strawberry'` (Task 1), `charKind` global (`'squirrel'`|`'penguin'`|`'axolotl'`).
- Produces: `COMMON_ITEM_BY_CHAR` (object, `charKind` → item type string) and `COMMON_ITEM_TYPES` (`Set` of the three item type strings) — both used by the pickup and idle-animation edits below, and available for Task 3/4 if needed (they aren't).

- [ ] **Step 1: Add the two lookup constants**

Immediately after the `makers` line from Task 1 (index.html:207), add:

```js
const COMMON_ITEM_BY_CHAR = {squirrel:'hazelnut', penguin:'herring', axolotl:'strawberry'};
const COMMON_ITEM_TYPES = new Set(['hazelnut', 'herring', 'strawberry']);
```

- [ ] **Step 2: Pick the common item by character in `populateSegment`**

Change index.html:283 from:

```js
      const it = getItem(Math.random() < 0.08 ? 'acorn' : 'berry');
```

to:

```js
      const it = getItem(Math.random() < 0.08 ? 'acorn' : COMMON_ITEM_BY_CHAR[charKind]);
```

- [ ] **Step 3: Update pickup handling**

Change index.html:730 from:

```js
      if(it.type === 'berry' && !P.air){ it.taken = true; it.mesh.visible = false; score += 10; popup('+10'); sfx.berry(); }
```

to:

```js
      if(COMMON_ITEM_TYPES.has(it.type) && !P.air){ it.taken = true; it.mesh.visible = false; score += 10; popup('+10'); sfx.berry(); }
```

- [ ] **Step 4: Update idle animation**

Change index.html:805 from:

```js
      if(it.type === 'berry' || it.type === 'acorn'){ it.mesh.rotation.y += dt*2; it.mesh.position.y = surfaceY(it.z, it.l)+1.0 + Math.sin(animT*3 + it.z)*0.15; }
```

to:

```js
      if(COMMON_ITEM_TYPES.has(it.type) || it.type === 'acorn'){ it.mesh.rotation.y += dt*2; it.mesh.position.y = surfaceY(it.z, it.l)+1.0 + Math.sin(animT*3 + it.z)*0.15; }
```

- [ ] **Step 5: Syntax-check the file**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
sed -n '65,828p' index.html > /tmp/splashdown-main.js
node --check /tmp/splashdown-main.js
```

Expected: no output, exit code 0.

- [ ] **Step 6: Manual verification — play all three characters**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
python3 -m http.server 8000
```

Open `http://localhost:8000/index.html` in a browser. For each of the three character-select buttons (Hazel/Mokus, Tux, Mochi/Axolotl):

- Start a run and confirm the common collectible you see floating in clusters matches the character (brown hazelnut for Hazel, silvery fish for Tux, red/green strawberry for Mochi) — not a pink sphere.
- Ride into one and confirm the score increases by 10 and a `+10` popup appears.
- Confirm the item still gently bobs/rotates while un-collected (idle animation still runs).
- Confirm the rare golden acorn still occasionally appears and is unchanged (gold sphere+cap, `+50` popup, "Goldene Eichel" text) for all three characters.

Stop the server with Ctrl+C when done.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: spawn and collect character-specific items"
```

---

### Task 3: Jump splash color

**Files:**
- Modify: `index.html:476-481` (splash-particle pool construction)
- Modify: `index.html:483-491` (`splash()` function)
- Modify: `index.html:693` (jump-landing call site)

**Interfaces:**
- Consumes: `charKind` global.
- Produces: `splash(x, y, z, n, spd, color=0xd8f2fa)` — the `color` parameter is new and optional; every existing call site that doesn't pass it keeps today's behavior unchanged. `JUMP_SPLASH_COLOR_BY_CHAR` (object, `charKind` → hex number) is new and only consumed by the landing call site in this task.

- [ ] **Step 1: Give each pooled splash particle its own material**

The splash pool currently shares one `THREE.MeshBasicMaterial` instance across all 40 pooled particles (`m.material = splashMat` implicitly via the `Mesh` constructor). Setting a color on one particle would recolor every particle system-wide, since they all point at the same material object — so each particle needs its own clone before per-call coloring can work.

Change index.html:476-481 from:

```js
const splashMat = new THREE.MeshBasicMaterial({color:0xd8f2fa, transparent:true, opacity:0.85});
for(let i=0; i<40; i++){
  const m = new THREE.Mesh(new THREE.SphereGeometry(0.14, 6, 5), splashMat);
  m.visible = false; scene.add(m);
  splashes.push({m, vx:0, vy:0, vz:0, life:0});
}
```

to:

```js
const splashMat = new THREE.MeshBasicMaterial({color:0xd8f2fa, transparent:true, opacity:0.85});
for(let i=0; i<40; i++){
  const m = new THREE.Mesh(new THREE.SphereGeometry(0.14, 6, 5), splashMat.clone());
  m.visible = false; scene.add(m);
  splashes.push({m, vx:0, vy:0, vz:0, life:0});
}
```

- [ ] **Step 2: Add a `color` parameter to `splash()`**

Change index.html:483-491 from:

```js
function splash(x, y, z, n, spd){
  for(let i=0; i<n; i++){
    const s = splashes[splashIdx++ % splashes.length];
    s.m.visible = true; s.m.position.set(x, y, z);
    s.vx = (Math.random()-0.5)*spd; s.vy = Math.random()*spd*0.9 + 1; s.vz = (Math.random()-0.5)*spd;
    s.life = 0.5 + Math.random()*0.3;
    s.m.scale.setScalar(0.7 + Math.random()*0.8);
  }
}
```

to:

```js
function splash(x, y, z, n, spd, color=0xd8f2fa){
  for(let i=0; i<n; i++){
    const s = splashes[splashIdx++ % splashes.length];
    s.m.visible = true; s.m.position.set(x, y, z);
    s.m.material.color.set(color);
    s.vx = (Math.random()-0.5)*spd; s.vy = Math.random()*spd*0.9 + 1; s.vz = (Math.random()-0.5)*spd;
    s.life = 0.5 + Math.random()*0.3;
    s.m.scale.setScalar(0.7 + Math.random()*0.8);
  }
}
```

- [ ] **Step 3: Add the per-character color map and pass it at the landing call site**

Add, right after `let splashIdx = 0;` (index.html:482):

```js
const JUMP_SPLASH_COLOR_BY_CHAR = {squirrel:0x6b4423, penguin:0x000000, axolotl:0xf28fc0};
```

Change index.html:693 from:

```js
      splash(centerX(P.z)+P.l, P.y+0.2, P.z, 12, 5);
```

to:

```js
      splash(centerX(P.z)+P.l, P.y+0.2, P.z, 12, 5, JUMP_SPLASH_COLOR_BY_CHAR[charKind]);
```

Leave the other three `splash(...)` call sites (boost pickup ~index.html:732, obstacle hit ~index.html:743, paddling wake ~index.html:788) unchanged — they'll use the new default `0xd8f2fa`, identical to their current fixed color.

- [ ] **Step 4: Syntax-check the file**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
sed -n '65,828p' index.html > /tmp/splashdown-main.js
node --check /tmp/splashdown-main.js
```

Expected: no output, exit code 0.

- [ ] **Step 5: Manual verification — jump landing color per character**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
python3 -m http.server 8000
```

Open `http://localhost:8000/index.html`. For each character, start a run, ride onto a ramp (pink-topped box obstacle) or off a waterfall drop to get airborne, and watch the landing splash:

- Hazel/Mokus: splash particles are brown.
- Tux: splash particles are black.
- Mochi/Axolotl: splash particles are pink.

Also confirm the *other* splash triggers (riding through a boost ring, hitting a rock/log, paddling wake trail) still look the same light blue-white as before — only jump landings changed color.

Stop the server with Ctrl+C when done.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: tint jump-landing splash by character"
```

---

### Task 4: Random boost ring color

**Files:**
- Modify: `index.html:309-316` (boost-ring spawn block in `populateSegment`)

**Interfaces:**
- Consumes: `getItem('boost')` (unchanged, index.html:208-214).
- Produces: nothing consumed elsewhere — self-contained visual change.

- [ ] **Step 1: Randomize the ring's color at spawn time**

`getItem()` reuses pooled mesh instances, so the color must be set on the existing material each time a ring is placed — not baked into `makeBoost()`, which would only run once per pooled instance.

Change index.html:309-316 from:

```js
  // boost ring
  if(Math.random() < 0.3){
    const z = z0 + 10 + Math.random()*(SEG_LEN-20), l = (Math.random()*2-1)*6;
    const it = getItem('boost');
    it.z = z; it.l = l; it.r = 2.2;
    it.mesh.position.set(centerX(z)+l, baseY(z)+WATER_Y+0.1, z);
    seg.items.push(it);
  }
```

to:

```js
  // boost ring
  if(Math.random() < 0.3){
    const z = z0 + 10 + Math.random()*(SEG_LEN-20), l = (Math.random()*2-1)*6;
    const it = getItem('boost');
    it.mesh.material.color.setHSL(Math.random(), 0.6, 0.68);
    it.z = z; it.l = l; it.r = 2.2;
    it.mesh.position.set(centerX(z)+l, baseY(z)+WATER_Y+0.1, z);
    seg.items.push(it);
  }
```

(`0.6` saturation / `0.68` lightness match the current fixed teal `0x7fe0d0`'s own HSL — `H≈0.473, S≈0.61, L≈0.688` — so only the hue varies, keeping every ring's brightness consistent with the game's existing pastel-toon look.)

- [ ] **Step 2: Syntax-check the file**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
sed -n '65,828p' index.html > /tmp/splashdown-main.js
node --check /tmp/splashdown-main.js
```

Expected: no output, exit code 0.

- [ ] **Step 3: Manual verification — varied ring colors**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
python3 -m http.server 8000
```

Open `http://localhost:8000/index.html`, start a run with any character, and play long enough to pass several boost rings (they spawn ~30% of the time per segment). Confirm:

- Rings show varying colors across the run (not all the same fixed teal).
- Each individual ring is a single solid color (not multi-colored/gradient).
- Riding through a ring still grants the "Turbo!" speed boost and blue popup as before.

Stop the server with Ctrl+C when done.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: randomize boost ring color per spawn"
```

---

### Task 5: Full regression pass and close the loop

**Files:**
- None (no code changes — this task verifies the finished feature end-to-end).

- [ ] **Step 1: Play a full run as each character**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
python3 -m http.server 8000
```

Open `http://localhost:8000/index.html`. For Hazel/Mokus, Tux, and Mochi/Axolotl in turn, play a run of at least a minute and confirm together:

- Only that character's own themed collectible spawns (never another character's, never the old pink berry).
- Collecting it scores +10 with a `+10` popup.
- The golden acorn still spawns rarely, is unchanged, and scores +50.
- Jump landings splash the character's own color (brown / black / pink).
- Other splash effects (boost pickup, obstacle hit, paddling wake) are unaffected.
- Boost rings vary in color from ring to ring and still grant the speed boost.
- Tricks, scoring, obstacles, and waterfalls behave exactly as before (no regressions).

Stop the server with Ctrl+C when done.

- [ ] **Step 2: Push the branch**

```bash
cd ~/repos/github/freaxnx01/public/game-splashdown
git push origin main
```

- [ ] **Step 3: Close the loop with the issue**

```bash
gh issue comment 2 --repo freaxnx01/game-splashdown --body "Implemented via commits on main: per-character collectibles (hazelnut/herring/strawberry), jump-landing splash color per character, and randomized boost ring colors. Golden acorn and all other mechanics unchanged. Verified by playing a full run as each of the three characters."
gh issue close 2 --repo freaxnx01/game-splashdown
```
