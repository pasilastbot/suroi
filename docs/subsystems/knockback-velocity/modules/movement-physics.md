# Movement Physics

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/knockback-velocity/README.md -->
<!-- @source: server/src/objects/player.ts -->

## Purpose

Player movement velocity system, acceleration/deceleration mechanics, and terrain friction effects. Core of character locomotion: transforms input direction + modifiers into position updates via velocity vectors. Handles normal terrain (instant acceleration) and ice terrain (gradual acceleration with friction).

## Key Files

| File | Purpose | Complexity |
|------|---------|-----------|
| `server/src/objects/player.ts` (lines 611, 1200–1280) | Velocity state (`_movementVector`), acceleration calculation, position update loop | High |
| `common/src/constants.ts` | Physics constants (baseSpeed, ice friction, drag) | Low |
| `common/src/utils/math.ts` | Vector utils (scale, add, sub, normalize, fromPolar) | Medium |
| `server/src/utils/terrain.ts` | Floor type definitions with speedMultiplier | Medium |

## Velocity System

### State & Accessor

```typescript
// server/src/objects/player.ts line 611
private _movementVector = Vec(0, 0);  // Internal velocity vector
get movementVector(): Vector { return Vec.clone(this._movementVector); }
```

**Properties:**
- **Type:** 2D vector `Vector = { x: number, y: number }`
- **Units:** Game units per millisecond (updated every 25ms at 40 TPS)
- **Range:** Unbounded; clamped indirectly via speed modifiers

**Read-only public accessor** prevents external code from directly mutating velocity state. All velocity changes go through the physics tick update.

### Maximum Velocity Cap

**Player velocity cap is NOT hard-limited.** Instead, max speed is controlled implicitly through the `speed` multiplier:

```typescript
// server/src/objects/player.ts lines 1207–1219
const speed = this.baseSpeed                                    // 0.06
    * (FloorTypes[this.floor].speedMultiplier ?? 1)             // terrain
    * recoilMultiplier                                           // gun kickback
    * perkSpeedMod                                               // perk bonuses
    * (this.action?.speedMultiplier ?? 1)                        // action (heal, revive)
    * adrenSpeedMod                                              // adrenaline
    * (this.downed ? 0.5 : (this.activeItemDefinition.speedMultiplier ?? 1))  // held item
    * (this.beingRevivedBy ? 0.5 : 1)                            // being revived
    * this.effectSpeedMultiplier                                 // status effects
    * this._modifiers.baseSpeed;                                 // on-wearer modifiers
```

**On normal terrain:** `_movementVector` is set directly to `desiredVelocity = Vec.scale(movement, speed)` (line 1273)
- E.g., if baseSpeed=0.06 and all modifiers=1.0, then max velocity = 0.06 units/ms = 60 units/s at 40 TPS

**On ice:** Acceleration limits max velocity scaling. See [Acceleration & Deceleration](#acceleration--deceleration) section.

### Network Serialization

Velocity is sent to clients in the UpdatePacket:

```typescript
// common/src/packets/updatePacket.ts (player movement data)
movementVector: { x: number, y: number }
```

Clients use this for **position extrapolation** during network latency. Position = `lastPos + movementVector * dt` until next packet.

## Movement Input

### Input Direction Calculation

```typescript
// server/src/objects/player.ts lines 1233–1250
const playerMovement = this.movement;
if (this.isMobile && playerMovement.moving) {
    movement = Vec.fromPolar(playerMovement.angle);
} else {
    let x = +playerMovement.right - +playerMovement.left;    // WASD booleans: right=d, left=a
    let y = +playerMovement.down - +playerMovement.up;       // down=s, up=w
    
    if (x * y !== 0) {
        // Diagonal movement: both components non-zero
        x *= Math.SQRT1_2;  // = 1/√2 ≈ 0.707
        y *= Math.SQRT1_2;
    }
    
    movement = Vec(x, y);
}
```

**Directional Encoding:**
- **Cardinal (N, E, S, W):** magnitude = 1.0 → velocity magnitude = 0.06
- **Diagonal (NE, SE, SW, NW):** magnitude = √2 ≈ 1.414 → scaled by 1/√2 → magnitude = 1.0
  - Prevents diagonal movement being 1.41× faster (equal speed in all directions)
- **Mobile joystick:** Angle from `playerMovement.angle`, normalized to unit vector

### Perk Modifiers (AchingKnees)

```typescript
// server/src/objects/player.ts lines 1252–1256
if (this.hasPerk(PerkIds.AchingKnees) && this.reversedMovement) {
    movement.x = -movement.x;
    movement.y = -movement.y;
}
```

**AchingKnees** (negative perk) reverses movement direction when callback activates. Movement vector is negated **after** input normalization.

### Sprint Multiplier (Adrenaline)

```typescript
// server/src/objects/player.ts lines 1210–1211
const adrenSpeedMod = (() => {
    const ratio = this._adrenaline / GameConstants.player.maxAdrenaline;
    // Logarithmic curve: fits Desmos equation...
    const a = 0.944297822457;
    const b = -0.0158132859327;
    const c = 0.699999999995;
    const d = 3.51269916486;
    return b * Math.log(this.adrenaline + d) / Math.log(c) + a;
})();
```

**Adrenaline speed boost:**
- **0 adrenaline:** adrenSpeedMod ≈ 0.944 (−5.6% slowdown)
- **50 adrenaline (mid):** adrenSpeedMod ≈ 0.994 (closer to 1.0)
- **100 adrenaline (max):** adrenSpeedMod → 1.0 (no penalty, slight bonus)

Logarithmic curve prevents adrenaline sprint from being too overpowered. No hard "sprint" mode; speed scaling is smooth continuous curve based on adrenaline level.

## Acceleration & Deceleration

### Normal Terrain (Grass, Sand, Water, Concrete)

```typescript
// server/src/objects/player.ts line 1273
this._movementVector = desiredVelocity;
```

**Behavior:** Velocity **instantaneously snaps** to `desiredVelocity = Vec.scale(movement, speed)` each tick.

**Why?** Normal terrain is non-slippery. Player has full grip. No gradual acceleration needed.

**Consequences:**
- Rapid direction changes (180° turn) are instant
- No "slide" or "momentum" on normal terrain
- Deceleration is also instant: input no movement → `movement = Vec(0, 0)` → velocity = 0

### Ice Terrain (Slippery)

```typescript
// server/src/objects/player.ts lines 1262–1271
const dtSeconds = dt / 1000;  // Delta time in seconds (dt is milliseconds)
const onIce = FloorTypes[this.floor].slippery ?? false;
const hasMovementInput = movement.x !== 0 || movement.y !== 0;

if (onIce) {
    const accelFactor = Numeric.clamp(
        GameConstants.player.ice.acceleration * dtSeconds, 
        0, 1
    );
    
    if (hasMovementInput) {
        // Blend current velocity toward desired velocity
        this._movementVector = Vec.add(
            this._movementVector,
            Vec.scale(
                Vec.sub(desiredVelocity, this._movementVector), 
                accelFactor
            )
        );
    }
    
    // Apply friction
    const friction = hasMovementInput 
        ? GameConstants.player.ice.movingFriction  // 0.45
        : GameConstants.player.ice.friction;      // 0.2
    this._movementVector = Vec.scale(
        this._movementVector, 
        1 / (1 + dtSeconds * friction)
    );
}
```

**Ice Mechanics:**

1. **Acceleration Phase (if movement input exists):**
   - `accelFactor = clamp(2.5 * dtSeconds, 0, 1)`
     - At dt=25ms: `accelFactor = 2.5 * 0.025 = 0.0625` (≈ 6.25%)
     - Velocity approaches desired velocity gradually
   - Formula: `velocity += (desiredVel - velocity) * accelFactor`
     - First tick: move 6.25% toward target
     - Convergence: ~3–4 seconds to reach target velocity from rest

2. **Friction Phase (always applied after acceleration):**
   - `velocity *= 1 / (1 + dt * friction)`
   - **No movement input:** friction = 0.2 (slow decay)
   - **With movement input:** friction = 0.45 (faster decay, prevents uncontrolled buildup)
   - Example with friction=0.45, dt=25ms:
     - `velocity *= 1 / (1 + 0.025 * 0.45) = 1 / 1.01125 ≈ 0.989` (1.1% loss per tick)

**Ice Constants:**
```typescript
// common/src/constants.ts
player: {
    ice: {
        acceleration: 2.5,        // units/s² (game units per second squared)
        friction: 0.2,            // static friction (no input)
        movingFriction: 0.45      // kinetic friction (with input)
    }
}
```

**Analogy:** 
- On ice, pressing WASD is like pushing, not walking. Player slides with inertia.
- Player stops pushing (release keys) → friction brings them to a halt (~5 seconds)
- Friction while moving prevents velocity explosion

### Friction Formula Explanation

Formula: `v_new = v_old / (1 + Δt × f)`

This is **exponential decay without clamping to zero bounds:**
- Always reduces velocity
- Never overshoots below zero
- Mathematically: `v(t) = v₀ / (1 + f·t)` is solution to `dv/dt = -f·v` (standard drag equation)

**Why not `v *= (1 - f*dt)`?**
- That's Euler integration, less stable, can oscillate
- Suroi uses implicit form for numerical stability

## Surface Types & Speed Modifiers

### All Terrain Types

| Terrain | Type | Speed Multiplier | Friction | Drag (projectile) | Notes |
|---------|------|------------------|----------|-------------------|-------|
| Grass | Default | 1.0 | — | 0.003 | Baseline terrain |
| Sand | Walkable | ~0.9 | — | 0.003 | Slightly slower |
| Water | Liquid | 0.7 | — | 5.0 | 30% speed reduction; extreme projectile drag |
| Ice | Slippery | 1.0 | 0.2/0.45 | 0.0008 | Acceleration model (see above); low projectile drag |
| Concrete | Hard | 1.0 | — | 0.003 | Urban terrain; instant accel |
| Lava | Hazard | ~0.8* | — | 0.003 | *estimated, applies slow effect |

Speed multipliers from `common/src/utils/terrain.ts` `FloorTypes[floorId].speedMultiplier`. Applied multiplicatively in speed calculation:

```typescript
const speed = baseSpeed * (FloorTypes[this.floor].speedMultiplier ?? 1) * ...;
```

**Water Special Case:**
- Direct movement penalty (0.7×)
- Extreme projectile drag (5 vs 0.003) to slow thrown/fired projectiles significantly
- No bespoke player acceleration model (uses instant acceleration like grass)

## Obstacle Collision & Velocity Handling

### Collision Resolution Loop

```typescript
// server/src/objects/player.ts lines 1275–1310
this.position = Vec.add(
    this.position,
    Vec.scale(this.movementVector, dt)
);

// Find and resolve collisions
this.nearObjects = this.game.grid.intersectsHitbox(this._hitbox, this.layer);

for (let step = 0; step < 10; step++) {
    let collided = false;
    
    for (const potential of this.nearObjects) {
        const { isObstacle, isBuilding } = potential;
        
        if (
            (isObstacle || isBuilding)
            && this.mapPerkOrDefault(PerkIds.AdvancedAthletics, ..., true)  // perk check
            && potential.collidable
            && potential.hitbox?.collidesWith(this._hitbox)
        ) {
            if (isObstacle && potential.definition.isStair) {
                // Handle stair layer transition
                potential.handleStairInteraction(this);
                this.activeStair = potential;
            } else if (!this._noClip) {
                collided = true;
                this._hitbox.resolveCollision(potential.hitbox);  // Separate hitboxes
            }
        }
    }
    
    if (!collided) break;  // Exit early if no collision this step
}
```

**Sequence:**
1. Update position: `position += velocity × dt`
2. Check for overlaps with collidable obstacles/buildings
3. Call `hitbox.resolveCollision()` to separate geometries
4. Repeat up to 10 times (for complex geometry)
5. Exit early if no collisions detected

**Velocity on Wall Hit:**
- Velocity is **NOT zeroed** automatically on collision
- Only position is adjusted (geometric separation)
- Next tick: velocity is recalculated from input
  - If still holding movement key toward wall → velocity = 0 (can't push through)
  - If holding perpendicular input → velocity is tangent to wall (slide along wall)

**No Momentum Loss:**
- Collision does not dampen velocity
- Velocity persists; next input tick recalculates from desired direction

### Stairs (Layer Transitions)

```typescript
// server/src/objects/player.ts line 1302–1305
if (isObstacle && potential.definition.isStair) {
    const oldLayer = this.layer;
    potential.handleStairInteraction(this);
    if (this.layer !== oldLayer) this.setDirty();
    this.activeStair = potential;
}
```

Stairs allow upward layer transitions (e.g., ground → building interior) without collision blocking. Velocity is **preserved** during stair traversal; no friction applied.

### AdvancedAthletics Perk

```typescript
// server/src/objects/player.ts line 1290
&& this.mapPerkOrDefault(PerkIds.AdvancedAthletics, ..., true)
```

Players with **AdvancedAthletics** perk can climb/vault obstacles that would normally block movement:
- Tree obstacles: can walk through
- Windows: can climb through
- Excludes buildings and stairs

Perk does NOT affect velocity; only expands collidable geometry.

## Knockback System

### Current Status: NOT IMPLEMENTED FOR PLAYERS

Explosions apply damage to players but **do NOT apply velocity knockback:**

```typescript
// server/src/objects/explosion.ts lines 87–95  (from explosion.ts)
if (isPlayer || isObstacle || isBuilding) {
    object.damage({
        amount: ...,
        source: this.source,
        weaponUsed: this
    });
    // ^ Only damage call, NO push/knockback for players
}
```

**Knockback mechanics exist only for:**
- **Loot items** (small drops, ammo, weapons)
- **Projectiles** (grenades, throwables bouncing)

See [Knockback & Velocity Ballistics](../../../projectiles-ballistics/modules/collision-damage.md#knockback-on-explosion) for explosion knockback formulas on non-player objects.

### Future Knockback Implementation (Hypothetical)

If player knockback were added, mechanics would likely be:

```typescript
// Hypothetical knockback application in explosion.ts
const knockbackMagnitude = (maxRadius - dist) * knockbackMultiplier;
const knockbackDir = Angle.betweenPoints(this.position, playerPos);
const knockbackVel = Vec.fromPolar(knockbackDir, knockbackMagnitude);
player._movementVector = Vec.add(player._movementVector, knockbackVel);  // Additive
```

**Knockback would:**
- Be **additive** to existing velocity (no clamping/damping)
- Ignore current movement direction (direct impulse)
- Affect both X and Y components of velocity simultaneously
- Interact poorly with ice (could accumulate infinite velocity if multiple explosions overlap)

### Knockback Resistance: NOT APPLICABLE

No perks currently reduce knockback for players (since knockback is not implemented).

## Air Movement (Hypothetical)

Currently, **players cannot be airborne.** No jumping mechanic exists.

If jumping were added:

```typescript
// Hypothetical air physics (not in code)
if (this.altitude > 0) {
    // Apply gravity
    this._velocityZ -= GameConstants.gravity.player * dt;  // default: 10 units/s²
    this.altitude += this._velocityZ * dt;
    
    // Horizontal air control (reduced input)
    const airInputMod = 0.5;
    const airDesiredVel = Vec.scale(movement, speed * airInputMod);
    this._movementVector = Vec.add(
        this._movementVector,
        Vec.scale(Vec.sub(airDesiredVel, this._movementVector), 0.1)
    );
}
```

Actual game mechanics:
- Players are **always grounded**
- Altitude is used only for obstacles (rooftops) and special geometry (stairs)
- No Z-axis velocity simulation

## Speed Modifiers

### Weapon Penalties

Holding active weapon reduces movement speed:

```typescript
// common/src/constants.ts
defaultSpeedModifiers: {
    [DefinitionType.Gun]: 0.88,      // −12%
    [DefinitionType.Melee]: 1,       // No penalty
    [DefinitionType.Throwable]: 0.92 // −8%
}
```

Applied as: `speed *= activeItemDefinition.speedMultiplier`

**Weapons with custom modifiers** override default:
- Heavy sniper → 0.75 (−25%)
- Fists (default melee) → 1.0 (−0%)
- Different guns have different penalties in definition

### Recoil Slowdown

After firing weapon, recoil temporarily slows movement:

```typescript
// server/src/objects/player.ts lines 1204–1208
const recoilMultiplier = owner.recoil.active 
    ? 1 - (owner.recoil.multiplier * ratio)
    : 1;
// recoil.multiplier comes from gun definition
// ratio = (recoilCooldown - elapsed) / recoilCooldown ∈ [0, 1]
```

- Recoil starts **immediately after firing**
- Decays over gun's `recoilCooldown` duration
- At peak (ratio=1): `speed *= (1 - recoilMultiplier)` (up to −30% for heavy guns)
- Subsides to `speed *= 1.0` as cooldown expires

### Action Modifiers (Healing, Reviving)

```typescript
// server/src/objects/player.ts line 1216
* (this.action?.speedMultiplier ?? 1)
```

While performing action (healing, reviving teammate, etc.):
- Healing: `speedMultiplier = 0.5` (−50%, can still move slowly while healing)
- Reviving: `speedMultiplier = 0.5`
- Interacting with object: `speedMultiplier` varies

No movement input during action **stops** animation; forced to restart.

### Perk Modifiers (Examples)

**Positive perks:**
- AdvancedAthletics: `speedMod = 1.05` (+5%)
- Runner: `speedMod = 1.5` (+50%)
- Adrenaline Junkie: `speedMod = 1.3` (+30% adrenaline sprint bonus)

**Negative perks:**
- Hypothermia: `speedMod = 0.75` (−25%)

Applied multiplicatively: `speed *= perkSpeedMod`

**Stacking:**
- All modifiers multiply together:
  ```
  speed = base × terrain × weapon × recoil × action × adrenaline × perks
  ```
- E.g.: `0.06 × 1.0 × 0.88 × (1 - 0.2) × 0.5 × 0.98 × 1.3` = final speed

### Status Effects

```typescript
// server/src/objects/player.ts line 1219
* this.effectSpeedMultiplier
```

- **Frostbite** (negative status): −20% speed
- **Vaccinator slowdown**: modulates movement speed via effectSpeedMultiplier
- Per-effect modifiers applied here

## Velocity Capping

### No Hard Cap

**There is NO hard velocity limiter (`if (speed > MAX) speed = MAX`).**

Velocity is naturally bounded by:

1. **Speed multiplier cap** (~0.06 base × modifiers):
   - Highest realistic value: `0.06 × 1.0 (terrain) × 1.0 (weapon) × 1.0 (recoil) × 1.0 (action) × 1.0 (adrenaline) × 1.5 (perk) = 0.09`
   - Fastest player speed ≈ 0.09 units/ms = 90 units/s

2. **Terrain friction** (ice):
   - Acceleration model prevents unbounded velocity growth
   - Friction always dampens: `v *= 1/(1+f×dt)`

3. **Knockback** (not for players):
   - Would be additive; could theoretically exceed bounds if desired

### Burst vs Sustained

- **Burst speed:** Instant from input (normal terrain)
- **Sustained speed:** Same as burst; no sprint mode exists
- Adrenaline provides smooth ramp (logarithmic), not discrete "sprint"

## Network Synchronization

### UpdatePacket PlayerData

```typescript
// common/src/packets/updatePacket.ts
interface PlayerData {
    position: { x: number, y: number };
    movementVector: { x: number, y: number };  // Velocity sync
    rotation: number;
    inventory: InventoryData;
    // ... other fields
}
```

**Serialization:**
- Position: float32 × 2
- MovementVector: float32 × 2
- Sent at **40 TPS** (every 25ms)

### Client-Side Extrapolation

Clients don't wait for next packet to update player position:

```typescript
// Hypothetical client code
playerPos = lastPacketPos + player.movementVector * timeSinceLastPacket;
```

Reduces perceived latency from ~50–100ms to <20ms.

### Velocity State Sync Frequency

- **Server tick rate:** 40 TPS (25ms per tick)
- **Velocity updates:** Every tick
- **Network send rate:** 40 Hz (full UpdatePacket to all nearby players)

If network latency = 100ms, client is 4 ticks behind server. Extrapolation bridges this gap.

### Packet Loss Handling

If UpdatePacket is dropped:
- Client **continues extrapolation** with last known velocity
- Next packet corrects position if prediction was off
- Network smoothing (Lerp) blends corrections over 1–2 frames

## Collision with Moving Objects

When two dynamically-moved objects collide, collision resolution affects only **position**, not velocity:

```typescript
// server/src/objects/loot.ts lines 166–170 (loot-loot collision)
const relativeVelocity = Vec(
    this.velocity.x - object.velocity.x,
    this.velocity.y - object.velocity.y
);
const speed = (relativeVelocity.x * normal.x + relativeVelocity.y * normal.y) * 0.5;
this.push(direction, -speed);  // Apply elastic bounce
```

**Players do NOT collide elastically with loot.** Player collision is strict geometric separation (velocity not factored).

## Complex Functions & Edge Cases

### Movement Vector Getter (Line 612)

```typescript
get movementVector(): Vector { return Vec.clone(this._movementVector); }
```

**Why clone?** Prevents accidental external mutation of internal state. Callers get a **copy**, not a reference.

**Cost:** Vector allocation per read. Justified by encapsulation benefit.

### Desired Velocity Formula

```typescript
const desiredVelocity = Vec.scale(movement, speed);
```

- `movement` ∈ [0, 1] (unit vector or cardinal)
- `speed` ∈ [0, 0.09] (typically)
- Result: velocity vector scaled by input direction

On normal terrain, this is **immediately applied** each tick (no smoothing).

### Stair Interaction & Layer Changes

```typescript
if (isObstacle && potential.definition.isStair) {
    potential.handleStairInteraction(this);
    this.activeStair = potential;
}
```

- `handleStairInteraction()` changes `this.layer` if player is moving upward
- Velocity carries through layer change (no friction reset)
- Used for multi-story buildings and elevated terrain

### DtSeconds Precision

```typescript
const dtSeconds = dt / 1000;
```

- `dt` is milliseconds (25ms typical)
- Converted to seconds for physics formula readability
- All constants (ice.acceleration, friction) are per-second values
- Important for frame-rate independent physics

## Known Gotchas & Technical Debt

### 1. **Knockback Not Implemented But Designed**

Explosions deal damage to players but don't push them. The `push()` method exists for loot/projectiles but not called for players.

**Impact:** Players are immune to knockback from explosions, grenades, etc.
**Fix:** Add `this.velocity += knockback vector` in explosion damage code, handle infinite velocity accumulation.

### 2. **Velocity Accumulation on Ice**

If player is pushed onto ice (future knockback feature) while already moving, velocity can accumulate without damping:

```typescript
// Bad scenario (hypothetical):
player.velocity = Vec(0.05, 0);  // Moving east at speed
explosion.push(west_direction, 0.03);  // Knockback west
// Result: velocity = Vec(0.02, 0)  NO, actually Vec.add applied
// Next frame: friction applies, velocity decays
```

Friction eventually caps accumulation, but brief spikes are possible.

**Fix:** Cap velocity magnitude or apply knockback resistance perk properly.

### 3. **Instant Acceleration on Normal Terrain**

Players snap to desired velocity instantly. No "easing in" from stop to movement. Makes turning feel snappy but unrealistic.

**Alternative:** Apply ice-like gradual acceleration everywhere (slower gameplay).
**Current design philosophy:** Arcade feel preferred.

### 4. **Stair Layer Transition Unvalidated**

No check that player is actually moving **toward** the stair (not falling/colliding). Players can get stuck on stairs if geometry is malformed.

**Fix:** Add directional check in `handleStairInteraction()`.

### 5. **No Movement Prediction for Obstacles**

Collision detection uses current position only, not predicted next position. Fast-moving players (or future projectiles) can **clip through thin obstacles** if velocity is high enough.

**Fix:** Swept collision shape (capsule/rect from last to current pos).

### 6. **Friction Asymmetry**

- Normal terrain: instant acceleration
- Ice: gradual acceleration + friction
- No unified model; code has multiple branches

Makes it hard to add new terrain types or tune globally.

**Future:** Abstract acceleration/friction into `TerrainPhysics` configuration object.

### 7. **Recoil Slowdown Not Networked**

Recoil multiplier is calculated server-side and affects player speed, but `recoil` state is **not sync'd to clients.**

Clients see correct position (position is sync'd), but local extrapolation might differ if client doesn't know recoil status.

**Impact:** Negligible (position sync overrides), but edge case exposed.

### 8. **Speed Modifier Stacking (Multiplicative Boom)**

Stacking 5+ speed modifiers can produce extreme values:
- `0.06 × 1.5 (perk) × 1.3 (adrenaline) × 1.05 (athletics) × ...` = explosion

No saturation or soft cap. By design (high-skill reward), but untested at extremes.

### 9. **Water Drag on Projectiles vs Players**

Projectiles have extreme drag on water (5×). Players have normal speed penalty (0.7×). No interaction between player-projectile projectiles underwater.

Inconsistent physical model, but intentional for balance.

## Related Documents

### Tier 1
- [System Architecture](../../../architecture.md#physics-tick) — Physics tick ordering
- [Data Model](../../../datamodel.md#vectors) — Vector type definition
- [Development Guide](../../../development.md#testing) — Testing physics changes

### Tier 2
- [Knockback & Velocity](../README.md) — Parent subsystem overview
- [Game Loop](../../game-loop/README.md) — Tick scheduling (40 TPS, 25ms)
- [Game Objects (Server)](../../game-objects-server/README.md) — BaseGameObject position/layer
- [Collision & Hitbox](../../collision-hitbox/README.md) — `hitbox.resolveCollision()` internals
- [Projectiles & Ballistics](../../projectiles-ballistics/README.md) — Gravity, drag, knockback for projectiles
- [Perks & Passive](../../perks-passive/README.md) — Speed-modifying perk definitions

### Tier 3 (Modules in this subsystem)
- [Acceleration Curves & Ice Physics](./ice-physics.md) — Deep dive on ice terrain acceleration formulas
- [Perk Speed Interactions](./perk-interactions.md) — Cross-perk speed stacking edge cases

### Code References
- `@file server/src/objects/player.ts` (lines 611–1330) — Movement update loop, velocity state
- `@file common/src/constants.ts` (player, ice) — Physics configuration
- `@file common/src/utils/math.ts` — Vector utilities
- `@file server/src/utils/terrain.ts` — Floor type speed multipliers
- `@file server/src/game.ts` — Game tick integration
