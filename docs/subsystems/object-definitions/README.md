# Object Definitions Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/object-definitions/modules/ -->
<!-- @source: common/src/definitions/, common/src/utils/objectDefinitions.ts -->

## Purpose

The Object Definitions subsystem is the **single source of truth for all static game-entity data**. Every gun, obstacle, building, item, emote, and other game concept is represented as a plain TypeScript object stored in a typed `ObjectDefinitions<T>` registry. The registry provides O(1) lookup by `idString`, and — critically — deterministic numeric indices used for compact binary serialization over the WebSocket wire.

Nothing in this subsystem has runtime side effects: it contains no server or game logic, only pure data and the `ObjectDefinitions<T>` container class.

---

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/utils/objectDefinitions.ts` | `ObjectDefinitions<T>` class, `DefinitionType` enum, shared definition interfaces (`ObjectDefinition`, `ItemDefinition`, `InventoryItemDefinition`, `WearerAttributes`, `ReferenceTo<T>`, etc.) |
| `common/src/definitions/items/guns.ts` | `Guns` registry, `GunDefinition`, `Tier` enum, `InventoryItemDefinitions` subclass usage |
| `common/src/definitions/items/ammos.ts` | `Ammos` registry, `AmmoDefinition` |
| `common/src/definitions/items/armors.ts` | `Armors` registry, `ArmorDefinition`, `ArmorType` enum |
| `common/src/definitions/items/backpacks.ts` | `Backpacks` registry, `BackpackDefinition` |
| `common/src/definitions/items/healingItems.ts` | `HealingItems` registry, `HealingItemDefinition`, `HealType` enum |
| `common/src/definitions/items/melees.ts` | `Melees` registry, `MeleeDefinition` |
| `common/src/definitions/items/perks.ts` | `Perks` registry, `PerkDefinition`, `PerkIds` enum, `PerkCategories` enum |
| `common/src/definitions/items/scopes.ts` | `Scopes` registry, `ScopeDefinition` |
| `common/src/definitions/items/skins.ts` | `Skins` registry, `SkinDefinition` |
| `common/src/definitions/items/throwables.ts` | `Throwables` registry, `ThrowableDefinition` |
| `common/src/definitions/items/items.ts` | `InventoryItemDefinitions<T>` subclass (applies speed multipliers at construction) |
| `common/src/definitions/obstacles.ts` | `Obstacles` registry, `ObstacleDefinition`, `Materials` array |
| `common/src/definitions/buildings.ts` | `Buildings` registry, `BuildingDefinition`, `BuildingObstacle`, `SubBuilding`, etc. |
| `common/src/definitions/bullets.ts` | `Bullets` registry, `BulletDefinition` — derived at startup from `Guns` + `Explosions` |
| `common/src/definitions/explosions.ts` | `Explosions` registry, `ExplosionDefinition` |
| `common/src/definitions/emotes.ts` | `Emotes` registry, `EmoteDefinition`, `EmoteCategory` enum |
| `common/src/definitions/badges.ts` | `Badges` registry, `BadgeDefinition` |
| `common/src/definitions/decals.ts` | `Decals` registry, `DecalDefinition` |
| `common/src/definitions/loots.ts` | `Loots` combined registry, `LootDefinition` union type, `WeaponDefinition`, `ItemType` |
| `common/src/definitions/mapPings.ts` | `MapPings` registry, `MapPingDefinition`, `PlayerPing` / `MapPing` subtypes |
| `common/src/definitions/mapIndicators.ts` | `MapIndicators` registry, `MapIndicatorDefinition` |
| `common/src/definitions/syncedParticles.ts` | `SyncedParticles` registry, `SyncedParticleDefinition` |
| `common/src/definitions/modes.ts` | `Modes` plain record (NOT an `ObjectDefinitions` instance), `ModeDefinition`, `ModeName` |
| `common/src/utils/baseBullet.ts` | `BaseBulletDefinition` — shared bullet ballistics fields used by `GunDefinition` and `ExplosionDefinition` |

---

## Architecture

All definitions follow the same pattern:

1. **Static array** — Definitions are written as plain TypeScript object literals in a `const` array.
2. **`ObjectDefinitions<T>` constructor** — The array is passed into `new ObjectDefinitions<T>(array)`, which builds two internal maps: `idStringToDef` (`idString → Def`) and `idStringToNumber` (`idString → numeric index`). The numeric index is simply the array position (0-based).
3. **Named registry export** — A module-level `const Guns = new ObjectDefinitions<GunDefinition>([...])` (for example) is exported and consumed everywhere.
4. **`overLength` flag** — If the registry holds more than 255 entries, `overLength` is `true` and the stream codec uses `uint16` (2 bytes) instead of `uint8` (1 byte).

### Special Registries

- **`Loots`** — A meta-registry that merges all item registries (`Guns`, `Ammos`, `Melees`, `Throwables`, `HealingItems`, `Armors`, `Backpacks`, `Scopes`, `Skins`, `Perks`) into one `ObjectDefinitions<LootDefinition>`. Used by the server to serialize any loot drop regardless of category.
- **`Bullets`** — Dynamically derived at module initialization time from `Guns.definitions` and `Explosions.definitions`. One `BulletDefinition` entry is generated per gun/explosion (dual variants excluded). It is not hand-authored.
- **`Modes`** — A plain `Record<ModeName, ModeDefinition>` (not an `ObjectDefinitions` instance). Mode selection is never transmitted as a binary index; it is sent as a `ModeName` string.
- **`InventoryItemDefinitions<T>`** (in `items.ts`) — A subclass of `ObjectDefinitions<T>` that multiplies each weapon's `speedMultiplier` by `GameConstants.defaultSpeedModifiers[defType]` at construction, so every consumer sees the final combined value.

---

## The `ObjectDefinitions<T>` Class

`@file common/src/utils/objectDefinitions.ts`

```typescript
export class ObjectDefinitions<Def extends ObjectDefinition = ObjectDefinition> {
    readonly definitions: readonly Def[];
    readonly idStringToDef: ReadonlyRecord<string, Def>;
    readonly idStringToNumber: ReadonlyRecord<string, number>;
    readonly overLength: boolean; // true when entry count > 255 (uses uint16 on wire)
}
```

### Constructor

```typescript
constructor(definitions: readonly Def[])
```

Iterates the array once, populating `idStringToDef` and `idStringToNumber`. Throws `Error` on a duplicate `idString`. Sets `overLength = (count > 255)`.

### Public Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `reify` | `reify<U extends Def>(type: ReifiableDef<Def>): U` | Accepts either a plain `Def` object or an `idString` string and always returns the resolved `Def`. Used where callers might have either form. |
| `fromString` | `fromString<Spec extends Def>(idString: ReferenceTo<Spec>): Spec` | Looks up a definition by `idString`. Throws `ReferenceError` if not found. |
| `fromStringSafe` | `fromStringSafe<Spec extends Def>(idString: ReferenceTo<Spec>): Spec \| undefined` | Same as `fromString` but returns `undefined` instead of throwing. |
| `hasString` | `hasString(idString: string): boolean` | Returns `true` if the `idString` exists in this registry. |
| `writeToStream` | `writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void` | Writes the definition's numeric index to a `ByteStream` — 1 byte if `overLength` is `false`, 2 bytes otherwise. |
| `readFromStream` | `readFromStream<Spec extends Def>(stream: ByteStream): Spec` | Reads a 1- or 2-byte index from a `ByteStream` and returns the corresponding definition. Throws `RangeError` for an out-of-bounds index. |
| `[Symbol.iterator]` | `(): Iterator<Def>` | Delegates to `definitions[Symbol.iterator]()` so the registry can be spread with `...registry` or iterated with `for...of`. |

---

## Definition Interfaces

### `ObjectDefinition` (base for all)

`@file common/src/utils/objectDefinitions.ts`

```typescript
export interface ObjectDefinition {
    readonly idString: string      // kebab-case unique key, used in code and HJSON configs
    readonly name: string          // human-readable display name
    readonly defType: DefinitionType
}
```

### `ItemDefinition` (extends `ObjectDefinition`)

```typescript
export interface ItemDefinition extends ObjectDefinition {
    readonly noDrop?: boolean        // item cannot be dropped from inventory
    readonly noSwap?: boolean        // item cannot be swapped out
    readonly devItem?: boolean       // only available in dev builds
    readonly reskins?: ModeName[]    // mode-specific alternative sprites to load
    readonly mapIndicator?: string   // idString of a MapIndicatorDefinition to show on minimap
    readonly hideInHUD?: boolean     // suppresses item from the HUD
}
```

### `InventoryItemDefinition` (extends `ItemDefinition`)

```typescript
export interface InventoryItemDefinition extends ItemDefinition {
    readonly fists?: {
        readonly left: Vector
        readonly right: Vector
    }
    readonly killfeedFrame?: string
    readonly translationString?: string
    readonly lootAndKillfeedTranslationString?: boolean
    readonly killstreak?: boolean
    readonly speedMultiplier: number   // stored raw; InventoryItemDefinitions subclass multiplies by defaultSpeedModifiers
    readonly wearerAttributes?: {
        readonly passive?: WearerAttributes
        readonly active?: WearerAttributes
        readonly on?: Partial<EventModifiers>
    }
}
```

### `WearerAttributes`

Multiplicative modifiers (applied as `× multiplier`): `maxHealth`, `maxAdrenaline`, `minAdrenaline`, `baseSpeed/speedBoost`, `sizeMod`, `adrenDrain`.  
Additive modifiers: `hpRegen`.

### `GunDefinition`

`@file common/src/definitions/items/guns.ts`

```typescript
type BaseGunDefinition = InventoryItemDefinition & {
    readonly defType: DefinitionType.Gun
    readonly ammoType: ReferenceTo<AmmoDefinition>
    readonly ammoSpawnAmount: number
    readonly tier: Tier                      // S | A | B | C | D
    readonly spawnScope?: ReferenceTo<ScopeDefinition>
    readonly capacity: number
    readonly extendedCapacity?: number
    readonly reloadTime: number
    readonly shotsPerReload?: number
    readonly infiniteAmmo?: boolean
    readonly fireDelay: number               // ms between shots
    readonly switchDelay: number             // ms weapon-switch penalty
    readonly recoilMultiplier: number
    readonly recoilDuration: number
    readonly shotSpread: number
    readonly moveSpread: number
    readonly bulletOffset?: number
    readonly bulletOffsets?: number[]
    readonly fsaReset?: number               // first-shot accuracy reset (ms)
    readonly jitterRadius?: number
    readonly consistentPatterning?: boolean
    readonly noQuickswitch?: boolean
    readonly bulletCount?: number
    readonly length: number
    readonly shootOnRelease?: boolean
    readonly summonAirdrop?: boolean
    readonly fists: { readonly animationDuration: number; ... }
    readonly casingParticles?: Array<{ ... }>
    readonly gasParticles?: { ... }
    readonly image: { readonly angle?: number; readonly zIndex?: number; ... }
    readonly cameraShake?: { readonly duration: number; readonly intensity: number }
    readonly backblast?: { ... }
    readonly cycle?: { readonly delay: number; readonly shotsRequired: number }
    readonly noMuzzleFlash?: boolean
    readonly ballistics: BaseBulletDefinition
    // + ReloadOnEmptyMixin | BurstFireMixin | DualDefMixin
}

// Final exported type includes dual variant link:
export type GunDefinition = BaseGunDefinition & { readonly dualVariant?: ReferenceTo<GunDefinition> };
```

Dual guns are generated automatically: the `dual` sub-block in a `RawGunDefinition` causes a second entry to be inserted with `isDual: true` and `singleVariant` pointing back to the base.

### `AmmoDefinition`

`@file common/src/definitions/items/ammos.ts`

```typescript
export interface AmmoDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Ammo
    readonly maxStackSize: number
    readonly minDropAmount: number
    readonly characteristicColor: { readonly hue: number; readonly saturation: number; readonly lightness: number }
    readonly ephemeral?: boolean           // all players start with max; never depleted or dropped; hidden from HUD
    readonly defaultCasingFrame?: string
    readonly hideUnlessPresent?: boolean
}
```

### `ArmorDefinition`

`@file common/src/definitions/items/armors.ts`

```typescript
export type ArmorDefinition = ItemDefinition & {
    readonly defType: DefinitionType.Armor
    readonly level: number
    readonly damageReduction: number       // fraction of incoming damage absorbed (0–1)
    readonly perk?: ReferenceTo<PerkDefinition>
    readonly positionOverride?: number
    readonly positionOverrideDowned?: number
    readonly emitSound?: string
} & (
    | { readonly armorType: ArmorType.Helmet }
    | { readonly armorType: ArmorType.Vest; readonly color: number; readonly worldImage?: string }
);
```

### `BackpackDefinition`

`@file common/src/definitions/items/backpacks.ts`

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

### `HealingItemDefinition`

`@file common/src/definitions/items/healingItems.ts`

```typescript
export interface HealingItemDefinition extends ItemDefinition {
    readonly defType: DefinitionType.HealingItem
    readonly healType: HealType           // Health | Adrenaline | Special
    readonly restoreAmount: number
    readonly useTime: number              // seconds
    readonly effect?: {
        readonly removePerk: PerkIds
        readonly restoreAmounts?: Array<{ healType: HealType; restoreAmount: number }>
    }
    readonly hideUnlessPresent?: boolean
}
```

### `MeleeDefinition`

`@file common/src/definitions/items/melees.ts`

```typescript
export interface MeleeDefinition extends InventoryItemDefinition {
    readonly defType: DefinitionType.Melee
    readonly tier: Tier
    readonly maxHardness?: number          // max obstacle hardness this weapon can damage (undefined = cannot damage hardened obstacles)
    readonly fireMode?: FireMode
    readonly damage: number
    readonly obstacleMultiplier: number
    readonly piercingMultiplier?: number
    readonly stonePiercing?: boolean
    readonly iceMultiplier?: number        // default 0.01
    readonly swingSound?: string
    readonly stopSound?: string
    readonly hitSound?: string
    readonly switchSound?: string
    readonly hitDelay?: number
    readonly radius: number
    readonly offset: Vector
    readonly cooldown: number
    readonly attackCooldown?: number
    readonly maxTargets?: number
    readonly numberOfHits?: number
    readonly delayBetweenHits?: number
    readonly fists: { readonly animationDuration: number; readonly randomFist?: boolean; ... }
    readonly image?: { readonly position: Vector; readonly angle?: number; ... }
    readonly animation: ReadonlyArray<{ readonly duration: number; readonly fists: { left: Vector; right: Vector }; readonly image?: { position: Vector; angle: number } }>
    readonly reflectiveSurface?: { readonly pointA: Vector; readonly pointB: Vector }
    readonly onBack?: { readonly angle: number; readonly position: Vector; ... }
}
```

### `ThrowableDefinition`

`@file common/src/definitions/items/throwables.ts`

```typescript
export type ThrowableDefinition = InventoryItemDefinition & {
    readonly defType: DefinitionType.Throwable
    readonly tier: Tier
    readonly fuseTime: number              // ms until detonation after throw
    readonly cookTime: number
    readonly throwTime: number
    readonly cookable: boolean
    readonly fireDelay: number
    readonly health?: number
    readonly noSkin?: boolean
    readonly cookSpeedMultiplier: number
    readonly physics: {
        readonly maxThrowDistance: number
        readonly initialZVelocity: number
        readonly initialAngularVelocity: number
        readonly initialHeight: number
        readonly noSpin?: boolean
        readonly drag?: { air: number; ground: number; water: number; ice?: number }
    }
    readonly image: { readonly position: Vector; readonly angle?: number; readonly zIndex: number; readonly anchor?: Vector }
    readonly hitboxRadius: number
    readonly impactDamage?: number
    readonly obstacleMultiplier?: number
    readonly detonation: {
        readonly explosion?: ReferenceTo<ExplosionDefinition>
        readonly particles?: ReferenceTo<SyncedParticleDefinition>
        readonly decal?: ReferenceTo<DecalDefinition>
    }
    readonly animation: { readonly pinImage?: string; readonly liveImage: string; readonly cook: { leftFist: Vector; rightFist: Vector }; readonly throw: { leftFist: Vector; rightFist: Vector } }
    readonly c4?: boolean
    readonly summonAirdrop?: boolean
    readonly maxSwapCount?: number
}
```

### `ScopeDefinition`

`@file common/src/definitions/items/scopes.ts`

```typescript
export interface ScopeDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Scope
    readonly zoomLevel: number
    readonly giveByDefault?: boolean
}
```

### `SkinDefinition`

`@file common/src/definitions/items/skins.ts`

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

### `PerkDefinition`

`@file common/src/definitions/items/perks.ts`

Each perk is a plain object satisfying `BasePerkDefinition & { [perk-specific fields] }`. The actual exported type `PerkDefinition` is computed as `LoosenNumerics<typeof perks[number]> & BasePerkDefinition` — meaning every numeric literal is widened to `number` so consumers don't have to match exact values.

```typescript
interface BasePerkDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Perk
    readonly category: PerkCategories      // Normal | Halloween | Hunted | Infection
    readonly mechanical?: boolean
    readonly updateInterval?: number
    readonly quality?: PerkQualities       // "positive" | "neutral" | "negative"
    readonly noSwap?: boolean
    readonly alwaysAllowSwap?: boolean
    readonly plumpkinGambleIgnore?: boolean
    readonly infectedEffectIgnore?: boolean
}
```

### `ObstacleDefinition`

`@file common/src/definitions/obstacles.ts`

```typescript
type CommonObstacleDefinition = ObjectDefinition & {
    readonly defType: DefinitionType.Obstacle
    readonly hardness?: number
    readonly material: typeof Materials[number]   // "tree" | "stone" | "bush" | "crate" | "metal_light" | "metal_heavy" | "wood" | "pumpkin" | "glass" | "porcelain" | "cardboard" | "appliance" | ...
    readonly health: number
    readonly indestructible?: boolean
    readonly impenetrable?: boolean
    readonly noHitEffect?: boolean
    readonly noDestroyEffect?: boolean
    readonly noResidue?: boolean
    readonly invisible?: boolean
    readonly hideOnMap?: boolean
    readonly scale?: { readonly spawnMin: number; readonly spawnMax: number; readonly destroy: number }
    readonly hitbox: Hitbox
    readonly spawnHitbox?: Hitbox
    readonly noCollisions?: boolean
    readonly rotationMode: RotationMode
    readonly particleVariations?: number
    readonly zIndex?: ZIndexes
    readonly allowFlyover?: FlyoverPref
    readonly collideWithLayers?: Layers
    readonly visibleFromLayers?: Layers
    readonly hasLoot?: boolean
    readonly lootTable?: string
    readonly spawnWithLoot?: boolean
    readonly explosion?: string
    readonly detector?: boolean
    readonly noMeleeCollision?: boolean
    readonly noBulletCollision?: boolean
    readonly reflectBullets?: boolean
    readonly damage?: number
    readonly animationFrames?: string[]
    readonly regenerateAfterDestroyed?: number
    readonly frames?: { base?: string; particle?: string; residue?: string; ... }
    readonly wall?: { readonly color: number; readonly borderColor: number }
    readonly spawnMode?: MapObjectSpawnMode
    readonly tint?: number
    // + TreeMixin | DoorMixin | StairMixin | ActivatableMixin
}
```

### `BuildingDefinition`

`@file common/src/definitions/buildings.ts`

```typescript
export interface BuildingDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Building
    readonly spawnHitbox: Hitbox
    readonly hitbox?: Hitbox
    readonly ceilingHitbox?: Hitbox
    readonly ceilingScope?: ReferenceTo<ScopeDefinition>
    readonly noCollisions?: boolean
    readonly noBulletCollision?: boolean
    readonly reflectBullets?: boolean
    readonly hasDamagedCeiling?: boolean
    readonly collideWithLayers?: Layers
    readonly visibleFromLayers?: Layers
    readonly hideOnMap?: boolean
    readonly rotationMode?: RotationMode.Limited | RotationMode.Binary | RotationMode.None
    readonly spawnMode?: MapObjectSpawnMode
    readonly obstacles?: readonly BuildingObstacle[]
    readonly lootSpawners?: readonly LootSpawner[]
    readonly subBuildings?: readonly SubBuilding[]
    readonly floorImages?: readonly BuildingImageDefinition[]
    readonly ceilingImages?: readonly BuildingImageDefinition[]
    readonly floors?: ReadonlyArray<{ readonly type: FloorNames; readonly hitbox: Hitbox; readonly layer?: number }>
    readonly graphics?: readonly BuildingGraphicsDefinition[]
    readonly puzzle?: { readonly triggerOnSolve?: ReferenceTo<ObstacleDefinition>; readonly delay: number; ... }
    readonly sounds?: { normal?: string; solved?: string | null; maxRange: number; falloff: number; position?: Vector }
    readonly wallsToDestroy?: number
    readonly hasSecondFloor?: boolean
    // ... many more optional fields
}
```

### `ExplosionDefinition`

`@file common/src/definitions/explosions.ts`

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

### `BulletDefinition`

`@file common/src/definitions/bullets.ts`

```typescript
export type BulletDefinition = BaseBulletDefinition & ObjectDefinition & {
    readonly defType: DefinitionType.Bullet
};
```

`Bullets` is auto-generated at module init: for each gun (excluding duals) and each explosion, a `BulletDefinition` is synthesized with `idString = "${source}_bullet"` and ballistics copied from `source.ballistics`. Tracer colours are computed from the ammo type if not overridden.

### `EmoteDefinition`

`@file common/src/definitions/emotes.ts`

```typescript
export interface EmoteDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Emote
    readonly category: EmoteCategory    // People | Text | Memes | Icons | Misc | Team | Weapon
    readonly hideInLoadout?: boolean
}
```

The `Emotes` registry is partly hand-authored (People, Icons, Memes, Text, Misc categories) and partly derived: Team emotes are generated from `Ammos` + `HealingItems`, and Weapon emotes from `Guns` + `Melees` + `Throwables`. All are merged at module init. The `isEmoteBadge` / `getBadgeIdString` helpers detect whether a badge maps to an existing emote (badges prefixed with `bdg_`).

### `BadgeDefinition`

`@file common/src/definitions/badges.ts`

```typescript
export interface BadgeDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Badge
    readonly roles?: readonly string[]     // server role IDs that are allowed to display this badge
}
```

### `DecalDefinition`

`@file common/src/definitions/decals.ts`

```typescript
export interface DecalDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.Decal
    readonly image?: string
    readonly scale?: number                // default 1
    readonly rotationMode: RotationMode
    readonly zIndex?: ZIndexes
    readonly alpha?: number
}
```

Several decals are auto-generated: each `HealingItem` produces a `${idString}_residue` decal.

### `MapPingDefinition`

`@file common/src/definitions/mapPings.ts`

```typescript
export interface MapPingDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.MapPing
    readonly color: number
    readonly showInGame: boolean
    readonly lifetime: number             // seconds
    readonly isPlayerPing: boolean
    readonly ignoreExpiration: boolean
    readonly sound?: string
}
```

### `MapIndicatorDefinition`

`@file common/src/definitions/mapIndicators.ts`

```typescript
export interface MapIndicatorDefinition extends ObjectDefinition {
    readonly defType: DefinitionType.MapIndicator
}
```

### `SyncedParticleDefinition`

`@file common/src/definitions/syncedParticles.ts`

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
    readonly spawner?: { readonly count: number; readonly radius: number; readonly staggering?: { delay: number; initialAmount?: number } }
    readonly hasCreatorID?: boolean
    // + optional hitbox with scope snap-to behaviour
}
```

---

## `DefinitionType` Enum

`@file common/src/utils/objectDefinitions.ts`

```typescript
export enum DefinitionType {
    Ammo,
    Armor,
    Backpack,
    Badge,
    Building,
    Bullet,
    Decal,
    Emote,
    Explosion,
    Gun,
    HealingItem,
    MapPing,
    MapIndicator,
    Melee,
    Obstacle,
    Perk,
    Scope,
    Skin,
    SyncedParticle,
    Throwable
}
```

Every definition object carries `readonly defType: DefinitionType.X` as a discriminant, enabling exhaustive type narrowing in switch statements.

---

## Shared Utility Types

`@file common/src/utils/objectDefinitions.ts`

```typescript
// Brand type for referencing a definition by its idString
export type ReferenceTo<T extends ObjectDefinition> = T["idString"];

// Reference that can also be the sentinel NullString symbol
export type ReferenceOrNull<T extends ObjectDefinition> = ReferenceTo<T> | typeof NullString;

// Either a fixed idString reference OR a weighted-random table { idString: weight, ... }
export type ReferenceOrRandom<T extends ObjectDefinition> = Partial<Record<ReferenceOrNull<T>, number>> | ReferenceTo<T>;

// Either an already-resolved definition object or an idString
export type ReifiableDef<T extends ObjectDefinition> = ReferenceTo<T> | T;

// Sentinel that represents "no definition / not applicable"
export const NullString: unique symbol;
```

---

## Definition Types Registry

| Entity | Registry Export | Definition Interface | Source File | Approx. Entries |
|--------|----------------|---------------------|-------------|----------------|
| Guns | `Guns` | `GunDefinition` | `definitions/items/guns.ts` | ~66 base (+ auto-generated dual variants) |
| Ammos | `Ammos` | `AmmoDefinition` | `definitions/items/ammos.ts` | ~15 |
| Armors | `Armors` | `ArmorDefinition` | `definitions/items/armors.ts` | 8 (4 helmets + 4 vests) |
| Backpacks | `Backpacks` | `BackpackDefinition` | `definitions/items/backpacks.ts` | 5 |
| HealingItems | `HealingItems` | `HealingItemDefinition` | `definitions/items/healingItems.ts` | 5 |
| Melees | `Melees` | `MeleeDefinition` | `definitions/items/melees.ts` | ~18 |
| Perks | `Perks` | `PerkDefinition` | `definitions/items/perks.ts` | 44 |
| Scopes | `Scopes` | `ScopeDefinition` | `definitions/items/scopes.ts` | 5 |
| Skins | `Skins` | `SkinDefinition` | `definitions/items/skins.ts` | ~80+ |
| Throwables | `Throwables` | `ThrowableDefinition` | `definitions/items/throwables.ts` | ~10 |
| Loots | `Loots` | `LootDefinition` (union) | `definitions/loots.ts` | Sum of all item registries |
| Obstacles | `Obstacles` | `ObstacleDefinition` | `definitions/obstacles.ts` | ~200+ (many auto-generated walls) |
| Buildings | `Buildings` | `BuildingDefinition` | `definitions/buildings.ts` | ~100+ (many auto-generated layouts) |
| Bullets | `Bullets` | `BulletDefinition` | `definitions/bullets.ts` | Derived from Guns + Explosions |
| Explosions | `Explosions` | `ExplosionDefinition` | `definitions/explosions.ts` | 24 |
| Emotes | `Emotes` | `EmoteDefinition` | `definitions/emotes.ts` | ~200+ (partly derived from weapon/item lists) |
| Badges | `Badges` | `BadgeDefinition` | `definitions/badges.ts` | ~21 |
| Decals | `Decals` | `DecalDefinition` | `definitions/decals.ts` | ~15 (partly derived from HealingItems) |
| MapPings | `MapPings` | `MapPingDefinition` | `definitions/mapPings.ts` | 6 |
| MapIndicators | `MapIndicators` | `MapIndicatorDefinition` | `definitions/mapIndicators.ts` | 6 |
| SyncedParticles | `SyncedParticles` | `SyncedParticleDefinition` | `definitions/syncedParticles.ts` | 6 |
| Modes | `Modes` (plain `Record`) | `ModeDefinition` | `definitions/modes.ts` | 8 — **not** serialized as binary index |

---

## Binary Serialization

`@file common/src/utils/objectDefinitions.ts`

Definitions are referred to in network packets exclusively by their 0-based array index. The `writeToStream` / `readFromStream` methods implement the codec:

```typescript
writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void {
    const idx = this.idStringToNumber[idString];
    if (this.overLength) {
        stream.writeUint16(idx);  // 2 bytes — used when registry has > 255 entries
    } else {
        stream.writeUint8(idx);   // 1 byte
    }
}

readFromStream<Spec extends Def>(stream: ByteStream): Spec {
    const idx = this.overLength ? stream.readUint16() : stream.readUint8();
    return this.definitions[idx] as Spec;
}
```

**Critical constraint:** The order of entries in each definition array is part of the network protocol. If an entry is inserted, removed, or reordered, **`GameConstants.protocolVersion` must be incremented** (see `common/src/constants.ts:protocolVersion`) so that mismatched clients disconnect cleanly instead of silently reading the wrong definition.

---

## Data Flow

```
Author writes plain TS object literal:
  { idString: "ak47", name: "AK-47", defType: DefinitionType.Gun, ... }

Static array:
  const rawGuns: RawGunDefinition[] = [ ... ]

InventoryItemDefinitions<GunDefinition> constructor:
  multiplies each speedMultiplier by GameConstants.defaultSpeedModifiers[Gun]
  → builds idStringToDef map  (O(n) once at module load)
  → builds idStringToNumber map
  → sets overLength flag

Export:
  export const Guns = new InventoryItemDefinitions<GunDefinition>(rawGuns)

Server usage:
  Guns.fromString("ak47")       → GunDefinition   (O(1))
  Guns.writeToStream(stream, "ak47")  → writes byte index

Client usage:
  Guns.readFromStream(stream)   → GunDefinition   (O(1))

Loot union:
  export const Loots = new ObjectDefinitions<LootDefinition>([
      ...Guns, ...Ammos, ...Melees, ...Throwables, ...HealingItems,
      ...Armors, ...Backpacks, ...Scopes, ...Skins, ...Perks
  ])
```

---

## Dependencies

- **Depends on:** Nothing at runtime — this subsystem is pure data. Only `common/src/utils/byteStream.ts` (for stream codec) and `common/src/utils/math.ts` / `hitbox.ts` / `vector.ts` (for hitbox and position types) are imported.
- **Depended on by:**
  - [Networking](../networking/) — every entity packet serializes and deserializes definitions by numeric index
  - [Game Loop](../game-loop/) — all `BaseGameObject` subclasses hold a reference to their definition
  - [Client Rendering](../client-rendering/) — reads definitions to determine sprite frames, animations, and hitbox sizes
  - [Inventory](../inventory/) — `InventoryItemDefinition` subtypes govern capacity, speed modifiers, and wearerAttributes
  - `common/src/constants.ts` — `GameConstants.lootRadius` and `defaultSpeedModifiers` are keyed by `DefinitionType`

---

## Known Gotchas & Tech Debt

- **`Modes` is not an `ObjectDefinitions`** — It is a plain `Record<ModeName, ModeDefinition>`. Mode selection travels as a string, not a byte index.
- **`Loots` index space vs individual registries** — The `Loots` registry has its own independent index sequence. Do not assume that `Guns.idStringToNumber["ak47"]` equals `Loots.idStringToNumber["ak47"]`.
- **Dual gun generation** — The `dual` block on a `RawGunDefinition` causes a second entry to be inserted immediately after the base entry at construction time. Any array-index-dependent code must account for this.
- **`Bullets` derived registry** — `Bullets` is generated from `Guns.definitions` and `Explosions.definitions` at module init, so its index space depends on the combined order of those two registries. Reordering either affects bullet indices.
- **`PerkDefinition` type widening** — The `LoosenNumerics<...>` mapped type means every numeric field on a perk is typed as `number`, not the specific literal it was declared with. This is intentional to avoid overly tight types in consumers.
- **`overLength` is per-registry** — `Obstacles` and `Buildings` (and `Emotes`, `Skins`) exceed 255 entries and use `uint16` on the wire. Smaller registries use `uint8`. Check `registry.overLength` before assuming byte width.

---

## Module Index (Tier 3)

For implementation details of key registries and serialization, see:

- [Registry Module](modules/registry.md) — `ObjectDefinitions<T>` class, registry patterns, `idStringToDef` / `idStringToNumber` maps

---

## Related Documents

- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — Core entity relationships
- **Tier 1:** [docs/architecture.md](../../architecture.md) — System-wide overview
- **Tier 1:** [docs/api-reference.md](../../api-reference.md) — Binary packet protocol (how indexed definitions are used on the wire)
- **Tier 2:** [Networking](../networking/) — Packet serialization using `writeToStream` / `readFromStream`
- **Tier 2:** [Game Loop](../game-loop/) — How `BaseGameObject` references definitions
- **Tier 2:** [Inventory](../inventory/) — `InventoryItemDefinition` consumption
- **Tier 2:** [Client Rendering](../client-rendering/) — How clients use definitions to render objects
- **Patterns:** [patterns.md](patterns.md) — Reusable patterns for this subsystem
