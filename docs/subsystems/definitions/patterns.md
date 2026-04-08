# Definitions Subsystem — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @updated: 2026-03-04 -->

## Pattern: Adding a New Definition Type

**When to use:** Adding a new kind of game object (e.g. a new item type).

**Implementation:**

1. Create `common/src/definitions/items/<type>.ts` (or appropriate file)
2. Define the TypeScript interface extending `ObjectDefinition` or `InventoryItemDefinition`
3. Create the array of definitions
4. Export `new ObjectDefinitions<YourDef>([...])`
5. If it's loot: add to `Loots` in `common/src/definitions/loots.ts`
6. Add serialization in `common/src/utils/objectsSerializations.ts` if it's a runtime object
7. Bump `GameConstants.protocolVersion`
8. Run `bun validateDefinitions`

**Example files:** `@file common/src/definitions/items/guns.ts`, `common/src/definitions/items/healingItems.ts`

---

## Pattern: ReferenceTo and Cross-References

**When to use:** A definition references another definition (e.g. gun → ammo type, obstacle → loot table).

**Implementation:**

```typescript
// Use ReferenceTo<Def> for a required reference
readonly ammoType: ReferenceTo<AmmoDefinition>

// Use ReferenceOrNull for optional
readonly spawnScope?: ReferenceTo<ScopeDefinition>

// Use ReferenceOrRandom for weighted random
readonly lootTable?: ReferenceOrRandom<LootDefinition>
```

Look up at runtime via `Defs.fromString(idString)` or `Defs.fromStringSafe(idString)`.

**Example files:** `@file common/src/definitions/items/guns.ts`, `common/src/definitions/obstacles.ts`

---

## Pattern: ObjectDefinitions Index Serialization

**When to use:** Serializing a definition reference over the network.

**Implementation:**

- `ObjectDefinitions.writeToStream(stream, def)` — writes 1 or 2 bytes (index)
- `ObjectDefinitions.readFromStream(stream)` — reads index, returns definition
- If the collection has >255 definitions, `overLength` is true and 2 bytes are used

**Gotcha:** Array order matters. Inserting a definition in the middle shifts all subsequent indices and breaks protocol compatibility.

**Example files:** `@file common/src/utils/objectDefinitions.ts`, `common/src/packets/joinPacket.ts`

---

## Pattern: WearerAttributes (Item Modifiers)

**When to use:** An item modifies the player when equipped (armor, perks, etc.).

**Implementation:**

```typescript
readonly wearerAttributes?: {
    readonly passive?: WearerAttributes   // when in inventory
    readonly active?: WearerAttributes    // when active item (stacks on passive)
    readonly on?: Partial<EventModifiers> // on kill, damageDealt
}
```

All attributes stack. Removed when item is dropped.

**Example files:** `@file common/src/definitions/items/armors.ts`, `common/src/definitions/items/perks.ts`
