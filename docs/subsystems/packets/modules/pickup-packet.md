# Packets — PickupPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/pickupPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the pickup packet: sent when a player interacts with loot. Contains either the picked-up item or an `InventoryMessages` error (e.g. NotEnoughSpace). The client uses it to update inventory UI and play feedback.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/pickupPacket.ts | `PickupPacket`, `PickupData` | Low |

## PickupData Structure

| Field | Purpose |
|-------|---------|
| `item?` | Picked-up item definition (success) |
| `message?` | InventoryMessages enum (failure) |

## InventoryMessages

- **Success:** No message; `item` contains the loot definition
- **Failure:** `message` indicates reason (NotEnoughSpace, WrongSlotType, etc.)

## Serialization

- **pickupData byte:** `hasItem` (bit 7), `hasMessage` (bit 6), `message` (bits 0–2 if hasMessage)
- If `hasItem`: `Loots.writeToStream(stream, item)`
- If `!hasItem && hasMessage`: message in low 3 bits

## When Sent

- After player interacts with loot (F key)
- Server validates pickup, sends success (item) or failure (message)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../inventory/](../../inventory/) — addItem, pickup logic
- **Tier 2:** [../objects/](../../objects/) — Loot interaction
