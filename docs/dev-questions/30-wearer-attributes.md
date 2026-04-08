# Q: How do WearerAttributes work and how do I make an item passively or actively modify a player stat?

<!-- @tags: definitions, items, inventory, player, perks -->
<!-- @related: docs/subsystems/definitions/modules/items.md, docs/subsystems/inventory/README.md -->

## Overview

`wearerAttributes` is an optional field on any `InventoryItemDefinition` (guns,
melees, throwables, armor, perks, etc.) that modifies the player while the item
is in their inventory.

## The Three Scopes

```typescript
wearerAttributes?: {
    passive?: WearerAttributes   // applied while item is anywhere in inventory
    active?:  WearerAttributes   // applied additionally when this is the active item
    on?:      Partial<EventModifiers>  // triggered on kill or damage dealt
}
```

- **passive** — Always active as long as the item is in the inventory
- **active** — Stacks on top of passive when the item is equipped and in hand
- **on** — One-shot event callbacks (e.g. heal on kill)

## WearerAttributes Fields

All modifiers are **multiplicative** (default 1.0) unless noted:

| Field | Default | Effect |
|-------|---------|--------|
| `maxHealth` | 1.0 | Multiplies max health |
| `maxAdrenaline` | 1.0 | Multiplies max adrenaline |
| `baseSpeed` | 1.0 | Multiplies movement speed |
| `size` | 1.0 | Multiplies player hitbox size |
| `reload` | 1.0 | Multiplies reload time |
| `fireRate` | 1.0 | Multiplies fire rate (fire delay) |
| `adrenDrain` | 1.0 | Multiplies adrenaline drain rate |
| `hpRegen` | 0 | **Additive** — HP regenerated per tick |
| `minAdrenaline` | 0 | **Additive** — Minimum adrenaline floor |

## EventModifiers

```typescript
on?: {
    kill?: WearerAttributes    // applied when player gets a kill
    damageDealt?: WearerAttributes  // applied when player deals damage
}
```

These are temporary stat boosts that stack on top of passive/active until the
next tick.

## Stacking

All `wearerAttributes` from all items in inventory are accumulated. The server
computes an effective modifier by multiplying all applicable passive multipliers,
then adding active multipliers for the equipped item.

## Example: A Perk That Increases Speed and Heals on Kill

```typescript
// common/src/definitions/items/perks.ts
{
    idString: "speedy_medic",
    name: "Speedy Medic",
    itemType: ItemType.Perk,
    wearerAttributes: {
        passive: {
            baseSpeed: 1.15,    // 15% speed increase
        },
        on: {
            kill: {
                hpRegen: 20,    // +20 HP on kill (additive, one tick)
            }
        }
    }
}
```

## Example: Armor That Reduces Movement

```typescript
// common/src/definitions/items/armors.ts
{
    idString: "heavy_vest",
    name: "Heavy Vest",
    wearerAttributes: {
        passive: {
            baseSpeed: 0.85,   // 15% slower while worn
        }
    }
}
```

## Example: Active-Only Bonus (Gun Passive)

```typescript
// common/src/definitions/items/guns.ts
{
    idString: "sniper",
    wearerAttributes: {
        active: {
            baseSpeed: 0.5,    // 50% slower only while aiming the sniper
        }
    }
}
```

## Removing Attributes

When an item is dropped, its `wearerAttributes` are removed and player stats
recalculated. This happens automatically in `server/src/inventory/`.

## Related

- [Definitions Patterns](../subsystems/definitions/patterns.md) — WearerAttributes pattern
- [Game Data Model](../datamodel.md) § Player Modifiers — full modifier table
- [Inventory](../subsystems/inventory/README.md) — how attributes are applied at runtime
- [Add Perk](07-add-new-perk.md) — end-to-end perk addition walkthrough
