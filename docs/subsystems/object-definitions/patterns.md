# Object Definitions — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/object-definitions/README.md -->

---

## Pattern: Adding a New Entity Type

**When to use:** When introducing a completely new category of game entity that requires its own `DefinitionType` variant and registry (e.g., a new weapon class, a new cosmetic type).

**Implementation:**

1. **Extend `DefinitionType`** in `common/src/utils/objectDefinitions.ts`:
   ```typescript
   export enum DefinitionType {
       // ... existing entries ...
       MyNewType   // add at end to preserve existing numeric values
   }
   ```

2. **Define the interface** — extend `ObjectDefinition` (or `ItemDefinition` / `InventoryItemDefinition` if it is a loot item):
   ```typescript
   export interface MyNewDefinition extends ItemDefinition {
       readonly defType: DefinitionType.MyNewType
       // ... type-specific fields
   }
   ```

3. **Write the static array and instantiate the registry** in a new file `common/src/definitions/myNewThings.ts`:
   ```typescript
   import { DefinitionType, ObjectDefinitions } from "../utils/objectDefinitions";
   import type { MyNewDefinition } from "./myNewThings";

   export const MyNewThings = new ObjectDefinitions<MyNewDefinition>([
       {
           idString: "first_thing",
           name: "First Thing",
           defType: DefinitionType.MyNewType,
           // ... required fields
       }
   ]);
   ```

4. **If it is a loot item**, add it to the `Loots` union in `common/src/definitions/loots.ts`:
   ```typescript
   export type LootDefinition =
       | ... // existing types
       | MyNewDefinition;

   export const Loots = new ObjectDefinitions<LootDefinition>([
       ...Guns, ..., ...MyNewThings
   ]);
   ```

5. **Increment `protocolVersion`** in `common/src/constants.ts` because the binary index space has changed.

**Example files:** `@file common/src/definitions/items/ammos.ts`, `@file common/src/definitions/badges.ts`

---

## Pattern: Adding an Entry to an Existing Registry

**When to use:** Adding a new gun, obstacle, emote, skin, etc. to an existing definition array.

**Implementation:**

1. Append (or insert) a new object literal to the array passed to the `ObjectDefinitions<T>` constructor in the relevant file.
2. Ensure `idString` is globally unique within that registry (the constructor throws `Error` on duplicates).
3. Ensure `defType` matches the registry's `<T>`.
4. **Increment `protocolVersion`** in `common/src/constants.ts` — inserting anywhere other than the end shifts all subsequent numeric indices.

**Key rule:** Appending to the end of the array is the safest change. Inserting in the middle reorders indices for every entry after the insertion point, which is a breaking wire-protocol change.

**Example files:** `@file common/src/definitions/items/guns.ts`, `@file common/src/definitions/obstacles.ts`

---

## Pattern: `ReferenceTo<T>` — Referencing One Definition from Another

**When to use:** When a definition needs to point at another definition (e.g., a gun references its ammo type, an explosion references a decal, a building references an obstacle).

**Implementation:**

`ReferenceTo<T>` is semantically equivalent to `string` but carries the *intent* that it must be a valid `idString` for registry `T`. It is declared as:

```typescript
export type ReferenceTo<T extends ObjectDefinition> = T["idString"];
```

**Author-time usage** (in a definition object):
```typescript
{
    idString: "ak47",
    defType: DefinitionType.Gun,
    ammoType: "762mm" satisfies ReferenceTo<AmmoDefinition>,
    // ...
}
```

**Runtime resolution** — call `reify()` or `fromString()` on the appropriate registry:
```typescript
// Accepts either string or object; always returns the resolved Def
const ammoDef: AmmoDefinition = Ammos.reify(gun.ammoType);

// String-only version, throws ReferenceError if not found
const ammoDef: AmmoDefinition = Ammos.fromString(gun.ammoType);

// Safe version, returns undefined if not found
const ammoDef = Ammos.fromStringSafe(gun.ammoType);
```

**Example files:** `@file common/src/utils/objectDefinitions.ts`, `@file common/src/definitions/items/guns.ts`

---

## Pattern: `ReferenceOrRandom<T>` — Weighted Random Selection

**When to use:** When a map placement or loot spawner should randomly pick from a set of definitions with varying probabilities (used extensively in `BuildingObstacle.idString` and `SubBuilding.idString`).

**Implementation:**

```typescript
export type ReferenceOrRandom<T extends ObjectDefinition> =
    | Partial<Record<ReferenceOrNull<T>, number>>  // weighted table { idString: weight }
    | ReferenceTo<T>;                              // or a fixed idString
```

**Static fixed reference:**
```typescript
const obs: BuildingObstacle = { idString: "barrel", position: Vec(5, 0) };
```

**Weighted random table** — keys are `idString` strings (or `NullString` for "spawn nothing"), values are relative weights:
```typescript
const randomBarrel: BuildingObstacle = {
    idString: { barrel: 3, super_barrel: 1, [NullString]: 0.5 },
    position: Vec(5, 0)
};
```

The map generator resolves `ReferenceOrRandom` at spawn time using the weighted picker in `common/src/utils/random.ts`.

**Example files:** `@file common/src/definitions/buildings.ts`, `@file common/src/definitions/obstacles.ts`

---

## Pattern: `NullString` Sentinel

**When to use:** When a `ReferenceOrRandom` table needs a "spawn nothing" option, or when a reference field must be assignable but intentionally left blank.

**Implementation:**

```typescript
import { NullString } from "common/src/utils/objectDefinitions";

// In a weighted table, NullString means "do not spawn anything here"
idString: { barrel: 2, [NullString]: 1 }

// As a standalone value on a ReferenceOrNull<T> field
const ref: ReferenceOrNull<ObstacleDefinition> = NullString;
```

`NullString` is a `unique symbol`, so it cannot accidentally collide with any real `idString` string. Code that calls `Obstacles.fromString(ref)` must first guard `if (ref !== NullString)`.

**Example files:** `@file common/src/utils/objectDefinitions.ts`

---

## Pattern: Factory Functions for Programmatic Definitions

**When to use:** When many similar definitions differ only in a few fields — for example, all wall variants of a building, all hatchets, or all scopes. Writing a helper function avoids repetition and keeps the array DRY.

**Implementation:**

Any plain function that returns a definition object (or an array of them) and is called inside the registry array can serve as a factory. Examples from the codebase:

```typescript
// scopes.ts — generates all five scope entries from a compact tuple
const Scopes = new ObjectDefinitions<ScopeDefinition>(
    ([ ["1x", 70, true], ["2x", 100, true], ["4x", 130], ["8x", 160], ["16x", 220] ])
    .map(([magnification, zoomLevel, defaultScope]) => ({
        idString: `${magnification}_scope`,
        name: `${magnification} Scope`,
        defType: DefinitionType.Scope,
        noDrop: defaultScope,
        giveByDefault: defaultScope,
        zoomLevel
    }))
);

// obstacles.ts — generates wall variants with a named helper
const houseWalls = (lengthNumber: number): RawObstacleDefinition => ({
    idString: `house_wall_${lengthNumber}`,
    // ...
});

// badges.ts — thin wrapper for common badge shape
const badge = (name: string, roles?: string[]): BadgeDefinition => ({
    idString: `bdg_${name.toLowerCase().split(" ").join("_")}`,
    name,
    defType: DefinitionType.Badge,
    roles
});
```

**Rules:**
- Factory output must satisfy the same `ObjectDefinition` interface as hand-authored entries.
- `idString` must still be unique within the registry — factories that take an identifier argument must ensure uniqueness at the call site.
- If a factory generates N entries, those N indices are contiguous in the array, which is relevant to the `protocolVersion` policy.

**Example files:** `@file common/src/definitions/items/scopes.ts`, `@file common/src/definitions/obstacles.ts`, `@file common/src/definitions/badges.ts`

---

## Pattern: `InventoryItemDefinitions<T>` Subclass

**When to use:** When defining a registry for `InventoryItemDefinition` weapon subtypes (`GunDefinition`, `MeleeDefinition`, `ThrowableDefinition`) so that speed multipliers are pre-resolved.

**Implementation:**

`InventoryItemDefinitions<T>` (in `common/src/definitions/items/items.ts`) is a subclass of `ObjectDefinitions<T>`. Its constructor mutates each entry's `speedMultiplier` field before calling `super()`:

```typescript
export class InventoryItemDefinitions<Def extends WeaponDefinition>
    extends ObjectDefinitions<Def> {

    constructor(definitions: readonly Def[]) {
        super(
            definitions.map(i => {
                (i as Mutable<Def>).speedMultiplier *=
                    GameConstants.defaultSpeedModifiers[i.defType];
                return i;
            })
        );
    }
}
```

This means the `speedMultiplier` stored in each `GunDefinition` / `MeleeDefinition` / `ThrowableDefinition` is already the final combined value (`raw × defaultForType`). Callers must **not** multiply again.

`Guns`, `Melees`, and `Throwables` all use `new InventoryItemDefinitions(...)` instead of `new ObjectDefinitions(...)`.

**Example files:** `@file common/src/definitions/items/items.ts`, `@file common/src/definitions/items/guns.ts`

---

## Pattern: Derived Registries from Existing Ones

**When to use:** When a registry's entries are computed from another registry rather than hand-authored (e.g., `Bullets` derived from `Guns` + `Explosions`, `Decals` partially derived from `HealingItems`, `Emotes` partially derived from weapon lists).

**Implementation:**

The pattern is `.map()` (or `.flatMap()`) over an existing registry's `.definitions` array, transforming each entry into the new definition shape:

```typescript
// bullets.ts — one BulletDefinition per gun/explosion
export const Bullets = new ObjectDefinitions<BulletDefinition>(
    [...Guns.definitions, ...Explosions.definitions]
        .filter(def => !("isDual" in def) || !def.isDual)
        .map(def => ({
            idString: `${def.idString}_bullet`,
            name: `${def.name} Bullet`,
            defType: DefinitionType.Bullet,
            ...def.ballistics,
            tracer: { /* computed color */ }
        }))
);

// decals.ts — residue decal per healing item
...HealingItems.definitions.map(healingItem => ({
    idString: `${healingItem.idString}_residue`,
    name: `${healingItem.name} Residue`,
    defType: DefinitionType.Decal,
    rotationMode: RotationMode.Full
}))
```

**Gotcha:** The derived registry's index space depends on the *order* of entries in the source registry. Adding or reordering entries in the source (e.g., `Guns`) changes indices in the derived registry (e.g., `Bullets`), requiring a `protocolVersion` bump.

**Example files:** `@file common/src/definitions/bullets.ts`, `@file common/src/definitions/emotes.ts`, `@file common/src/definitions/decals.ts`

---

## Pattern: `idString` / Numeric Index Usage Contract

**When to use:** Deciding whether to use an `idString` string or a numeric index when referencing a definition.

**Rule:**
| Context | Use |
|---------|-----|
| Definition cross-references (`ammoType`, `explosion`, `decal`, etc.) | `ReferenceTo<T>` (`idString` string) |
| Config files, HJSON translations, map data files | `idString` string |
| Network packets (server → client, client → server) | Numeric index via `writeToStream` / `readFromStream` |
| Runtime lookups in game logic | `fromString()` or `reify()` |
| Checking whether a definition exists | `hasString()` |

The `idString` form is the canonical identifier in source code. The numeric index is an internal implementation detail of the binary protocol — it must never be hard-coded in game logic.

**Example files:** `@file common/src/utils/objectDefinitions.ts`
