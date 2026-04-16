# Data Structures Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/utils/*.ts, common/src/utils/*.ts -->

## Purpose

Provides server-side and shared custom data structures optimized for game object management, bidirectional lookups, object pooling, and set operations.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/utils/objectPool.ts` | `ObjectPool<Mapping>` generic pool for typed object storage | High |
| `server/src/utils/misc.ts` | Utility functions (map mode, username cleaning, patterning shapes) | Medium |
| `common/src/utils/misc.ts` | Type utilities (DeepPartial, Result types, cloneDeep) | Medium |
| `server/src/utils/idAllocator.ts` | ID generation / ID recycling (if present) | Low |

## Business Rules

- **Type-Safe Pooling:** `ObjectPool<Mapping>` maintains separate sets per `ObjectCategory` (Player, Bullet, Building, etc.)
- **O(1) Lookup:** Pool stores objects by ID; lookup is Map-based, O(1)
- **Category Separation:** Objects stored in category-specific Sets for efficient iteration by type
- **ExtendedMap Utility:** Server uses custom Map variant with helper methods (`ifPresent`, `getOrCompute`, etc.)
- **Immutable Result Type:** `Result<Res, Err>` union type enforces error handling (no nullable returns)
- **Deep Clone on Demand:** `cloneDeep()` recursively copies objects while preserving Symbols and custom descriptors

## Data Lineage

```
Game Objects Created (Player, Bullet, Building, etc.)
    ↓
Grid.addObject(object) calls ObjectPool.add(object)
    ↓
ObjectPool._objects: Map<id, GameObject> — stores all objects
ObjectPool._byCategory[ObjectCategory.X]: Set<GameObject> — stores by type
    ↓
Queries:
  - pool.get(id) → single object lookup
  - pool.getCategory(ObjectCategory.Player) → all players
                                                                ↓
When object removed → pool.delete(object) removes from both maps
```

## Complex Functions

### `ObjectPool<Mapping>.constructor()`  
**Purpose:** Initialize empty pool with category-specific sets.

**Implicit Behavior:**
- Creates 2D map structure for all object categories
- Initializes Set for each `ObjectCategory` enum value
- Filters enum to avoid double-indexing (numeric + string keys)

**File:** `common/src/utils/objectPool.ts:23–31`

### `ObjectPool<Mapping>.add(object: Mapping[C]): void`  
**Purpose:** Register object in pool and category set.

**Implicit Behavior:**
- Adds to `_objects` map (key = object.id)
- Adds to `_byCategory[object.type]` set for fast iteration by type
- Called by `Grid.addObject()` during spawn

**File:** `common/src/utils/objectPool.ts:37–41`

### `ObjectPool<Mapping>.getCategory<C>(key: C): Set<Mapping[C]>`  
**Purpose:** Get all objects of a specific category (all players, all bullets, etc.).

**Implicit Behavior:**
- Returns the Set for the given ObjectCategory
- Set is empty if no objects of that type exist
- Iteration order is insertion order

**Pattern:** Frequently used in game loops:
```typescript
for (const player of pool.getCategory(ObjectCategory.Player)) {
  player.update();
}
```

**Performance:** O(1) set retrieval + O(n) iteration of objects in set

**File:** `common/src/utils/objectPool.ts:44–46`

### `cloneDeep<T>(object: T): T` — @file common/src/utils/misc.ts  
**Purpose:** Recursively clone an object, preserving types and custom descriptors.

**Implicit Behavior:**
- Primitives returned as-is (effective clone)
- Objects recursively cloned
- Arrays, Maps, Sets receive special handling (preserved subclass)
- Custom Symbol-based clone methods honored (`DeepCloneable`)
- Cyclical references detected via `cloneNodes` map (preservation of cycles)

**Gotcha:** Does not clone Functions (returned as reference). Class instances need `DeepCloneable` interface for custom logic.

**Performance:** O(total properties recursively), may be slow for deep structures

**File:** `common/src/utils/misc.ts:200–280`

## Collection Type Reference

| Type | Purpose | Complexity |
|------|---------|------------|
| `ObjectPool<Mapping>` | Typed pool by ObjectCategory | High |
| `ExtendedMap<K, V>` | Map with utility methods (server) | Medium |
| `Result<Res, Err>` | Discriminated union for error handling | Low |
| `DeepPartial<T>` | Type utility (properties optional recursively) | Low |
| `ReadonlyRecord<K, T>` | Readonly key-value collection | Low |

## WeakMap Usage Patterns

**Current:**
- Plugin data storage (associate data with player without preventing GC)
- Temporary object metadata (hitbox cache, velocity state, etc.)

**Not Current (why):**
- Object lifecycle management (explicit addition/removal via `Grid` pool is clearer)
- Cache invalidation (weakly-held references harder to debug)

## Related Documents

- **Tier 2:** [Server Utilities Subsystem](../README.md) — Utility functions overview
- **Tier 1:** [Data Model](../../../datamodel.md) — Object types and schema
- **Patterns:** [Object Pool Patterns](../patterns.md) — Pool usage and lifecycle
