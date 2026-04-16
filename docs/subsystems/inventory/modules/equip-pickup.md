# Inventory System — Equip & Pickup

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/inventory.ts, server/src/objects/loot.ts -->

## Purpose

Detailed mechanics of item pickup, equipping, weapon switching, and inventory slot management. Covers how players acquire loot, manage active items, swap weapons, and handle full inventory constraints.

---

## Inventory Structure

The Inventory class manages a player's weapons, equipment, and stackable items:

```typescript
class Inventory {
  // Weapon slots (fixed 4 slots)
  readonly weapons: Array<InventoryItem | undefined> = [
    gun0,      // slot 0: gun
    gun1,      // slot 1: gun
    melee,     // slot 2: melee (always has "fists" at minimum)
    throwable  // slot 3: throwable
  ]

  // Active weapon tracking
  _activeWeaponIndex = 2      // currently equipped slot (0-3)
  _lastWeaponIndex = 0        // previously used slot for quick-swap

  // Equipment (non-slot items)
  helmet?: ArmorDefinition
  vest?: ArmorDefinition
  backpack: BackpackDefinition = bag (default level 0)
  _scope: ScopeDefinition

  // Stackable items (ammo, healing, throwables)
  readonly items: ItemCollection = { gauze: 5, cola: 2, ... }

  // Slot locking (bitmask) — prevents accidental weapon swaps
  _lockedSlots: number = 0
}
```

### Slot Layout & Typing

| Slot # | Type | Purpose | Hotkey | Min Items |
|--------|------|---------|--------|-----------|
| 0 | `DefinitionType.Gun` | Primary gun | `1` | 0 |
| 1 | `DefinitionType.Gun` | Secondary gun | `2` | 0 |
| 2 | `DefinitionType.Melee` | Melee weapon | `E` | Always "fists" |
| 3 | `DefinitionType.Throwable` | Grenade/throwable | `G` | 0 |

**Constraint:** Slot 2 can never be empty. If a melee item is removed, `MeleeItem("fists")` is automatically placed there.

---

## Pickup Detection

### Distance & Interaction Range

Pickup is triggered by the player pressing `InputActions.Loot` or `InputActions.Interact`:

```typescript
// In Player.processInputs() → InputActions.Loot
const detectionHitbox = new CircleHitbox(3 * this._sizeMod, this.position);
//                                        ^^^^^^^^^^^^^^^
//                                        interaction radius
//                                        (default ~3 units, scaled by size modifier)

for (const object of this.game.grid.intersectsHitbox(detectionHitbox, this.layer)) {
  if (object.isLoot && object.hitbox.collidesWith(detectionHitbox)) {
    // Can interact with loot
  }
}
```

**Key:** Pickup uses **hitbox collision**, not distance. The loot's hitbox radius varies by item type (see [GameConstants.lootRadius](../../../../common/src/constants.ts#L70)).

### Loot Radius by Type

| Item Type | Radius | Source |
|-----------|--------|--------|
| Gun | 3.4 units | `GameConstants.lootRadius[DefinitionType.Gun]` |
| Ammo | 2.0 units | `GameConstants.lootRadius[DefinitionType.Ammo]` |
| Melee | 3.0 units | `GameConstants.lootRadius[DefinitionType.Melee]` |
| Throwable | 3.0 units | `GameConstants.lootRadius[DefinitionType.Throwable]` |
| HealingItem | 2.5 units | `GameConstants.lootRadius[DefinitionType.HealingItem]` |
| Armor | 3.0 units | `GameConstants.lootRadius[DefinitionType.Armor]` |
| Backpack | 3.0 units | `GameConstants.lootRadius[DefinitionType.Backpack]` |
| Scope | 3.0 units | `GameConstants.lootRadius[DefinitionType.Scope]` |

---

## Pickup Flow

### 1. Detection Phase

When a player presses the loot key, the server:
1. Creates a detection hitbox (3×playerSize units radius) around player position
2. Queries spatial grid for nearby objects in same layer
3. Iterates through loot items and checks hitbox collision
4. Calls `loot.canInteract(player)` to validate

### 2. Validation Phase (`canInteract()`)

The loot checks if interaction is allowed and returns:
- `true` — pickup is allowed
- `false` — pickup not allowed (dead loot, downed player, etc.)
- `InventoryMessages` number — pickup rejected with error message (e.g., `InventoryMessages.NotEnoughSpace`)

**Validation Examples:**

```typescript
// Gun pickup
canInteract(gun) {
  // Can upgrade to dual if single variant matches
  if (singleGunExists && gunHasDualVariant) return true

  // Can fit in empty gun slot (0 or 1)
  if (!inventory.hasWeapon(0) && !inventory.isLocked(0)) return true
  if (!inventory.hasWeapon(1) && !inventory.isLocked(1)) return true

  // Can replace active weapon if different gun
  if (activeSlotIsGun && newGun !== activeGun) return true

  return false
}

// Healing/Ammo
canInteract(ammo) {
  const currentCount = inventory.items.getItem(ammo.idString)
  const maxCapacity = inventory.backpack.maxCapacity[ammo.idString]

  if (currentCount + 1 > maxCapacity) {
    return InventoryMessages.NotEnoughSpace
  }
  return true
}

// Melee
canInteract(melee) {
  return melee !== inventory.getWeapon(2)?.definition
      && !inventory.isLocked(2)
}

// Armor
canInteract(armor) {
  const currentLevel = armor.armorType === Helmet
    ? inventory.helmet?.level ?? 0
    : inventory.vest?.level ?? 0

  return armor.level > currentLevel
}
```

### 3. Pickup Phase (`interact()`)

If validation passes, `loot.interact(player)` is called. Item handling varies by type:

#### **Guns**

```typescript
// Step 1: Check for dual upgrades
for (each gun in inventory.weapons) {
  if (gun exists && gun.isDual === false && gun.dualVariant && gun === newGun) {
    return inventory.upgradeToDual(slot)  // Combine dual!
  }
}

// Step 2: Try to append to first available gun slot
const slot = inventory.appendWeapon(newGun)
if (slot !== -1) {
  // Picked up successfully into new slot
  if (activeSlot is melee) {
    setActiveWeaponIndex(slot)  // Auto-switch to gun
  }
  return
}

// Step 3: All gun slots full → try to replace active weapon
if (activeWeapon is gun && activeGun !== newGun) {
  inventory.addOrReplaceWeapon(activeWeaponIndex, newGun)
  return
}

// Step 4: Can't fit → pickup fails
countToRemove = 0  // Loot stays on ground
```

**Dual Gun Upgrade:** If a player picks up the same single-variant gun while one is equipped, it automatically upgrades to the dual variant (fires both).

#### **Melee**

```typescript
const meleeSlot = 2
if (newMelee !== inventory.getWeapon(2)?.definition) {
  inventory.addOrReplaceWeapon(meleeSlot, newMelee)
}
```

**Constraint:** Only one melee weapon at a time (slot 2 only).

#### **Healing Items & Ammo**

```typescript
const currentCount = inventory.items.getItem(itemId)
const maxCapacity = inventory.backpack.maxCapacity[itemId]

if (currentCount + lootCount <= maxCapacity) {
  inventory.items.incrementItem(itemId, lootCount)
  countToRemove = lootCount  // Remove all
} else {
  // Partial pickup: fill to capacity
  inventory.items.setItem(itemId, maxCapacity)
  countToRemove = maxCapacity - currentCount  // Partial
}

// Loot.interact() then:
if (countToRemove < lootCount) {
  this._count -= countToRemove
  createNewItem()  // Drop remainder on ground
}
```

**Backpack Capacity:** Managed per item by `backpack.maxCapacity[itemId]`. Backpack level defines limits:

| Backpack | Level | Ammo (9mm) | Healing (Gauze) | Grenades |
|----------|-------|-----------|-----------------|----------|
| Bag | 0 | 120 | 5 | 3 |
| Basic Pack | 1 | 240 | 10 | 6 |
| Regular Pack | 2 | 330 | 15 | 9 |
| Tactical Pack | 3 | 420 | 30 | 12 |
| AMP-4 Mule | 4 | 420 | 30 | 12 |

#### **Throwables (stackable grenades)**

Throwables have **two representations:**
1. **Slot weapon** (`weapons[3]`): The active `ThrowableItem` instance being thrown
2. **Inventory stack** (`items[grenadeName]`): Count of this grenade type

```typescript
// Step 1: Validate slot exists
const slot = inventory.slotsByDefType[DefinitionType.Throwable][0]
if (slot === undefined) {
  countToRemove = 0  // No grenade slot, can't pickup
  return
}

// Step 2: Add to stack (respects backpack capacity)
const currentCount = inventory.items.getItem(grenade.idString)
const maxCapacity = inventory.backpack.maxCapacity[grenade.idString]

if (currentCount + 1 <= maxCapacity) {
  inventory.items.incrementItem(grenade.idString, count)
  countToRemove = count
}

// Step 3: Update active throwable reference
inventory.useItem(grenade.idString)  // Equips this grenade type
inventory.throwableItemMap.get(grenade.idString).count = newCount
```

**Auto-equip:** When pickupgrenade is picked up, it becomes the active throwable.

#### **Armor (Helmet/Vest)**

```typescript
// If already wearing armor of lower level, drop it
if (inventory.helmet && newHelmet.level > inventory.helmet.level) {
  createNewItem({ type: inventory.helmet, count: 1 })
}

inventory.helmet = newHelmet

if (newHelmet.perk !== undefined) {
  player.addPerk(newHelmet.perk)
}
```

**Equipment Swap:** Old item is dropped to ground with `push()` velocity.

#### **Backpack**

Similar to armor:
```typescript
if (inventory.backpack.level > 0) {
  createNewItem({ type: inventory.backpack, count: 1 })
}

inventory.backpack = newBackpack

// If backpack is now smaller, drop excess items
for (each item in inventory) {
  const excess = currentCount - newBackpack.maxCapacity[item]
  if (excess > 0) {
    this._dropItem(item, { count: excess })
  }
}
```

#### **Scope**

```typescript
inventory.items.setItem(scope.idString, 1)

// Auto-equip if better zoom level
if (scope.zoomLevel > inventory.scope.zoomLevel) {
  inventory.scope = scope
}
```

#### **Skin & Perks**

Skins and perks replace current loadout/perk:

```typescript
// Skin
createNewItem({ type: currentSkin, count: 1 })
player.loadout.skin = newSkin
player.setDirty()

// Perk
const oldPerk = player.perks.find(p => p.category === newPerk.category)
if (oldPerk && !oldPerk.noDrop) {
  createNewItem({ type: oldPerk, count: 1 })
}
player.addPerk(newPerk)
```

---

## Equip & Switch Mechanics

### Setting Active Weapon Slot

```typescript
setActiveWeaponIndex(newSlot: number): boolean {
  if (!Inventory.isValidWeaponSlot(newSlot)) throw RangeError
  if (!this.hasWeapon(newSlot) || newSlot === this._activeWeaponIndex) return false

  const old = this._activeWeaponIndex
  this._activeWeaponIndex = newSlot
  this._lastWeaponIndex = old  // Save for quick-swap

  // Stop using old weapon
  const oldItem = this.weapons[old]
  if (oldItem) {
    oldItem.isActive = false
    oldItem.stopUse()
  }

  // Start using new weapon
  const newItem = this.weapons[newSlot]
  this._reloadTimeout?.kill()
  if (newItem.isGun) {
    newItem.cancelAllTimers()
  }

  newItem.isActive = true
  this.owner.setDirty()
  this.owner.dirty.activeWeaponIndex = true

  return true
}
```

**Constraints:**
- Can only switch to slots with weapons (returns `false` if slot empty)
- Cancels reload/reload timeout
- Marks player dirty for sync to clients

### Quick Swap (Back to Last Weapon)

```typescript
// Player presses weapon key that matches last active slot
if (newSlot === this._lastWeaponIndex && this.hasWeapon(newSlot)) {
  setActiveWeaponIndex(newSlot)  // Switch back
}
```

### Finding Next Populated Slot

When current slot becomes empty (e.g., gun dropped):

```typescript
private _findNextPopulatedSlot(): number {
  let target = this._activeWeaponIndex
  while (!this.hasWeapon(target)) {
    target = (target + 1) % this.weapons.length  // Circular search
  }
  return target
}
```

Searches forward in circular order: 0 → 1 → 2 → 3 → 0 → ...

---

## Drop Mechanics

### Dropping Active Weapon

Called when player presses the drop key or slot is removed:

```typescript
dropWeapon(slot: number, force?: boolean): InventoryItem | undefined {
  if (!Inventory.isValidWeaponSlot(slot)) throw RangeError
  if (!force && this.isLocked(slot)) return  // Slot locked, no drop

  const item = this.weapons[slot]

  // Check: Don't drop if marked noDrop or cooking throwable
  if (item === undefined || item.definition.noDrop) return
  if (item.category === DefinitionType.Throwable && item.cooking) return

  // Branch 1: Throwable
  if (inventorySlotTypings[slot] === DefinitionType.Throwable) {
    return removeThrowable(item.definition, drop=true)
  }

  // Branch 2: Gun or Melee
  _dropItem(item.definition, { data: item.itemData() })

  // Return ammo to inventory if gun is being dropped
  if (item.isGun && item.ammo > 0) {
    giveItem(item.definition.ammoType, item.ammo)
    item.ammo = 0
  }

  this._setWeapon(slot, undefined)

  // Auto-switch to next populated slot
  const nextSlot = this._findNextPopulatedSlot()
  if (nextSlot !== slot) {
    setActiveWeaponIndex(nextSlot)
  }

  player.setDirty()
  player.dirty.items = true
  player.dirty.weapons = true

  return item
}
```

**Special Cases:**
- **Guns** → Ammo in magazine is returned to inventory
- **Dual guns** → Both single variants are dropped separately
- **Throwables** → Only part of the stack is dropped; rest stays in `items`
- **Locked slots** → Can't drop (returns early)
- **noDrop items** → Can't be dropped (e.g., starting melee)

### Dropping Stackable Items

```typescript
dropItem(itemString: string): void {
  const definition = Loots.reify(itemString)
  const { idString } = definition

  switch (definition.defType) {
    case DefinitionType.HealingItem:
    case DefinitionType.Ammo: {
      const currentCount = this.items.getItem(idString)
      const dropAmount = Math.ceil(currentCount / 2)  // Half
      const dropCount = Math.min(currentCount, dropAmount)

      this._dropItem(definition, { count: dropCount })
      this.items.decrementItem(idString, dropCount)
      break
    }

    case DefinitionType.Throwable: {
      this.removeThrowable(definition, drop=true)
      break
    }

    // ... other types handle equip swaps
  }

  player.setDirty()
  player.dirty.items = true
}
```

**Drop Amount Rule:**
- **Ammo:** `Math.ceil(count / 2)` with minimum `definition.minDropAmount`
- **Healing:** `Math.ceil(count / 2)` (half the stack)
- **Throwables:** Half the stack

---

## Slot Locking

Prevents accidental weapon swaps during action sequences:

```typescript
lock(slot: number): boolean {
  if (!Inventory.isValidWeaponSlot(slot)) throw RangeError

  const mask = 1 << slot
  const wasLocked = !!(this._lockedSlots & mask)

  this._lockedSlots |= mask  // Set bit
  this.owner.dirty.slotLocks = wasLocked ? false : true

  return wasLocked  // Returns whether state changed
}

unlock(slot: number): boolean {
  if (!Inventory.isValidWeaponSlot(slot)) throw RangeError

  const mask = 1 << slot
  const wasLocked = !!(this._lockedSlots & mask)

  this._lockedSlots &= ~mask  // Clear bit
  this.owner.dirty.slotLocks = wasLocked ? true : false

  return wasLocked  // Returns whether state changed
}

isLocked(slot: number): boolean {
  if (!Inventory.isValidWeaponSlot(slot)) throw RangeError
  return !!(this._lockedSlots & (1 << slot))
}
```

**Usage:**
- Gun reloading locks slot to prevent mid-reload swaps
- Throwable cooking locks slot
- Player perks (Lycanthropy) may prevent unlocking

---

## Auto-Switch Rules

The inventory automatically switches weapons under specific conditions:

### 1. **Gun Out of Ammo**

When equipped gun's ammo reaches 0 and magazine is empty:

```typescript
if (activeGun.ammo === 0 && activeGun.magazine === 0) {
  inventory.switchToNextGun()  // Equips first non-empty gun
}
```

### 2. **Weapon Destroyed**

If active weapon's slot becomes null:

```typescript
if (this._setWeapon(slot, undefined) && slot === this._activeWeaponIndex) {
  this.setActiveWeaponIndex(this._findNextPopulatedSlot())
}
```

### 3. **Throwable Depleted**

When player uses all grenades of one type:

```typescript
if (inventory.items.getItem(grenade.idString) === 0) {
  // Discard ThrowableItem cache
  inventory.throwableItemMap.delete(grenade.idString)

  // Find next throwable to equip
  for (const otherGrenade of Throwables) {
    if (inventory.items.getItem(otherGrenade.idString) > 0) {
      inventory.useItem(otherGrenade.idString)
      break
    }
  }
}
```

---

## Data Synchronization

### Dirty Flags

After inventory changes, flags mark data for sync:

| Flag | Synced | Triggers |
|------|--------|----------|
| `player.dirty.weapons` | `UpdatePacket.inventory.weapons[...]` | Weapon swap, pickup gun, equip melee |
| `player.dirty.items` | `UpdatePacket.inventory.items` | Pickup/drop ammo, healing, throwable |
| `player.dirty.healing` | `UpdatePacket.health` | Consume healing item |
| `player.dirty.slotLocks` | `UpdatePacket.slotLocks` | Lock/unlock slot |
| `player.dirty.activeWeaponIndex` | `UpdatePacket.activeWeaponIndex` | Switch weapon |

### Serialization

```typescript
// Inventory sent to client as UpdatePacket
{
  activeWeaponIndex: 0,
  weapons: [
    { definition: gun_id, ammo: 45, ... },
    undefined,
    { definition: melee_id, ... },
    { definition: grenade_id, count: 3 }
  ],
  items: {
    "9mm": 120,
    "gauze": 5,
    "cola": 2,
    ...
  },
  equipment: {
    helmet: { level: 2 },
    vest: { level: 3 },
    backpack: { level: 2 },
    scope: { zoomLevel: 4 }
  }
}
```

---

## Gotchas & Edge Cases

### 1. **Dual Gun Upgrade Consumes Both Singles**

When upgrading to dual, both single variant instances are consumed / dropped:

```typescript
if (pickingUp gun and dualVariant exists) {
  upgradeToDual()  // ← Consumes single from slot & inventory
}
```

Only one dual gun can be held at a time (uses one gun slot).

### 2. **Backpack Downgrade Drops Excess**

Picking up smaller backpack auto-drops items exceeding new capacity:

```typescript
if (newBackpack.level < current.level) {
  for (each item) {
    const excess = count - newBackpack.maxCapacity[item]
    if (excess > 0) {
      _dropItem(item, { count: excess })
    }
  }
}
```

### 3. **Slot 2 Can Never Be Empty**

Melee slot always has at least the "fists" melee:

```typescript
if (slot === 2 && item === undefined) {
  this._setWeapon(2, new MeleeItem("fists", owner))
}
```

Attempting to unequip melee is only possible if another weapon exists to swap to.

### 4. **Locked Slots Can't Receive New Items**

If target slot is locked, pickup fails **silently**:

```typescript
replaceWeapon(slot, item, force=false) {
  if (this.isLocked(slot) && !force) return null  // Silent fail
}
```

Returns `null` to indicate no pickup occurred.

### 5. **Gun Slot Type Enforcement**

Putting wrong item type in slot throws error:

```typescript
if (item.defType !== inventorySlotTypings[slot]) {
  throw Error(`Can't put ${item.defType} in slot ${slot}`)
}
```

### 6. **Ammo Reload Priority**

When player picks up correct ammo type with empty active gun:

```typescript
const activeWeapon = inventory.activeWeapon
if (
  activeWeapon.isGun
  && activeWeapon.ammo === 0
  && pickedUpAmmo === activeWeapon.definition.ammoType
) {
  activeWeapon.reload()  // ← Auto-reload triggered
}
```

---

## Related Documents

### Tier 2
- [Inventory Subsystem](../README.md) — Full system overview, action types, serialization
- [Game Loop](../../game-loop/) — When inventory updates occur each tick
- [Game Objects (Server)](../../game-objects-server/) — Loot entity, player object structure

### Tier 3
- Coming next: Gun Item mechanics (fire, reload, ammo), Melee Item (swing hitbox), Throwable Item (cooking, throwing)

### Cross-References
- [Networking — Packets](../../networking/) — `UpdatePacket`, `PickupPacket`, `InputPacket` structures
- [Object Definitions](../../object-definitions/) — `@common/utils/objectDefinitions.ts` item registry
- [Spatial Grid](../../spatial-grid/) — `game.grid.intersectsHitbox()` for pickup detection

---

## References

- **Source:** [server/src/inventory/inventory.ts](../../../../server/src/inventory/inventory.ts)
- **Loot Interaction:** [server/src/objects/loot.ts](../../../../server/src/objects/loot.ts#L281-L521)
- **Player Input:** [server/src/objects/player.ts](../../../../server/src/objects/player.ts#L3451-L3510)
- **Constants:** [common/src/constants.ts](../../../../common/src/constants.ts)
- **Backpack Capacity:** [common/src/definitions/items/backpacks.ts](../../../../common/src/definitions/items/backpacks.ts)
