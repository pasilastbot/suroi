# Actions Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/action.ts -->

## Purpose
Implements timed player actions (healing, reloading, and reviving) as cancellable, single-slot state that slows the player during execution and applies its effect only on successful completion.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/action.ts` | Abstract `Action` base class + all three concrete action classes | High |
| `server/src/inventory/inventory.ts` | `Inventory.useItem()` — guards and creates `HealingAction` | Medium |
| `server/src/objects/player.ts` | `executeAction()` entry point; action cancellation triggers; partial serialization | Medium |
| `common/src/definitions/items/healingItems.ts` | `HealingItemDefinition` — heal type, restore amount, use time | Low |
| `common/src/definitions/items/guns.ts` | `GunDefinition` — `reloadTime`, `fullReloadTime`, `shotsPerReload`, `reloadFullOnEmpty`, `extendedCapacity` | Low |
| `common/src/constants.ts` | `GameConstants.player.reviveTime` (8 s); `PlayerActions` enum | Low |

## Business Rules
- **One action at a time.** `Player.executeAction()` calls `this.action?.cancel()` before assigning the new action, so the previous action is always aborted without applying its effect.
- **Downed players cannot act.** `executeAction()` returns immediately if `player.downed` is `true`. `Action.execute()` also short-circuits on downed state.
- **Health/adrenaline cap guards.** `Inventory.useItem()` refuses to start a `HealingAction` if the stat is already at maximum, or if trying to use a Special item whose associated perk the player does not have.
- **Already-consuming guard.** If `player.action` is already a `HealingAction`, `useItem()` returns early — you cannot queue a second consumable.
- **Auto-reload on empty gun.** When an action slot clears (setter in player.ts) and the active weapon is a gun with 0 ammo, the server immediately creates a `ReloadAction` — unless `isConsumingItem` is `true`, preventing a phantom reload while using a healing item.
- **`isConsumingItem` flag.** Set to `true` by `Inventory.useItem()` before creating `HealingAction`; cleared inside `HealingAction.execute()`. Prevents the auto-reload logic from firing mid-heal.
- **Cancelling does not refund.** `Action.cancel()` kills the timeout and clears `player.action`; no stat or inventory change occurs. Items are only consumed in `execute()`.
- **ReviveAction range check.** Each game tick checks whether the reviver has moved ≥ 7 units from the downed target; if so, `action.cancel()` is called automatically.
- **ReviveAction cancel resets animation.** Both `ReviveAction.cancel()` and `ReviveAction.execute()` set the reviver's animation to `AnimationType.None`.
- **Reload is cancelled by weapon switch or certain perk changes.** Weapon-swap code calls `this.action?.cancel()`. Adding or removing `Butterfingers`/`CombatExpert` perks also cancels any in-progress reload.
- **Speed penalty during actions.** `ReloadAction` has no `speedMultiplier` override (inherits `1`). `HealingAction` and `ReviveAction` both set `speedMultiplier = 0.5`, halving the player's movement speed.

## Action Base Class (`Action`)
`@file server/src/inventory/action.ts`

### Properties
| Property | Type | Description |
|----------|------|-------------|
| `player` | `Player` | The acting player |
| `_timeout` | `Timeout` (private) | Scheduled callback; killed on cancel |
| `speedMultiplier` | `number` | Movement speed modifier while action is active (default `1`) |

### Abstract / Overridable Members
| Member | Kind | Description |
|--------|------|-------------|
| `type` | abstract getter → `PlayerActions` | Action type constant sent to clients |
| `execute()` | overridable | Applies the action's effect; base clears `player.action` and calls `setPartialDirty()` |
| `cancel()` | overridable | Kills the timeout and clears `player.action`; base calls `setPartialDirty()` |

**Constructor** takes `player` and `time` (seconds). The timeout is registered as `execute.bind(this)` with `time * 1000` ms delay. `setPartialDirty()` is called immediately so clients receive the new action state in the next tick.

---

## Action Types

### `ReloadAction`
`PlayerActions.Reload` | `speedMultiplier = 1` (no override, inherits base)

**Duration:** Computed at construction:
- If `reloadFullOnEmpty === true` **and** `item.ammo <= 0` → uses `definition.fullReloadTime` (full reload)
- Otherwise → uses `definition.reloadTime`
- Both values are divided by `player.reloadMod` (the `CombatExpert` perk modifier; defaults to `1`)
- If the player has the `CombatExpert` perk, `updateAndApplyModifiers()` is called before the timer is set

**`fullReload` property:** Captures whether the full-reload path was taken; used in `execute()` to decide the shot budget.

**`execute()` — ammo restoration logic:**
1. Determines effective `capacity`: uses `extendedCapacity` if the player has the `ExtendedMags` perk, otherwise `capacity`.
2. Calculates `desiredLoad`:
   - If `shotsPerReload` is defined **and** this is not a `fullReload`: loads `shotsPerReload` rounds (doubled for dual weapons with `isDual`).
   - Otherwise: loads up to `capacity − item.ammo` (fill to full).
3. Clamps `desiredLoad` against the items the player carries (skipped if player has `InfiniteAmmo` perk).
4. Adds `toLoad` rounds to `item.ammo`; decrements the player's ammo items accordingly.
5. **Chain reload:** If `item.ammo < capacity` after loading, calls `item.reload()` to start another `ReloadAction` automatically (per-shot reload for guns like shotguns).
6. Clears `player.attacking`; marks `weapons` and `items` dirty.

---

### `HealingAction`
`PlayerActions.UseItem` | `speedMultiplier = 0.5`

**Duration:** `item.useTime / player.mapPerkOrDefault(PerkIds.FieldMedic, ({ usageMod }) => usageMod, 1)`  
The `FieldMedic` perk divides `useTime`; without the perk the divisor is `1`.

**Item definition resolved** from `Loots.reify()` at construction; stored as `this.item: HealingItemDefinition`.

**`execute()` — application order:**

1. `super.execute()` — clears action slot and marks partial dirty
2. `inventory.items.decrementItem(item.idString)` — **item consumed before the heal is applied**
3. Switch on `item.healType`:

| `healType` | Effect |
|-----------|--------|
| `HealType.Health` | `player.health += item.restoreAmount` |
| `HealType.Adrenaline` | `player.adrenaline += item.restoreAmount` |
| `HealType.Special` | Iterates `item.effect.restoreAmounts[]` applying each sub-heal; calls `player.removePerk(item.effect.removePerk)` if defined |

4. Spawns a decal at the player's position: `` game.addDecal(`${item.idString}_residue`, position, randomRotation(), layer) ``
5. `player.dirty.items = true`
6. `player.isConsumingItem = false`

---

### `ReviveAction`
`PlayerActions.Revive` | `speedMultiplier = 0.5`

**Duration:** `GameConstants.player.reviveTime / player.mapPerkOrDefault(PerkIds.FieldMedic, ({ usageMod }) => usageMod, 1)`  
`GameConstants.player.reviveTime` = **8 seconds** (`common/src/constants.ts:31`).  
The `FieldMedic` perk shortens the revive timer by dividing by `usageMod`.

**Properties:**
| Property | Type | Description |
|----------|------|-------------|
| `target` | `Player` | The downed player being revived |

**`execute()`:**
1. Calls `super.execute()` (clears action, marks partial dirty)
2. Calls `this.target.revive()` to bring the downed player back up
3. Resets the reviver's animation to `AnimationType.None` and calls `setDirty()`

**`cancel()`:**
1. Calls `super.cancel()` (kills timeout, clears action)
2. Sets `this.target.beingRevivedBy = undefined` and calls `target.setDirty()`
3. Resets the reviver's animation to `AnimationType.None` and calls `setDirty()`

**Automatic cancellation conditions (from player.ts):**
- Reviver moves ≥ 7 units away from the target (checked every tick)
- Target dies before the revive completes (handled in the player death path)
- A new action preempts the revive via `executeAction()`

---

## Action Lifecycle

```
Player input (use item / auto-reload / start revive)
  │
  ▼
Inventory.useItem() / GunItem.reload() / player.startRevive()
  │  Guard checks (downed? stat at cap? already consuming?)
  │
  ▼
player.executeAction(new XxxAction(...))
  │  ① cancel previous action if any (no effect applied)
  │  ② assign player.action = new action
  │
  ▼
Action constructor
  │  ③ speedMultiplier recorded on action object
  │  ④ game.addTimeout(execute, durationMs) stored as _timeout
  │  ⑤ player.setPartialDirty() → clients see action start
  │
  ├──[timeout fires]──▶ execute()
  │                        apply effect (heal / ammo / revive)
  │                        clear player.action
  │                        player.setPartialDirty() → clients see action end
  │                        (chain-reload? → item.reload() → new ReloadAction)
  │
  └──[cancel() called]──▶ cancel()
                           _timeout.kill()
                           clear player.action (NO effect applied)
                           player.setPartialDirty() → clients see action end
```

---

## Player Action Integration

`Player.executeAction(action: Action)` (`player.ts:3653`) is the single entry point:
```
if (this.downed) return;
this.action?.cancel();
this.action = action;
```

`player.action` is backed by a private `_action` field with a setter that:
- Tracks the `PlayerActions` type and `dirty` flag
- On clear (`value === undefined`): if the cleared action was **not** a reload and the active gun is empty, immediately creates a new `ReloadAction` (auto-reload on empty, blocked while `isConsumingItem`)

**Network serialization** (in `player.data` getter):
- When `_action.dirty` is set, the partial update includes:
  - `HealingAction`: `{ type: PlayerActions.UseItem, item: this.action.item }` (item definition sent so the client can display the progress bar)
  - All others: `{ type: this.action?.type ?? PlayerActions.None }`

---

## `HealingItemDefinition` Fields
`@file common/src/definitions/items/healingItems.ts`

```typescript
interface HealingItemDefinition extends ItemDefinition {
    readonly defType: DefinitionType.HealingItem
    readonly healType: HealType          // Health | Adrenaline | Special
    readonly restoreAmount: number       // Stat points restored (0 for Special-only effects)
    readonly useTime: number             // Seconds to complete (before FieldMedic perk division)
    readonly effect?: {
        readonly removePerk: PerkIds     // Perk to remove on completion (Special type)
        readonly restoreAmounts?: Heal[] // Additional heals applied alongside perk removal
    }
    readonly hideUnlessPresent?: boolean // Hide item from HUD unless player has it
}
```

**Registered items:**

| `idString` | `healType` | `restoreAmount` | `useTime` |
|-----------|-----------|----------------|----------|
| `gauze` | `Health` | 20 | 3 s |
| `medikit` | `Health` | 100 | 6 s |
| `cola` | `Adrenaline` | 25 | 3 s |
| `tablets` | `Adrenaline` | 50 | 4 s |
| `vaccine_syringe` | `Special` | 0 | 2 s (removes `Infected` perk; also grants 50 Adrenaline) |

---

## Data Lineage

```
Player presses use-item key
  → InputPacket InputActions.UseItem { item: idString }
  → player.processInput() → inventory.useItem(idString)
  → guards pass
  → player.isConsumingItem = true
  → player.executeAction(new HealingAction(player, idString))
      → HealingAction constructor: resolve def, start timer, setPartialDirty
  → client receives partial update: { action: { type: UseItem, item: def } }
  → [after useTime / FieldMedic] execute()
      → inventory.items.decrementItem(idString)
      → player.health += restoreAmount  (or adrenaline / special logic)
      → addDecal(`${idString}_residue`, position)
      → player.isConsumingItem = false
      → player.setPartialDirty()
  → client receives partial update: { action: { type: None } }
  → auto-reload check fires if active gun is empty
```

---

## Dependencies
- **Internal (same subsystem):**
  - `GunItem.reload()` creates a `ReloadAction`; `ReloadAction.execute()` can call `item.reload()` again for chain reloads
  - `Inventory.useItem()` creates `HealingAction`; guards access `this.items`
- **External:**
  - [Game Loop](../../game-loop/README.md) — `game.addTimeout()` drives all action timers
  - [Networking](../../networking/README.md) — action state (type + item def) is included in the player's partial dirty packet; clients use it to render progress bars
  - [Object Definitions](../../object-definitions/README.md) — `HealingItemDefinition` and `GunDefinition` supply timing and capacity values at construction time

---

## Complex Functions

### `Action.cancel()` — @file server/src/inventory/action.ts
**Purpose:** Abort the action without applying any effect.  
**Implicit behavior:** Kills the internal timeout, clears `player.action` to `undefined`, and marks the player partial-dirty so the client hides the action progress bar. Does **not** refund any item or ammo. For `ReviveAction`, the override additionally clears `target.beingRevivedBy` and resets both players' animations.  
**Called by:** `Player.executeAction()` on its `existing` action before assigning a new one; weapon-switch paths; tick-level range check for `ReviveAction`; death handler when target dies.

### `ReloadAction.execute()` — @file server/src/inventory/action.ts
**Purpose:** Restore ammo to the active gun from the player's carried ammo pool.  
**Implicit behavior:** Calculates the exact `toLoad` value accounting for `shotsPerReload` (per-shot guns), `fullReload` flag (full-drum guns), `ExtendedMags` perk, and `InfiniteAmmo` perk. After loading, if the gun is still not full, calls `item.reload()` which creates a **new** `ReloadAction` — this chain continues until full or the player runs out of ammo. `CombatExpert` perk is recalculated via `updateAndApplyModifiers()` before the first `ReloadAction` is constructed (not on chained calls).  
**Called by:** Timeout callback (automatic); never called manually.

### `HealingAction.execute()` — @file server/src/inventory/action.ts
**Purpose:** Apply healing/adrenaline/special effect and consume the item.  
**Implicit behavior:** Decrements the item first, then applies stat change. Spawns a `${idString}_residue` decal at `player.position` with a random rotation on the player's current layer. Clears `isConsumingItem` flag so the auto-reload logic can fire if the gun happens to be empty post-heal.  
**Called by:** Timeout callback only.

### `Inventory.useItem()` — @file server/src/inventory/inventory.ts
**Purpose:** Gate-keeper that validates all preconditions before creating a `HealingAction`.  
**Implicit behavior:** Returns silently (no error to caller) if: item not in inventory, already in a `HealingAction`, stat is at cap, or Special item's target perk is absent. Downed players are also blocked. Sets `isConsumingItem = true` **before** calling `executeAction`, which means the setter on `player.action` will not trigger auto-reload while this flag is active.  
**Called by:** `player.processInput()` on `InputActions.UseItem`.

---

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Inventory subsystem overview
- **Tier 3:** [gun-item.md](gun-item.md) — `GunItem.reload()` triggers `ReloadAction`
- **Tier 1:** [../../../datamodel.md](../../../datamodel.md) — Core entity definitions including `HealingItemDefinition`
- **Patterns:** [../patterns.md](../patterns.md) — Timed action pattern
- **Subsystem:** [../../game-loop/README.md](../../game-loop/README.md) — `game.addTimeout()` used by all action timers
- **Subsystem:** [../../networking/README.md](../../networking/README.md) — partial player serialization carries action type + item
