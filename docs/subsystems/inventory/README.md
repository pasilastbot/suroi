# Inventory Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/inventory/modules/ -->
<!-- @source: server/src/inventory/ -->

## Purpose

Manages a player's equipped weapons, stackable items (ammo, healing, scopes, throwables), active action (reload/heal/revive), and the stat modifiers that weapons confer. Everything from pulling the trigger to expending ammo to landing a melee strike passes through this subsystem.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/inventory/inventory.ts` | `Inventory` class — weapon slots, stackable-item collection, pickup/drop/swap logic; `ItemCollection` utility class |
| `server/src/inventory/inventoryItem.ts` | Abstract `InventoryItemBase` (mixin pattern via `.derive()`); abstract `CountableInventoryItem` |
| `server/src/inventory/gunItem.ts` | `GunItem` — firing logic, fire modes, bullet spawning, dual-gun, ammo management |
| `server/src/inventory/meleeItem.ts` | `MeleeItem` — hitbox-based attack, multi-hit, obstacle/player/building/projectile damage |
| `server/src/inventory/throwableItem.ts` | `ThrowableItem` — cook-and-throw state machine, fuse timer, projectile spawn |
| `server/src/inventory/action.ts` | `Action` abstract base + `ReloadAction`, `HealingAction`, `ReviveAction` |
| `common/src/defaultInventory.ts` | `DEFAULT_INVENTORY` record — initial stack counts for every stackable item |

## Architecture

```
Player (server/src/objects/player.ts)
└── Inventory
    ├── weapons: Array<InventoryItem | undefined>   [4 slots]
    │   ├── [0] GunItem (gun slot A)
    │   ├── [1] GunItem (gun slot B)
    │   ├── [2] MeleeItem (melee slot — active by default)
    │   └── [3] ThrowableItem (throwable slot)
    ├── items: ItemCollection                        [ammo, healing, scopes, throwable counts]
    ├── helmet, vest, backpack                       [armor references]
    └── scope                                        [active scope reference]

Player
├── action?: Action                                  [current timed action, if any]
└── (imports all weapon item classes and action classes directly)
```

`GunItem`, `MeleeItem`, and `ThrowableItem` are produced by calling `InventoryItemBase.derive(DefinitionType.X)`, a static mixin method that generates a concrete typed subclass once per category at module load time. This enforces that each weapon class is a registered subtype of `InventoryItemBase` and provides the `isGun` / `isMelee` / `isThrowable` type-predicate properties automatically.

## Inventory Slots

Defined in `GameConstants.player.inventorySlotTypings` (`common/src/constants.ts`):

```typescript
const inventorySlotTypings = Object.freeze(
    [DefinitionType.Gun, DefinitionType.Gun, DefinitionType.Melee, DefinitionType.Throwable] as const
);
```

| Slot index | Type | Notes |
|---|---|---|
| 0 | `DefinitionType.Gun` | Primary gun |
| 1 | `DefinitionType.Gun` | Secondary gun |
| 2 | `DefinitionType.Melee` | Melee weapon — default active slot on spawn |
| 3 | `DefinitionType.Throwable` | Throwable (grenade, etc.) |

`GameConstants.player.maxWeapons = 4` (derived from the length of the array above).

Slots can be individually locked with `Inventory.lock(slot)` / `unlock(slot)`. Lock state is stored as a bitmask (`_lockedSlots`). Locked slots cannot be replaced or dropped unless `force = true` is passed.

## Item Classes

### `InventoryItemBase` (`server/src/inventory/inventoryItem.ts`)

Abstract base for all weapon items. Concrete subclasses are obtained via the `.derive(defType)` mixin — never by extending directly.

Key members:

| Member | Description |
|--------|-------------|
| `owner: Player` | The player this item belongs to |
| `definition` | `LootDefForType<Type>` resolved from an `idString` or definition object |
| `category: Type` | The `DefinitionType` enum value for this weapon |
| `isActive: boolean` | Set by `Inventory.setActiveWeaponIndex()`; fires `inv_item_equip`/`inv_item_unequip` plugin events |
| `stats.kills / stats.damage` | Accumulated kills and damage — trigger `refreshModifiers()` and mark `dirty.weapons` |
| `lastUse: number` | Timestamp of the last firing / use (for cooldown enforcement) |
| `switchDate: number` | Timestamp when this item was last equipped (for switch delay enforcement) |
| `modifiers` | Clone of the internal `PlayerModifiers` this item contributes |
| `refreshModifiers()` | Re-evaluates `wearerAttributes` (passive/active/on-kill/on-damage) and emits `inv_item_modifiers_changed` if changed |
| `_bufferAttack(cooldown, cb)` | Fires `cb` immediately if both cooldown and switch-delay have elapsed; otherwise schedules it up to 200 ms in the future |
| `useItem()` | Abstract — each concrete class implements fire/attack logic |
| `stopUse()` | Called when the player stops attacking or switches away; overridden by `ThrowableItem` to trigger the throw |
| `itemData()` | Abstract — returns `{ kills, damage }` for network serialization |
| `destroy()` | Called when the item instance is discarded; no-op by default |

### `CountableInventoryItem` (`server/src/inventory/inventoryItem.ts`)

Thin abstract subclass of `InventoryItemBase` that adds an abstract `count: number` field. Used by `ThrowableItem` to track how many of that throwable type the player holds.

### `GunItem` (`server/src/inventory/gunItem.ts`)

Represents a firearm.

| Member | Description |
|--------|-------------|
| `ammo: number` | Rounds currently chambered |
| `fireDelay: number` | Current effective fire delay (ms); can be temporarily overridden by the `cycle` mechanic |
| `_consecutiveShots` | Counts shots in the current burst or auto-fire sequence |
| `_shots` | Lifetime shot counter (serialized as `totalShots`) |
| `_altFire` | Toggles for dual guns — alternates left/right barrel each shot |
| `_burstTimeout` | `NodeJS.Timeout` scheduling the next shot in a burst |
| `_autoFireTimeout` | `NodeJS.Timeout` scheduling the next shot in auto mode |
| `_reloadTimeout` | `Timeout` scheduling automatic reload after the last round |

Fire modes (`FireMode` enum from `common/src/constants.ts`):

| Mode | Behaviour |
|---|---|
| `FireMode.Single` | One shot per trigger press; auto-fires on mobile |
| `FireMode.Auto` | Continuous fire via `_autoFireTimeout`; continues while `owner.attacking` |
| `FireMode.Burst` | Fires `burstProperties.shotsPerBurst` rounds then waits `burstProperties.burstCooldown` |

Each trigger pull goes through `_useItemNoDelayCheck(skipAttackCheck)`, which:
1. Guards against dead/downed/disconnected states and empty ammo
2. Cancels the current action
3. Applies perk modifiers (Flechettes, SabotRounds, Toploaded, HollowPoints, etc.) to create a `BulletOptions.modifiers` object
4. Calculates spread, offset, and bullet start position (including wall-clipping check against the spatial grid)
5. Spawns `bulletCount` bullets via `owner.game.addBullet()`
6. Decrements `ammo`; schedules `reload()` if empty; schedules next shot if not `Single` mode

`reload()` creates a `ReloadAction` and assigns it to `player.action`.

Dual guns alternate `_altFire` on every shot and use `definition.leftRightOffset` to space the barrels.

### `MeleeItem` (`server/src/inventory/meleeItem.ts`)

Represents a melee weapon.

`_useItemNoDelayCheck(skipAttackCheck)`:
1. Sets `AnimationType.Melee` on the owner
2. After `definition.hitDelay ?? 50` ms, queries the spatial grid via `getMeleeHitbox()` / `getMeleeTargets()` for objects within reach
3. Iterates hits; applies `obstacleMultiplier` / `piercingMultiplier` / `iceMultiplier`; applies perk damage mods (Berserker, Lycanthropy)
4. Calls `target.damage()` on each hit object
5. If `definition.fireMode === FireMode.Auto` or on mobile, schedules the next swing automatically (using `attackCooldown` if a target was hit, otherwise `cooldown`)

Supports `definition.numberOfHits` and `definition.delayBetweenHits` for weapons that hit multiple times per swing (e.g. dual knives).

### `ThrowableItem` (`server/src/inventory/throwableItem.ts`)

Represents a grenade or throwable. Extends `CountableInventoryItem` — the `count` field tracks how many of that throwable type the player has equipped in slot 3.

State machine:

```
useItem() called
  └─> _useItemNoDelayCheck()
        └─> _cook()       — sets _cooking=true, applies cookSpeedMultiplier recoil,
                            optionally schedules auto-throw at fuseTime
stopUse() called
  └─> _throw()            — resets _cooking, creates Projectile via game.addProjectile(),
                            decrements count, updates dirty.weapons
```

`ThrowableItem` instances are cached in `Inventory.throwableItemMap` (keyed by `idString`) to avoid re-instantiation every time the player swaps between throwable types.

When `count` reaches zero, `removeThrowable()` deletes the map entry and switches to another available throwable, or clears slot 3 if none remain.

## Interfaces & Contracts

### `Inventory` class (`server/src/inventory/inventory.ts`)

Public API exposed to `Player` and other subsystems:

| Method / Property | Type | Purpose |
|---|---|---|
| `weapons: Array<InventoryItem \| undefined>` | property | 4-slot weapon array (guns, melee, throwable) |
| `items: ItemCollection` | property | Stackable item counts (ammo, healing, scopes) |
| `helmet: ArmorItem \| undefined` | property | Equipped helmet armor |
| `vest: ArmorItem \| undefined` | property | Equipped vest armor |
| `backpack: BackpackItem \| undefined` | property | Equipped backpack |
| `scope: ScopeItem \| undefined` | property | Equipped scope |
| `getActiveWeapon()` | method `→ InventoryItem \| undefined` | Returns currently active weapon |
| `setActiveWeaponIndex(index)` | method | Changes active weapon; fires equip/unequip events |
| `addOrDrop(item)` | method | Adds item to inventory or drops at player location |
| `removeItem(slot, count)` | method | Removes count of item from slot |
| `lock(slot) / unlock(slot)` | method | Prevents/allows dropping/swapping that weapon slot |

### `InventoryItemBase` class (mixin)

Generated via `.derive(DefinitionType.X)` for each item category:

| Property | Type | Purpose |
|---|---|---|
| `definition: T` | The resolved item definition |
| `isActive: boolean` | `true` if this item is currently equipped |
| `category: DefinitionType` | Gun \| Melee \| Throwable \| etc. |
| `useItem()` | Abstract — firing, attacking, or throwing logic |
| `stopUse()` | Called when attacking stops or item is unequipped |
| `itemData()` | Returns `{ kills, damage }` for network serialization |

### Events (from plugin system)

| Event | When | Cancellable |
|---|---|---|
| `inv_item_equip` | Weapon equipped | No |
| `inv_item_unequip` | Weapon unequipped | No |
| `inv_item_use` | Weapon/item used (fire/cook/heal) | Yes — cancels action |
| `inv_item_stop_use` | Weapon/item deactivated | Yes |
| `inv_item_stats_changed` | Weapon stats updated (kills/damage) | No |
| `inv_item_modifiers_changed` | Weapon modifiers re-evaluated | No |

---

## Player Health / Status

All values from `GameConstants.player` in `common/src/constants.ts`:

| Stat | Default | Max | Notes |
|------|---------|-----|-------|
| Health | 200 (`defaultHealth`) | Multiplied by `maxHealth` modifier | Weapons can raise/lower max via `wearerAttributes` |
| Adrenaline | 0 | 100 (`maxAdrenaline`) | Drains over time; provides speed/regen benefits |
| Shield | 0 | 100 (`maxShield`) | Used by specific game modes / perks |
| Infection | 0 | 100 (`maxInfection`) | Used by specific game modes |

Modifiers (`PlayerModifiers`) that weapons affect include: `maxHealth`, `maxAdrenaline`, `maxShield`, `baseSpeed`, `size`, `reload`, `fireRate`, `adrenDrain`, `minAdrenaline`, `hpRegen`, `shieldRegen`.

## Item Actions

Actions are timed operations that occupy `player.action`. Only one action can be active at a time — starting a new one (or firing a weapon) calls `action.cancel()` on the previous one.

`Action` (abstract base, `server/src/inventory/action.ts`):

| Member | Description |
|--------|-------------|
| `player: Player` | The acting player |
| `type: PlayerActions` | Enum value broadcast to clients via the update packet |
| `speedMultiplier: number` | Applied to player movement speed; default 1 |
| `cancel()` | Kills the timeout; clears `player.action`; marks partial dirty |
| `execute()` | Runs on timeout completion; clears `player.action`; marks partial dirty |

Concrete actions:

### `ReloadAction`

- Initiated by `GunItem.reload()` or automatically after the last round is fired
- Duration: `definition.reloadTime` (or `fullReloadTime` if `reloadFullOnEmpty` and current ammo is 0), divided by `player.reloadMod`
- On execute: moves ammo from `inventory.items` into `gun.ammo`; respects `shotsPerReload` (partial reload), `extendedCapacity` perk, `InfiniteAmmo` perk
- Chain-reloads until the gun is full, then sets `player.attacking = false`

### `HealingAction`

- Initiated when the player uses a `HealingItem` (bandage, medkit, cola, etc.)
- `speedMultiplier = 0.5`
- Duration: `item.useTime` (ms, converted to seconds), divided by `FieldMedic` perk modifier
- On execute: decrements the item count in `inventory.items`; applies `item.healType`:
  - `HealType.Health` → `player.health += restoreAmount`
  - `HealType.Adrenaline` → `player.adrenaline += restoreAmount`
  - `HealType.Special` → iterates `effect.restoreAmounts`, may also call `player.removePerk()`
- Spawns a residue decal at the player's position

### `ReviveAction`

- Initiated when a standing player begins reviving a downed teammate
- `speedMultiplier = 0.5`
- Duration: `GameConstants.player.reviveTime` (8 s), divided by `FieldMedic` perk modifier on the reviver
- On execute: calls `target.revive()` and resets the reviver's animation
- On cancel: clears `target.beingRevivedBy` and resets both animations

## Ammo System

Ammo is stored in `Inventory.items` (an `ItemCollection`) keyed by the ammo's `idString` (e.g. `"762mm"`, `"12g"`).

- `GunDefinition.ammoType` references the ammo `idString` that gun consumes
- `GunItem.ammo` is the in-chamber round count (max = `definition.capacity`)
- When a gun is fired, `GunItem.ammo` is decremented
- When a gun is reloaded, `ReloadAction.execute()` moves rounds from `inventory.items` into `GunItem.ammo`
- When a gun is dropped, any remaining `GunItem.ammo` is returned to `inventory.items` before the weapon loot is spawned
- Ammo whose `AmmoDefinition.ephemeral === true` is initialized to `Infinity` in `DEFAULT_INVENTORY` — it is never actually consumed

Backpack capacity limits are enforced in `Inventory.giveItem()`: if the incoming amount would exceed `backpack.maxCapacity[itemString]`, the excess is dropped as loot.

## Loot Pickup

Pickup is handled in `Player` (not in the `Inventory` class itself). The player's `InputActions.PickupItem` handler calls methods on `Inventory`:

- `appendWeapon(item)` — inserts into the first free slot of the matching type; returns slot index or `-1`
- `addOrReplaceWeapon(slot, item)` — replaces and drops the previous item in that slot
- `giveItem(item, amount)` — adds stackable items (ammo, healing) to `inventory.items`; drops excess if over capacity

Dual guns are a special case: picking up a gun whose `dual` variant exists, when the matching single is already held, upgrades the single to the dual variant automatically.

## Data Flow

```
Player presses use key
  └─> InputPacket received by server
        └─> player.attacking = true
              ├─> player.activeItem.useItem()
              │     ├─> GunItem: _bufferAttack(cooldown, _useItemNoDelayCheck)
              │     │     └─> game.addBullet() × bulletCount
              │     ├─> MeleeItem: _bufferAttack(cooldown, _useItemNoDelayCheck)
              │     │     └─> grid.intersectsHitbox() → target.damage()
              │     └─> ThrowableItem: _bufferAttack(fireDelay, _useItemNoDelayCheck)
              │           └─> _cook() — starts fuse
Player releases use key
  └─> player.attacking = false
        └─> ThrowableItem.stopUse() → _throw() → game.addProjectile()

Player presses heal key
  └─> new HealingAction(player, item) → player.action
        └─> timeout fires → execute() → player.health / adrenaline += restoreAmount

Gun runs dry
  └─> GunItem → new ReloadAction(player, gun) → player.action
        └─> timeout fires → execute() → ammo reserve → gun.ammo
```

## Dependencies

- **Depends on:**
  - [Object Definitions](../object-definitions/) — `GunDefinition`, `MeleeDefinition`, `ThrowableDefinition`, `HealingItemDefinition`, `AmmoDefinition`; `Loots.reify()` for idString → definition resolution
  - [Game Loop](../game-loop/) — `player.game.addTimeout()`, `player.game.addBullet()`, `player.game.addProjectile()` called by item and action code; `player.game.now` for timestamp tracking
  - Plugin System (`server/src/pluginManager.ts`) — `inv_item_equip`, `inv_item_unequip`, `inv_item_use`, `inv_item_stop_use`, `inv_item_modifiers_changed` events
  - Spatial Grid (`server/src/utils/grid.ts`) — melee and gun items query `game.grid.intersectsHitbox()` for hit detection and wall-clipping

- **Depended on by:**
  - Client Rendering — weapon slot state, active weapon, ammo counts flow through `UpdatePacket` to the client HUD (`client/src/scripts/managers/uiManager.ts`)
  - [Game Loop](../game-loop/) — `Player` is a game object iterated every tick; dirty flags set here (`dirty.weapons`, `dirty.items`) control what the `UpdatePacket` sends

## Module Index (Tier 3)

No Tier 3 module docs exist yet. Candidates in priority order:

- `gun-item.md` — fire modes, burst logic, perk modifier matrix, dual-gun mechanics
- `action.md` — action lifecycle, cooldown / speed interaction, chain-reload
- `inventory.md` — slot management, ItemCollection, ammo capacity, throwable cycling

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System architecture
- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — Item definition interfaces
- **Tier 2:** [Object Definitions](../object-definitions/) — item definition registries
- **Tier 2:** [Game Loop](../game-loop/) — tick and timeout integration
- **Tier 2:** [Networking](../networking/) — dirty flags → UpdatePacket serialization
- **Patterns:** [patterns.md](patterns.md) — Reusable inventory patterns
