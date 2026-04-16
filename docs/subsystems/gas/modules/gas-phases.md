# Gas System — Gas Phases & Progression

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/gas/README.md -->
<!-- @source: server/src/gas.ts, server/src/data/gasStages.ts, common/src/constants.ts -->

## Purpose
Implements the game's core gas zone mechanic: phase sequence definition, gas position/radius interpolation, damage scaling per phase, and safe zone shrinking animation timing.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/gas.ts` | Gas state machine, phase advancement, position/damage calculation | High |
| `server/src/data/gasStages.ts` | Phase definitions, duration, radius progression, damage rates | High |
| `common/src/constants.ts` | `GameConstants.gas.*` — damage formula, unscaled distance threshold | Medium |
| `common/src/utils/math.ts` | Lerp, distance calculation for gas damage | Medium |

## Business Rules

- **Phase Sequence:** Gas progresses through discrete stages (e.g., 15 phases in a typical BR game)
- **Phase 0 (Inactive):** Game start; gas not yet advancing
- **Phase Advancing:** Each phase has a start position (oldPosition), target position (newPosition), and transition duration
- **Safe Zone Shrinking:** Distance from oldRadius → newRadius over phase duration (linear lerp + 1 second per damage tick)
- **Damage Ticks:** Gas deals damage once per 1000ms (1 per second); calculation happens on tick, applied to doDamage flag
- **Damage Falloff:** Damage increases with distance into gas zone: `dps + max(0, distIntoGas - unscaledDamageDist) × damageScaleFactor`
- **Final Phase:** Last stage has `finalStage = true`; signals end-game imminent
- **Phase Duration Override:** Config can force all phases to same duration (for testing)

## Data Lineage

### Phase Progression
```
Game start: stage = 0, state = Inactive
  ↓
Time elapses, game populates (shrinking safezone encourages engagement)
  ↓
Trigger advanceGasStage() (from game event or timeout)
  ↓
Load GasStages[stage + 1]
  ↓
Copy oldPosition ← currentPosition (previous target becomes new start)
  ↓
newPosition = random point within oldRadius (stage-specific algorithm)
  ↓
newRadius = GasStages[stage].newRadius × mapSize
  ↓
state = Advancing (triggers damage)
  ↓
Completion ratio: (now - countdownStart) / (1000 × duration)
  ↓
While completionRatio < 1.0:
    currentPosition = lerp(oldPosition, newPosition, ratio)
    currentRadius = lerp(oldRadius, newRadius, ratio)
  ↓
When completionRatio >= 1.0:
    Advance to next stage or end game
```

### Damage Calculation
```
Player position known
  ↓
distIntoGas = distance(playerPos, currentGasPos) - currentRadius
  ↓
If distIntoGas <= 0: player in safe zone, no damage
  ↓
If distIntoGas > 0:
    unscaledDist = GameConstants.gas.unscaledDamageDist (e.g., 10 units)
    if distIntoGas > unscaledDist:
        extraDist = distIntoGas - unscaledDist
        scaledDamage = dps + (extraDist × GameConstants.gas.damageScaleFactor)
    else:
        scaledDamage = dps (base gas damage for this phase)
  ↓
Apply scaledDamage to player.health per damage tick (1/second)
```

### Position Progression Within Phase
```
Phase start time: countdownStart = now
Duration: currentDuration (e.g., 75 seconds for first shrink)
  ↓
Each server tick (40 TPS = 25ms):
    completionRatio = (now - countdownStart) / (1000 × duration)
    
    If state == Advancing:
        currentPosition = Vec.lerp(oldPosition, newPosition, ratio)
        currentRadius = Numeric.lerp(oldRadius, newRadius, ratio)
  ↓
Result: Safe zone smoothly shrinks from oldRadius to newRadius over duration
```

## Complex Functions

### `Gas.advanceGasStage()` — @file server/src/gas.ts
**Purpose:** Transition to next gas phase, calculate new safe zone position/radius.

**Implicit behavior:**
- Stage increments: `this.stage++`
- Loads `GasStages[this.stage]` — must exist or returns early
- Duration determined: if Config.gas.forceDuration exists, use that; otherwise use stage.duration
- Saves old endpoint as new start: `oldPosition ← currentPosition`, `oldRadius ← currentRadius`
- Generates new target position (algorithm depends on game mode, may be random or fixed)
- Sets `newRadius` from stage definition, scaled by mapSize
- Sets `state` to `Advancing`, `countdownStart` to now
- Sets `finalStage` flag if this is the last phase from GasStages
- Special: In Hunted Mode, checks if this stage unlocks bunker doors (`mode.unlockStage`)

**Called by:** Game timeout (e.g., every 75–200 seconds) or phase scheduler

**Side effects:**
- Emits plugin event or broadcast to players (new safe zone position)
- May unlock obstacles (doors in Hunted mode)

**Example:**
```typescript
// Gas is at stage 3, about to enter phase 4
// oldRadius = 1000 units, currentPosition = (350, 350)
gas.advanceGasStage();
// Result: stage = 4, oldPosition = (350, 350), oldRadius = 1000
//         newPosition = (250, 250), newRadius = 800, state = Advancing
```

### `Gas.scaledDamage(position)` — @file server/src/gas.ts
**Purpose:** Calculate damage per tick for player at given position.

**Implicit behavior:**
- Calculates `distIntoGas = distance(position, currentPosition) - currentRadius`
- If distIntoGas ≤ 0, player is safe, returns 0
- If distIntoGas > 0:
  - Clamps extra distance:  `extraDist = max(0, distIntoGas - GameConstants.gas.unscaledDamageDist)`
  - Scales damage: `resultDamage = dps + extraDist × GameConstants.gas.damageScaleFactor`
- Returns scaled damage (damage per second, applied once per 1000ms)

**Called by:** `Player.takeDamage()` during damage phase of game loop

**Example:**
```typescript
// Player at (0, 0), gas center at (200, 200), radius 100
// distIntoGas = distance((0, 0), (200, 200)) - 100 ≈ 183
// extraDist = 183 - 10 = 173
// dps = 3, damageScaleFactor = 0.1
// scaledDamage = 3 + (173 × 0.1) = 20.3 DPS
```

### `Gas.tick()` — @file server/src/gas.ts
**Purpose:** Per-server-tick gas update; increment completion ratio, interpolate position/radius, set damage flag.

**Implicit behavior:**
- Recalculates `completionRatio` if gas is not Inactive:
  - `ratio = (now - countdownStart) / (1000 × duration)`
- If 1000ms has passed since last damage tick:
  - Sets `_doDamage = true` (so next player update applies damage)
  - Resets damage timestamp
- If state == Advancing and enough time passed:
  - Interpolates: `currentPosition = lerp(oldPosition, newPosition, ratio)`
  - Interpolates: `currentRadius = lerp(oldRadius, newRadius, ratio)`
- Sets `completionRatioDirty` to signal clients of new ratio (for UI countdown)

**Called by:** Main game loop at 40 TPS (once per 25ms)

## Configuration
| Setting | Source | Effect |
|---------|--------|--------|
| `GameConstants.gas.unscaledDamageDist` | constants.ts | Distance into gas before damage scaling starts (e.g., 10 units) |
| `GameConstants.gas.damageScaleFactor` | constants.ts | Damage per unit distance into gas (e.g., 0.1 DPS/unit) |
| `Config.gas.disabled` | config.json | If true, gas disabled (no advancing) |
| `Config.gas.forceDuration` | config.json | Override all phase durations to this value (for testing) |

## Gas Stage Definition Format

```typescript
// From GasStages[]
{
    duration: 75,              // Seconds to move from old to new safe zone
    oldRadius: 0.7,            // Relative to map size (0-1)
    newRadius: 0.5,            // Target radius
    state: GasState.Advancing,
    dps: 3,                    // Damage per second at zone edge
    scaleDamageFactor?: 0.15,  // Optional: override default damage scale
    finalStage?: true          // Optional: marks end-game phase
}
```

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Damage application, plugin events, hunted mode integration
- **Tier 2:** [../../game-loop/README.md](../../game-loop/README.md) — Game tick loop, phase synchronization
- **Tier 2:** [../../health-damage/README.md](../../health-damage/README.md) — Player damage application
- **Tier 1:** [../../../../datamodel.md](../../../../datamodel.md) — GasState enum, gas packet structure
- **Patterns:** [../patterns.md](../patterns.md) (if exists) — Gas phase event lifecycle
