# Definitions — Obstacles Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/obstacles.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

Obstacles are world objects: barrels, crates, doors, trees, etc. They can be destructible, have loot tables, support rotation modes, and control bullet/throwable collision behavior.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `obstacles.ts` | Obstacle definitions, materials, spawn modes | High |

## Business Rules

- `hardness` determines which melee can destroy (fists vs knife vs axe)
- `rotationMode`: Full, Limited (4 directions), Binary (2), None
- `lootTable` / `spawnWithLoot` for loot-dropping obstacles
- `allowFlyover` controls whether throwables can sail over
- `impenetrable` / `noBulletCollision` / `reflectBullets` for bullet behavior
- `interactObstacleIdString` for doors (linked obstacle when opened)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 2:** [../patterns.md](../patterns.md) — ReferenceOrRandom for loot tables
- **Tier 1:** [../../../datamodel.md](../../../datamodel.md) — RotationMode, FlyoverPref
