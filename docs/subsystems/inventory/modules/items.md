# Inventory — Items Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/inventory.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents non-weapon inventory items: healing items, armor, backpack, scope, ammo. These are stored in `ItemCollection` and equipped in dedicated slots.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/inventory/inventory.ts | `Inventory`, `ItemCollection` | High |
| @file common/src/defaultInventory.ts | `DEFAULT_INVENTORY` — default loadout | Low |

## Item Slots (non-weapon)

| Slot | Type | Purpose |
|------|------|---------|
| helmet | ArmorDefinition | Head armor |
| vest | ArmorDefinition | Body armor |
| backpack | BackpackDefinition | Capacity |
| scope | ScopeDefinition | Zoom level |

## ItemCollection

- **Keyed by:** definition idString
- **Values:** count (number)
- **Capacity:** From `backpack` definition
- **Default:** `DEFAULT_INVENTORY` — initial items on spawn

## Item Types

- **Healing:** Bandages, medikit, cola, tablets — `HealingAction` for use
- **Armor:** Helmets, vests — equip to helmet/vest slots
- **Backpacks:** Capacity upgrade — equip to backpack slot
- **Scopes:** Zoom — equip to scope slot
- **Ammo:** Stored by ammo type idString; consumed by guns

## Business Rules

- `addItem(def, count)` — Check capacity, swap logic
- `InventoryMessages` — NotEnoughSpace, etc. on pickup failure
- Armor/backpack/scope affect player stats (damage reduction, capacity, zoom)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Inventory overview
- **Tier 3:** [weapons.md](weapons.md) — Weapon slots
- **Tier 2:** [../definitions/](../../definitions/) — Item definitions
