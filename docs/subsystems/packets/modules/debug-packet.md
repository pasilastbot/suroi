# Packets — DebugPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/debugPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the debug packet: Server→Client, dev-only. Sends debug state (speed override, no-clip, invulnerable, spawn loot, dummy) for development and testing.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/debugPacket.ts | `DebugPacket`, `DebugPacket` interface | Medium |

## DebugPacket Structure

| Field | Purpose |
|-------|---------|
| `speed` | Movement speed override |
| `overrideZoom` | Use custom zoom |
| `zoom` | Zoom level (if overrideZoom) |
| `noClip` | Pass through obstacles |
| `invulnerable` | No damage |
| `spawnLootType?` | Loot to spawn on interact |
| `spawnDummy` | Spawn test dummy |
| `dummyVest?`, `dummyHelmet?` | Dummy armor |
| `layerOffset` | Layer visibility offset |

## Serialization

- `writeBooleanGroup` — 7 booleans (overrideZoom, noClip, invulnerable, spawnLoot, spawnDummy, dummyVest, dummyHelmet)
- `writeFloat32(speed)`, `writeInt8(layerOffset)`
- Conditional: zoom, spawnLootType (Loots), dummyVest/Helmet (Armors)

## When Sent

- Dev mode only; server sends when debug state changes
- Used for testing movement, damage, loot spawning

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 1:** [docs/development.md](../../../development.md) — Dev mode
