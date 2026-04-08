# Map — Terrain Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/map/README.md -->
<!-- @source: common/src/utils/terrain.ts, server/src/map.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents terrain types and floor definitions: `FloorNames`, `FloorTypes`, `Terrain`, `River`. Used for map generation (grass, water, beach) and player movement (speed multiplier, slippery).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/utils/terrain.ts | `FloorNames`, `FloorTypes`, `Terrain`, `River` | Medium |
| @file server/src/map.ts | Uses Terrain for map generation | High |

## FloorNames

`grass`, `stone`, `wood`, `log`, `sand`, `metal`, `carpet`, `water`, `ice`, `void`

## FloorDefinition

| Field | Purpose |
|-------|---------|
| `debugColor` | Debug visualization color |
| `speedMultiplier?` | Player movement modifier (e.g. water 0.72) |
| `overlay?` | Render as overlay (water) |
| `particles?` | Spawn particles (water) |
| `slippery?` | Ice — reduced friction |

## Terrain

- Represents map floor layout
- `getFloor(position)` — Returns floor type at position
- Used for: movement speed, river/ocean boundaries, map packet

## River

- River definitions for map generation
- Width, points, obstacles
- `replaceWaterBy` (modes) — e.g. winter replaces water with ice

## Business Rules

- Water, ice have `speedMultiplier`; ice has `slippery`
- Mode `replaceWaterBy` swaps floor type (e.g. FloorNames.Ice for winter)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Map overview
- **Tier 3:** [generation.md](generation.md) — Terrain in generation flow
- **Tier 2:** [../definitions/](../../definitions/) — Modes.replaceWaterBy
