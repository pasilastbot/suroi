# Data Model

<!-- @tier: 1 -->
<!-- @see-also: docs/subsystems/object-definitions/, docs/subsystems/networking/, docs/subsystems/inventory/ -->

## Overview

This document describes the complete data model for suroi: every runtime entity the game operates on, how those entities are typed in TypeScript, how they are registered in the `ObjectDefinitions` registry, and how they are serialized over the binary WebSocket protocol. Every field name and value in this document was read directly from source files — nothing is invented.

Key source files:
- `common/src/constants.ts` — enumerations and `GameConstants`
- `common/src/typings.ts` — shared interfaces (`PlayerModifiers`, `CustomTeamMessage`, etc.)
- `common/src/defaultInventory.ts` — initial inventory state
- `common/src/utils/objectDefinitions.ts` — `ObjectDefinitions<T>` registry class
- `common/src/utils/vector.ts` — `Vector` interface
- `common/src/utils/hitbox.ts` — hitbox class hierarchy
- `common/src/definitions/` — all definition registries
- `common/src/utils/objectsSerializations.ts` — network serialization types per `ObjectCategory`
- `server/src/objects/player.ts` — runtime `Player` properties

---

## Core Enumerations

All enumerations below come directly from `common/src/constants.ts` unless noted otherwise.

### ObjectCategory

Runtime object categories. Each live game object belongs to exactly one category; the category determines which network serialization schema is used (`ObjectsNetData[Cat]`).

```typescript
export enum ObjectCategory {
    Player,          // 0
    Obstacle,        // 1
    DeathMarker,     // 2
    Loot,            // 3
    Building,        // 4
    Decal,           // 5
    Parachute,       // 6
    Projectile,      // 7
    SyncedParticle   // 8
}
```

### Layer

Vertical layers for indoor/outdoor separation and basement/upstairs areas.

```typescript
export enum Layer {
    Basement  = -2,
    ToBasement = -1,
    Ground    = 0,
    ToUpstairs = 1,
    Upstairs  = 2
}
```

### Layers (collision policy, `const enum`)

```typescript
export const enum Layers {
    All,      // Collide with objects on all layers
    Adjacent, // Collide with objects on the same or adjacent layers
    Equal     // Only collide with objects on the same layer
}
```

### GasState (`const enum`)

```typescript
export const enum GasState {
    Inactive,   // 0
    Waiting,    // 1
    Advancing   // 2
}
```

### TeamMode

```typescript
export enum TeamMode {
    Solo  = 1,
    Duo   = 2,
    Squad = 4
}
```

### ZIndexes

Rendering order for all visual elements (0 = bottom, highest = top).

```typescript
export enum ZIndexes {
    Ground,           // 0
    BuildingsFloor,   // 1
    Decals,           // 2
    DeadObstacles,    // 3
    DeathMarkers,     // 4
    Explosions,       // 5
    ObstaclesLayer1,  // 6  (default obstacle layer)
    Loot,             // 7
    GroundedThrowables, // 8
    ObstaclesLayer2,  // 9
    TeammateName,     // 10
    Bullets,          // 11
    DownedPlayers,    // 12
    Players,          // 13
    ObstaclesLayer3,  // 14  (bushes, tables, etc.)
    AirborneThrowables, // 15
    ObstaclesLayer4,  // 16  (trees)
    BuildingsCeiling, // 17
    ObstaclesLayer5,  // 18  (show on top of ceilings)
    Emotes,           // 19
    Gas               // 20
}
```

### AnimationType (`const enum`)

```typescript
export const enum AnimationType {
    None,           // 0
    Melee,          // 1
    Downed,         // 2
    ThrowableCook,  // 3
    ThrowableThrow, // 4
    GunFire,        // 5
    GunFireAlt,     // 6
    GunClick,       // 7
    Revive          // 8
}
```

### FireMode (`const enum`)

```typescript
export const enum FireMode {
    Single, // 0
    Burst,  // 1
    Auto    // 2
}
```

### InputActions (`const enum`)

```typescript
export const enum InputActions {
    EquipItem,        // 0
    EquipLastItem,    // 1
    EquipOtherWeapon, // 2
    DropWeapon,       // 3
    DropItem,         // 4
    SwapGunSlots,     // 5
    LockSlot,         // 6
    UnlockSlot,       // 7
    ToggleSlotLock,   // 8
    Interact,         // 9
    Reload,           // 10
    Cancel,           // 11
    UseItem,          // 12
    Emote,            // 13
    MapPing,          // 14
    Loot,             // 15
    ExplodeC4         // 16
}
```

### SpectateActions (`const enum`)

```typescript
export const enum SpectateActions {
    BeginSpectating,       // 0
    SpectatePrevious,      // 1
    SpectateNext,          // 2
    SpectateSpecific,      // 3
    SpectateKillLeader,    // 4
    Report                 // 5
}
```

### PlayerActions (`const enum`)

```typescript
export const enum PlayerActions {
    None,    // 0
    Reload,  // 1
    UseItem, // 2
    Revive   // 3
}
```

### InventoryMessages

```typescript
export enum InventoryMessages {
    NotEnoughSpace,       // 0
    ItemAlreadyEquipped,  // 1
    BetterItemEquipped,   // 2
    CannotUseFlare        // 3
}
```

### FlyoverPref

Controls whether throwables can sail over an obstacle.

```typescript
export enum FlyoverPref {
    Always,    // 0 — always allow flyover
    Sometimes, // 1 — allow if throwable velocity > 0.04 u/ms
    Never      // 2 — never allow flyover
}
```

### MapObjectSpawnMode

```typescript
export enum MapObjectSpawnMode {
    Grass,        // 0
    GrassAndSand, // 1 — grass, beach, and river banks
    River,        // 2
    Beach,        // 3
    Trail,        // 4
    Ring          // 5
}
```

### RotationMode

```typescript
export enum RotationMode {
    Full,    // 0 — arbitrary angle
    Limited, // 1 — four cardinal directions (0, 90, 180, 270°)
    Binary,  // 2 — two directions (normal / flipped)
    None     // 3 — no rotation
}
```

---

## Known Issues & Gotchas

- **NetworkID is uint16:** Object network IDs are serialized as `uint16`, limiting theoretical max to ~65,535 simultaneous objects per game. In practice, games do not approach this limit, but extremely large games with thousands of bullets, obstacles, and players could hit this boundary.
- **ObjectCategory determines wire format:** Each object category has a different serialization schema defined in `ObjectSerializations`. Sending an object as the wrong category will deserialize with corrupted data on the client.
- **Layer is int8, not Layer enum:** Layer is serialized as `int8` over the wire, not as the `Layer` enum. -2 (Basement), -1 (ToBasement), 0 (Ground), +1 (ToUpstairs), +2 (Upstairs). Out-of-range values are undefined behavior.
- **Inventory slot indices are hardcoded:** `inventorySlotTypings: [Gun, Gun, Melee, Throwable]` is the immutable slot order. Code frequently assumes slot 0-1 are guns, slot 2 is melee, slot 3 is throwable. Changing this requires versioning all inventory logic.
- **Hitbox types persist across object lifecycle:** Once an object is created with a hitbox type (Circle, Rect, Group, Polygon), changing it requires deserialization/reserialization. No dynamic hitbox swaps mid-game.
- **rotationMode affects wire size:** RotationMode.Full uses 2-byte float; Limited/Binary use 1-byte compact encoding or packed bits. Definition changes from Full→Limited reduce packet size but are not backward compatible without version bump.

## Dependencies on This Document

This document defines the **canonical data model** that all other subsystems serialize, transmit, and render:

- [Object Definitions](docs/subsystems/object-definitions/) — Extends GameConstants with per-definition data; uses enumerations defined here
- [Networking](docs/subsystems/networking/) — Uses ObjectCategory to dispatch serialization, encodes enumerations as the defined integer values
- [Game Loop](docs/subsystems/game-loop/) — Creates and updates objects with enumerations and GameConstants properties
- [Spatial Grid](docs/subsystems/spatial-grid/) — Uses hitbox types and layer collision policies
- [Client Rendering](docs/subsystems/client-rendering/) — Uses ObjectCategory to map objects to renderable classes; uses ZIndexes for display order
- [Inventory](docs/subsystems/inventory/) — Depends on inventory slot types and item definitions
- [Gas System](docs/subsystems/gas/) — Uses GasState enum and GameConstants.gas parameters
- [Map Generation](docs/subsystems/map/) — Uses MapObjectSpawnMode and FlyoverPref enumerations

## Related Documents

### Tier 1 — Architecture & High-Level
- [Project Description](description.md) — Business domain and game features
- [System Architecture](architecture.md) — Process model and deployment structure
- [Development Guide](development.md) — Setup and code standards
- [API Reference](api-reference.md) — Binary encoding of these data types

### Tier 2 — Subsystems
- [Object Definitions](docs/subsystems/object-definitions/) — Extends this model with gun, obstacle, building definitions
- [Networking](docs/subsystems/networking/) — Serializes these types via SuroiByteStream
- [Game Loop](docs/subsystems/game-loop/) — Manages object lifecycle using these types
- [Spatial Grid](docs/subsystems/spatial-grid/) — Collision queries using hitbox types
- [Client Rendering](docs/subsystems/client-rendering/) — Renders objects mapped by ObjectCategory
- [Inventory](docs/subsystems/inventory/) — Manages inventory items and slots
- [Gas System](docs/subsystems/gas/) — State machine using GasState enum
- [Map Generation](docs/subsystems/map/) — Places objects using MapObjectSpawnMode

### Tier 3 — Key Modules
- [SuroiByteStream Encoding](docs/subsystems/networking/modules/stream.md) — Implementation of float→int mapping
- [Hitbox Collision](docs/subsystems/spatial-grid/modules/hitbox.md) — Hitbox type-specific collision tests

## GameConstants Reference

Full `GameConstants` object from `common/src/constants.ts`:

```typescript
export const GameConstants = {
    protocolVersion: 73,        // increment on any byte-stream change
    tps: 40,                    // server ticks per second → 25 ms/tick
    gridSize: 32,               // spatial hash grid cell size (world units)
    maxPosition: 1924,          // map boundary (world units)
    objectMinScale: 0.15,
    objectMaxScale: 3,
    defaultMode: "normal",

    player: {
        radius: 2.25,           // collision circle radius
        baseSpeed: 0.06,
        defaultHealth: 200,
        maxAdrenaline: 100,
        maxShield: 100,
        maxInfection: 100,
        inventorySlotTypings: [Gun, Gun, Melee, Throwable],  // slot→DefinitionType
        maxWeapons: 4,
        nameMaxLength: 16,
        defaultName: "Player",
        defaultSkin: "hazel_jumpsuit",
        killLeaderMinKills: 3,
        maxMouseDist: 256,
        reviveTime: 8,          // seconds
        maxReviveDist: 5,
        bleedOutDPMs: 0.002,    // = 2 dps
        maxPerkCount: 1,
        maxPerks: 4,
        buildingVisionSize: 20,
        rateLimitPunishmentTrigger: 10,
        emotePunishmentTime: 5000,  // ms
        rateLimitInterval: 1000,    // ms
        combatLogTimeoutMs: 12000,
        defaultModifiers: {
            maxHealth: 1,
            maxAdrenaline: 1,
            maxShield: 1,
            baseSpeed: 1,
            size: 1,
            reload: 1,
            fireRate: 1,
            adrenDrain: 1,
            minAdrenaline: 0,
            hpRegen: 0,
            shieldRegen: 0
        },
        ice: {
            acceleration: 2.5,
            friction: 0.2,
            movingFriction: 0.45
        }
    },

    gas: {
        damageScaleFactor: 0.005,  // extra damage per distance unit inside gas
        unscaledDamageDist: 12     // no scaling for first this many units into gas
    },

    lootSpawnMaxJitter: 0.7,
    loot: {
        drag: 0.003,
        iceDrag: 0.0008
    },

    // Loot pickup radius by item type (DefinitionType → world units)
    lootRadius: {
        Gun: 3.4, Ammo: 2, Melee: 3, Throwable: 3,
        HealingItem: 2.5, Armor: 3, Backpack: 3,
        Scope: 3, Skin: 3, Perk: 3
    },

    // Default movement speed penalty by held weapon type
    defaultSpeedModifiers: {
        Gun: 0.88, Melee: 1, Throwable: 0.92
    },

    airdrop: {
        fallTime: 8000,   // ms
        flyTime: 30000,   // ms
        damage: 300
    },

    projectiles: {
        maxHeight: 5,
        gravity: 10,
        distanceToMouseMultiplier: 1.5,
        drag: { air: 0.7, ground: 3, ice: 1, water: 5 }
    },

    explosionMaxDistSquared: 16384,  // 128² world units²
    trailPadding: 384,
    explosionRayDistance: 2
};
```

---

## Core Value Types

### Vector

From `common/src/utils/vector.ts`:

```typescript
export interface Vector {
    x: number
    y: number
}
```

The `Vec` export is a function `(x, y) => Vector` extended with utility methods: `add`, `addComponent`, `sub`, `subComponent`, `scale`, `clone`, `rotate`, `squaredLen`, `len`, `normalize`, `invert`, `dot`, `addAdjust` (adds + rotates by `Orientation`).

### Orientation and Variation

From `common/src/typings.ts`:

```typescript
export type Orientation = 0 | 1 | 2 | 3;
export type Variation  = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7;
```

---

## Hitbox Types

From `common/src/utils/hitbox.ts`.

### HitboxType enum

```typescript
export enum HitboxType {
    Circle,   // 0
    Rect,     // 1
    Group,    // 2
    Polygon   // 3
}
```

### Hitbox type union

```typescript
export type Hitbox = CircleHitbox | RectangleHitbox | GroupHitbox | PolygonHitbox;
export type ShapeHitbox = CircleHitbox | RectangleHitbox; // polygon support pending
```

### HitboxJSONMapping (serializable forms)

```typescript
export interface HitboxJSONMapping {
    [HitboxType.Circle]: {
        readonly type: HitboxType.Circle
        readonly radius: number
        readonly position: Vector
    }
    [HitboxType.Rect]: {
        readonly type: HitboxType.Rect
        readonly min: Vector
        readonly max: Vector
    }
    [HitboxType.Group]: {
        readonly type: HitboxType.Group
        readonly hitboxes: HitboxJSONMapping[HitboxType.Circle | HitboxType.Rect][]
    }
    [HitboxType.Polygon]: {
        readonly type: HitboxType.Polygon
        readonly points: Vector[]
        readonly center: Vector
    }
}
```

### CircleHitbox

```typescript
class CircleHitbox extends BaseHitbox<HitboxType.Circle> {
    readonly type = HitboxType.Circle;
    position: Vector;   // center
    radius: number;
}
```

### RectangleHitbox

```typescript
class RectangleHitbox extends BaseHitbox<HitboxType.Rect> {
    readonly type = HitboxType.Rect;
    min: Vector;   // top-left corner
    max: Vector;   // bottom-right corner
}
```

Static factory methods: `RectangleHitbox.fromLine(a, b)`, `RectangleHitbox.fromRect(width, height, center?)`, `RectangleHitbox.simple(width, height, center?)`.

### GroupHitbox

```typescript
class GroupHitbox extends BaseHitbox<HitboxType.Group> {
    readonly type = HitboxType.Group;
    hitboxes: ShapeHitbox[];
}
```

### PolygonHitbox

```typescript
class PolygonHitbox extends BaseHitbox<HitboxType.Polygon> {
    readonly type = HitboxType.Polygon;
    points: Vector[];
    center: Vector;
}
```

---

## ObjectDefinitions Registry Pattern

From `common/src/utils/objectDefinitions.ts`.

```typescript
export class ObjectDefinitions<Def extends ObjectDefinition = ObjectDefinition> {
    readonly definitions: readonly Def[];
    readonly idStringToDef: ReadonlyRecord<string, Def>;    // O(1) lookup by idString
    readonly idStringToNumber: ReadonlyRecord<string, number>; // O(1) lookup → numeric index
    readonly overLength: boolean;  // true when > 255 entries → uses uint16 in wire format

    // Write definition to binary stream: 1 byte (uint8) if ≤255 entries, else 2 bytes (uint16)
    writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void
    // Read definition from binary stream
    readFromStream<Spec extends Def>(stream: ByteStream): Spec
}
```

**Key rule:** The numeric index of a definition is its position in the `definitions` array. Adding a definition in the middle changes all subsequent indices and requires a `protocolVersion` bump.

### DefinitionType enum

```typescript
export enum DefinitionType {
    Ammo,           // 0
    Armor,          // 1
    Backpack,       // 2
    Badge,          // 3
    Building,       // 4
    Bullet,         // 5
    Decal,          // 6
    Emote,          // 7
    Explosion,      // 8
    Gun,            // 9
    HealingItem,    // 10
    MapPing,        // 11
    MapIndicator,   // 12
    Melee,          // 13
    Obstacle,       // 14
    Perk,           // 15
    Scope,          // 16
    Skin,           // 17
    SyncedParticle, // 18
    Throwable       // 19
}
```

### Base definition interfaces

```typescript
export interface ObjectDefinition {
    readonly idString: string
    readonly name: string
    readonly defType: DefinitionType
}

export interface ItemDefinition extends ObjectDefinition {
    readonly noDrop?: boolean
    readonly noSwap?: boolean
    readonly devItem?: boolean
    readonly reskins?: ModeName[]
    readonly mapIndicator?: string
    readonly hideInHUD?: boolean
}

export interface InventoryItemDefinition extends ItemDefinition {
    readonly fists?: {
        readonly left: Vector
        readonly right: Vector
    }
    readonly killfeedFrame?: string
    readonly translationString?: string
    readonly lootAndKillfeedTranslationString?: boolean
    readonly killstreak?: boolean
    readonly speedMultiplier: number
    readonly wearerAttributes?: {
        readonly passive?: WearerAttributes
        readonly active?: WearerAttributes
        readonly on?: Partial<EventModifiers>
    }
}
```

### WearerAttributes and ExtendedWearerAttributes

```typescript
export interface WearerAttributes {
    readonly maxHealth?: number
    readonly maxAdrenaline?: number
    readonly minAdrenaline?: number
    readonly speedBoost?: number
    readonly sizeMod?: number
    readonly adrenDrain?: number
    readonly hpRegen?: number
}

export interface ExtendedWearerAttributes extends WearerAttributes {
    readonly limit?: number          // upper bound; effect stops reapplying past this
    readonly healthRestored?: number
    readonly adrenalineRestored?: number
}

export interface EventModifiers {
    readonly kill: readonly ExtendedWearerAttributes[]
    readonly damageDealt: readonly ExtendedWearerAttributes[]
}
```

### Reference types

```typescript
export type ReferenceTo<T extends ObjectDefinition> = T["idString"];  // string alias
export type ReferenceOrNull<T extends ObjectDefinition> = ReferenceTo<T> | typeof NullString;
export type ReferenceOrRandom<T extends ObjectDefinition> = Partial<Record<ReferenceOrNull<T>, number>> | ReferenceTo<T>;
export type ReifiableDef<T extends ObjectDefinition> = ReferenceTo<T> | T;
```

---

## Game Entity Definitions

All definition registries live under `common/src/definitions/`.

### Entity Definitions Table

| Entity Type | Registry Export | Definition Interface | `defType` | Primary File |
|-------------|----------------|---------------------|-----------|-------------|
| Gun | `Guns` | `GunDefinition` | `DefinitionType.Gun` | `definitions/items/guns.ts` |
| Ammo | `Ammos` | `AmmoDefinition` | `DefinitionType.Ammo` | `definitions/items/ammos.ts` |
| Melee | `Melees` | `MeleeDefinition` | `DefinitionType.Melee` | `definitions/items/melees.ts` |
| Throwable | `Throwables` | `ThrowableDefinition` | `DefinitionType.Throwable` | `definitions/items/throwables.ts` |
| HealingItem | `HealingItems` | `HealingItemDefinition` | `DefinitionType.HealingItem` | `definitions/items/healingItems.ts` |
| Armor | `Armors` | `ArmorDefinition` | `DefinitionType.Armor` | `definitions/items/armors.ts` |
| Backpack | `Backpacks` | `BackpackDefinition` | `DefinitionType.Backpack` | `definitions/items/backpacks.ts` |
| Scope | `Scopes` | `ScopeDefinition` | `DefinitionType.Scope` | `definitions/items/scopes.ts` |
| Skin | `Skins` | `SkinDefinition` | `DefinitionType.Skin` | `definitions/items/skins.ts` |
| Perk | `Perks` | `PerkDefinition` | `DefinitionType.Perk` | `definitions/items/perks.ts` |
| Obstacle | `Obstacles` | `ObstacleDefinition` | `DefinitionType.Obstacle` | `definitions/obstacles.ts` |
| Building | `Buildings` | `BuildingDefinition` | `DefinitionType.Building` | `definitions/buildings.ts` |
| Decal | `Decals` | `DecalDefinition` | `DefinitionType.Decal` | `definitions/decals.ts` |
| Explosion | `Explosions` | `ExplosionDefinition` | `DefinitionType.Explosion` | `definitions/explosions.ts` |
| Emote | `Emotes` | `EmoteDefinition` | `DefinitionType.Emote` | `definitions/emotes.ts` |
| Badge | `Badges` | `BadgeDefinition` | `DefinitionType.Badge` | `definitions/badges.ts` |
| Bullet | `Bullets` | `BulletDefinition` | `DefinitionType.Bullet` | `definitions/bullets.ts` |
| MapPing | `MapPings` | `MapPingDefinition` | `DefinitionType.MapPing` | `definitions/mapPings.ts` |
| MapIndicator | `MapIndicators` | `MapIndicatorDefinition` | `DefinitionType.MapIndicator` | `definitions/mapIndicators.ts` |
| SyncedParticle | `SyncedParticles` | `SyncedParticleDefinition` | `DefinitionType.SyncedParticle` | `definitions/syncedParticles.ts` |

The `Loots` aggregate registry (`definitions/loots.ts`) combines all ten inventory-item registries into a single `ObjectDefinitions<LootDefinition>` for network serialization of dropped items.

### Loot type aliases (from `definitions/loots.ts`)

```typescript
export type LootDefinition =
    | GunDefinition | AmmoDefinition | MeleeDefinition | HealingItemDefinition
    | ArmorDefinition | BackpackDefinition | ScopeDefinition | SkinDefinition
    | ThrowableDefinition | PerkDefinition;

export type WeaponDefinition = GunDefinition | MeleeDefinition | ThrowableDefinition;
export type WeaponTypes = WeaponDefinition["defType"];

export type ItemType =
    | DefinitionType.Gun | DefinitionType.Ammo | DefinitionType.Melee
    | DefinitionType.Throwable | DefinitionType.HealingItem | DefinitionType.Armor
    | DefinitionType.Backpack | DefinitionType.Scope | DefinitionType.Skin
    | DefinitionType.Perk;
```

---

### AmmoDefinition

```typescript
export interface AmmoDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Ammo
    readonly maxStackSize: number
    readonly minDropAmount: number
    readonly characteristicColor: {
        readonly hue: number
        readonly saturation: number
        readonly lightness: number
    }
    readonly ephemeral?: boolean         // infinite, never shown in HUD, cannot drop
    readonly defaultCasingFrame?: string
    readonly hideUnlessPresent?: boolean
}
```

Ammo types: `12g`, `556mm`, `762mm`, `9mm`, `50cal`, `338lap`, `545mm`, `power_cell`, `flare`, `firework_rocket`, `bb`, `shrapnel` (among others).

### ArmorDefinition

```typescript
export type ArmorDefinition = ItemDefinition & {
    readonly defType: DefinitionType.Armor
    readonly level: number
    readonly damageReduction: number       // fraction 0–1
    readonly perk?: ReferenceTo<PerkDefinition>
    readonly positionOverride?: number
    readonly positionOverrideDowned?: number
    readonly emitSound?: string
} & (
    | { readonly armorType: ArmorType.Helmet }
    | { readonly armorType: ArmorType.Vest; readonly color: number; readonly worldImage?: string }
);

export enum ArmorType {
    Helmet, // 0
    Vest    // 1
}
```

Armors: `basic_helmet` (L1, 10%), `regular_helmet` (L2, 15%), `tactical_helmet` (L3, 20%), `power_helmet` (L4, 25%), and corresponding vest tiers.

### BackpackDefinition

```typescript
export interface BackpackDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Backpack
    readonly level: number
    readonly defaultTint?: number
    readonly maxCapacity: Record<ReferenceTo<HealingItemDefinition | AmmoDefinition | ThrowableDefinition>, number>
    readonly perk?: ReferenceTo<PerkDefinition>
    readonly noTint?: boolean
}
```

Backpacks: `bag` (L0, noDrop), `basic_pack` (L1), `regular_pack` (L2), `tactical_pack` (L3).

### GunDefinition (abbreviated key fields)

```typescript
type BaseGunDefinition = InventoryItemDefinition & {
    readonly defType: DefinitionType.Gun
    readonly ammoType: ReferenceTo<AmmoDefinition>
    readonly ammoSpawnAmount: number
    readonly tier: Tier           // enum: S, A, B, C, D (0–4)
    readonly spawnScope?: ReferenceTo<ScopeDefinition>
    readonly capacity: number
    readonly extendedCapacity?: number
    readonly reloadTime: number
    readonly shotsPerReload?: number
    readonly infiniteAmmo?: boolean
    readonly fireDelay: number
    readonly switchDelay: number
    readonly recoilMultiplier: number
    readonly recoilDuration: number
    readonly shotSpread: number
    readonly moveSpread: number
    readonly bulletCount?: number
    readonly length: number
    // ... (fists, casingParticles, gasParticles, image, ballistics fields)
}

export enum Tier { S, A, B, C, D }  // from guns.ts
```

### MeleeDefinition (key fields)

```typescript
export interface MeleeDefinition extends InventoryItemDefinition {
    readonly defType: DefinitionType.Melee
    readonly tier: Tier
    readonly maxHardness?: number    // can only damage obstacles with hardness ≤ this
    readonly damage: number
    readonly obstacleMultiplier: number
    readonly piercingMultiplier?: number
    readonly stonePiercing?: boolean
    readonly iceMultiplier?: number
    readonly radius: number          // attack hitbox radius
    readonly offset: Vector
    readonly cooldown: number
    readonly attackCooldown?: number
    readonly maxTargets?: number
    readonly numberOfHits?: number
    readonly delayBetweenHits?: number
    readonly animation: ReadonlyArray<{
        readonly duration: number
        readonly fists: { readonly left: Vector; readonly right: Vector }
        readonly image?: { readonly position: Vector; readonly angle: number }
    }>
}
```

### ThrowableDefinition (key fields)

```typescript
export type ThrowableDefinition = InventoryItemDefinition & {
    readonly defType: DefinitionType.Throwable
    readonly tier: Tier
    readonly fuseTime: number         // ms
    readonly cookTime: number         // ms
    readonly throwTime: number        // ms (animation duration)
    readonly cookable: boolean        // if true, cooking runs fuse timer
    readonly fireDelay: number
    readonly health?: number
    readonly cookSpeedMultiplier: number
    readonly physics: {
        readonly maxThrowDistance: number
        readonly initialZVelocity: number
        readonly initialAngularVelocity: number
        readonly initialHeight: number
        readonly noSpin?: boolean
        readonly drag?: { readonly air: number; readonly ground: number; readonly water: number; readonly ice?: number }
    }
    readonly hitboxRadius: number
    readonly impactDamage?: number
    readonly obstacleMultiplier?: number
    readonly detonation: {
        readonly explosion?: ReferenceTo<ExplosionDefinition>
        readonly particles?: ReferenceTo<SyncedParticleDefinition>
        readonly spookyParticles?: ReferenceTo<SyncedParticleDefinition>
        readonly decal?: ReferenceTo<DecalDefinition>
    }
}
```

### HealingItemDefinition

```typescript
export interface HealingItemDefinition extends ItemDefinition {
    readonly defType: DefinitionType.HealingItem
    readonly healType: HealType        // Health | Adrenaline | Special
    readonly restoreAmount: number
    readonly useTime: number           // seconds
    readonly effect?: {
        readonly removePerk: PerkIds
        readonly restoreAmounts?: Array<{ readonly healType: HealType; readonly restoreAmount: number }>
    }
    readonly hideUnlessPresent?: boolean
}

export enum HealType {
    Health,      // 0
    Adrenaline,  // 1
    Special      // 2
}
```

Items: `gauze` (+20 HP, 3s), `medikit` (+100 HP, 6s), `cola` (+25 adren, 3s), `tablets` (+50 adren, 4s), `vaccine_syringe` (removes Infected perk, 2s).

### ScopeDefinition

```typescript
export interface ScopeDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Scope
    readonly zoomLevel: number
    readonly giveByDefault?: boolean
}
```

Scopes: `1x_scope` (zoom 70, default), `2x_scope` (100, default), `4x_scope` (130), `8x_scope` (160), `16x_scope` (220).

### SkinDefinition

```typescript
export interface SkinDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Skin
    readonly hideFromLoadout?: boolean
    readonly grassTint?: boolean
    readonly baseImage?: string
    readonly fistImage?: string
    readonly baseTint?: number
    readonly fistTint?: number
    readonly backpackTint?: number
    readonly hideEquipment?: boolean
    readonly rolesRequired?: string[]
    readonly hideBlood?: boolean
    readonly noSwap?: boolean
    readonly sound?: boolean
}
```

### PerkDefinition

```typescript
interface BasePerkDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Perk
    readonly category: PerkCategories
    readonly mechanical?: boolean
    readonly updateInterval?: number
    readonly quality?: PerkQualities
    readonly noSwap?: boolean
    readonly alwaysAllowSwap?: boolean
    readonly plumpkinGambleIgnore?: boolean
    readonly infectedEffectIgnore?: boolean
}

export const enum PerkCategories {
    Normal,     // 0
    Halloween,  // 1
    Hunted,     // 2
    Infection   // 3
}

export const enum PerkQualities {
    Positive = "positive",
    Neutral  = "neutral",
    Negative = "negative"
}
```

PerkIds string enum (selected values from `definitions/items/perks.ts`):

| PerkIds key | idString value |
|-------------|---------------|
| `SecondWind` | `"second_wind"` |
| `Flechettes` | `"flechettes"` |
| `SabotRounds` | `"sabot_rounds"` |
| `ExtendedMags` | `"extended_mags"` |
| `DemoExpert` | `"demo_expert"` |
| `AdvancedAthletics` | `"advanced_athletics"` |
| `Toploaded` | `"toploaded"` |
| `InfiniteAmmo` | `"infinite_ammo"` |
| `FieldMedic` | `"field_medic"` |
| `Berserker` | `"berserker"` |
| `CloseQuartersCombat` | `"close_quarters_combat"` |
| `LowProfile` | `"low_profile"` |
| `CombatExpert` | `"combat_expert"` |
| `PrecisionRecycling` | `"precision_recycling"` |
| `LootBaron` | `"loot_baron"` |
| `Overclocked` | `"overclocked"` |
| `PlumpkinGamble` | `"plumpkin_gamble"` |
| `PlumpkinShuffle` | `"plumpkin_shuffle"` |
| `Lycanthropy` | `"lycanthropy"` |
| `Bloodthirst` | `"bloodthirst"` |
| `PlumpkinBomb` | `"plumpkin_bomb"` |
| `Shrouded` | `"shrouded"` |
| `EternalMagnetism` | `"eternal_magnetism"` |
| `LastStand` | `"last_stand"` |
| `PlumpkinBlessing` | `"plumpkin_blessing"` |
| `ExperimentalTreatment` | `"experimental_treatment"` |
| `Infected` | `"infected"` |
| `ThermalGoggles` | `"thermal_goggles"` |

### ObstacleDefinition (key fields)

```typescript
type CommonObstacleDefinition = ObjectDefinition & {
    readonly defType: DefinitionType.Obstacle
    readonly hardness?: number
    readonly material: typeof Materials[number]
    readonly health: number
    readonly indestructible?: boolean
    readonly impenetrable?: boolean
    readonly hitbox: Hitbox
    readonly spawnHitbox?: Hitbox
    readonly rotationMode: RotationMode
    readonly scale?: {
        readonly spawnMin: number
        readonly spawnMax: number
        readonly destroy: number
    }
    readonly zIndex?: ZIndexes
    readonly allowFlyover?: FlyoverPref
    readonly collideWithLayers?: Layers
    readonly visibleFromLayers?: Layers
    readonly hasLoot?: boolean
    readonly lootTable?: string
    readonly explosion?: string
    readonly reflectBullets?: boolean
    readonly noMeleeCollision?: boolean
    readonly noBulletCollision?: boolean
    readonly damage?: number          // damage dealt on contact
    readonly requiresPower?: boolean
    // ... (many more optional fields)
}
```

### BuildingDefinition (key fields)

```typescript
export interface BuildingDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Building
    readonly spawnHitbox: Hitbox
    readonly hitbox?: Hitbox
    readonly ceilingHitbox?: Hitbox
    readonly bunkerSpawnHitbox?: Hitbox
    readonly obstacles: readonly BuildingObstacle[]
    readonly subBuildings?: readonly SubBuilding[]
    readonly lootSpawners?: readonly LootSpawner[]
    readonly groundGraphics?: readonly BuildingGraphicsDefinition[]
    readonly ceilingGraphics?: readonly BuildingCeilingGraphics[]
    readonly ceilingImages: readonly BuildingImageDefinition[]
    readonly groundImages: readonly BuildingImageDefinition[]
    readonly rotationMode: RotationMode
    readonly collideWithLayers?: Layers
    readonly visibleFromLayers?: Layers
    readonly allowFlyover?: FlyoverPref   // default: FlyoverPref.Never
    readonly hideOnMap?: boolean
}

export interface BuildingObstacle {
    readonly idString: ReferenceOrRandom<ObstacleDefinition>
    readonly position: Vector
    readonly rotation?: number
    readonly layer?: Layer
    readonly variation?: Variation
    readonly scale?: number
    readonly lootSpawnOffset?: Vector
    readonly puzzlePiece?: string | boolean
    readonly locked?: boolean
    readonly activated?: boolean
    readonly outdoors?: boolean
    readonly waterOverlay?: boolean
    readonly replaceableBy?: ReferenceTo<ObstacleDefinition>
}

export interface SubBuilding {
    readonly idString: ReferenceOrRandom<BuildingDefinition>
    readonly position: Vector
    readonly orientation?: Orientation
    readonly layer?: Layer
    readonly replaceableBy?: ReferenceTo<BuildingDefinition>
}
```

### DecalDefinition

```typescript
export interface DecalDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Decal
    readonly image?: string
    readonly scale?: number         // default: 1
    readonly rotationMode: RotationMode
    readonly zIndex?: ZIndexes
    readonly alpha?: number
}
```

### ExplosionDefinition

```typescript
export interface ExplosionDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Explosion
    readonly damage: number
    readonly obstacleMultiplier: number
    readonly radius: { readonly min: number; readonly max: number }
    readonly cameraShake: { readonly duration: number; readonly intensity: number }
    readonly animation: { readonly duration: number; readonly tint: number | `#${string}`; readonly scale: number }
    readonly sound?: string
    readonly killfeedFrame?: string
    readonly decal?: ReferenceTo<DecalDefinition>
    readonly decalFadeTime?: number
    readonly shrapnelCount: number
    readonly ballistics: Omit<BaseBulletDefinition, "goToMouse" | "lastShotFX">
}
```

### EmoteDefinition

```typescript
export interface EmoteDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Emote
    readonly category: EmoteCategory
    readonly hideInLoadout?: boolean
}

export enum EmoteCategory {
    People, // 0
    Text,   // 1
    Memes,  // 2
    Icons,  // 3
    Misc,   // 4
    Team,   // 5
    Weapon  // 6
}
```

### BadgeDefinition

```typescript
export interface BadgeDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Badge
    readonly roles?: readonly string[]   // role strings that may equip this badge
}
```

### BulletDefinition

Auto-generated from `Guns` and `Explosions` registries. Each gun and explosion produces a `<idString>_bullet` entry.

```typescript
export type BulletDefinition = BaseBulletDefinition & ObjectDefinition & {
    readonly defType: DefinitionType.Bullet
};
```

### MapPingDefinition

```typescript
export interface MapPingDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.MapPing
    readonly color: number
    readonly showInGame: boolean      // add sprite to game camera
    readonly lifetime: number         // seconds
    readonly isPlayerPing: boolean
    readonly ignoreExpiration: boolean
    readonly sound?: string
}
```

Pings: `airdrop_ping`, `unlock_ping` (game pings); `arrow_ping`, `gift_ping`, `heal_ping`, `warning_ping` (player pings).

### MapIndicatorDefinition

```typescript
export interface MapIndicatorDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.MapIndicator
}
```

Indicators: `helmet_indicator`, `vest_indicator`, `pack_indicator`, `juggernaut_indicator`, `player_indicator`, `priority_target`.

### SyncedParticleDefinition (key fields)

```typescript
export type SyncedParticleDefinition = ObjectDefinition & {
    readonly defType: DefinitionType.SyncedParticle
    readonly scale: Animated<number> | NumericSpecifier
    readonly alpha: (Animated<number> & { readonly creatorMult?: number }) | NumericSpecifier
    readonly lifetime: NumericSpecifier
    readonly angularVelocity: NumericSpecifier
    readonly velocity: VectorSpecifier & { readonly easing?: keyof typeof EaseFunctions; readonly duration?: number }
    readonly zIndex: ZIndexes
    readonly frame: string
    readonly tint?: number
    readonly depletePerMs?: { readonly health: number; readonly adrenaline: number }
    readonly hasCreatorID?: boolean
    // optional hitbox and scope-snap fields
}
```

---

## Network Serialization Types (ObjectsNetData)

From `common/src/utils/objectsSerializations.ts`. This interface maps each `ObjectCategory` to its partial (per-tick delta) and full (initial/respawn) network payload shapes.

```typescript
export interface ObjectsNetData {
    [ObjectCategory.Player]: {
        readonly position: Vector
        readonly rotation: number
        readonly animation?: AnimationType
        readonly action?: {
            readonly type: Exclude<PlayerActions, PlayerActions.UseItem>
        } | {
            readonly type: PlayerActions.UseItem
            readonly item: HealingItemDefinition
        }
        readonly full?: {
            readonly layer: Layer
            readonly dead: boolean
            readonly downed: boolean
            readonly beingRevived: boolean
            readonly teamID: number
            readonly invulnerable: boolean
            readonly activeItem: WeaponDefinition
            readonly sizeMod: number
            readonly reloadMod?: number
            readonly skin: SkinDefinition
            readonly helmet?: ArmorDefinition
            readonly vest?: ArmorDefinition
            readonly backpack: BackpackDefinition
            readonly halloweenThrowableSkin: boolean
            readonly activeDisguise?: ObstacleDefinition
            readonly infected: boolean
            readonly backEquippedMelee?: MeleeDefinition
            readonly hasBubble: boolean
            readonly activeOverdrive: boolean
            readonly hasMagneticField: boolean
            readonly isCycling: boolean
            readonly emitLowHealthParticles: boolean
        }
    }

    [ObjectCategory.Obstacle]: {
        readonly scale: number
        readonly dead: boolean
        readonly playMaterialDestroyedSound: boolean
        readonly waterOverlay: boolean
        readonly powered: boolean
        readonly full?: {
            readonly definition: ObstacleDefinition
            readonly position: Vector
            readonly layer: Layer
            readonly rotation: { readonly orientation: Orientation; readonly rotation: number }
            readonly variation?: Variation
            readonly activated?: boolean
            readonly door?: { readonly offset: number; readonly locked: boolean }
        }
    }

    [ObjectCategory.Loot]: {
        readonly position: Vector
        readonly layer: Layer
        readonly full?: {
            readonly definition: LootDefinition
            readonly count: number
            readonly isNew: boolean
        }
    }

    [ObjectCategory.DeathMarker]: {
        readonly position: Vector
        readonly layer: Layer
        readonly playerID: number
        readonly isNew: boolean
    }

    [ObjectCategory.Building]: {
        readonly dead: boolean
        readonly puzzle: undefined | { readonly solved: boolean; readonly errorSeq: boolean }
        readonly layer: number
        readonly full?: {
            readonly definition: BuildingDefinition
            readonly position: Vector
            readonly orientation: Orientation
        }
    }

    [ObjectCategory.Decal]: {
        readonly position: Vector
        readonly rotation: number
        readonly layer: Layer
        readonly definition: DecalDefinition
    }

    [ObjectCategory.Parachute]: {
        readonly height: number
        readonly full?: { readonly position: Vector }
    }

    // Projectile and SyncedParticle entries also exist in the file
}
```

The `full?` sub-object is only sent on initial visibility or when `fullDirtyObjects` flag is set; subsequent ticks only send the non-`full` fields as a delta (partial update) via `partialDirtyObjects`.

---

## Shared Interfaces (typings.ts)

From `common/src/typings.ts`:

```typescript
export type Orientation = 0 | 1 | 2 | 3;
export type Variation  = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7;

export interface PlayerModifiers {
    // Multiplicative modifiers (1.0 = no change)
    maxHealth: number
    maxAdrenaline: number
    maxShield: number
    baseSpeed: number
    size: number
    reload: number
    fireRate: number
    adrenDrain: number

    // Additive modifiers (0 = no change)
    minAdrenaline: number
    hpRegen: number
    shieldRegen: number
}

export interface PunishmentMessage {
    readonly message: "warn" | "temp" | "perma" | "vpn" | "noname"
    readonly reason?: string
    readonly reportID?: string
}

export const enum CustomTeamMessages {
    Join,        // 0
    Update,      // 1
    Settings,    // 2
    KickPlayer,  // 3
    Start,       // 4
    Started,     // 5
    KeepAlive    // 6
}

export interface CustomTeamPlayerInfo {
    id: number
    isLeader?: boolean
    ready: boolean
    name: string
    skin: string
    badge?: string
    nameColor?: number
}

export type CustomTeamMessage =
    | { type: CustomTeamMessages.Join; teamID: string; isLeader: boolean; autoFill: boolean; locked: boolean; forceStart: boolean }
    | { type: CustomTeamMessages.Update; players: CustomTeamPlayerInfo[]; isLeader: boolean; ready: boolean; forceStart: boolean }
    | { type: CustomTeamMessages.Settings; autoFill?: boolean; locked?: boolean; forceStart?: boolean }
    | { type: CustomTeamMessages.KickPlayer; playerId: number }
    | { type: CustomTeamMessages.Start | CustomTeamMessages.Started | CustomTeamMessages.KeepAlive };
```

---

## DefaultInventory

From `common/src/defaultInventory.ts`. The `DEFAULT_INVENTORY` record is built at startup by iterating `HealingItems`, `Ammos`, `Scopes`, and `Throwables`:

- **Ephemeral ammo** (e.g. `power_cell`): initial count = `Infinity`
- **Default scopes** (`1x_scope`, `2x_scope`): initial count = `1`
- **All other items**: initial count = `0`

```typescript
export const DEFAULT_INVENTORY: Record<string, number> = { /* auto-generated */ };
export const itemKeys: readonly string[]  // all item idStrings tracked in inventory
export const itemKeysLength: number
```

---

## Player State

Key properties on the `Player` class at `server/src/objects/player.ts` (first ~200 lines):

```typescript
export interface PlayerSocketData {
    player?: Player
    readonly ip?: string
    readonly teamID?: string
    readonly autoFill: boolean
    readonly noName?: boolean
    readonly role?: string
    readonly isDev: boolean
    readonly nameColor?: number
    readonly lobbyClearing: boolean
    readonly weaponPreset: string
}

class Player extends BaseGameObject.derive(ObjectCategory.Player) {
    // Network budget
    override readonly fullAllocBytes = 16;
    override readonly partialAllocBytes = 12;
    override readonly damageable = true;

    // Collision
    private _hitbox: CircleHitbox;  // radius = GameConstants.player.radius (2.25)

    // Identity
    name: string;
    readonly noName: boolean;
    readonly ip?: string;

    // Cosmetic state
    halloweenThrowableSkin: boolean;
    activeBloodthirstEffect: boolean;
    activeDisguise?: ObstacleDefinition;

    // Combat tracking
    bulletTargetHitCount: number;
    overdriveKills: number;
    activeOverdrive: boolean;
    canUseOverdrive: boolean;

    // Team
    teamID?: number;
    colorIndex: number;

    // Rate limiting
    emoteCount: number;
    lastRateLimitUpdate: number;
    blockEmoting: boolean;

    // Loadout (cosmetics)
    readonly loadout: {
        badge?: BadgeDefinition
        skin: SkinDefinition
        emotes: ReadonlyArray<EmoteDefinition | undefined>
    };

    // Stats (tracked via getters/setters with dirty flags)
    kills: number;
    maxHealth: number;       // default: GameConstants.player.defaultHealth (200)
    health: number;
    normalizedHealth: number;  // health remapped 0..1
    maxAdrenaline: number;   // default: GameConstants.player.maxAdrenaline (100)
    normalizedAdrenaline: number;

    // Lifecycle
    joined: boolean;
    disconnected: boolean;
    team?: Team;
}
```

---

## Mode System

From `common/src/definitions/modes.ts`:

```typescript
export type ModeName =
    | "normal" | "fall" | "halloween" | "infection"
    | "hunted" | "birthday" | "winter" | "nye";

export interface ModeDefinition {
    readonly similarTo?: ModeName
    readonly colors: Record<ColorKeys, string>      // terrain colors in HSL/HSLA
    readonly spriteSheets: readonly SpritesheetNames[]
    readonly ambience?: string
    readonly defaultScope?: ReferenceTo<ScopeDefinition>
    readonly obstacleVariants?: boolean
    readonly weaponSwap?: boolean
    readonly canvasFilters?: { readonly brightness: number; readonly saturation: number }
    readonly bulletTrailAdjust?: string
    readonly particleEffects?: { readonly frames: string | readonly string[]; readonly delay: number; readonly tint?: number; readonly gravity?: boolean }
    // ... (many more optional fields controlling events and special rules)
}
```

---

## Entity Relationships

```
Player
  ├─ hitbox: CircleHitbox (radius 2.25)
  ├─ loadout: { skin: SkinDefinition, badge?: BadgeDefinition, emotes: EmoteDefinition[] }
  ├─ helmet?: ArmorDefinition  (ArmorType.Helmet)
  ├─ vest?:   ArmorDefinition  (ArmorType.Vest)
  ├─ backpack: BackpackDefinition  (level 0–3, determines maxCapacity per item)
  ├─ inventory: Inventory
  │     ├─ slots[0]: GunItem → GunDefinition
  │     ├─ slots[1]: GunItem → GunDefinition
  │     ├─ slots[2]: MeleeItem → MeleeDefinition
  │     ├─ slots[3]: ThrowableItem → ThrowableDefinition
  │     └─ items: Record<idString, count>   (uses DEFAULT_INVENTORY as initial state)
  ├─ perks: PerkDefinition[]  (max 1 active at a time, max 4 total)
  └─ modifiers: PlayerModifiers  (computed from wearerAttributes of all held items + perks)

Loot (in world)
  └─ definition: LootDefinition  (one of the ten Loots registry entries)

Obstacle
  ├─ definition: ObstacleDefinition
  └─ hitbox: Hitbox (from definition)

Building
  ├─ definition: BuildingDefinition
  ├─ obstacles: Obstacle[]   (placed per BuildingObstacle[] in definition)
  └─ subBuildings: Building[]

Projectile
  ├─ definition: BulletDefinition (from Bullets registry)
  └─ position / velocity / layer: runtime physics state

SyncedParticle
  └─ definition: SyncedParticleDefinition

DeathMarker
  └─ playerID: numeric object ID of the dead player
```

---

## Related Documents

- **Tier 1:** [docs/architecture.md](architecture.md) — System overview, packages, build commands
- **Tier 2:** [docs/subsystems/object-definitions/](subsystems/object-definitions/) — `ObjectDefinitions<T>` registry, definition authoring patterns
- **Tier 2:** [docs/subsystems/networking/](subsystems/networking/) — `SuroiByteStream`, `UpdatePacket`, `ObjectsNetData` serialization
- **Tier 2:** [docs/subsystems/inventory/](subsystems/inventory/) — `Inventory`, `GunItem`, `MeleeItem`, `ThrowableItem` runtime management
- **Tier 2:** [docs/subsystems/gas/](subsystems/gas/) — `GasState` lifecycle, damage calculation
