# Packets — JoinedPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/joinedPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the server→client join confirmation packet. Sent in response to `JoinPacket`; confirms team assignment and loadout (emotes). The client uses it to finalize connection state before receiving `MapPacket` and `UpdatePacket`.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/joinedPacket.ts | `JoinedPacket`, `JoinedData` | Medium |

## JoinedData Structure

| Field | Purpose |
|-------|---------|
| `teamMode` | Solo, Duo, or Squad |
| `teamID?` | Team index (only when teamMode ≠ Solo) |
| `emotes` | 8 emote slots (EmoteDefinition \| undefined) |

## Serialization

- `writeUint8(teamMode)` — TeamMode enum
- If teamMode ≠ Solo: `writeUint8(teamID)`
- `writeBooleanGroup` — 8 booleans for which emote slots are defined
- For each defined emote: `Emotes.writeToStream(stream, emote)`

## Connection Flow

```
Client → JoinPacket
Server → Validate, create Player, assign team
Server → JoinedPacket (teamMode, teamID?, emotes)
Client → Process JoinedPacket, request MapPacket
Server → MapPacket
Client → Start game loop, send InputPacket
```

## DataSplit

- `recordTo(DataSplitTypes.PlayerData)` — Byte count for analytics

## Related Documents

- **Tier 3:** [join-packet.md](join-packet.md) — Client join request
- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../game-loop/](../../game-loop/) — Player activation
