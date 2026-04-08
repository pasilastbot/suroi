# Packets Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/packets/modules/ -->
<!-- @source: common/src/packets/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Packets subsystem implements the binary network protocol for client-server communication. Each packet type has a `serialize` function (sender) and `deserialize` function (receiver). All packets use `SuroiByteStream` for dense binary encoding over WebSocket frames.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/packets/packet.ts` | `Packet` class, `PacketType` enum, `DataSplitTypes` |
| `common/src/packets/packetStream.ts` | `PacketStream`, `Packets` array — framing and dispatch |
| `common/src/packets/joinPacket.ts` | Client → Server: initial connection, player setup |
| `common/src/packets/joinedPacket.ts` | Server → Client: join confirmation, initial state |
| `common/src/packets/inputPacket.ts` | Client → Server: movement, actions, aim (every frame) |
| `common/src/packets/updatePacket.ts` | Server → Client: game state delta (40 TPS) |
| `common/src/packets/mapPacket.ts` | Server → Client: map data (terrain, obstacles, etc.) |
| `common/src/packets/killPacket.ts` | Server → Client: kill events |
| `common/src/packets/gameOverPacket.ts` | Server → Client: game end stats |
| `common/src/packets/pickupPacket.ts` | Server → Client: item pickup result |
| `common/src/packets/spectatePacket.ts` | Client → Server: spectate actions |
| `common/src/packets/reportPacket.ts` | Client → Server: player report |
| `common/src/packets/debugPacket.ts` | Server → Client: debug visuals (dev only) |
| `common/src/utils/suroiByteStream.ts` | Game-specific serialization primitives |
| `common/src/utils/byteStream.ts` | Base byte stream (uint8, uint16, float, etc.) |

## Architecture

```
PacketStream (wraps SuroiByteStream)
    │
    ├── serialize(data) → writes type byte + packet payload
    └── deserialize(splits?) → reads type byte, dispatches to Packets[type].deserialize

Packet<DataIn, DataOut>
    ├── type: PacketType
    ├── serialize(stream, data)
    ├── deserialize(stream, splits?)
    └── create(data?) → factory for packet data
```

The `Packets` array is indexed by `PacketType - 1` (0-based). `Disconnect` is in the enum but not in the array — it is handled at the WebSocket level.

## Data Flow

```
Client                          Server
  │                                │
  │  JoinPacket ─────────────────> │  (protocol version check)
  │  <───────────────── JoinedPacket
  │  <───────────────── MapPacket
  │                                │
  │  InputPacket ────────────────> │  (every frame)
  │  <───────────────── UpdatePacket  (40 TPS)
  │                                │
  │  SpectatePacket ────────────> │
  │  <───────────────── GameOverPacket
```

## Packet Direction Summary

| Packet | Direction | When |
|--------|-----------|------|
| Join | Client → Server | Once, after WebSocket connect |
| Joined | Server → Client | Response to Join |
| Map | Server → Client | Once, after Join |
| Input | Client → Server | Every frame (if changed) |
| Update | Server → Client | Every tick (40 TPS) |
| Kill | Server → Client | On any kill |
| Pickup | Server → Client | On item pickup |
| GameOver | Server → Client | On game end |
| Spectate | Client → Server | After death, spectate actions |
| Report | Client → Server | Player report |
| Debug | Server → Client | Dev only |

## DataSplit (UpdatePacket Bandwidth Tracking)

`UpdatePacket` uses `DataSplit` to record byte counts per category for analytics:

| Split | Content |
|-------|---------|
| `PlayerData` | Local player stats |
| `Players` | Other players |
| `Obstacles` | Obstacles |
| `Loots` | Loot items |
| `SyncedParticles` | Particles |
| `GameObjects` | Buildings, decals, parachutes, projectiles, death markers |
| `Killfeed` | Kill feed entries |

## Protocol Considerations

- **Affects protocol:** Yes. Any packet format change requires `GameConstants.protocolVersion` bump.
- **Backward compatibility:** None. Old clients cannot connect to new servers (and vice versa) if protocol differs.
- **Validation:** Protocol version is checked in `JoinPacket`; server disconnects on mismatch.

## Dependencies

- **Depends on:** Definitions (for `writeToStream`/`readFromStream`), `SuroiByteStream`, `ObjectSerializations`
- **Depended on by:** Server (sends/receives), Client (sends/receives), Game loop

## Module Index (Tier 3)

For implementation details, see:

- [JoinPacket](modules/join-packet.md) — Client join handshake, protocol version, loadout
- [JoinedPacket](modules/joined-packet.md) — Server join confirmation, team assignment, emotes
- [MapPacket](modules/map-packet.md) — Terrain, rivers, obstacles, buildings (sent once after join)
- [UpdatePacket](modules/update-packet.md) — Delta-compressed game state, full vs partial updates
- [InputPacket](modules/input-packet.md) — Movement, actions, mobile/turning mixins
- [KillPacket](modules/kill-packet.md) — Kill events, damage source, weapon, killfeed
- [GameOverPacket](modules/game-over-packet.md) — Game end, rank, teammate stats
- [PickupPacket](modules/pickup-packet.md) — Loot pickup result, InventoryMessages
- [SpectatePacket](modules/spectate-packet.md) — Spectate actions (next, prev, specific player)
- [ReportPacket](modules/report-packet.md) — Player report (moderation)
- [DebugPacket](modules/debug-packet.md) — Dev-only debug state (no-clip, spawn loot, etc.)

## Related Documents

- **Tier 1:** [docs/protocol.md](../../protocol.md) — Full protocol reference
- **Tier 1:** [docs/architecture.md](../../architecture.md) — Connection lifecycle
- **Tier 2:** [../definitions/](../definitions/) — Definition serialization
- **Tier 2:** [../objects/](../objects/) — Object serialization in UpdatePacket
