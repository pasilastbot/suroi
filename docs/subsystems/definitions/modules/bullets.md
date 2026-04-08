# Definitions — Bullets Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/bullets.ts, common/src/utils/baseBullet.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents bullet definitions: derived from `Guns` and `Explosions`, ballistics (damage, speed, range, tracer), and how they are used in combat.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/definitions/bullets.ts | `Bullets` — derived from Guns + Explosions | Medium |
| @file common/src/utils/baseBullet.ts | `BaseBulletDefinition` — ballistics fields | High |

## Bullet Source

- **Guns:** Each gun (except dual variants) produces `{idString}_bullet` from `def.ballistics`
- **Explosions:** Each explosion produces `{idString}_bullet` from `def.ballistics`
- **Dual guns:** Excluded (use single variant's bullet)

## BaseBulletDefinition Fields

| Field | Purpose |
|-------|---------|
| `damage` | Base damage per hit |
| `obstacleMultiplier` | Damage multiplier vs obstacles |
| `speed` | Bullet velocity |
| `range` | Max travel distance |
| `rangeVariance?` | Random range variation |
| `shrapnel?` | Shrapnel type (uses shrapnel color) |
| `tracer?` | opacity, width, length, color, image |
| `trail?` | interval, frame, scale, alpha |

## Tracer Colors

- Default: derived from ammo type (9mm, 12g, 556mm, etc.) or `shrapnel`
- Override: `def.ballistics.tracer?.color`, `saturatedColor`

## Business Rules

- `Bullets` is built at module load; no runtime additions
- Order in `Bullets` affects serialization index

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 3:** [items.md](items.md) — Guns, ballistics source
- **Tier 2:** [../objects/](../../objects/) — Bullet server object
