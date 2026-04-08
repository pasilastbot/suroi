# Q: How do I add a new perk and make it affect gameplay?

<!-- @tags: definitions, perks, inventory, player -->
<!-- @related: docs/subsystems/definitions/modules/items.md, docs/subsystems/inventory/ -->

## 1. Add the perk definition

Open `common/src/definitions/items/perks.ts` and append:

```typescript
{
    idString: "my_perk",
    name: "My Perk",
    defType: DefinitionType.Perk,
    itemType: ItemType.Perk,

    // What does this perk do?
    wearerAttributes: {
        passive: {
            baseSpeed: 1.15,       // 15% faster
            maxHealth: 1.2,        // 20% more max health
        },
        on: {
            kill: {
                hpRegen: 30,       // heal 30 HP on kill
            }
        }
    },

    // Display
    noDrop: false,    // true = perk cannot be dropped
}
```

See [WearerAttributes](30-wearer-attributes.md) for the full list of modifiers.

## 2. Add a sprite

```
client/public/img/game/shared/loot_<idString>.svg
```

The perk icon is shown in the loadout/HUD.

```bash
bun validateSvgs
```

## 3. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

## 4. Add to the `Loots` collection

If the perk is droppable/pickupable, ensure `perks.ts` items are included in
`common/src/definitions/loots.ts`. This is typically already set up for the
`Perks` collection.

## 5. Add to a loot table

In `server/src/data/lootTables.ts`, add the perk to wherever it should appear:

```typescript
LootTables["normal"]["airdrop_limited"] = [
    // ...
    { item: "my_perk", weight: 1 },
];
```

Or create a dedicated perk spawn table.

## 6. Validate

```bash
bun validateDefinitions
```

## 7. Test

```bash
bun dev
```

Use `give my_perk` in the in-game console to test the effects.

## How Perks Work at Runtime

1. Player picks up the perk → `Inventory.addItem(perkDef)` is called
2. `wearerAttributes.passive` is immediately applied to the player's stat modifiers
3. `wearerAttributes.on.kill` is triggered when the player gets a kill
4. When the perk is dropped, modifiers are removed and stats recalculated

Perks use the same `WearerAttributes` system as armor and guns. They stack with
any other equipped items that also have `wearerAttributes`.

Only **1 perk** can be active at a time (`GameConstants.player.maxPerkCount = 1`).
Picking up a new perk replaces the old one.

## Custom Logic Beyond WearerAttributes

If your perk needs behavior beyond stat multipliers (e.g., triggering a special
effect, modifying game logic), you have two options:

**Option A — Event modifiers only:**
Use `on.kill` or `on.damageDealt` event modifiers. Limited to stat changes.

**Option B — Server plugin:**
Create a server plugin that listens to `player_did_kill` or other events,
checks if the player has the perk (`player.inventory.hasPerk("my_perk")`),
and applies custom logic.

See [Create Server Plugin](05-create-server-plugin.md).

## Related

- [WearerAttributes](30-wearer-attributes.md) — modifier system
- [Items Module](../subsystems/definitions/modules/items.md) — perks.ts
- [Inventory](../subsystems/inventory/README.md) — how perks integrate with inventory
- [Definitions Patterns](../subsystems/definitions/patterns.md) — WearerAttributes pattern
