# Packets — GameOverPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/gameOverPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the game-over packet: sent when a game ends, contains placement rank and teammate stats. The client uses it to show the game-over screen (placement, kills, damage, spectate option).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/gameOverPacket.ts | `GameOverPacket`, `GameOverData`, `TeammateGameOverData` | Low |

## GameOverData Structure

| Field | Purpose |
|-------|---------|
| `rank` | Placement (1 = winner) |
| `teammates` | Array of teammate stats |

## TeammateGameOverData

| Field | Purpose |
|-------|---------|
| `playerID` | Object ID |
| `kills` | Kill count |
| `damageDone` | Total damage dealt |
| `damageTaken` | Total damage received |
| `timeAlive` | Seconds alive |
| `alive` | Still alive at game end |

## Serialization

- `writeUint8(rank)`
- `writeArray(teammates)` — each: objectId, kills, damageDone (uint16), damageTaken (uint16), timeAlive (uint16), alive (1/0)

## When Sent

- After win condition (last player/team standing)
- Or when game ends (e.g. timeout)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../game-loop/](../../game-loop/) — Win condition check
- **Tier 2:** [../ui/](../../ui/) — Game-over screen
