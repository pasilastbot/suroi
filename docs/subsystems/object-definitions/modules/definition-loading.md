# Definition Loading Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: common/src/utils/objectDefinitions.ts, common/src/definitions/ -->

## Purpose

Handles the loading, validation, indexing, and in-memory storage of all game object definitions (weapons, items, buildings, etc.) using the `ObjectDefinitions<T>` generic registry class.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/utils/objectDefinitions.ts` | `ObjectDefinitions<T>` registry class, `DefinitionType` enum, definition type system | High |
| `common/src/definitions/*.ts` | Specific definition files (guns.ts, ammos.ts, buildings.ts, etc.) | Medium |
| `common/src/utils/byteStream.ts` | Binary serialization / deserialization for definition indices | Medium |

## Business Rules

- **Single Registry Per Type:** Each object type (Gun, Item, Building) has one `ObjectDefinitions<T>` instance with O(1) lookup
- **Index-Based Serialization:** Definitions are serialized as single-byte (0–255) or two-byte (256+) indices, not as full objects → compact network bandwidth
- **Validation on Construction:** Duplicate `idString` values throw errors immediately; invalid references caught at runtime
- **Immutable After Creation:** Definition objects are readonly; cannot be modified after registration (prevents desync)
- **Fallback Not Implemented:** If a definition is not found at load time, the entire application should fail loudly (no graceful degradation)

## Data Lineage

```
HJSON Definition Files (common/src/definitions/*.ts)
    ↓
Parser / Import (HJSON → TypeScript object)
    ↓
ObjectDefinitions<T> Constructor (builds idStringToDef map, allocates indices)
    ↓
In-Memory Registry (idString → Definition lookup, fast binary I/O)
    ↓
Network Packets (write index to stream, read index back)
    ↓
Game Objects (apply definition, spawn with properties)
```

## Dependencies

- **Internal:** `ByteStream` (serialization), `Hitbox` (for building/obstacle definitions)
- **External:** None (self-contained)
- **Uses:** Definition types defined in `common/src/definitions/` (Guns, Ammos, Armors, Buildings, etc.)

## Complex Functions

### `ObjectDefinitions<Def>.constructor(definitions: readonly Def[])`  
**Purpose:** Initialize the registry, build lookup tables, detect Schema overflow (>255 definitions).

**Implicit Behavior:**
- Iterates all definitions, assigns sequential indices 0, 1, 2, ...
- Populates `idStringToDef` (Fast O(1) lookup by string ID)
- Populates `idStringToNumber` (Maps ID → index for serialization)
- Sets `overLength = idx > 255` flag (determines if index uses 1 or 2 bytes)
- Throws if duplicate `idString` encountered (e.g., two guns with `idString: "ak47"`)

**Error Handling:** Synchronous throw on detect; prevents registration of malformed definitions

**File:** `common/src/utils/objectDefinitions.ts`

### `ObjectDefinitions<Def>.fromString(idString: string): Def`  
**Purpose:** Resolve a string ID to a definition at runtime.

**Implicit Behavior:**
- Calls `fromStringSafe` and asserts result is defined
- Throws `ReferenceError` if ID not found (e.g., server sent unknown gun)
- Used when parsing config, spawn requests, or other ID-based lookups

**Error Handling:** Throws on undefined; prevents use of deleted/misnamed items

**File:** `common/src/utils/objectDefinitions.ts`

### `ObjectDefinitions<Def>.writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void`  
**Purpose:** Serialize a definition to a binary stream as its index.

**Implicit Behavior:**
- Converts string ID or definition object to index
- If `overLength` is false: writes 1 byte (uint8)
- If `overLength` is true: writes 2 bytes (uint16)
- Validates ID exists before writing; throws on unknown ID

**Gotcha:** Must use same registry instance on both sender and receiver, or indices will desync (e.g., server sends index 5, client has different definition at index 5)

**File:** `common/src/utils/objectDefinitions.ts`

### `ObjectDefinitions<Def>.readFromStream(stream: ByteStream): Def`  
**Purpose:** Deserialize a definition from a binary stream.

**Implicit Behavior:**
- Reads index (1 or 2 bytes depending on `overLength`)
- Looks up definition by index
- Returns definition or throws if index out of bounds

**Gotcha:** Index must be within bounds; no fallback to default. If packet received from outdated client, read may fail.

**Error Handling:** Throws `RangeError` if index >= definitions.length

**File:** `common/src/utils/objectDefinitions.ts`

## Related Documents

- **Tier 2:** [Object Definitions Subsystem](../README.md) — Registry overview, definition types
- **Tier 1:** [Data Model](../../../datamodel.md) — Object schema and definition structure
- **Tier 1:** [API Reference](../../../api-reference.md) — Binary packet protocol and serialization
- **Patterns:** [Object Definitions Patterns](../patterns.md) — Registry usage patterns
