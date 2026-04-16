# Knockback & Velocity — Ice Floor Physics

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/knockback-velocity/README.md -->
<!-- @source: server/src/objects/player.ts, common/src/utils/terrain.ts, common/src/constants.ts -->

## Purpose
Implements slippery ice surface physics: reduced friction/acceleration on ice, deceleration curve on normal surfaces, surface transitions, and momentum conservation for gameplay balance.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/player.ts` | Player velocity updates, ice floor detection, friction application | High |
| `common/src/utils/terrain.ts` | FloorTypes enum, ice material identification | Medium |
| `common/src/constants.ts` | `GameConstants.player.ice*` friction, acceleration coefficients | Medium |
| `common/src/utils/vector.ts` | Vec type, velocity math (lerp, magnitude, normalize) | Medium |

## Business Rules

- **Ice Surface Detection:** Player's current floor checked each tick; determines friction coefficient
- **Reduced Acceleration:** On ice, `acceleration *= iceAccelerationMultiplier` (e.g., 0.5 = half normal)
- **Friction Coefficient:** Ice has lower friction than grass/concrete
  - Grass/normal: `friction = 0.15` (fast deceleration)
  - Ice: `friction = 0.05` (slow deceleration, player slides)
- **Velocity Capping:** Max velocity clamped per surface type; ice allows higher momentum retention
- **Momentum Conservation:** Switching from ice → normal ground maintains current velocity, then decelerates
- **Ice Sliding:** Player continues moving after input stops; input duration doesn't match distance traveled
- **Knockback Interaction:** Explosive/melee knockback applies to velocity vector; ice doesn't reduce knockback, but deceleration is slower
- **Frame-Perfect Movement:** Timing inputs on ice to maintain momentum is skill expression

## Data Lineage

### Surface Detection & Friction Application
```
Player position known
  ↓
Terrain grid lookup: terrain[x, y]
  ↓
FloorType determined (Grass, Ice, Water, etc.)
  ↓
Each server tick (40 TPS = 25ms):
    if floor == Ice:
        frictionCoeff = GameConstants.player.iceFriction
        accelMult = GameConstants.player.iceAccelerationMultiplier
    else:
        frictionCoeff = GameConstants.player.normalFriction
        accelMult = 1.0
  ↓
Input acceleration applied: vel += inputAccel × accelMult
  ↓
Friction applied: vel *= (1 - frictionCoeff)
  ↓
Velocity capped: vel = clamp(vel, -maxVel, maxVel)
```

### Momentum Transfer on Impact
```
Explosion detonates near player
  ↓
calcKnockback(explosionPos, playerPos, force)
  ↓
Direction = normalize(playerPos - explosionPos)
  ↓
Knockback velocity = direction × force
  ↓
Applied to player.velocity (adds to existing velocity)
  ↓
On ice: high knockback, slow deceleration → slides far
  ↓
On grass: high knockback, fast deceleration → stops sooner
```

### Deceleration Curve Over Time
```
Player on ice, velocity = 5.0 units/tick
  ↓
No input, coast to stop
  ↓
Tick 0: vel = 5.0
Tick 1: vel = 5.0 × (1 - 0.05) = 4.75
Tick 2: vel = 4.75 × 0.95 = 4.5125
Tick 3: vel = 4.29...
...continues...
Tick 30: vel ≈ 0.0

Total distance = Σ vel[i] ≈ 100 units
  ↓
Player on grass, same starting velocity
  ↓
Tick 1: vel = 5.0 × (1 - 0.15) = 4.25
Tick 2: vel = 3.6...
...
Tick 10: vel ≈ 0.0

Total distance = Σ vel[i] ≈ 35 units (3× shorter!)
```

## Complex Functions

### `Player.updateVelocity()` — @file server/src/objects/player.ts
**Purpose:** Update player velocity per server tick based on acceleration input and friction.

**Implicit behavior:**
1. Detect current floor type from terrain grid
2. Choose friction coefficient based on floor (ice vs normal)
3. Apply input acceleration (from keys held):
   - `inputAccel = moveDir × moveSpeed × accelMult`
   - If on ice: `accelMult = GameConstants.player.iceAccelerationMultiplier` (0.5)
   - Else: `accelMult = 1.0`
4. Add to velocity: `velocity += inputAccel × deltaTime`
5. Apply friction (deceleration): `velocity *= (1 - friction)`
6. Cap velocity:
   - Ice: `maxSpeed = GameConstants.player.maxVelocityOnIce` (e.g., 15)
   - Normal: `maxSpeed = GameConstants.player.maxSpeed` (e.g., 10)
   - `velocity = clamp(velocity, -maxSpeed, -maxSpeed)`
7. Update position: `position += velocity × deltaTime`
8. Clamp position to map bounds

**Called by:** Game loop update phase per server tick

**Example:**
```typescript
// Player on ice, pressing forward (inputDir = 1.0), no prior velocity
// Tick 0: 
//   inputAccel = 1.0 × 2.5 × 0.5 = 1.25 (normal=2.5, ice×0.5)
//   velocity = 0 + 1.25 = 1.25
//   velocity *= (1 - 0.05) = 1.1875
//   position += 1.1875 × (25/1000) ≈ 0.03 units

// Tick 1:
//   velocity = 1.1875 + 1.25 = 2.4375
//   velocity *= 0.95 = 2.31
//   position += 2.31 × 0.025 ≈ 0.06 units

// After 20 ticks of input: velocity ≈ 12 (ramping up)
// Stop input at tick 20:
// Tick 21:
//   velocity = 12 × 0.95 = 11.4 (slow coast)
// Tick 40:
//   velocity ≈ 0.5 (still sliding)
```

### `calculateFriction(floor: FloorType)` — @file server/src/objects/player.ts
**Purpose:** Determine friction coefficient based on current floor surface.

**Implicit behavior:**
- Lookup floor type in `FloorTypes` enum
- Return corresponding friction value:
  - `Ice` → `GameConstants.player.iceFriction`
  - `Water` → `GameConstants.player.waterFriction`
  - `Grass/Concrete/etc` → `GameConstants.player.normalFriction`
- Some surfaces (water) may have additional drag or slow effect

**Called by:** `updateVelocity()` each tick

**Example:**
```typescript
const floor = terrain.grid[playerX][playerY];
const friction = calculateFriction(floor);
// floor = FloorTypes.Ice → friction = 0.05
// floor = FloorTypes.Grass → friction = 0.15
```

## Ice Physics Constants

| Constant | Value | Effect |
|----------|-------|--------|
| `iceFriction` | 0.05 | Deceleration per tick on ice (5% per tick) |
| `normalFriction` | 0.15 | Deceleration per tick on grass (15% per tick) |
| `iceAccelerationMultiplier` | 0.5 | Acceleration reduced to 50% on ice |
| `maxSpeed` | ~10 units/tick | Velocity cap on normal floor |
| `maxVelocityOnIce` | ~15 units/tick | Higher velocity allowed on ice |
| `moveSpeed` | 2.5 units/tick | Acceleration magnitude per input direction |

## Surface Transitions

### Normal → Ice (`lastFloor = Grass, currentFloor = Ice`)
- Velocity preserved immediately
- Friction changes to 0.05 (deceleration slows)
- Acceleration multiplier changes to 0.5 (harder to change direction)
- Player maintains momentum but harder to control

### Ice → Normal (`lastFloor = Ice, currentFloor = Grass`)
- Velocity preserved initially
- Friction jumps to 0.15 (deceleration speeds up)
- Acceleration multiplier resets to 1.0 (normal control restored)
- If player coasting on ice at velocity 10, stops quickly on grass

### Water Floor
- Friction similar to grass (maybe slightly higher)
- Slow effect applied (animation + speed reduction)
- Cannot maintain high momentum

## Gameplay Implications

| Scenario | Normal Floor | Ice Floor |
|----------|--------------|-----------|
| **Acceleration** | Fast (100% accel) | Slow (50% accel) |
| **Deceleration** | Fast (15% friction) | Slow (5% friction) |
| **Max Speed** | 10 u/tick | 15 u/tick |
| **Travel Time to Stop (from max speed)** | ~7 ticks | ~30 ticks |
| **Distance Traveled (max speed → stop)** | ~50 units | ~150 units |
| **Knockback Distance** | ~70 units | ~200 units |
| **Skill Expression** | Precise movement | Momentum conservation |

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Knockback mechanics, velocity physics
- **Tier 2:** [../../explosions-hazards/README.md](../../explosions-hazards/README.md) — Knockback from explosions
- **Tier 2:** [../../game-loop/README.md](../../game-loop/README.md) — Server tick at 40 TPS, update phase
- **Tier 1:** [../../../../development.md](../../../../development.md) — Testing ice physics, config
- **Tier 1:** [../../../../constants.md](../../../../constants.md) — GameConstants.player friction values
