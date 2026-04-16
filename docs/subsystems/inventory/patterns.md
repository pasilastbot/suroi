# Inventory — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/inventory/README.md -->

## Pattern: Weapon Subclass via `.derive()`

**When to use:** Understanding why `GunItem`, `MeleeItem`, and `ThrowableItem` are not
simple direct subclasses of `InventoryItemBase`, and how to introduce a new weapon
category.

**Implementation:**

`InventoryItemBase` is an abstract class whose constructor is inaccessible except
through the `derive` static method:

```typescript
// inventoryItem.ts
static derive<This extends AbstractConstructor, Type extends WeaponTypes>(
    this: This,
    defType: Type
): new (...args: ConstructorParameters<This>) => InstanceType<This> & PredicateForItem<Type>
```

Calling `.derive(DefinitionType.Gun)` at the top of `gunItem.ts` generates a concrete
class that:
- Hard-codes `readonly category = DefinitionType.Gun`
- Resolves the `definition` field from the passed `idString | GunDefinition`
- Throws a `TypeError` if the resolved definition has the wrong `defType`
- Stamps `isGun = true` (type predicate) for all runtime `instanceof`-style checks
- Registers the subclass in a private static map so the same category cannot be
  registered twice

```typescript
// gunItem.ts
export class GunItem extends InventoryItemBase.derive(DefinitionType.Gun) { ... }
```

This means `InventoryItemCtorMapping` in `inventory.ts` can also be written as a
type-safe map from `DefinitionType → constructor`:

```typescript
export const InventoryItemCtorMapping = {
    [DefinitionType.Gun]:       GunItem,
    [DefinitionType.Melee]:     MeleeItem,
    [DefinitionType.Throwable]: ThrowableItem,
} satisfies { [K in WeaponTypes]: AbstractConstructor<InventoryItem & PredicateFor<WeaponItemTypeMap, K>, ...> };
```

**Example files:**
- `@file server/src/inventory/inventoryItem.ts` — `InventoryItemBase.derive` implementation
- `@file server/src/inventory/gunItem.ts` — `GunItem extends InventoryItemBase.derive(DefinitionType.Gun)`
- `@file server/src/inventory/inventory.ts` — `InventoryItemCtorMapping`

---

## Pattern: Input Buffering (`_bufferAttack`)

**When to use:** Firing any weapon — understanding how a trigger press that arrives
slightly early (within 200 ms) is still honoured.

**Implementation:**

All three weapon `useItem()` methods delegate to `_bufferAttack(cooldown, callback)`,
defined on `InventoryItemBase`:

```
_bufferAttack(cooldown, internalCallback)
  ├─ timeToFire  = cooldown  - (now - this.lastUse)
  └─ timeToSwitch = switchDelay - (now - this.switchDate)

  if both ≤ 0  →  execute callback immediately
  else if max(timeToFire, timeToSwitch) < 200 ms
              →  schedule owner.bufferedAttack timeout
  else         →  discard input (player pressed too early)
```

Only one buffered attack is kept at a time (`owner.bufferedAttack?.kill()` before each
schedule). The buffered callback re-checks `owner.attacking` when it fires so that a
player who released the key before the buffer expired does not accidentally fire.

**Example files:**
- `@file server/src/inventory/inventoryItem.ts` — `_bufferAttack()` implementation (near bottom of the IIFE)

---

## Pattern: Fire Mode Dispatch

**When to use:** Understanding the three gun fire modes and how they schedule subsequent
shots.

**Implementation:**

`GunItem._useItemNoDelayCheck(skipAttackCheck)` is the single method that fires one
shot. After spawning bullets it dispatches differently per `FireMode`:

| `FireMode` | After-shot scheduling |
|---|---|
| `Single` | Nothing — waits for next `useItem()` call (or auto-fires on mobile) |
| `Auto` | `clearTimeout(_autoFireTimeout); _autoFireTimeout = setTimeout(_useItemNoDelayCheck, fireDelay * fireRateMod)` |
| `Burst` | Counts `_consecutiveShots`; once `shotsPerBurst` reached, `_burstTimeout = setTimeout(…, burstCooldown)` and resets counter |

Burst mode uses **Node.js** `setTimeout`/`clearTimeout` (not the game's `addTimeout`)
because burst fire delays can be shorter than a single tick (25 ms at 40 TPS). Auto
mode does the same.

The `cycle` mechanism (optional in `GunDefinition`) lets a weapon alternate between two
different fire delays every `cycle.shotsRequired` shots. `_shotsCounter` tracks progress
and `_previousFireDelay` stores the normal delay while the cycle delay is active.

**Example files:**
- `@file server/src/inventory/gunItem.ts` — `_useItemNoDelayCheck()`, fire mode branch at end of method

---

## Pattern: Timed Action

**When to use:** Adding or understanding any time-gated operation (healing, reloading,
reviving).

**Implementation:**

```
new XxxAction(player, params)
  constructor
    ├─ super(player, durationSeconds)
    │    └─ _timeout = game.addTimeout(this.execute.bind(this), duration * 1000)
    │    └─ player.setPartialDirty()   ← broadcasts action type to clients immediately
    └─ store params (item def, gun ref, …)

action.cancel()
  ├─ _timeout.kill()
  ├─ player.action = undefined
  └─ player.setPartialDirty()

action.execute()   ← called by the game timeout
  ├─ (guard: if player.downed → return)
  ├─ super.execute()  → player.action = undefined, setPartialDirty
  └─ perform effect (apply heal, transfer ammo, call target.revive(), …)
```

Any weapon fire, `InputActions.UseItem`, or `InputActions.Revive` input first calls
`player.action?.cancel()` — ensuring only one action can run at a time.

Action durations are modified by perks:
- `FieldMedic` perk provides a `usageMod` divisor applied to both `HealingAction` and `ReviveAction` durations
- `CombatExpert` perk triggers `player.updateAndApplyModifiers()` at the start of `ReloadAction` (the `reloadMod` field on `Player` is used as the divisor)

**Example files:**
- `@file server/src/inventory/action.ts` — `Action`, `ReloadAction`, `HealingAction`, `ReviveAction`

---

## Pattern: `ItemCollection` for Stackable Items

**When to use:** Adding to, consuming, or checking quantities of ammo, healing items,
scopes, or throwable counts.

**Implementation:**

`ItemCollection<ItemDef>` (defined at the bottom of `inventory.ts`) wraps a
`Map<ReferenceTo<ItemDef>, number>`. The `Inventory` class exposes one instance:

```typescript
readonly items = new ItemCollection(Object.entries(DEFAULT_INVENTORY));
```

Key methods:

```typescript
items.hasItem(idString)             // → true if count > 0
items.getItem(idString)             // → raw number (call hasItem first)
items.setItem(idString, n)          // → sets count directly
items.incrementItem(idString, n=1)  // → setItem(key, getItem(key) + n)
items.decrementItem(idString, n=1)  // → setItem(key, max(getItem(key) - n, 0))
items.asRecord()                    // → Record<string, number> (cached)
```

Callers are responsible for capacity checks. `Inventory.giveItem()` performs the
capacity check against `backpack.maxCapacity[idString]` and drops surplus loot when
the player is alive.

`_recordCache` is invalidated on every `setItem` call so `asRecord()` stays consistent.

**Example files:**
- `@file server/src/inventory/inventory.ts` — `ItemCollection` class (end of file)
- `@file common/src/defaultInventory.ts` — `DEFAULT_INVENTORY` seed values

---

## Pattern: Throwable Cycling

**When to use:** Understanding what happens when a player has multiple throwable types
or runs out of a throwable.

**Implementation:**

`Inventory.throwableItemMap: ExtendedMap<idString, ThrowableItem>` caches one
`ThrowableItem` instance per throwable type so a player carrying both grenades and
molotovs does not lose state when switching between them.

Removing a throwable goes through `removeThrowable(definition, drop, removalCount)`:

```
itemAmount = items.getItem(idString)
removalAmount = min(itemAmount, removalCount ?? ceil(itemAmount/2))

drop ? _dropItem(definition, { count: removalAmount })
items.decrementItem(idString, removalAmount)

if itemAmount === removalAmount:
    throwableItemMap.delete(idString)

    found = false
    for def of Throwables:
        if items.getItem(def.idString) > 0:
            found = true; useItem(def); break

    if !found:
        unlock(throwableSlot)
        weapons[throwableSlot] = undefined
        setActiveWeaponIndex(_findNextPopulatedSlot())
else:
    throwableItemMap.get(idString).count -= removalAmount
```

`ThrowableItem` also auto-triggers removal when fully thrown: `_throw()` calls
`inventory.removeThrowable(this.definition, false, 1)` (no drop because the projectile
has already been spawned).

**Example files:**
- `@file server/src/inventory/inventory.ts` — `removeThrowable()`, `throwableItemMap`
- `@file server/src/inventory/throwableItem.ts` — `_throw()` and `_cook()`

---

## Pattern: Weapon Stat Modifiers (`wearerAttributes`)

**When to use:** Understanding how equipping a weapon changes player movement speed,
max health, adrenaline drain, etc.

**Implementation:**

Any `InventoryItemDefinition` may carry a `wearerAttributes` block with three sub-keys:
- `passive` — applied while the weapon is in the inventory (regardless of active slot)
- `active` — applied only while this weapon is the active weapon (`isActive === true`)
- `on.kill` / `on.damageDealt` — stackable bonuses that accumulate per kill or per
  damage unit up to an optional `limit`

`InventoryItemBase.refreshModifiers()` recalculates the combined modifiers and calls
`owner.updateAndApplyModifiers()` if anything changed, which propagates the new values
to `player.maxHealth`, `player.baseSpeed`, etc.

`refreshModifiers()` is called whenever:
- `isActive` changes (equip / unequip)
- `stats.kills` or `stats.damage` changes (kill / hit)

**Example files:**
- `@file server/src/inventory/inventoryItem.ts` — `refreshModifiers()` implementation
- `@file common/src/utils/objectDefinitions.ts` — `WearerAttributes` / `ExtendedWearerAttributes` interfaces
