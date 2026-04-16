# Knockback & Velocity Physics

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/objects/player.ts -->

## Purpose

Movement velocity and knockback physics system. Handles player movement acceleration/deceleration, terrain-specific friction and drag effects, and impulse knockback from explosions and projectiles. Provides physics state (_movementVector) and update mechanics integrated into the server game loop.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/objects/player.ts` | Player velocity state (`_movementVector`), movement acceleration, friction/drag, speed modifiers |
| `server/src/objects/loot.ts` | Loot velocity state, impulse push method (`push` for knockback) |
| `server/src/objects/projectile.ts` | Projectile velocity, drag mechanics, bounce reflection |
| `server/src/objects/explosion.ts` | Knockback application to loot/projectiles on detonation |
| `common/src/constants.ts` | Physics constants (baseSpeed, ice friction, drag coefficients) |
| `common/src/utils/math.ts` | Vector operations (normalize, scale, fromPolar) |
| `server/src/game.ts` | Physics tick integration |

## Player Velocity Management

### Velocity State

```typescript
// server/src/objects/player.ts line 611
private _movementVector = Vec(0, 0);  // Current velocity vector
get movementVector(): Vector { return Vec.clone(this._movementVector); }
```

Player velocity is a 2D vector representing current movement direction and speed in game units per tick.

### Acceleration Model

On **normal terrain** (grass, sand), velocity matches desired input immediately:

```typescript
// server/src/objects/player.ts line 1273
this._movementVector = desiredVelocity;
```

Where `desiredVelocity = Vec.scale(movement, speed)` and `speed` includes:
- Base speed: `gameConstants.player.baseSpeed = 0.06`
- Floor type multiplier (terrain speedModifier)
- Recoil multiplier (gun recoil slowdown)
- Perk modifiers (speed bonuses/penalties)
- Action speed multiplier (reviving, healing, etc.)

### Ice Terrain Acceleration & Friction

On **slippery ice**, acceleration is gradual and friction applies:

```typescript
// server/src/objects/player.ts lines 1264-1271
const accelFactor = Numeric.clamp(GameConstants.player.ice.acceleration * dtSeconds, 0, 1);
if (hasMovementInput) {
    this._movementVector = Vec.add(
        this._movementVector,
        Vec.scale(Vec.sub(desiredVelocity, this._movementVector), accelFactor)
    );
}
const friction = hasMovementInput 
    ? GameConstants.player.ice.movingFriction 
    : GameConstants.player.ice.friction;
this._movementVector = Vec.scale(this._movementVector, 1 / (1 + dtSeconds * friction));
```

**Ice constants:**
- `acceleration: 2.5` — acceleration rate on ice (game units/s²)
- `friction: 0.2` — friction when not moving (decay per second)
- `movingFriction: 0.45` — friction while moving (controls momentum decay)

Friction formula: `velocity *= 1 / (1 + dt * friction)`
- dt = time delta in seconds
- As friction increases, velocity decays faster
- Blending of acceleration and deceleration creates realistic slide behavior

### Position Update

```typescript
// server/src/objects/player.ts line 1275
this.position = Vec.add(
    this.position,
    Vec.scale(this.movementVector, dt)
);
```

Position updates each tick by velocity × delta time. Collision resolution occurs after position update in a loop trying up to 10 iterations.

## Knockback System

### Knockback via Explosions

Explosions apply impulse knockback to nearby loot and projectiles (player knockback is **not currently implemented**):

```typescript
// server/src/objects/explosion.ts lines 99-100
object.push(
    Angle.betweenPoints(object.position, this.position),
    (max - dist) * multiplier
);
```

Where:
- `angle`: Direction away from explosion center toward object
- `multiplier`: Distance-based falloff (0.01 for loot, 0.002 for projectiles)
- `max`: Explosion max radius; `dist`: Distance from explosion center

**Falloff formula**: `knockback_magnitude = (max - dist) * multiplier`

At maximum distance, knockback ≈ 0. At explosion center, knockback is maximum (capped by multiplier).

### Push Method

Loot and projectiles have `push()` method that applies a velocity impulse:

```typescript
// server/src/objects/loot.ts line 187-188
push(angle: number, velocity: number): void {
    this.velocity = Vec.add(this.velocity, Vec.fromPolar(angle, velocity));
}
```

Impulse is added directly to current velocity (no deceleration). If object is already moving, knockback combines with existing velocity (momentum is not conserved; knockback is purely additive).

### Projectile Knockback

For projectiles, push multiplier in explosions: `multiplier = 0.002`

```typescript
// server/src/objects/projectile.ts line 355
push(angle: number, speed: number): void {
    this._velocity = Vec.add(this._velocity, Vec.fromPolar(angle, speed * 1000));
    if (!this.definition.physics.noSpin) this._angularVelocity = 10;
}
```

Knockback is scaled by 1000 here (differs from loot's push).

##  Loot Drag & Friction

Loot items lose velocity over time via exponential drag:

```typescript
// server/src/objects/loot.ts line 134
this.velocity = Vec.scale(this.velocity, 1 / (1 + dt * drag));
```

**Drag coefficients** (`common/src/constants.ts`):
```typescript
loot: {
    drag: 0.003,        // Normal terrain
    iceDrag: 0.0008     // Ice terrain (less drag)
}
```

Drag formula: `velocity *= 1 / (1 + dt * drag)`
- Higher drag = faster deceleration
- Same exponential model as player ice friction
- Loot on ice decays slower (0.0008 vs 0.003)

## Projectile Physics

### Velocity State

```typescript
// server/src/objects/projectile.ts lines 57-60
private _velocity: Vector;      // Horizontal velocity
private _velocityZ: number;     // Vertical (Z-axis) velocity
private _angularVelocity: number; // Rotation speed
```

Projectiles track separate XY (horizontal) and Z (vertical/height) velocity for ballistics.

### Gravity & Air Drag

```typescript
// server/src/objects/projectile.ts lines 242-244, 262, 265
this._velocityZ -= GameConstants.projectiles.gravity * dt;  // gravity: 10
this._height += this._velocityZ * dt;
this._velocity = Vec.scale(this._velocity, 1 / (1 + dt * speedDrag));
this.position = Vec.add(this.position, Vec.scale(this._velocity, dt));
```

**Gravity constant**: `10` (game units/s²)
**Drag by surface** (`common/src/constants.ts`):
```typescript
projectiles: {
    drag: {
        air: 0.7,
        ground: 3,
        ice: 1,
        water: 5
    }
}
```

Drag is selected based on terrain the projectile is over. Water provides maximum drag (slowest projectiles).

### Angular Velocity (Spin)

```typescript
// server/src/objects/projectile.ts line 273
this._angularVelocity *= (1 / (1 + dt * 1.2));
```

Rotation decays faster than linear velocity (coefficient 1.2 vs 0.7 for air drag). Prevents grenades from spinning indefinitely.

## Terrain Effects

Different floors provide speed and drag modifiers:

| Terrain | Speed Multiplier | Friction | Drag | Notes |
|---------|------------------|----------|------|-------|
| Grass | 1.0 | — | 0.003 | Baseline |
| Sand | 0.9* | — | 0.003 | *estimates from FloorTypes |
| Water | 0.7* | — | 5 | High drag slows projectiles significantly |
| Ice | 1.0 | 0.2 (0.45 moving) | 0.0008 | Slippery: low friction, custom acceleration |

*Speed multipliers per `FloorTypes[floor].speedMultiplier` in terrain definitions; exact values depend on map config.

## Speed Modifiers

### Weapons & Equipment

Holding weapons reduces movement speed:

```typescript
// common/src/constants.ts
defaultSpeedModifiers: {
    [DefinitionType.Gun]: 0.88,      // Guns 12% slower
    [DefinitionType.Melee]: 1,        // No penalty for melee
    [DefinitionType.Throwable]: 0.92  // Throwables 8% slower
}
```

Applied as `effectiveSpeed *= speedModifier` when weapon is active.

### Recoil Slowdown

After firing, recoil applies a temporary movement penalty:

```typescript
// server/src/objects/player.ts line 1206
recoilMultiplier = owner.recoil.active 
    ? 1 - (owner.recoil.multiplier * ratio)
    : 1;
```

Where `ratio` transitions from 1 → 0 as recoil cooldown expires. Multiplier comes from gun definition's `recoilMultiplier`.

### Perk Modifiers

Perks apply global speed multipliers (stored in player's modifier cache). Examples:
- Advanced Athletics: +5% speed
- Berserk: Speed penalty while active
- Frostbite (negative perk): -20% speed

Modifiers are calculated and applied each tick:

```typescript
// server/src/objects/player.ts line 1205
const speed = this.baseSpeed
    * (FloorTypes[this.floor].speedMultiplier ?? 1)
    * recoilMultiplier
    * perkSpeedMod
    * (this.action?.speedMultiplier ?? 1);
```

## Physics Tick Integration

Per-tick update sequence in `server/src/game.ts` and player update:

```
1. [Game Tick] Update all loots (velocity decay, collision)
2. [Game Tick] Update all projectiles (gravity, drag, bounce)
3. [Player Update] Calculate desired velocity from input
4. [Player Update] Apply acceleration (instant or ice-based)
5. [Player Update] Apply friction/decay
6. [Player Update] Update position: pos += velocity × dt
7. [Player Update] Collision detection & resolution (10 iterations max)
8. [Player Update] Clamp position to world boundaries
9. [Network] Serialize movement vector to UpdatePacket
```

Explosions occur during bullet update (`bullet.update()` may trigger explosion).
Knockback is applied immediately when explosion.explode() runs, before next physics update.

## Known Issues & Gotchas

1. **Player knockback not implemented** — Explosions call `damage()` on players but do not apply velocity impulse. Players are immune to knockback effects from weapons/explosions. Only loot and projectiles are pushed.

2. **Knockback stacks additively** — Rapid explosions apply multiple impulses without damping or saturation. No limit on velocity magnitude for loot/projectiles.

3. **No velocity clamp for loot/projectiles** — Knockback can create arbitrarily high speeds. Constraint is via position updates (can clip through obstacles if velocity is extreme).

4. **Friction asymmetric** — Ice friction applies exponentially, different from player's instant acceleration on normal terrain. No unified "velocity cap" for players (but speed modifiers control max velocity indirectly).

5. **Collision happens after position update** — Physics checks 10 collision resolution iterations, but does not predict future position. Fast-moving objects may clip through thin obstacles.

6. **Terrain drag is not layer-aware** — Projectile drag is selected once per update; Z-axis movement (height) is unaffected by horizontal drag coefficient.

7. **Momentum not conserved** — Knockback impulses ignore current velocity direction. Object moving west hit by east knockback results in velocity = (west vec + east vec), not reflection/reduced impact.

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Physics tick ordering and subsystem integration
- [Data Model](../../datamodel.md) — Vector/position data types

### Tier 2
- [Game Loop](../game-loop/) — Tick timing, object update order (40 TPS, 25ms per tick)
- [Game Objects (Server)](../game-objects-server/) — BaseGameObject parent class, position/rotation state
- [Collision & Hitbox](../collision-hitbox/) — Collision detection after position update
- [Projectiles & Ballistics](../projectiles-ballistics/) — Projectile physics details (gravity, spin, bouncing)
- [Status Effects](../status-effects/) — Health effects that may modify speed (Frostbite, etc.)
- [Perks & Passive](../perks-passive/) — Speed-modifying perks and how they apply

### Tier 3
- Game Loop — Modules: tick.md, update-sequence.md
- Projectiles & Ballistics — Modules: ballistics.md, drag.md

### Code References
- `@file server/src/objects/player.ts` — Player movement vector and update logic
- `@file server/src/objects/loot.ts` — Loot velocity and push mechanics
- `@file server/src/objects/explosion.ts` — Knockback impulse application
- `@file common/src/constants.ts` — Physics constants (baseSpeed, ice, projectiles)
