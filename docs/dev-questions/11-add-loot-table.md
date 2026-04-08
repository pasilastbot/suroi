# Q: How do I add or modify a loot table?

<!-- @tags: map, loot, items, definitions -->
<!-- @related: docs/subsystems/map/modules/loot-tables.md -->

## Where Loot Tables Live

`server/src/data/lootTables.ts` — the `LootTables` object, keyed first by
mode name (`normal`, `halloween`, etc.) then by table ID string:

```typescript
LootTables["normal"]["ground_loot"] = { ... }
LootTables["halloween"]["barrel"] = { ... }
```

## Loot Table Types

### FullLootTable

Spawns a random count of items between `min` and `max`:

```typescript
{
    min: 1,
    max: 3,
    noDuplicates: true,  // optional — each item can only appear once
    loot: [
        { item: "9mm", weight: 3 },
        { item: "556mm", weight: 1.5 },
        { table: "guns", weight: 2 },  // nested table reference
    ]
}
```

### SimpleLootTable

Array of weighted items (one pick total), or array-of-arrays (one pick per
inner array):

```typescript
// One random pick from the whole list:
[
    { item: "bandage", weight: 5 },
    { item: "medikit", weight: 1 },
]

// One pick from each group:
[
    [{ item: "9mm_pack", weight: 3 }, { item: "556mm_pack", weight: 2 }],
    [{ item: "bandage", weight: 4 }, { item: "medikit", weight: 1 }],
]
```

### WeightedItem

```typescript
{ item: "idString", weight: number }         // specific item
{ table: "tableIdString", weight: number }   // nested table lookup
```

## Adding a New Table

1. Open `server/src/data/lootTables.ts`
2. Add your table under the relevant mode:

```typescript
LootTables["normal"]["my_new_crate"] = [
    { item: "ak47", weight: 1 },
    { item: "m16a4", weight: 1 },
    { item: "556mm", weight: 3 },
];
```

3. Reference it from an obstacle definition:

```typescript
// common/src/definitions/obstacles.ts
{
    idString: "my_crate",
    // ...
    lootTable: "my_new_crate",
    spawnWithLoot: true,
}
```

## Modifying an Existing Table

Edit the existing entry in `LootTables`. Adjust weights to change rarity —
higher weight = more likely to be chosen.

## Airdrop Quality

Airdrop crates use a `quality` parameter passed to `getLootFromTable()`. Tables
can use this to vary what drops based on quality tier.

## Data Flow

```
Obstacle destroyed / opened
    → getLootFromTable(modeName, tableID, quality?)
    → resolveTable() → LootTables[mode][tableID]
    → getLoot() — weighted random selection
    → Return LootItem[] (idString + count)
    → Game.addLoot() for each item
```

## Related

- [Loot Tables](../subsystems/map/modules/loot-tables.md) — full data flow and type reference
- [Map Generation](../subsystems/map/modules/generation.md) — how obstacles use loot tables during generation
- [Obstacles](../subsystems/definitions/modules/obstacles.md) — `lootTable`, `spawnWithLoot` fields
