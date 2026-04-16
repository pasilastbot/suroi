# Q19: How does the ObjectDefinitions<T> registry work, and what is the contract for adding a new definition type?

**Answer:**

`ObjectDefinitions<T>` is a **typed, dual-indexed registry** that manages all static game-entity data in Suroi. It's a single source of truth for every gun, armor, building, obstacle, item, emote, and other static game concept.

## Overview

The registry provides O(1) lookup by `idString` and maintains stable numeric indices used for compact binary serialization on the wire.

**Core contract:** Every definition must satisfy the `ObjectDefinition` interface:

```typescript
export interface ObjectDefinition {
    readonly idString: string      // kebab-case unique key (e.g., "ak47")
    readonly name: string          // human-readable display name
    readonly defType: DefinitionType
}
```

## Structure (Maps, Indices, Serialization)

`ObjectDefinitions<Def>` is structured as:

```typescript
export class ObjectDefinitions<Def extends ObjectDefinition = ObjectDefinition> {
    readonly definitions: readonly Def[]           // Original array, insertion order
    readonly idStringToDef: ReadonlyRecord<string, Def>      // O(1) lookup: "ak47" → GunDef
    readonly idStringToNumber: ReadonlyRecord<string, number> // O(1) lookup: "ak47" → index
    readonly overLength: boolean                   // true if > 255 entries
}
```

**Key structural rules:**

- **Numeric index = insertion order.** Definition at position 0 gets index 0, position 1 gets index 1, etc. Order is stable across server and client or deserialization fails.
- **`overLength` flag determines wire format:** If entries > 255, indices use `uint16` (2 bytes); otherwise `uint8` (1 byte).
- **Maps use `Object.create(null)`** — avoids prototype-key collisions.

## Adding a New Definition Type: Steps & Contract

### Step 1: Define the Type Interface

```typescript
// In common/src/definitions/mytype.ts

import { DefinitionType, type ObjectDefinition, type ItemDefinition } from "../../utils/objectDefinitions";

export interface MyTypeDefinition extends ItemDefinition {
    readonly defType: DefinitionType.MyType
    readonly myCustomField: string
    readonly myCustomNumber: number
}
```

### Step 2: Add DefinitionType Enum Member

In `common/src/utils/objectDefinitions.ts`:

```typescript
export enum DefinitionType {
    // ... existing types
    MyType  // New type
}
```

### Step 3: Create the Definition Array

```typescript
const RawDefinitions: MyTypeDefinition[] = [
    {
        idString: "my-type-1",
        name: "My Type Item 1",
        defType: DefinitionType.MyType,
        myCustomField: "value",
        myCustomNumber: 42,
    },
    // ... more definitions
];
```

### Step 4: Instantiate the Registry

```typescript
// Simple type:
export const MyTypes = new ObjectDefinitions<MyTypeDefinition>(RawDefinitions);

// Or if it's an inventory item:
export const MyTypes = new InventoryItemDefinitions<MyTypeDefinition>(RawDefinitions);
```

### Step 5: Export from Central Index

In `common/src/definitions/index.ts`:

```typescript
export { MyTypes } from "./mytype";
```

### Step 6: Contract Validation

The constructor enforces:

- **Duplicate `idString` → throws immediately** during module load
- **Required fields on every definition:** `idString`, `name`, `defType` are mandatory
- **Index overflow handled automatically:** If entries > 255, `overLength` is set and binary serialization uses `uint16`

## Serialization/Deserialization via Indices

**Writing:**

```typescript
Guns.writeToStream(stream, gunDefinition);
```

Under the hood:

```typescript
writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void {
    const idString = typeof def === "string" ? def : def.idString;
    const idx = this.idStringToNumber[idString];
    if (this.overLength) {
        stream.writeUint16(idx);  // 2 bytes for > 255 definitions
    } else {
        stream.writeUint8(idx);   // 1 byte for ≤ 255 definitions
    }
}
```

**Reading:**

```typescript
const gun = Guns.readFromStream<GunDefinition>(stream);
```

**Critical:** Client and server must have the same definition order. If the server sends index 5 for a gun but the client has a different definition at index 5, deserialization reads the wrong object.

## Runtime Validation

**Load-time validation:**

- **Duplicate `idString` detection** — constructor throws immediately
- **No explicit schema validation** — relies on TypeScript's type system

**Query-time validation:**

- **`fromString(idString)`** → throws `ReferenceError` if unknown
- **`fromStringSafe(idString)`** → returns `undefined` instead of throwing
- **`readFromStream(stream)`** → throws `RangeError` if index out of bounds

## Dependencies

**Object Definitions depends on:**

- **Serialization System** — `ByteStream` for binary encoding/decoding
- **Constants** — `GameConstants`, `DefinitionType` enum members

**Depended on by (virtually all subsystems):**

- **Game Loop** — Tick phase uses definitions to spawn objects
- **Networking** — All packets serialize/deserialize definition indices
- **Game Objects** — Objects created from definitions
- **Inventory** — Gun, armor, healing item definitions
- **Map Generation** — Building, obstacle, loot definitions

## References

- **Tier 2:** [docs/subsystems/object-definitions/README.md](docs/subsystems/object-definitions/README.md)
- **Source:** [ObjectDefinitions class](common/src/utils/objectDefinitions.ts)
- **Example:** [Guns registry](common/src/definitions/items/guns.ts)
