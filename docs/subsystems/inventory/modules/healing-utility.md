# Inventory — Healing & Utility Items

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/inventoryItem.ts, common/src/definitions/items/healingItems.ts, server/src/inventory/action.ts -->

## Purpose
Manages healing item types (gauze, medikit, cola, tablets), their use animations, restoration timing, stack management, and special effects (perk removal via vaccine syringe).

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/definitions/items/healingItems.ts` | Healing item definitions (types, amounts, use times) | Medium |
| `server/src/inventory/action.ts` | HealingAction — server-side action execution | High |
| `server/src/objects/player.ts` | Player health/adrenaline properties, perk application | High |
| `client/src/scripts/objects/player.ts` | Client healing animation state synchronization | Medium |

## Business Rules

- **Healing Types:** Health (gauze, medikit), Adrenaline (cola, tablets), Special (vaccine syringe)
- **Use Time:** Defined per item (e.g., gauze = 3s, medikit = 6s); player locked into animation for duration
- **Healing Ticks:** Some items apply healing in ticks during use (e.g., medkit heals continuously over 6s)
- **Interruption:** Using another healing item cancels current healing; canceling partially refunds duration
- **Stack Management:** Player can carry multiple stacks of same healing item; use decreases count
- **Adrenaline Rush:** High adrenaline unlocks attack/movement bonuses (configurable at thresholds)
- **Special Effects:** Vaccine syringe removes specific perk (Infected) and grants adrenaline bonus
- **Conditional Display:** Items marked `hideUnlessPresent` only show in inventory UI if carried
- **Max Caps:** Health ≤ 100, Adrenaline ≤ 100 (exact caps vary by game mode)

## Data Lineage

### Healing Item Use Flow
```
Player input: Healing key pressed (E key, mobile tap)
  ↓
Inventory slot selected (if not already in use)
  ↓
Healing item definition loaded (restoreAmount, useTime, healType)
  ↓
HealingAction created:
    duration = useTime × 1000 (ms)
    restoreAmount = definition.restoreAmount
  ↓
Player locked in healing animation
  ↓
Each game tick: tick() called on action
  ↓
At tick completion: restore performed
    if healType === Health: health += restoreAmount
    if healType === Adrenaline: adrenaline += restoreAmount
  ↓
Item count decremented: inventory[item] -= 1
  ↓
If special effect: perk removed / bonus applied
  ↓
Action removed from active actions list
```

### Stack Count Management
```
Loot pickup: gauze (count=3)
  ↓
Inventory[gauze] += 3 (now have 5 total if had 2)
  ↓
Use gauze: count -= 1 (inventory[gauze] = 4)
  ↓
Drop item: new Loot(gauze, count=4) created
  ↓
Inventory[gauze] = 0
```

## Complex Functions

### `HealingAction.tick()` — @file server/src/inventory/action.ts
**Purpose:** Update healing action per server tick; detect completion and apply restoration.

**Implicit behavior:**
- Checks if `elapsed >= duration`
- If incomplete, waits for next tick
- When complete:
  1. Extract restoration amounts from item definition
  2. For Health restoration: `player.health = min(100, player.health + restoreAmount)`
  3. For Adrenaline restoration: `player.adrenaline = min(100, player.adrenaline + restoreAmount)`
  4. Apply special effects if present:
     - Remove perk: `player.perks.remove(effectConfig.removePerk)`
     - Additional bonus: apply `effectConfig.restoreAmounts` (e.g., adrenaline +50)
  5. Decrement item count: `inventory[item.idString] -= 1`
  6. Emit action completion event
  7. Remove self from active actions

**Called by:** Game loop update phase (once per 25ms at 40 TPS)

**Side effects:**
- Player health/adrenaline modified
- Perk state changed (if applicable)
- Inventory count changed
- UpdatePacket includes new health/adrenaline on next network send

**Example:**
```typescript
// Player uses Medikit (100 health restore, 6 second use time)
// endTime = now + 6000
// Tick 0 (t=0ms): elapsed = 0, action awaiting completion
// Tick 120 (t=3000ms): elapsed = 3000, action half-done
// Tick 240 (t=6000ms): elapsed = 6000 >= 6000
//    → player.health += 100 (capped at 100)
//    → medikit count -= 1
//    → action removed
```

### `Player.useHealingItem(item: InventoryItem)` — @file server/src/objects/player.ts
**Purpose:** Initiate healing action; validate item and create HealingAction.

**Implicit behavior:**
- Checks if healing is allowed (player not stunned/frozen, healing not already in progress)
- Validates healing item is in inventory with count > 0
- Cancels current healing action if any (refunds partial duration or instant cancel)
- Creates new HealingAction with:
  - `player: this`
  - `item: healingItem`
  - `startTime: now`
  - `duration: item.definition.useTime × 1000`
- Adds action to `this.actions[]`
- Sets animation state: `this.animationType = AnimationType.Healing`
- Broadcasts to clients: new animation state in UpdatePacket
- Emits plugin event: `"healing_used"`

**Called by:** Input manager (when player presses heal key)

**Side effects:**
- Active action list modified
- Player animation state changed (visible to other clients)
- Input locked (player cannot move/attack during healing)

## Healing Item Types

| Item | Heal Type | Restore Amount | Use Time | Special Effect |
|------|-----------|----------------|----------|-----------------|
| **Gauze** | Health | 20 | 3s | — |
| **Medikit** | Health | 100 | 6s | — |
| **Cola** | Adrenaline | 25 | 3s | — |
| **Tablets** | Adrenaline | 50 | 4s | — |
| **Vaccine Syringe** | Special | 0 | 2s | Remove Infected perk, +50 Adrenaline |

## Configuration

| Setting | Source | Effect | Default |
|---------|--------|--------|---------|
| `restoreAmount` | Definition | Health/Adrenaline restored | Item-specific |
| `useTime` | Definition | Healing duration (seconds) | Item-specific |
| `healType` | Definition | Which stat restored (Health/Adrenaline/Special) | Item-specific |
| `hideUnlessPresent` | Definition | Hide in UI unless player carries | false |
| `GameConstants.player.maxHealth` | constants.ts | Health cap | 100 |
| `GameConstants.player.maxAdrenaline` | constants.ts | Adrenaline cap | 100 |

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Inventory system, item management, loot drops
- **Tier 2:** [../../health-damage/README.md](../../health-damage/README.md) — Health state management
- **Tier 2:** [../../perks-passive/README.md](../../perks-passive/README.md) — Perk system, Infected perk
- **Tier 2:** [../../game-loop/README.md](../../game-loop/README.md) — Action tick scheduling (40 TPS)
- **Tier 1:** [../../../../datamodel.md](../../../../datamodel.md) — HealingItemDefinition structure
- **Patterns:** [../patterns.md](../patterns.md) — Inventory action lifecycle
