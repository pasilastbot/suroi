# Throwable/Grenade System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/throwable-system/modules/ -->
<!-- @source: server/src/inventory/throwableItem.ts -->

## Purpose

Server-side throwable weapons system (grenades, smoke bombs, etc.). Handles cooking timers, throw mechanics, projectile physics simulation, fuse-based detonation, and explosion effects. Grenades are player-held inventory items that transition through cook → throw → flight → detonation lifecycle.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [server/src/inventory/throwableItem.ts](../../../../server/src/inventory/throwableItem.ts) | `ThrowableItem` class — throw mechanics, cooking state, fuse integration |
| [common/src/definitions/items/throwables.ts](../../../../common/src/definitions/items/throwables.ts) | `ThrowableDefinition` — config (fuseTime, maxThrowDistance, physics, animations, detonation effects) |
| [server/src/objects/projectile.ts](../../../../server/src/objects/projectile.ts) | `Projectile` class — physics simulation (velocity, gravity, drag, bounce, collision) |
| [server/src/objects/explosion.ts](../../../../server/src/objects/explosion.ts) | `Explosion` class — damage calculation, falloff, raycast validation, shrapnel |
| [common/src/definitions/explosions.ts](../../../../common/src/definitions/explosions.ts) | `ExplosionDefinition` — damage, radius, effects (particles, decals, camera shake, shrapnel) |
| [server/src/game.ts](../../../../server/src/game.ts) | `Game.addProjectile()`, `Game.addExplosion()` — lifecycle integration, grid updates |

## Architecture

### Attack Flow

```
Player presses E (throw key)
  ↓
InputManager sends InputPacket.action = throw
  ↓
Player.use() calls ThrowableItem.useItem()
  ↓
ThrowableItem.useItem() → _useItemNoDelayCheck()
  ↓
COOK PHASE (active):
  ├─ owner.animation = AnimationType.ThrowableCook
  ├─ recoil.active = true
  ├─ _cooking = true
  ├─ _cookStart = game.now
  └─ timeout scheduled for fuseTime (if cookable)
  ↓
Player releases E (or timeout fires)
  ↓
ThrowableItem.stopUse() → _throw()
  ↓
THROW PHASE:
  ├─ Decrement count
  ├─ owner.animation = AnimationType.ThrowableThrow
  ├─ Calculate throw speed from distance-to-mouse
  ├─ Combine throw velocity + player movement vector
  ├─ Game.addProjectile(ProjectileParams)
  └─ If c4: add to owner.c4s
  ↓
IN-FLIGHT PHASE:
  ├─ Projectile.update() each tick
  ├─ Apply gravity to vertical velocity
  ├─ Apply drag deceleration to horizontal velocity
  ├─ Detect collisions (bounce off obstacles, walls, players)
  ├─ Track height above ground
  └─ Check if fuseTime expired
  ↓
DETONATION:
  ├─ Projectile._detonate()
  ├─ Explosion.explode() applies damage in radius
  ├─ Spawn synced particles
  ├─ Spawn decals
  └─ Remove projectile from game
```

### State Machine

| State | Duration | Entry | Exit | Key Actions |
|-------|----------|-------|------|-------------|
| **Held (cooking)** | 0–`fuseTime` ms | Player presses E | Player releases E or timeout | _cooking=true; recoil active; fuse timer running (if cookable) |
| **In-flight** | physics-dependent | Projectile spawned | Collision detonates or fuse expires | Physics loop: gravity, drag, collision, angle decay |
| **Detonated** | instant | Fuse expires or impact w/ cookable=false | — | Spawn explosion; apply damage; spawn effects; remove projectile |

### Throwable Lifecycle Properties

```typescript
// From ThrowableItem
_cooking: boolean           // True during hold phase
_cookStart?: number         // Game timestamp when cooking began
_throwTimer?: Timeout       // Timer for max cook time detonation
count: number               // Ammo remaining

// From Projectile
_fuseTime: number           // Remaining milliseconds until detonation
_height: number             // Height above ground (Z coordinate)
_velocity: Vector           // Horizontal velocity (X, Y)
_velocityZ: number          // Vertical velocity component
_angularVelocity: number    // Spin rate (radians per second)
inAir: boolean              // True if not touching ground/obstacle
activated: boolean          // True for C4 (detonated by player action)
```

## Throwable Definitions

### ThrowableDefinition Fields

Configuration for each grenade type (e.g., `frag_grenade`, `smoke_grenade`). See [throwables.ts](../../../../common/src/definitions/items/throwables.ts) for full list.

| Field | Type | Purpose | Example (Frag Grenade) |
|-------|------|---------|------------------------|
| `idString` | string | Unique identifier | `"frag_grenade"` |
| `name` | string | Display name | `"Frag Grenade"` |
| `cookable` | boolean | Can player hold to cook and increase damage? | `true` |
| `fuseTime` | number | Milliseconds before detonation (even during cook) | `4000` |
| `cookTime` | number | Client-side throw animation duration | `150` |
| `throwTime` | number | Client-side cool-down after throw | `150` |
| `fireDelay` | number | Minimum milliseconds between throws | `250` |
| `impactDamage` | number | Damage dealt on collision with obstacles/players | `1` |
| `obstacleMultiplier` | number | Damage multiplier vs obstacles for impact | `20` |
| `hitboxRadius` | number | Collision circle radius | `1` |
| `health` | number? | Grenade durability (if damageable; C4 only) | undefined |
| `cookSpeedMultiplier` | number | Recoil speed while cooking | `0.7` |
| `c4` | boolean? | Detonates on player action (F key), not fuse | `false` |
| `summonAirdrop` | boolean? | Flare gun behavior; detonates on hit/timer, summons airdrop | `false` |
| `pinSkin` | boolean? | Support for Halloween variant spawning | `false` |
| **physics** | object | Movement and trajectory | — |
| `maxThrowDistance` | number | Max mouse distance → throw speed | `128` |
| `initialZVelocity` | number | Initial upward velocity | `4` |
| `initialAngularVelocity` | number | Initial spin rate (rad/s) | `10` |
| `initialHeight` | number | Initial height above ground | `0.5` |
| `noSpin` | boolean? | Disable spinning animation | `false` |
| `drag` | object? | Friction coefficients (air, ground, water, ice) | see Physics below |
| **detonation** | object | Explosion effects | — |
| `explosion` | string | ExplosionDefinition ID | `"frag_grenade_explosion"` |
| `particles` | string? | SyncedParticleDefinition ID (normal) | undefined |
| `spookyParticles` | string? | SyncedParticleDefinition ID (Halloween) | undefined |
| `decal` | string? | DecalDefinition ID (scorch mark) | undefined |

### Example Definitions

**Frag Grenade:**
```typescript
{
    idString: "frag_grenade",
    name: "Frag Grenade",
    cookable: true,       // Players can hold to cook
    fuseTime: 4000,       // 4 second fuse
    fireDelay: 250,       // Can throw every 250 ms
    impactDamage: 1,      // 1 damage on obstacle hit
    obstacleMultiplier: 20, // 20x damage vs obstacles
    physics: {
        maxThrowDistance: 128,
        initialZVelocity: 4,
        initialAngularVelocity: 10,
        initialHeight: 0.5
    },
    detonation: {
        explosion: "frag_grenade_explosion"  // Damage: 120, Radius: 10–25
    }
}
```

**C4 (Deployable Bomb):**
```typescript
{
    idString: "c4",
    name: "C4",
    cookable: false,      // No cooking
    fuseTime: 3600000,    // 1 hour fuse (effectively never)
    c4: true,             // Activates on F key, not fuse
    health: 100,          // Can be destroyed
    physics: {
        maxThrowDistance: 80,
        initialZVelocity: 3,
        initialAngularVelocity: 8,
        initialHeight: 0.3
    }
}
```

## Projectile Physics

Physics simulation runs each server tick (40 TPS = 25 ms per frame). Projectiles are affected by gravity, drag, and collisions.

### Gravity and Height

```typescript
// Vertical motion
if (height > 0) {
    velocityZ -= GameConstants.projectiles.gravity * dt  // gravity = 10 m/s²
}
height += velocityZ * dt
height = clamp(height, 0, maxHeight)  // maxHeight = 5
```

**dt** = frame delta in seconds (0.025 for 40 TPS)

### Drag / Friction

Horizontal velocity is decayed based on surface:

```typescript
const drag = definition.physics.drag ?? GameConstants.projectiles.drag
// GameConstants.projectiles.drag defaults:
drag.air    = 0.7    // Air resistance
drag.ground = 3.0    // Ground/rolling friction
drag.water  = 5.0    // Water resistance
drag.ice    = 1.0    // Slippery ice

// Velocity decay each frame:
velocity *= 1 / (1 + dt * speedDrag)
```

**Example:** Grenade on ground with ground drag = 3.0:
- Frame 1: velocity *= 1 / (1 + 0.025 × 3) = 0.930 (7% loss per frame)
- Frame 2: velocity *= 0.930 × 0.930 = 0.865
- By frame 10: velocity ≈ 0.48 (52% of original)

### Bounce Mechanics

On collision with obstacle or player:

```typescript
private _reflect(normal: Vector): void {
    const length = Vec.len(velocity)          // Preserve speed
    const direction = velocity / length        // Normalize to unit vector
    const dot = dot_product(direction, normal)
    const newDir = normal * (dot * -2) + direction
    velocity = newDir * length * 0.3           // Elasticity = 0.3
}
```

**Key insights:**
- Bounce elasticity is **fixed at 0.3** (30% of pre-bounce speed retained)
- Not configurable per grenade type
- Bounces indefinitely until velocity becomes negligible or fuse detonates
- Angular velocity also decays (spinner *= 1 / (1 + dt * 1.2) per frame)

### Collision Detection

```
Each tick:
├─ Raycast from lastPosition to currentPosition
├─ Find all obstacles, buildings, players in path
├─ Sort by distance (closest first)
└─ For each collision:
    ├─ If obstacle/building with height > projectile.height: skip (fly over)
    ├─ If stair: handle special stair interaction
    ├─ If player or passable: collide, reflect, apply impact damage
    └─ Continue to next collision (or break if solid)
```

## Explosion Mechanics

### Explosion.explode() Algorithm

After fuse expires or C4 activates, the projectile detonates:

```typescript
Explosion.explode():
├─ Query grid for all objects within radius.max * 2
├─ For each angle (θ) from –π to π in steps of acos(...):
│   ├─ Fire ray from center to radius.max
│   ├─ Collect all objects intersecting ray
│   ├─ Sort by distance (closest = first blocked)
│   └─ Damage each object:
│       ├─ If object is blocked by obstacle: skip
│       ├─ If obstacle/wall: full damage block
│       └─ If unblocked:
│           ├─ distance = distance(center, hitpoint)
│           └─ damage = configuration.damage * falloff * perks
├─ Spawn shrapnel bullets (random directions)
├─ Create impact decal
└─ Remove projectile from game
```

### Damage Falloff Formula

```typescript
// ExplosionDefinition has radius.min and radius.max
falloff = (distance > min) 
    ? (max - distance) / (max - min)
    : 1.0

// Frag Grenade: min=10, max=25
if (distance <= 10):  falloff = 1.0   (full damage)
if (distance = 17.5): falloff = 0.5   (half damage)
if (distance >= 25):  falloff = 0.0   (no damage)
```

For a **Frag Grenade** (damage = 120, radius min=10, max=25):
- At 10 units: **120 damage** (full radius)
- At 17.5 units: **60 damage** (50% reduction)
- At 25 units: **0 damage** (outside radius)

### Damage Modifiers

```typescript
// Applied in order:
actualDamage = baseDamage
    * (isObstacle ? definition.obstacleMultiplier : 1)
    * (isPlayer ? player.perk(LowProfile).explosionMod : 1)
    * falloff
```

**Example:** Frag Grenade explodes near obstacle (obstacleMultiplier = 1.15) at distance 17.5:
- Base: 120 × 1.15 × 0.5 = **69 damage**

### Knockback

Loot and projectiles within radius are pushed away:

```typescript
Explosion.explode():
  if (isLoot || isProjectile):
      direction = angle_from_center_to_object
      speed = (max - distance) * multiplier
      object.push(direction, speed)
```

Knockback stacks: overlapping explosions apply additive velocity.

### Shrapnel

After damage pass, explosion spawns bullets:

```typescript
for (i = 0; i < definition.shrapnelCount; i++):
    game.addBullet(
        source: this,
        position: explosion.position,
        rotation: random_angle(),  // Random spread
        layer: explosion.layer
    )
```

**Frag Grenade:** `shrapnelCount = 10` (10 bullets fired in random directions)

## Cooking Mechanic

### Hold-to-Cook Behavior

The **cooking phase** begins when player presses the throw key (E):

```typescript
ThrowableItem._cook():
├─ _cooking = true
├─ _cookStart = game.now
├─ owner.animation = AnimationType.ThrowableCook
├─ recoil.active = true (visual/audio recoil effect)
└─ If definition.cookable:
    └─ Set timeout to fire / auto-detonate after fuseTime
```

**Fuse counting down during cook:**
```typescript
// In Projectile.update():
if (!definition.c4 || this.activated):
    fuseTime -= game.dt  // Fuse burns even during cook phase
```

**Key constraint:** If player holds grenade longer than `fuseTime`, the projectile is created with `fuseTime = 0`, detonating immediately after spawn.

### Cooking Time Calculation

When player releases E, the throw happens with adjusted fuse:

```typescript
ThrowableItem._throw():
├─ cookDuration = game.now - cookStart
├─ fuseTime_remaining = definition.fuseTime - cookDuration
└─ Projectile created with:
    └─ fuseTime: Math.max(0, fuseTime_remaining)
```

**Effect on damage:**
- Modern bomb physics: longer hold time = more fuse burned = shorter in-flight time = less damage buildup opportunity
- With Suroi's mechanic: damage is determined by **throw speed** (distance-to-mouse), not cook time directly
- **However:** player.mapPerkOrDefault(DemoExpert) provides range modifier bonus

### Throw Speed Calculation

```typescript
const speed = min(
    definition.physics.maxThrowDistance 
        * owner.mapPerkOrDefault(DemoExpert, ({ rangeMod }) => rangeMod, 1),
    owner.distanceToMouse
) * GameConstants.projectiles.distanceToMouseMultiplier

// GameConstants.projectiles.distanceToMouseMultiplier = 1.5
// So effective throw velocity = min(maxDistance * perk, mouseDistance) * 1.5
```

**Frag Grenade (maxThrowDistance = 128):**
- Mouse at 50 units: throw speed = 50 × 1.5 = 75 units/s
- Mouse at 200 units: throw speed = min(128, 200) × 1.5 = 192 units/s (capped)

## Dependencies

### Depends on:
- **[Inventory System](../inventory/)** (`InventoryItemBase`, `CountableInventoryItem`) — item holding, count management
- **[Game Loop](../game-loop/)** (`Game.update()`, `Game.addTimeout()`) — physics tick integration, timer system
- **[Game Objects (Server)](../game-objects-server/)** (`GameObject`, `Player`, `Obstacle`, `Building`) — collision, damage application
- **[Spatial Grid](../spatial-grid/)** (`Grid.intersectsHitbox()`, `Grid.updateObject()`) — broad-phase collision, object lookup
- **[Object Definitions](../object-definitions/)** — definition registry and reification
- **[Serialization System](../serialization-system/)** (`SuroiByteStream`) — packet encoding for updates

### Depended on by:
- **[Inventory System](../inventory/)** — item use/deactivation, ammo consumption
- **[Game Loop](../game-loop/)** — projectile update scheduling, cleanup
- **Game Objects (Server)** — damage handlers (`Player.damage()`, `Obstacle.damage()`)
- **[Team System](../team-system/)** — thrower team ID tracking (no friendly fire)
- **Particle System** — explosion effects (synced particles, decals)

## Known Issues & Gotchas

### 1. **Fixed Bounce Elasticity (0.3)**

Bounces are hardcoded to 30% velocity retention. Cannot configure per grenade type. Grenades bounce indefinitely unless fuse detonates.

**Implication:** Very low-velocity grenades can oscillate near obstacles for extended periods.

### 2. **Fuse Counting During Cook**

Fuse timer begins at projectile spawn, **not** throw release. Cooking phase counts down the fuse.

**Gotcha:** Hold grenade for 4 seconds (frag_grenade.fuseTime), then throw → projectile spawns with fuseTime = 0 → immediate detonation.

**Solution:** Player must throw before `definition.fuseTime` to avoid in-hand detonation.

### 3. **Max Cook Time Safety Mechanism**

If player holds throw key longer than `fuseTime`, the grenade detonates at feet:

```typescript
ThrowableItem._cook():
    this._throwTimer = game.addTimeout(
        () => this._throw(soft = true),  // Soft throw = 0 velocity
        this.definition.fuseTime
    )
```

`soft = true` sets throw velocity to 0, detonating in place.

### 4. **Collision Stair Handling**

Stairs are obstacles with `isStair = true`. Projectiles can pass through them without bouncing:

```typescript
if (object.isObstacle && object.definition.isStair):
    object.handleStairInteraction(this)
    continue  // Skip collision handling
```

**Implication:** Grenades can roll down staircases without bouncing.

### 5. **Height-Based Collision Bypass**

Projectiles flying above obstacles don't collide:

```typescript
if (projectile.height >= obstacle.height):
    obstaclesBelow.add(obstacle)
    continue  // No collision
```

**Gotcha:** If obstacle height = 0.5, projectile must have `height < 0.5` to collide.

### 6. **Explosion Ray Casting**

Explosions use angular raycast from center, not full-radius collision:

```
angle_step = acos(1 - (rayDistance / maxRadius)² / 2)
// rayDistance = 2 (GameConstants.explosionRayDistance)
// maxRadius varies per explosion
```

Tighter raycasts near center mean **more rays**. For large radius explosions, may miss thin obstacles.

### 7. **Knockback Stacking**

Multiple overlapping explosions apply additive velocity. No velocity cap.

**Implication:** Stacked grenades can ragdoll players to extreme speeds.

### 8. **C4 Special Behavior**

C4 detonates on player input (F key), not fuse:

```typescript
if (definition.c4 && !this.activated):
    this.activated = true
    // Fuse countdown ignored, waits for player 'activate' action
```

Must be placed (`inAir = false`) to activate. Activates on team basis (no friendly fire).

### 9. **Drag Coefficient Multiplication**

Drag is applied as `velocity *= 1 / (1 + dt * speedDrag)`. On high latency (large dt), drag effect exponentially greater:

**Example:** speedDrag = 3.0
- dt = 0.025 (40 TPS): velocity *= 0.930
- dt = 0.100 (10 TPS): velocity *= 0.769 (7.7% loss per frame = larger damage)

### 10. **Projectile Angular Velocity Variance**

Random ±10% variance added to initial angular velocity prevents perfectly aligned bounces:

```typescript
const vel = definition.physics.initialAngularVelocity
this._angularVelocity = vel + randomFloat(vel / -10, vel / 10)
```

**Effect:** Same grenade definition throws differently (visually) each time.

## Related Documents

### Tier 2
- **[Inventory System](../inventory/)** — weapon item lifecycle, holding, swapping
- **[Game Loop](../game-loop/)** — physics tick integration, frame time
- **[Game Objects (Server)](../game-objects-server/)** — damage handlers, collision response
- **[Spatial Grid](../spatial-grid/)** — broad-phase lookup, updates
- **[Object Definitions](../object-definitions/)** — definition registry pattern
- **[Serialization System](../serialization-system/)** — binary encoding for network sync

### Tier 1
- **[Architecture Overview](../../architecture.md)** — system-wide object model
- **[Data Model](../../datamodel.md)** — core entities and relationships
- **[API Reference](../../api-reference.md)** — packet protocol, binary format
