# Game Loop — Player Update Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/objects/player.ts -->

## Purpose
Encapsulates per-tick player state updates: movement, animation, input processing, status effects, and network dirty-flag management for the primary player update phase.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/player.ts` | Player class with `update()` / `secondUpdate()` methods | High |
| `server/src/inventory/` | Action system (reload, healing, revive) | Medium |
| `common/src/definitions/` | Player-affecting definitions (perks, items, status) | Medium |

## Player State Machine

Players exist in three primary states:

```
┌─────────┐  take_damage    ┌────────┐  down_timeout     ┌──────┐
│  Alive  │ ───────────────▶ │ Downed │ ──────────────────▶ │ Dead │
│ downed  │ (if HP > 0)      │ downed │ (if not revived)    │      │
│= false  │                  │= true  │                     │      │
└─────────┘                  └────────┘                     └──────┘
   △                             │                           △
   │                             │                           │
   └─────────────────────────────┘                           │
          revive_action           any_living_damage ──────────┘
          (healing action)
```

**State Flags:**
- `downed: boolean` — player knocked down but alive (can be revived)
- `dead: boolean` — game object destroyed; used for synchronization
- `joined: boolean` — client finished joining (prevents premature input processing)
- `activeBloodthirstEffect: boolean` — temporary state from perk or item effect

**Important:** `downed` and `dead` are independent. A player can be `downed && !dead` (revivable) or `downed && dead` (unrevivable in some modes).

## Per-Tick Update Sequence

The `update()` method runs once per game tick (40 TPS = 25 ms). The sequence is critical:

### 1. Building & Scope Detection (Lines 1125–1145)
```typescript
// Iterate nearObjects (from spatial grid)
// Check ceiling hitbox collisions
// Detect scope target (for zoom adjustment)
// Check synced particles (smoke effects)
```

**Output:** `isInsideBuilding`, `scopeTarget`, `syncedParticles` set

**Why it matters:** Scope and building state affects speed multipliers and visibility calculations. Must run before movement.

### 2. Movement Speed Calculation (Lines 1150–1230)
```
base_speed = GameConstants.player.baseSpeed
  × shoulder_multiplier (holster scope vs. equipped gun)
  × active_item_speed_multiplier (gun, throwable, melee slowdown)
  × downed_multiplier (0.5x if downed)
  × perk_speed_mods (Advanced Athletics water/smoke, Claustrophobic building)
  × adrenaline_speed_mod (logarithmic scaling 0–100 adrenaline)
  × recoil_multiplier (weapon recoil penalty lingering from last shot)
```

**Key function:** `Numeric.lerp()` for smooth adrenaline scaling
**Formula reference:** @file server/src/objects/player.ts:1188

**Output:** `this.velocity` updated; position updated next tick via `BaseGameObject.update()`

### 3. Animation & Emote Management (Lines 1230–1280)
- Expired emotes removed from `playingEmotes` set
- Animation frames advanced (idle or moving animation based on velocity)
- Perks that modify animation (e.g., Bloodthirst) applied

### 4. Input Processing

**Scope (if player `joined` and not `dead`):** @file server/src/game.ts:294
- Only process `InputPacket` if `!player.dead && !this.over`
- Input contains: `movement`, `rotation`, `action`, `weaponIndex`

**Input Actions** (enum `PlayerActions`):
- `Attack` — fire gun or swing melee
- `Interact` — pickup loot, revive teammate, open door
- `Emote` — play emote with rate limiting
- `Reload` — start reload action
- `EquipLastUsedWeapon` — quick swap

**Processing:** Player validates `InputData` and queues actions into inventory system

### 5. Status Effects Per-Tick (Lines 1300–1450)

#### Healing Effects
- `health`: regenerate if `adrenaline > 0` (1 HP/tick)
- `adrenaline`: decay if not healing (controlled by item use)
- `shield`: passthrough damage pool; regenerates if `Experimental Forcefield` perk

#### Damage Over Time (DoT)
- `infection`: increases if in cyan/green zone OR inside building with ceiling infection
- `gas damage`: applied per tick; cumulative damage storage resets on config

#### Knockback & Recoil
- Lingering `recoil.time` countdown — affects speed multiplier
- `knockback` velocity fades each tick (affects movement vector next frame)

#### Perk Effects
- **Bloodthirst:** `activeBloodthirstEffect` flag applied per tick (visual effect, speed boost)
- **Overcharge:** `activeOverdrive` cooldown tracked; kill counter reset after cooldown expires

### 6. Visibility Calculation (Lines 1450–1700)
**Purpose:** Determine which objects this player can see (for client-side visibility, map indicators)

**Algorithm:**
1. Iterate all game objects via `this.nearObjects` (from spatial grid)
2. For each object, test:
   - **Distance:** Is it within render distance?
   - **Layer:** Is it on same/adjacent layer?
   - **Occlusion:** Is there line-of-sight (LOS)?
3. Store visible objects for later network serialization

**LOS Calculation:** `Geometry.getIntersection()` vs building walls/obstacles
**Scope effect:** Scope can see further (adjusted hitbox for LOS checks)

**Output:** Visibility bitmasks set on objects for packet filtering

### 7. Dirty Flag Management (Lines 1750–1780)

After all state changes, mark what needs network synchronization:

```typescript
this.dirty.position = true;  // If velocity != 0
this.dirty.rotation = true;  // If input rotation changed
this.dirty.health = true;    // If health/adrenaline changed
this.dirty.animation = true; // If animation frame advanced
```

**Why:** Network packet construction reads these flags to avoid sending unchanged data

**Full update trigger:** If player joins this tick or major state change, set `_firstPacket = true`

## Inventory-Coupled Updates

The `update()` method coordinates with the **Inventory Subsystem**:

### Action Processing
- **Reload action:** Trigger via `InputPacket.action == PlayerActions.Reload`
- **Healing action:** Decouple from `update()` — processed asynchronously
- **Revive action:** Teammate reviving this downed player

**Code:** Actions queued in `this.inventory`; resolved in `secondUpdate()`

### Weapon State
- Active gun (`this.activeItem` if `instanceof GunItem`)
- Gun tracks ammo per clip, magazine count
- Reload timeout prevents instant firing after reload

## Network Dirty Flag System

**Full-update bytes:** 16 bytes per player per tick (if dirty flags all set)
**Partial-update bytes:** 12 bytes per player per tick (if only minor changes)

```typescript
override readonly fullAllocBytes = 16;
override readonly partialAllocBytes = 12;
```

**Dirty flags tracked:** @file server/src/objects/gameObject.ts
```typescript
this.dirty = {
  position: false,
  rotation: false,
  health: false,
  animation: false,
  inventory: false,
  teammates: false,
  // ... more flags
};
```

**Usage:** In `secondUpdate()`, check flags before serializing data into `UpdatePacket`

## Complex Functions

### `update()` — @file server/src/objects/player.ts:1125
**Purpose:** Main per-tick state machine update
**Complexity:** ~400 lines of state tracking
**Called by:** Game tick loop in `game.ts:465`
**Implicit behavior:**
- Modifies `this.position`, `this.velocity`, `this.rotation`
- Updates all status effect timers
- Sets visibility and dirty flags
- Does NOT send any packets (that's `secondUpdate()`'s job)

### `secondUpdate()` — @file server/src/objects/player.ts:1799
**Purpose:** Serialize state into network packet, post-update cleanup
**Complexity:** ~100 lines of serialization
**Called by:** Game tick loop in `game.ts:479`
**Implicit behavior:**
- Reads dirty flags from `update()`
- Constructs packet data via `fullUpdate()` or `partialUpdate()`
- Resets dirty flags for next tick
- Cleans up expired timers (emote rate limit, status effect timeouts)

### `updateWithInputData(data: InputData)` — @file server/src/objects/player.ts
**Purpose:** Process a single input packet frame
**Parameters:**
- `data.movement` — vector [-1, 1] for each axis
- `data.rotation` — player look direction
- `data.action` — active button press
- `weaponIndex` — inventory slot selected
**Implicit behavior:**
- Validates player state (not dead, game not over)
- Applies movement immediately
- Queues action if valid (rate-limited for emotes)

## Configuration & Tuning

Base player constants: @file common/src/constants.ts
```typescript
GameConstants.player = {
  baseSpeed: 7.5,           // px/ms
  speedSmoothing: 0.15,     // lerp factor
  radius: 2.4,              // hitbox radius
  defaultHealth: 100,       // max HP
  adrenalineDecayRate: 1.5  // units/tick
}
```

Zone damage multiplier: @file server/src/data/gasStages.ts
- Gas damage ramps up over time as safe zone shrinks
- Each gas stage has different damage rate

## Related Documents
- **Tier 2:** [Game Loop](../README.md) — subsystem overview  
- **Tier 2:** [Inventory](../../inventory/README.md) — action processing
- **Tier 3:** [Object Update](object-update.md) — sibling module
- **Tier 3:** [Game State](game-state.md) — player state transitions
- **Patterns:** [../patterns.md](../patterns.md) — subsystem patterns
