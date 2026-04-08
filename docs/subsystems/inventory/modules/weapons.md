# Inventory — Weapons Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/gunItem.ts, server/src/inventory/meleeItem.ts, server/src/inventory/throwableItem.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

Weapons are the three equipable item types: guns, melees, and throwables. Each has server-side logic for use (fire, swing, cook/throw), stats tracking, and serialization.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/gunItem.ts` | `GunItem` — firing, reload, burst/auto, fire delay | High |
| `server/src/inventory/meleeItem.ts` | `MeleeItem` — melee swing, hitbox, targets | Medium |
| `server/src/inventory/throwableItem.ts` | `ThrowableItem` — cook timer, throw, count | Medium |
| `server/src/inventory/inventoryItem.ts` | `InventoryItemBase`, `CountableInventoryItem` | High |
| `server/src/inventory/action.ts` | `ReloadAction`, `HealingAction` | Low |

## Game Rules

- **GunItem:** Tracks ammo, reload state, fire delay. Uses `setTimeout` for burst/auto fire (faster than tick rate). `useItem()` fires; `stopUse()` stops auto fire.
- **MeleeItem:** Single swing per use. Uses `getMeleeHitbox`, `getMeleeTargets` from common. Cooldown from definition.
- **ThrowableItem:** Cook phase (hold) then throw on release. `count` tracks stack size. Cached in `Inventory.throwableItemMap` by type.

## Data Flow

```
Player attacks
    → activeItem.useItem()
    → GunItem: fire bullet(s), consume ammo
    → MeleeItem: resolve targets, deal damage
    → ThrowableItem: start cook → stopUse() → throw

Reload (GunItem)
    → ReloadAction, consume ammo from Inventory.items
    → fireDelay reset after reload
```

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Inventory overview
- **Tier 2:** [../../definitions/](../../definitions/) — GunDefinition, MeleeDefinition, ThrowableDefinition
