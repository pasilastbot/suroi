# Packets — SpectatePacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/spectatePacket.ts, common/src/constants.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the spectate packet: sent when a dead player chooses spectate actions (next player, previous, specific player). Client→Server only. The server updates spectate target and sends UpdatePacket with that player's view.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/spectatePacket.ts | `SpectatePacket`, `SpectateData` | Low |
| @file common/src/constants.ts | `SpectateActions` enum | Low |

## SpectateData Structure

| Field | Purpose |
|-------|---------|
| `spectateAction` | SpectateActions enum |
| `playerID?` | Target player ID (only for SpectateSpecific) |

## SpectateActions

- **SpectateSpecific** — Spectate a specific player by ID
- **SpectateNext** — Cycle to next alive player
- **SpectatePrevious** — Cycle to previous alive player

## Serialization

- `writeUint8(spectateAction)`
- If SpectateSpecific: `writeObjectId(playerID)`

## When Sent

- After player death, when spectate UI is shown
- Client sends on button press (next/prev) or player select

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../ui/](../../ui/) — Spectate button, player list
- **Tier 2:** [../game-loop/](../../game-loop/) — Spectate target in UpdatePacket
