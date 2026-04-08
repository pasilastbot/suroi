# Definitions Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/definitions/modules/ -->
<!-- @source: common/src/definitions/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Definitions subsystem holds all static game object data: guns, obstacles, buildings, loot items, and more. Every game object type is described by a typed definition object stored in an `ObjectDefinitions` collection. The server and client both read these definitions to simulate and render the game.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/utils/objectDefinitions.ts` | `ObjectDefinitions` class, `DefinitionType` enum, `ObjectDefinition` interface |
| `common/src/definitions/loots.ts` | `Loots` — unified loot/items collection (guns, ammo, melees, etc.) |
| `common/src/definitions/obstacles.ts` | `Obstacles` — all obstacle definitions |
| `common/src/definitions/buildings.ts` | `Buildings` — building definitions |
| `common/src/definitions/modes.ts` | `Modes` — game mode definitions |
| `common/src/definitions/items/*.ts` | Per-type item definitions (guns, melees, throwables, etc.) |
| `common/src/definitions/bullets.ts` | `Bullets` — bullet definitions |
| `common/src/definitions/decals.ts` | `Decals` — decal definitions |
| `common/src/definitions/explosions.ts` | `Explosions` — explosion definitions |
| `common/src/definitions/emotes.ts` | `Emotes` — emote definitions |
| `common/src/definitions/syncedParticles.ts` | `SyncedParticles` — particle definitions |
| `common/src/definitions/mapPings.ts` | `MapPings` — map ping definitions |
| `common/src/definitions/mapIndicators.ts` | `MapIndicators` — map indicator definitions |
| `common/src/definitions/badges.ts` | `Badges` — badge definitions |

## Architecture

```
ObjectDefinition (base)
    ├── idString: string
    ├── name: string
    └── defType: DefinitionType

ItemDefinition extends ObjectDefinition
    └── noDrop?, noSwap?, devItem?, reskins?, mapIndicator?, hideInHUD?

InventoryItemDefinition extends ItemDefinition
    └── fists?, killfeedFrame?, speedMultiplier, wearerAttributes?, etc.

GunDefinition, MeleeDefinition, ThrowableDefinition, etc. extend InventoryItemDefinition
```

Each definition file exports:
1. **Type** — TypeScript interface for the definition shape
2. **Collection** — `ObjectDefinitions<Def>` instance (e.g. `Guns`, `Obstacles`)

## Data Flow

```
Definition file (e.g. guns.ts)
    → ObjectDefinitions instance (e.g. Guns)
    → idStringToDef / idStringToNumber lookup maps
    → Used by: packets (writeToStream/readFromStream), server logic, client rendering
```

Definitions are **read-only** at runtime. They are loaded once when the module is imported.

## Interfaces & Contracts

### ObjectDefinitions API

| Method | Purpose |
|--------|---------|
| `fromString(idString)` | Look up definition by idString (throws if not found) |
| `fromStringSafe(idString)` | Look up definition (returns undefined if not found) |
| `hasString(idString)` | Check if idString exists |
| `writeToStream(stream, def)` | Serialize definition index to byte stream |
| `readFromStream(stream)` | Deserialize definition from byte stream |
| `reify(type)` | Resolve string or definition to definition |

### Reference Types

- `ReferenceTo<T>` — string idString referencing a definition
- `ReferenceOrNull<T>` — `ReferenceTo<T> | NullString`
- `ReferenceOrRandom<T>` — reference or weighted random options
- `ReifiableDef<T>` — `ReferenceTo<T> | T` (string or definition object)

## Protocol Considerations

- **Affects protocol:** Yes. Adding or removing definitions changes indices. Bump `GameConstants.protocolVersion`.
- **Serialization:** Definitions are serialized by **index** (1 byte if ≤255 definitions, 2 bytes if >255). Order in the array matters.
- **Validation:** Run `bun validateDefinitions` after any change.

## Dependencies

- **Depends on:** `common/src/constants.ts` (enums, GameConstants), `common/src/utils/` (hitbox, vector, misc)
- **Depended on by:** Packets (serialization), Object Model (runtime objects), Server (game logic), Client (rendering)

## Module Index (Tier 3)

For implementation details, see:

- [Items](modules/items.md) — Guns, melees, throwables, healing, armor, backpacks, scopes, skins, perks
- [Buildings](modules/buildings.md) — Building definitions and structure
- [Obstacles](modules/obstacles.md) — Obstacle definitions, materials, loot tables
- [Bullets](modules/bullets.md) — Derived from Guns/Explosions, ballistics, tracer
- [Modes](modules/modes.md) — Game mode config, colors, spritesheets, features
- [Explosions](modules/explosions.md) — Damage, radius, shrapnel, decals, camera shake
- [SyncedParticles](modules/synced-particles.md) — Server particles, ValueSpecifier, Animated
- [Decals](modules/decals.md) — Ground marks, explosion residue, healing residue
- [Emotes](modules/emotes.md) — Emote categories, loadout slots, badge overlap

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview
- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — DefinitionType enum, ObjectCategory
- **Tier 2:** [../packets/](../packets/) — How definitions are serialized in packets
- **Tier 2:** [../objects/](../objects/) — How definitions are used by runtime objects
