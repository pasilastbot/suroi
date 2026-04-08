# Object Model — Hitbox Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/objects/README.md -->
<!-- @source: common/src/utils/hitbox.ts, common/src/utils/math.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents collision geometry: `Hitbox` types (Circle, Rect, Group, Polygon), intersection functions, and how hitboxes are used for collision detection and spatial queries.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/utils/hitbox.ts | `Hitbox`, `HitboxType`, `CircleHitbox`, `RectangleHitbox`, etc. | High |
| @file common/src/utils/math.ts | `Collision` — intersection responses | Medium |

## Hitbox Types

| Type | Shape | Fields |
|------|-------|--------|
| Circle | Circle | `position`, `radius` |
| Rect | Rectangle | `min`, `max` (corners) |
| Polygon | Convex polygon | `points`, `center` |
| Group | Multiple hitboxes | `hitboxes` (Circle or Rect) |

## Intersection

- **intersectionFunctions** — Lookup table for (typeA, typeB) → collision fn
- **Collision.circleCircleIntersection**, **rectCircleIntersection**, **rectRectIntersection**
- **CollisionResponse** — `intersected`, `overlap`, `direction`

## Usage

- **Server:** Bullet hit detection, loot pickup radius, obstacle collision
- **Client:** Visual hitbox debug (optional)
- **Definitions:** Obstacles, buildings, players define hitboxes
- **Grid:** `hitbox.toRectangle()` for spatial hash cell assignment

## Business Rules

- Hitboxes are defined in object definitions (e.g. obstacle.hitbox)
- `spawnHitbox` — Used before full hitbox (e.g. parachute)
- Group hitboxes — Used for complex shapes (e.g. buildings)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Object model overview
- **Tier 3:** [server-objects.md](server-objects.md) — Server objects use hitbox
- **Tier 2:** [../game-loop/](../../game-loop/) — Grid uses hitbox for cells
