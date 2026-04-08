# Packets — MapPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/mapPacket.ts, server/src/map.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the map data packet: sent once after join, contains terrain, rivers, obstacles, buildings, and places. The client uses it to render the static world.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/mapPacket.ts | `MapPacket`, `MapData`, `MapObject` | High |
| @file server/src/map.ts | `GameMap` — builds MapPacket buffer | High |

## MapData Structure

| Field | Purpose |
|-------|---------|
| `seed` | Map generation seed |
| `width`, `height` | Map dimensions |
| `oceanSize`, `beachSize` | Margins |
| `rivers` | River definitions (width, points, isTrail) |
| `objects` | MapObject[] — obstacles, buildings, loot |
| `places` | Named locations (position, name) |

## MapObject

- `type` — ObjectCategory (Obstacle, Building, Loot, etc.)
- `position` — World position
- `definition` — Obstacle/Building/BuildingObstacle definition
- `rotation` — Orientation
- `orientation?` — Building orientation
- `scale?`, `variation?` — Optional overrides

## Serialization

- Rivers: width, points array, isTrail
- Obstacles: definition index, rotation (packed by RotationMode), variation
- Buildings: definition, subparts, orientation
- Optimizations: variation + rotation packed into uint8 where possible

## When Sent

- Once per player, after `JoinedPacket`, before game start
- Cached in `GameMap.buffer` — same map for all players in a game

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../map/](../../map/) — Map generation, MapPacket build
- **Tier 3:** [../map/modules/generation.md](../../map/modules/generation.md) — Generation flow
