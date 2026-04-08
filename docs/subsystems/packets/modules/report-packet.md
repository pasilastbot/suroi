# Packets — ReportPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/reportPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the report packet: sent when a player reports another player. Client→Server only. Contains the reported player ID and a report identifier for moderation tracking.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/reportPacket.ts | `ReportPacket`, `ReportData` | Low |

## ReportData Structure

| Field | Purpose |
|-------|---------|
| `playerID` | Object ID of reported player |
| `reportID` | Report identifier (max 8 chars) |

## Serialization

- `writeObjectId(playerID)`
- `writeString(8, reportID)` — Fixed 8-char string

## When Sent

- When player uses report UI (e.g. after death, spectating)
- Server handles moderation (logging, bans, etc.)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../ui/](../../ui/) — Report UI
