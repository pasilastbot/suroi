# Object Model — Client Objects Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/objects/README.md -->
<!-- @source: client/src/scripts/objects/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

Client objects render game state received from the server. Each extends PixiJS `Container` and implements `updateFromData()` / `updateFull()` to apply `ObjectsNetData`. Position and rotation are interpolated for smooth visuals.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `gameObject.ts` | GameObject base, interpolation, container | High |
| `player.ts` | Player sprite, weapon, animations | High |
| `obstacle.ts` | Obstacle sprite, destruction animation | Medium |
| `loot.ts` | Loot sprite, pickup feedback | Medium |
| `building.ts` | Building parts, layers | Medium |

## Business Rules

- `updateContainerPosition()` / `updateContainerRotation()` lerp between old and new values using `Game.serverDt`
- `destroyed` objects are removed from the scene
- Sprites loaded via `loadSpritesheets`; paths in `client/public/img/game/`
- Each object has a `container` (PixiJS) added to the appropriate layer container

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Object model overview
- **Tier 2:** [../packets/](../packets/) — UpdatePacket, ObjectsNetData
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — Rendering pipeline
