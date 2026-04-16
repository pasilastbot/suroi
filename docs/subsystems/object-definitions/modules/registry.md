# Object Definitions Registry Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/object-definitions/README.md -->
<!-- @source: common/src/utils/objectDefinitions.ts -->

## Purpose

`ObjectDefinitions<Def>` is a typed, dual-indexed registry that maps `idString` keys and compact numeric indices to definition objects, and owns the binary serialization of those references into `ByteStream`.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/utils/objectDefinitions.ts` | `ObjectDefinitions<Def>` class, helper types, `DefinitionType` enum | Medium |
| `common/src/definitions/items/items.ts` | `InventoryItemDefinitions<Def>` — subclass that resolves speed multipliers | Low |
| `common/src/definitions/items/guns.ts` | `Guns = new InventoryItemDefinitions<GunDefinition>([ ... ])` — typical usage | Low |

## Business Rules

- **Duplicate `idString` is a hard error.** The constructor throws `Error("Duplicate idString '${idString}' in schema")` synchronously during module load; no deferred validation.
- **Numeric index = insertion order.** Definitions are indexed 0, 1, 2, … in the order they appear in the input array. This order must be stable across server and client builds or deserialization will read the wrong definition.
- **`overLength` determines serialization width.** If more than 255 definitions are registered (`idx > 255`), the registry uses `uint16` (2 bytes) for all reads/writes; otherwise `uint8` (1 byte). This flag is computed once at construction time.
- **`fromString` vs `fromStringSafe` — throw vs undefined.** `fromString` throws `ReferenceError` on unknown `idString`; `fromStringSafe` returns `undefined`. Callers that tolerate missing definitions must use `fromStringSafe`.
- **`Guns` is actually `InventoryItemDefinitions<GunDefinition>`.** That subclass wraps the constructor to multiply each definition's `speedMultiplier` by `GameConstants.defaultSpeedModifiers[defType]` before passing the array to `super()`. All other definition sets use `ObjectDefinitions` directly.
- **Definitions are not frozen/sealed at the type level**, but all fields are declared `readonly` in the `ObjectDefinition` interface. Mutation is possible only via an explicit cast (e.g., `(def as Mutable<Def>).speedMultiplier`).
- **Index lookup goes through the plain array.** `this.definitions[idx]` is a direct array access; the registry validates the result is non-undefined and that its `idString` is registered, then throws `RangeError`/`Error` if not.

## Generic Type Constraint

```typescript
export class ObjectDefinitions<Def extends ObjectDefinition = ObjectDefinition>
```

The bound `Def extends ObjectDefinition` requires every registered object to satisfy:

```typescript
export interface ObjectDefinition {
    readonly idString: string
    readonly name: string
    readonly defType: DefinitionType
}
```

All three fields are mandatory. `defType` must be one of the `DefinitionType` enum members (see below).

## Class API

### Constructor

```typescript
constructor(definitions: readonly Def[])
```

Accepts a read-only array of definitions. Iterates once to:
1. Detect duplicate `idString` — throws immediately.
2. Populate `idStringToDef` (string → Def map).
3. Populate `idStringToNumber` (string → numeric index map).
4. Set `overLength = idx > 255`.

**Called by:** every definition module, e.g. `export const Guns = new InventoryItemDefinitions<GunDefinition>([ ... ])`.

### Public Properties

```typescript
readonly definitions: readonly Def[]
readonly idStringToDef: ReadonlyRecord<string, Def>
readonly idStringToNumber: ReadonlyRecord<string, number>
readonly overLength: boolean
```

### `reify(type: ReifiableDef<Def>): U`

```typescript
reify<U extends Def = Def>(type: ReifiableDef<Def>): U
```

Accepts either a `string` (`ReferenceTo<Def>`) or a `Def` object. If a string is passed, delegates to `fromString`; if already an object, returns it cast to `U`. Used throughout the server when a caller may hold either form.

### `fromString<Spec extends Def = Def>(idString: ReferenceTo<Spec>): Spec`

```typescript
fromString<Spec extends Def = Def>(idString: ReferenceTo<Spec>): Spec
```

Throws `ReferenceError("Unknown idString '${idString}' for this schema")` if the key is not found. Use when the caller is certain the string is valid.

### `fromStringSafe<Spec extends Def = Def>(idString: ReferenceTo<Spec>): Spec | undefined`

```typescript
fromStringSafe<Spec extends Def = Def>(idString: ReferenceTo<Spec>): Spec | undefined
```

Returns `undefined` (not null, not an exception) for unknown keys.

### `hasString(idString: string): boolean`

```typescript
hasString(idString: string): boolean
```

Fast existence check via `in` operator on `idStringToDef`.

### `writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void`

```typescript
writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void
```

Resolves `def` to its numeric index, then writes 1 byte (`writeUint8`) when `overLength` is false, or 2 bytes (`writeUint16`) when true. Throws if the `idString` is unknown.

### `readFromStream<Spec extends Def>(stream: ByteStream): Spec`

```typescript
readFromStream<Spec extends Def>(stream: ByteStream): Spec
```

Reads 1 or 2 bytes depending on `overLength`, looks up `definitions[idx]`, validates the result, and returns the definition cast to `Spec`. Throws `RangeError` on a bad index or `Error` if the recovered `idString` is not in the map.

### `[Symbol.iterator](): Iterator<Def>`

Makes the registry directly iterable: `for (const def of Guns) { ... }` iterates in insertion order via the underlying `definitions` array.

## Helper Types

```typescript
// The idString of T, used to communicate "this string must be a valid idString for T"
export type ReferenceTo<T extends ObjectDefinition> = T["idString"];

// ReferenceTo<T> or the NullString symbol (no match / not applicable)
export type ReferenceOrNull<T extends ObjectDefinition> = ReferenceTo<T> | typeof NullString;

// Weighted options: Record<ReferenceOrNull<T>, weight> OR a plain ReferenceTo<T>
export type ReferenceOrRandom<T extends ObjectDefinition> = Partial<Record<ReferenceOrNull<T>, number>> | ReferenceTo<T>;

// Either a definition object or an idString referencing one
export type ReifiableDef<T extends ObjectDefinition> = ReferenceTo<T> | T;

// Symbol used to mean "no idString applies"
export const NullString = Symbol("null idString");
```

`ReferenceTo<T>` is semantically `string` — the type system carries the intent that this string is a valid key for a specific registry, but no runtime check enforces it until `fromString` is called.

## `DefinitionType` Enum

```typescript
export enum DefinitionType {
    Ammo,          // 0
    Armor,         // 1
    Backpack,      // 2
    Badge,         // 3
    Building,      // 4
    Bullet,        // 5
    Decal,         // 6
    Emote,         // 7
    Explosion,     // 8
    Gun,           // 9
    HealingItem,   // 10
    MapPing,       // 11
    MapIndicator,  // 12
    Melee,         // 13
    Obstacle,      // 14
    Perk,          // 15
    Scope,         // 16
    Skin,          // 17
    SyncedParticle,// 18
    Throwable      // 19
}
```

Every definition must include `defType` set to the appropriate member. Server-side code (e.g., `maps.ts`) uses `DefinitionType` to branch on the category of an item without an `instanceof` check.

## How Binary Serialization Uses the Registry

Binary serialization is **owned by the registry itself** via `writeToStream`/`readFromStream`; there is no separate `writeObjectType`/`readObjectType` in `suroiByteStream.ts`. The pattern used across all packets is:

```typescript
// Writing (e.g. inside a packet serializer)
Guns.writeToStream(stream, gunDefinition);
// → if Guns.overLength: stream.writeUint16(index)
// → else:              stream.writeUint8(index)

// Reading
const gun = Guns.readFromStream<GunDefinition>(stream);
// → idx = overLength ? stream.readUint16() : stream.readUint8()
// → return Guns.definitions[idx]
```

Because the numeric index is a 1-or-2-byte value (no string sent on the wire), the compact representation is only stable as long as definitions are in the same order on both ends.

## Data Lineage

```
Definition array literal (e.g. guns.ts)
  → new InventoryItemDefinitions<GunDefinition>(array)
      → speedMultiplier adjusted per entry
      → super(adjusted array)
          → idStringToDef populated (O(n))
          → idStringToNumber populated (O(n))
          → overLength computed
  → Guns singleton exported

Query path:
  Guns.fromString("ak47")       → O(1) Map.get → GunDefinition
  Guns.reify("ak47")            → delegates to fromString → GunDefinition
  Guns.reify(existingGunDef)    → returns existingGunDef cast to U

Binary path:
  Guns.writeToStream(stream, def)
    → idx = idStringToNumber["ak47"]
    → stream.writeUint8(idx)     // overLength = false for guns (< 256)

  Guns.readFromStream(stream)
    → idx = stream.readUint8()
    → Guns.definitions[idx]      // direct array access
```

## Complex Functions

### `ObjectDefinitions<Def>` constructor — @file common/src/utils/objectDefinitions.ts

**Purpose:** Builds both lookup structures from the input definition array in a single O(n) pass.

**Implicit behavior:**
- Uses `Object.create(null)` for `idStringToDef` and `idStringToNumber` to avoid prototype-key collisions (no inherited `constructor`, `__proto__`, etc.).
- Throws synchronously during module initialization if duplicates exist — a badly ordered import that creates a duplicate will crash the process at startup, not at call time.
- Does **not** freeze or seal the registered definitions; mutation of individual fields (with a cast) is intentional in `InventoryItemDefinitions`.

**Called by:** Every definition module at module load time: `Guns`, `Obstacles`, `Buildings`, `Ammos`, etc.

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Object Definitions subsystem overview
- **Tier 1:** [../../datamodel.md](../../datamodel.md) — Data model overview
- **Patterns:** [../patterns.md](../patterns.md) — Definition patterns
