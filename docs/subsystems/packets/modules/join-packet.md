# Packets — JoinPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/joinPacket.ts, common/src/packets/joinedPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the client→server connection flow: `JoinPacket` (client sends) and `JoinedPacket` (server responds). The join handshake establishes protocol version, player identity, and loadout.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/joinPacket.ts | `JoinPacket`, `JoinData` — client join request | Medium |
| @file common/src/packets/joinedPacket.ts | `JoinedPacket`, `JoinedData` — server response | High |

## JoinData (Client → Server)

| Field | Purpose |
|-------|---------|
| `protocolVersion` | Must match server; reject if mismatch |
| `name` | Player display name |
| `isMobile` | Mobile client flag |
| `skin` | Selected skin definition |
| `badge?` | Optional badge |
| `emotes` | 8 emote slots (EmoteDefinition \| undefined) |

## Serialization

- `writeBooleanGroup2` — 10 booleans (isMobile, hasBadge, emotes[0–7])
- `writeUint16` — protocol version
- `writePlayerName` — name string
- `Loots.writeToStream` — skin
- `Badges.writeToStream` — badge if present
- `Emotes.writeToStream` — each defined emote

## Connection Flow

```
Client → JoinPacket (protocolVersion, name, skin, loadout)
Server → Validate protocolVersion
    → If mismatch: close connection
    → Create Player, assign team
    → JoinedPacket (game state, player ID, etc.)
Client → Process JoinedPacket, start game
```

## Protocol Considerations

- **Protocol version mismatch:** Server closes connection; client must upgrade
- **JoinPacket format changes:** Require protocol bump

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 1:** [docs/protocol.md](../../../protocol.md) — Protocol version
- **Tier 2:** [../game-loop/](../../game-loop/) — Player activation
