# Feedback triage — 2026-08-04

status: approved

## Entries

| # | Note (normalized) | Att. | Topic | Kind | Disposition | Status |
|---|---|---|---|---|---|---|
| 1 | Ramp should be brown/wooden when playing squirrel | | Ramps | Improvement | Issue (combined with #2) | done |
| 2 | Ramp should be black when playing penguin | | Ramps | Improvement | Issue (combined with #1) | done |
| 3 | Penguin's fish collectible model looks awkward | | Collectibles | Improvement (bug) | Issue (combined with #4) | done |
| 4 | Golden item should be a "golden fish" for penguin | | Collectibles | New feature | Issue (combined with #3) | done |
| 5 | Schwimmreifen (water ring) should get random color every new game | | Boost rings | — | No action — already implemented | done |

## Rationale

- **#1 + #2** → filed as one issue: per-character ramp coloring. Ramp meshes are pooled
  (`itemPools`, never reset across restarts in `index.html`), so color must be applied
  per-spawn like the boost ring, not baked into the pooled mesh at creation — same
  non-trivial pattern as issue #2 (per-character collectibles).
- **#3 + #4** → filed as one issue: penguin fish collectible rework. #4 revisits an
  explicit prior decision from issue #2 ("golden acorn ... universal availability
  regardless of character" was a stated non-goal), so it needs discussion before
  implementation, not a blind fix.
- **#5** → already implemented in issue #2 (`index.html:333`,
  `it.mesh.material.color.setHSL(Math.random(), 0.6, 0.68)` on every ring spawn) — covers
  both within-game and new-game cases. No action needed.

## Raw notes

```
game splash down:
- Ramps playing squirrel should be of brown color/wooden        [#1]
- Ramps playing penguin should be of black color                [#2]
- penguin: the fish looks akward the golden one should be golden fish   [#3] [#4]
- schwimmreifen should get random color in every new game       [#5]
```
