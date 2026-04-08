# Inventory Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/inventory/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Inventory subsystem manages player inventory: weapons (guns, melees, throwables), armor, backpack, scope, healing items, and perks. It handles equipping, swapping, dropping, picking up, and using items.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/inventory/inventory.ts` | `Inventory` class — main inventory container |
| `server/src/inventory/inventoryItem.ts` | `InventoryItemBase` — base for weapon items |
| `server/src/inventory/gunItem.ts` | `GunItem` — gun handling, firing, reload |
| `server/src/inventory/meleeItem.ts` | `MeleeItem` — melee attacks |
| `server/src/inventory/throwableItem.ts` | `ThrowableItem` — grenades, cooking, throwing |
| `server/src/inventory/action.ts` | `HealingAction`, `ReloadAction` — action types |
| `common/src/defaultInventory.ts` | `DEFAULT_INVENTORY` — default loadout |

## Architecture

```
Inventory (owner: Player)
    ├── weapons[4] — Gun, Gun, Melee, Throwable
    ├── items — ItemCollection (healing, armor, ammo, etc.)
    ├── helmet, vest, backpack — equipped armor
    ├── scope — equipped scope
    ├── throwableItemMap — cached ThrowableItem instances
    └── lockedSlots — bitmask of locked weapon slots
```

### Slot Layout

| Slot | Type | Purpose |
|------|------|---------|
| 0 | Gun | Primary weapon |
| 1 | Gun | Secondary weapon |
| 2 | Melee | Melee weapon |
| 3 | Throwable | Grenades, C4, etc. |

### ItemCollection

- `items` holds non-weapon items: healing items, armor, ammo, etc.
- Backpack capacity from `BackpackDefinition`
- Items are keyed by definition idString

## Data Flow

```
Pickup (Loot interaction)
    → Inventory.addItem(itemDef, count)
    → Check slot type, capacity, swap logic
    → Return InventoryMessages (NotEnoughSpace, etc.)

Equip
    → Inventory.equipItem(slot)
    → Swap or equip from items
    → Emit inv_item_equip

Use (attack)
    → activeItem.useItem()
    → GunItem: fire bullet
    → MeleeItem: melee attack
    → ThrowableItem: cook / throw

Reload
    → GunItem.reload()
    → Consume ammo from items
```

## Key Concepts

### Weapon Items

- `GunItem` — Tracks ammo, reload state, fire delay, recoil
- `MeleeItem` — Single-use swing, cooldown
- `ThrowableItem` — Cook timer, throw physics

### Locked Slots

- Players can lock weapon slots to prevent accidental swap
- `lockedSlots` is a bitmask (1 << slot)

### Wearer Attributes

- Items with `wearerAttributes` modify player stats when equipped
- Passive (in inventory) vs active (equipped) modifiers
- Event modifiers (on kill, on damage dealt)

## Module Index (Tier 3)

- [Weapons](modules/weapons.md) — Guns, melees, throwables
- [Items](modules/items.md) — Healing, armor, backpack, scope, ammo
- [Actions](modules/actions.md) — HealingAction, ReloadAction, ReviveAction

## Protocol Considerations

- **Affects protocol:** Yes. Inventory changes are serialized in UpdatePacket PlayerData. New item types require protocol bump.

## Dependencies

- **Depends on:** Definitions (Loots, armor, weapons), Objects (Player, Loot)
- **Depended on by:** Game loop (player.update), Loot interaction, PickupPacket

## Related Documents

- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — Inventory slots, ItemType
- **Tier 2:** [../definitions/](../definitions/) — Item definitions
- **Tier 2:** [../objects/](../objects/) — Player, Loot
