# Game Loop — Grid Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/game-loop/README.md -->
<!-- @source: server/src/utils/grid.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the spatial hash grid used for collision detection and visibility queries. Objects are partitioned into cells by position or hitbox; the grid enables efficient range queries without checking every object.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/utils/grid.ts | `Grid` — spatial hash, ObjectPool | Medium |

## Architecture

- **Cell size:** 32 units (`GameConstants.gridSize`)
- **2D structure:** `_grid[width][height]` — Map<objectId, GameObject> per cell
- **ObjectPool:** `pool.getCategory(ObjectCategory.X)` — iterate objects by category
- **_objectsCells:** Tracks which cells each object occupies for fast removal

## Operations

| Method | Purpose |
|--------|---------|
| `addObject(object)` | Add to pool and grid |
| `removeObject(object)` | Remove from pool and grid |
| `updateObject(object)` | Recompute cells (e.g. after movement) |

## Cell Assignment

- **Point objects** (no hitbox): Single cell at `_roundToCells(position)`
- **Hitbox objects:** All cells intersecting hitbox bounding rect
- **SpawnHitbox:** Used for initial placement before full hitbox

## Business Rules

- Objects call `grid.updateObject(this)` when position changes
- `game.updateObjects = true` triggers visibility/culling updates
- Used for: bullet hit detection, loot pickup, visibility culling

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Game loop overview
- **Tier 3:** [tick-serialization.md](tick-serialization.md) — Tick order
- **Tier 2:** [../objects/](../../objects/) — GameObject, hitbox
