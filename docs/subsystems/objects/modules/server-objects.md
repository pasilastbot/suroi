# Object Model — Server Objects Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/objects/README.md -->
<!-- @source: server/src/objects/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

Server objects are the authoritative game state. They run game logic (movement, collision, damage), serialize to `UpdatePacket`, and interact with the game world (grid, other objects).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `gameObject.ts` | BaseGameObject, ObjectMapping, fullStream/partialStream | High |
| `player.ts` | Player state, inventory, input handling | High |
| `obstacle.ts` | Obstacle health, destruction, loot spawn | High |
| `loot.ts` | Loot physics, pickup | Medium |
| `building.ts` | Building parts, doors | Medium |
| `projectile.ts` | Thrown projectiles, trajectory | Medium |
| `bullet.ts` | Bullet trajectory, hit detection | High |

## Business Rules

- Each object has `fullAllocBytes` and `partialAllocBytes` for pre-allocated serialization buffers
- Objects register in `Game.grid` for spatial queries
- `damage()` flows through `DamageParams`; source can be player, gas, obstacle, etc.
- Dead objects are marked `dead = true` and may be removed from the grid

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Object model overview
- **Tier 2:** [../packets/](../packets/) — UpdatePacket serialization
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — Game loop, 40 TPS
