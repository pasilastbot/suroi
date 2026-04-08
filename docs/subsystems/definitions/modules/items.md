# Definitions — Items Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/items/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

Items are lootable objects that players can pick up and use. This module covers guns, melees, throwables, ammo, healing items, armor, backpacks, scopes, skins, and perks.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `items/guns.ts` | Gun definitions (fire rate, recoil, ammo, etc.) | High |
| `items/melees.ts` | Melee weapon definitions | Medium |
| `items/throwables.ts` | Grenades, C4, etc. | Medium |
| `items/ammos.ts` | Ammo types | Low |
| `items/healingItems.ts` | Bandages, medikit, cola, tablets | Low |
| `items/armors.ts` | Vests, helmets | Low |
| `items/backpacks.ts` | Backpack capacity | Low |
| `items/scopes.ts` | Scope zoom levels | Low |
| `items/skins.ts` | Player skins | Low |
| `items/perks.ts` | Perk definitions | Medium |
| `items/items.ts` | `InventoryItemDefinitions` — shared item config | Medium |
| `loots.ts` | `Loots` — unified `ObjectDefinitions` for all items | Medium |

## Business Rules

- All items have `speedMultiplier` (affects player movement when equipped)
- Inventory items have `wearerAttributes` for passive/active modifiers
- `noDrop` / `noSwap` items cannot be dropped or swapped
- `devItem` items only spawn in dev mode

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 2:** [../patterns.md](../patterns.md) — ReferenceTo, WearerAttributes
- **Tier 1:** [../../../datamodel.md](../../../datamodel.md) — ItemType, loot radius
