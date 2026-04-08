# Packets — UpdatePacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/updatePacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

`UpdatePacket` is the primary server→client packet, sent every tick (40 TPS). It carries delta-compressed game state: players, obstacles, loot, buildings, decals, projectiles, particles, and killfeed.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `updatePacket.ts` | UpdatePacket serialize/deserialize, PlayerData, object splits | High |
| `objectsSerializations.ts` | Per-category full/partial serialization | High |

## Business Rules

- **Full vs partial:** Objects send full data when first visible, partial (delta) on subsequent ticks
- **DataSplit:** Byte counts recorded per category for bandwidth analytics
- **PlayerData:** Local player gets dedicated `PlayerData` block (health, adrenaline, inventory, etc.)
- **Dirty tracking:** Server only serializes objects that changed

## Data Flow

```
Server tick
  → For each player: serialize PlayerData (if dirty)
  → For each visible object: serialize full (new) or partial (changed)
  → Write killfeed entries
  → Send buffer to client

Client
  → Deserialize with DataSplit (optional)
  → Dispatch to GameObject handlers
  → updateFromData() or updateFull()
```

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../objects/](../objects/) — ObjectSerializations, ObjectsNetData
- **Tier 1:** [../../../protocol.md](../../../protocol.md) — Protocol reference
