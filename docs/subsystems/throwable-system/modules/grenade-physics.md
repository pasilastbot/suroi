# Grenade Physics & Detonation

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/throwable-system/README.md -->
<!-- @source: server/src/objects/projectile.ts, common/src/definitions/items/throwables.ts, server/src/objects/explosion.ts -->

## Purpose

This module implements the complete physics simulation and detonation mechanics for throwable projectiles (grenades, smoke, C4, etc.). It covers how grenades are thrown, how they move through space with gravity/drag, how they bounce and collide with surfaces, how they detect fuse expiration, and how they trigger detonation effects.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [server/src/objects/projectile.ts](../../../../../server/src/objects/projectile.ts) | `Projectile` class — physics loop, collision detection, fuse timer, detonation trigger | High |
| [common/src/definitions/items/throwables.ts](../../../../../common/src/definitions/items/throwables.ts) | `ThrowableDefinition` physics config (velocity, drag, bounce) — 6 grenade types | Medium |
| [server/src/objects/explosion.ts](../../../../../server/src/objects/explosion.ts) | `Explosion` class — radius-based damage, falloff curves, ray-traced visibility, shrapnel | High |
| [common/src/definitions/explosions.ts](../../../../../common/src/definitions/explosions.ts) | `ExplosionDefinition` — damage, radius, camera shake, decals, particle spawning | Medium |
| [server/src/inventory/throwableItem.ts](../../../../../server/src/inventory/throwableItem.ts) | `ThrowableItem` — throw velocity calculation, player perk integration (DemoExpert) | Medium |

## Throwable Types

Suroi includes 6 core throwable types with distinct behavioral differences:

### 1. **Frag Grenade** (`frag_grenade`)
- **Purpose:** Primary damage-dealing explosive
- **Config:**
  - `cookable: true` — Players can hold to cook fuse
  - `fuseTime: 4000` ms (4 seconds)
  - `fireDelay: 250` ms between throws
  - `impactDamage: 1` (on collision)
  - `obstacleMultiplier: 20` (20× damage vs obstacles)
  - `hitboxRadius: 1`
- **Detonation:** `frag_grenade_explosion`
  - Damage: 120 (player), 138 (obstacles)
  - Radius: min 10, max 25
  - Shrapnel: 10 bullets at 0.08 speed, 20 range
  - Camera shake: 200 ms, intensity 30

### 2. **Smoke Grenade** (`smoke_grenade`)
- **Purpose:** Tactical cover (no damage)
- **Config:**
  - `cookable: false` — Cannot hold to cook
  - `fuseTime: 2000` ms (2 seconds)
  - `fireDelay: 250` ms
  - `impactDamage: 1`
  - `obstacleMultiplier: 20`
- **Detonation:** `smoke_grenade_explosion`
  - Damage: 0 (no blast damage)
  - Radius: 0 (visual effect only)
  - Particles: `smoke_grenade_particle` + Halloween variant `plumpkin_smoke_grenade_particle`
  - Decal: `smoke_explosion_decal`

### 3. **Confetti Grenade** (`confetti_grenade`)
- **Purpose:** Halloween seasonal/cosmetic explosive
- **Config:**
  - `cookable: true`
  - `fuseTime: 4000` ms
  - `noSkin: true` — Ignores player color skins
  - `fireDelay: 250` ms
  - `impactDamage: 1`
  - `obstacleMultiplier: 20`
- **Detonation:** `confetti_grenade_explosion`
  - Similar damage/radius to frag grenade
  - Cosmetic particles

### 4. **C4** (`c4`)
- **Purpose:** Placeable remote-detonated explosive
- **Config:**
  - `c4: true` — Requires manual detonation (F key)
  - `cookable: false`
  - `fuseTime: 750` ms (self-destruct if not activated)
  - `health: 40` — C4 can be damaged/destroyed
  - `fireDelay: 250` ms
  - `initialHeight: 0.5`
  - **Custom drag:** `air: 0.7, ground: 6, water: 8` (higher friction than grenades)
- **Lifecycle:**
  - Thrown like grenade, sticks to ground
  - `activated: true` when player presses F key
  - Detonates 750 ms after activation (or manual remotely)
  - Player can have multiple active C4s (tracked in `owner.c4s` Set)
  - Survives obstacles but dies at health ≤ 0
- **Detonation:** `c4_explosion`
  - Damage: 130
  - Radius: min 10, max 25
  - Decal fade: 30 seconds (longer than frags)

### 5. **Flare** (`flare`)
- **Purpose:** Summon airdrop via proximity or impact
- **Config:**
  - `summonAirdrop: true`
  - `cookable: false`
  - `fuseTime: 30000` ms (30 seconds — very long fuse)
  - `fireDelay: 1000` ms (slower throw rate)
  - `maxSwapCount: 2` — Limit swaps to flare (Halloween mode)
  - `pinSkin: true` — Halloween variant decals
  - `flicker` animation: blinks with `proj_flare_flicker` sprite
  - `activeSound: "flare"` — Emits audio while active
- **Lifecycle:**
  - Does NOT detonate on fuse — instead spawns airdrop at landing position
  - Airdrop summoned when `activated: true` (automatic on landing)
  - Flare cannot be used inside buildings
- **Detonation:** No explosion, just spawns decal `used_flare_decal`

### 6. **S.E.E.D.** (`sm56`)
- **Purpose:** Advanced high-damage grenade (Tier A)
- **Config:**
  - `tier: Tier.A`
  - `cookable: true`
  - `fuseTime: 3000` ms
  - `fireDelay: 250` ms
  - `impactDamage: 1`
  - `maxThrowDistance: 130` (longer range than frag)
  - `obstacleMultiplier: 20`
- **Detonation:** `sm56_explosion`
  - Similar damage/radius to frag grenade
  - Shrapnel: 10 bullets

**Infected Variants:** Two experimental seed variants generate shrapnel with infection mechanic:
- `seed_infected` → `infected_seed_explosion`
- `seed_infected_m202` → `infected_seed_explosion_m202`

## Throw Mechanics

### Throw Velocity Calculation

When player releases throw key (E), `ThrowableItem._throw()` calculates throw speed:

```typescript
// From server/src/inventory/throwableItem.ts

const speed = soft
    ? 0  // Soft throw (timeout fired while holding)
    : Numeric.min(
        definition.physics.maxThrowDistance 
            * owner.mapPerkOrDefault(PerkIds.DemoExpert, ({ rangeMod }) => rangeMod, 1),
        owner.distanceToMouse
    ) * GameConstants.projectiles.distanceToMouseMultiplier;
```

**Key points:**
1. **Distance-to-mouse independent variable:** `owner.distanceToMouse` (clamped to screen bounds)
2. **Max throw distance cap:** `definition.physics.maxThrowDistance` (e.g., 128 for frags)
3. **Perk scaling:** Demo Expert perk applies `rangeMod` multiplier (if player has it)
4. **Protocol multiplier:** `GameConstants.projectiles.distanceToMouseMultiplier = 1.5` (scales final speed)
5. **Soft throw:** If player holds beyond `fuseTime`, grenade auto-detonates with speed 0 (dropped at feet)

### Throw Direction & Height

```typescript
const projectile = game.addProjectile({
    definition,
    position: Vec.add(
        owner.position,
        Vec.rotate(definition.animation.cook.rightFist, owner.rotation)
    ),
    layer: owner.layer,
    rotation: owner.rotation,
    height: 0,  // Always starts at ground level after throw
    velocity: Vec.add(
        Vec.rotate(Vec(speed, 0), owner.rotation),  // Throw direction aligned with player rotation
        owner.velocity  // Additive player movement
    ),
    fuseTime: time,
    owner,
    source: throwableItem
});
```

**Components:**
- **Spawn position:** Player position + offset to right fist (cooked position on player model)
- **Spawn rotation:** Matches player rotation (direction player is facing)
- **Initial velocity:** Player's aim direction + player's current movement (additive physics)
- **Initial height:** 0 (ground level) — gravity and `initialZVelocity` govern upward motion during flight
- **Fuse time:** How long player held grenade (elapsed time since `_cookStart`)

## Physics Simulation

### Velocity Components

Each projectile tracks three velocity vectors:

```typescript
private _velocity: Vector;   // Horizontal (X, Y) velocity — affected by drag
private _velocityZ: number;  // Vertical (Z) velocity — affected by gravity
private _height: number;     // Not velocity, but Z position (height above ground)
```

### Gravity & Height

Gravity applies only when in air (not touching ground or obstacle):

```typescript
const onFloor = this._height <= 0;
const sittingOnObstacle = /* obstacle beneath projectile */;

if (this._height > 0) {
    this._velocityZ -= GameConstants.projectiles.gravity * dt;  // gravity = 10
}
this._height += this._velocityZ * dt;
this._height = Numeric.clamp(this._height, 0, GameConstants.projectiles.maxHeight);  // max 5 units
```

**Behavior:**
- Falling projectiles accelerate downward at 10 units/s²
- Height clamped to [0, 5] range
- Landing (height ≤ 0) immediately sets height to 0, stops Z velocity
- If landing on obstacle with height H, projectile sits on obstacle (height = H) until obstacle no longer supports it

### Drag & Friction

Drag reduces both horizontal and Z velocity based on surface type:

```typescript
const drag = definition.physics.drag ?? GameConstants.projectiles.drag;

// GameConstants.projectiles.drag = { air: 0.7, ground: 3, ice: 1, water: 5 }
// C4 overrides with: { air: 0.7, ground: 6, water: 8 }

let speedDrag: number = drag.air;
if (onWater) speedDrag = drag.water;
else if (onIce) speedDrag = drag.ice ?? drag.ground;
else if (onFloor || sittingOnObstacle) speedDrag = drag.ground;

this._velocity = Vec.scale(this._velocity, 1 / (1 + dt * speedDrag));
```

**Physics:**
- **Air drag (0.7):** Minimal friction; grenades maintain momentum in air
- **Ground drag (3–6):** High friction; grenades slow quickly after landing
- **Water drag (5–8):** Extreme friction; grenades almost stop in water
- **Ice drag (1):** Lower friction; grenades slide further on ice
- **Decay formula:** `v_new = v_old / (1 + dt × drag_coefficient)`

**Example:** Frag grenade with speed 30 landing on ground:
- Frame 1: `v ≈ 30 / (1 + 0.025 × 3) = 29.28`
- Frame 10: `v ≈ 23.7` (slowing down)
- Frame 40: `v ≈ 3.5` (nearly stopped)

### Angular Velocity & Spin

Grenades spin during flight, decaying over time:

```typescript
const vel = definition.physics.initialAngularVelocity;  // e.g., 10 rad/s
this._angularVelocity = vel + randomFloat(vel / -10, vel / 10);  // ±10% variation

// Each frame:
this.rotation = Angle.normalize(this. + this._angularVelocity * dt);
this._angularVelocity *= (1 / (1 + dt * 1.2));  // Decay factor: 1.2
```

**Behavior:**
- Each grenade gets slightly different spin (±10% variation)
- Prevents all grenades of same type from landing in exact same position
- Spin decays rapidly: within ~2–3 seconds, `_angularVelocity ≈ 0`
- Some throwables have `noSpin: true` (e.g., S.E.E.D seeds) — angular velocity stays 0

## Collision with Surfaces

### Collision Detection

Each physics frame, projectile sweeps a rectangle from `_lastPosition` to current position and tests against obstacles/buildings/players:

```typescript
const objects = this.game.grid.intersectsHitbox(
    RectangleHitbox.fromLine(this._lastPosition, this.position),
    this.layer
);

// Test: Building? Obstacle? Player? Door? Stairs?
// Skip if: object.dead | no hitbox | object === owner | different layer | no collision enabled
```

### Collision Response (Bouncing)

When collision detected:

```typescript
// 1. Resolve penetration (move projectile out of obstacle)
this.hitbox.resolveCollision(object.hitbox);

// 2. Calculate bounce reflection
const length = Vec.len(this._velocity);
const direction = Vec.scale(this._velocity, 1 / length);
const dot = Vec.dotProduct(direction, normal);  // normal = collision surface normal
const newDir = Vec.add(Vec.scale(normal, dot * -2), direction);
this._velocity = Vec.scale(newDir, length * 0.3);  // Retain 30% speed (energy loss)

// 3. Apply impact damage if cookable=false (impact detonation not enabled)
if (this.definition.impactDamage) {
    object.damage({
        amount: this.definition.impactDamage 
            * (object.isObstacle ? this.definition.obstacleMultiplier : 1),
        source: this.owner,
        weaponUsed: this.source
    });
}
```

**Bounce mechanics:**
- **Reflection angle:** Bounces off surface normal (like light reflecting off mirror)
- **Energy loss:** Retains 30% of velocity each bounce (decays quickly)
- **Impact damage:** Applied only to obstacles/players, not to grenades themselves
- **DemoExpert perk:** If owner has perk, `_fuseTime = -1` (forces immediate detonation)

### Stair Handling

Special case for stair obstacles:

```typescript
if (object.isObstacle && object.definition.isStair) {
    object.handleStairInteraction(this);
    continue;  // Skip collision; allow walking through stairs
}
```

Stairs are non-blocking to grenades (passthrough mechanic).

### Obstacle Height Tracking

Grenades track which obstacles they're resting on:

```typescript
private _obstaclesBelow = new Set<Obstacle | Building>();

// If projectile height >= obstacle height, add to set
// Later: use highest obstacle height as ground level
let height = this._height;
let sittingOnObstacle = false;
for (const obstacle of this._obstaclesBelow) {
    if (!obstacle.hitbox?.collidesWith(this.hitbox) || obstacle.dead) {
        this._obstaclesBelow.delete(obstacle);
        continue;
    }
    height = Numeric.max(height, obstacle.height);
    if (this._height <= obstacle.height) {
        sittingOnObstacle = true;
    }
}
```

**Use case:** Grenades landing on tables/barrels sit at height H (obstacle height), not ground height.

## Fuse Mechanics

### Fuse Timer

Each projectile has a countdown fuse:

```typescript
private _fuseTime: number;  // Milliseconds remaining

update() {
    if (!this.definition.c4 || this.activated) {
        this._fuseTime -= this.game.dt;
    }
    if (this._fuseTime < 0) {
        this._detonate();
        return;
    }
    // ... rest of physics simulation
}
```

**Behavior:**
- Ticks down each frame by `game.dt` (milliseconds per frame at 40 TPS = 25 ms)
- **Exception for C4:** Fuse only decrements if `activated: true` (manual detonation)
- When `_fuseTime < 0`: Immediately call `_detonate()`

### Cooking & Fuse Reduction

When player holds grenade during throw, elapsed time reduces fuse:

```typescript
// In ThrowableItem._throw()
const time = this.definition.cookable 
    ? (game.now - (this._cookStart ?? 0)) 
    : 0;

const projectile = game.addProjectile({
    // ...
    fuseTime: time,  // Pass elapsed cook time
    // ...
});
```

**Example:** Frag grenade with 4000 ms fuse
- Player holds for 1000 ms (1 second)
- Throws grenade
- Remaining fuse: 4000 - 1000 = 3000 ms (3 seconds)
- Detonates 3 seconds after throw

### Time-to-Explosion Constants

| Grenade Type | Fuse Time | Cook Time | Total if Held Max | Notes |
|--------------|-----------|-----------|-------------------|-------|
| Frag Grenade | 4000 ms | 150 ms | 3850 ms | Cookable; typical cook 0.5–1 s |
| Smoke Grenade | 2000 ms | 150 ms | 1850 ms | Not cookable; fixed 2 s |
| Confetti Grenade | 4000 ms | 150 ms | 3850 ms | Cookable; cosmetic |
| C4 | 750 ms | 250 ms | Manual | Requires F key activation; 750 ms self-destruct |
| Flare | 30000 ms | 250 ms | 29750 ms | Long-lasting; triggers on impact/timeout |
| S.E.E.D. | 3000 ms | 150 ms | 2850 ms | Advanced; cookbook |

**Key gotcha:** If `fuseTime > cookTime`, fuse may expire while player still holding grenade (soft throw scenario).

## Detonation Event

### Detonation Flow

```typescript
private _detonate(): void {
    if (this.detonated) return;
    this.detonated = true;

    const {
        explosion,
        particles,
        spookyParticles,
        decal
    } = this.definition.detonation;

    // 1. Spawn explosion & apply damage
    if (explosion !== undefined) {
        game.addExplosion(
            explosion,
            this.position,  // Epicenter
            this.owner,     // Source (for kill credit)
            this.layer,
            this.source,    // ThrowableItem
            this.halloweenSkin ? PerkData[PerkIds.PlumpkinBomb].damageMod : 1,  // Damage mod
            this._obstaclesBelow  // Ignore obstacles below
        );
    }

    // 2. Spawn particles (normal or Halloween variant)
    const effectiveParticles = (this.halloweenSkin && spookyParticles) 
        ? spookyParticles 
        : particles;
    if (effectiveParticles !== undefined) {
        game.addSyncedParticles(effectiveParticles, this.position, this.layer);
    }

    // 3. Spawn decal (scorch mark)
    if (decal !== undefined) {
        let decal_ = decal;
        if (this.halloweenSkin && this.definition.pinSkin) {
            decal_ += "_halloween";
        }
        game.addDecal(decal_, this.position, this.rotation, this.layer);
    }

    // 4. Remove projectile from game
    this.game.removeProjectile(this);
}
```

**Key points:**
- **Detonation guard:** `if (this.detonated) return` prevents double-detonation (safe)
- **Halloween perk:** `PlumpkinBomb` perk increases explosion damage (`damageMod`)
- **Obstacle culling:** Obstacles directly below projectile immune to damage (passed as `objectsToIgnore`)
- **Particle variants:** Halloween mode switches to spooky particles if available
- **Decal rotation:** Scorch mark matches grenade's landing rotation

## Explosion Mechanics

### Explosion Class

The `Explosion` class handles radius-based damage with ray-tracing to prevent damage through walls:

```typescript
// From server/src/objects/explosion.ts

class Explosion {
    definition: ExplosionDefinition;
    position: Vector;
    source: GameObject;
    layer: Layer;
    damageMod: number;
    objectsToIgnore: Set<GameObject>;

    explode(): void {
        const { radius, damage, obstacleMultiplier } = this.definition;

        // 1. Find all objects in 2× max radius (broad phase)
        const objects = this.game.grid.intersectsHitbox(
            new CircleHitbox(radius.max * 2, position),
            layer
        );

        // 2. For each direction, cast ray to find first obstacle blocking damage
        const step = Math.acos(1 - ((GameConstants.explosionRayDistance / radius.max) ** 2) / 2);
        for (let angle = -Math.PI; angle < Math.PI; angle += step) {
            const lineEnd = Vec.add(position, Vec.fromPolar(angle, radius.max));
            
            // 3. Collect line-of-sight collisions, sort by distance
            const lineCollisions = [];
            for (const object of objects) {
                const intersection = object.hitbox.intersectsLine(position, lineEnd);
                if (intersection) {
                    lineCollisions.push({
                        object,
                        pos: intersection.point,
                        squareDistance: Geometry.distanceSquared(position, intersection.point)
                    });
                }
            }

            // 4. Sort by closest first (prevent damage through walls)
            lineCollisions.sort((a, b) => a.squareDistance - b.squareDistance);

            // 5. Apply damage to first valid object per ray
            for (const collision of lineCollisions) {
                const object = collision.object;
                if (!damagedObjects.has(object.id)) {
                    damagedObjects.add(object.id);
                    const distance = Math.sqrt(collision.squareDistance);

                    // Damage falloff
                    const falloff = (distance > radius.min) 
                        ? (radius.max - distance) / (radius.max - radius.min) 
                        : 1;

                    object.damage({
                        amount: damageMod * damage 
                            * (isObstacle ? obstacleMultiplier : 1) 
                            * (isPlayer ? perkModifier : 1) 
                            * falloff,
                        source,
                        weaponUsed
                    });

                    break;  // Only first object per ray takes damage
                }
            }
        }
    }
}
```

### Damage Falloff

Explosion damage decreases with distance from epicenter:

```
Falloff = (radius.max - distance) / (radius.max - radius.min)
```

**Example: Frag Grenade Explosion**
- `radius.min = 10`, `radius.max = 25`
- Base damage: 120
- At distance 10: falloff = 100%, damage = 120
- At distance 17.5: falloff = 50%, damage = 60
- At distance 25: falloff = 0%, damage = 0

**Modifiers applied:**
1. `damageMod` — Perk scaling (PlumpkinBomb = 1.5×)
2. Obstacle multiplier (e.g., `obstacleMultiplier: 1.15`)
3. Player perk (e.g., `LowProfile` reduces explosive damage)
4. Falloff curve (distance-based)

### Shrapnel Generation

Some explosions spawn shrapnel bullets (secondary projectiles):

```typescript
const { shrapnelCount, ballistics } = this.definition;

if (shrapnelCount > 0) {
    for (let i = 0; i < shrapnelCount; i++) {
        const angle = (Math.PI * 2 * i) / shrapnelCount + randomFloat(-0.1, 0.1);
        game.addBullet(
            ballistics.damage,
            position,
            angle,
            ballistics.speed,
            ballistics.range,
            source
        );
    }
}
```

**Frag Grenade example:**
- `shrapnelCount: 10` — 10 shrapnel bullets spread in circle
- `ballistics.damage: 15` — Each bullet deals 15 damage
- `ballistics.speed: 0.08` — Shrapnel velocity (80% speed)
- `ballistics.range: 20` — Max distance before bullet dissipates

Smoke grenade has `shrapnelCount: 0` (no shrapnel).

### Camera Shake & Effects

Client receives explosion data and applies visual effects:

```typescript
const { cameraShake, animation } = this.definition;

// Server sends UpdatePacket with explosion position/type
// Client receives and:
// 1. Shakes camera for `duration` ms with `intensity` magnitude
// 2. Animates sprite tint to `tint` color for `duration` ms
// 3. Animates sprite scale to `scale` multiplier during animation
```

**Frag Grenade example:**
- `cameraShake: { duration: 200, intensity: 30 }`
- `animation: { duration: 1000, tint: 0x91140b (dark red), scale: 1.5 }`

## Special Abilities

### C4 Remote Detonation

C4 differs from grenades: detonates only when player presses F key (no fuse countdown):

```typescript
// In Projectile.update()
if (!this.definition.c4 || this.activated) {
    this._fuseTime -= this.game.dt;
}

// In Projectile.activateC4()
activateC4(): boolean {
    if (!this.definition.c4) throw new Error("Not C4");
    if (this.inAir) return false;  // Can only activate on ground
    this.activated = true;
    this.setDirty();
    return true;
}
```

**Lifecycle:**
1. Throw C4 (behaves like normal grenade during flight)
2. Lands on ground (`inAir = false`)
3. Player presses F to activate (`activated = true`)
4. Fuse countdown starts (750 ms remaining)
5. Auto-detonates if 750 ms expires without manual detonation
6. Player can have multiple active C4s (tracked in `owner.c4s`: `Set<Projectile>`)

**Network sync:** C4 activation sent to all clients via UpdatePacket dirty flag.

### Flare Remote Activation

Flare (airdrop beacon) detonates via proximity/impact instead of fuse:

```typescript
// In Projectile.update()
if (!this.activated && this.definition.summonAirdrop) {
    this.activated = true;
    this.game.summonAirdrop(this.position, this.halloweenSkin);
}
```

**Trigger condition:** Projectile comes to rest (very low velocity + on ground).

**Building restriction:** Player cannot throw flare indoors:
```typescript
if (this.definition.summonAirdrop && owner.isInsideBuilding) {
    owner.sendPacket(PickupPacket.create({ 
        message: InventoryMessages.CannotUseFlare 
    }));
    return;  // Throw fails
}
```

### Grenade Pickup & Ammo

Grenades are stackable inventory items (unlike guns):

```typescript
// From ThrowableItem (extends CountableInventoryItem)
count: number;  // Ammo quantity

override itemData(): ItemData<ThrowableDefinition> {
    return {
        kills: this.stats.kills,
        damage: this.stats.damage
    };
}
```

**Pickup rules:**
- Grenades killed mid-air cannot be recovered
- Grenades detonated spawn no loot
- Max capacity per grenade type set in backpack definition (e.g., Frag Grenade: 3–6 max)
- Armory backpacks increase grenade capacity

## Throwable Pickup & Reuse

### Mid-Air Pickup

Once thrown, grenades are game objects (projectiles) with physics. **They cannot be picked up mid-air** (by design). Only loot items on ground can be picked up.

### Detonated Grenades

Grenades that detonate create no loot pickup. The player loses the ammo count on throw, not on detonation.

### Stackable Ammo System

Grenades use a count-based inventory system:

```typescript
// In ThrowableItem constructor
this.count = count;  // e.g., 3 frag grenades

// On throw
owner.inventory.removeThrowable(this.definition, false, 1);  // Decrement by 1
```

If count reaches 0, the `ThrowableItem` is removed from inventory (cannot equip).

## Animation & Effects

### Throwable Sprites

Each grenade type has multiple sprite states:

```typescript
const animation = {
    pinImage?: string;           // Before cooking starts (pin on grenade)
    liveImage: string;           // While cooking and in flight
    leverImage?: string;         // While lever held down
    activatedImage?: string;     // C4-specific: activated state
    cook: {
        cookingImage?: string;   // Replace live image while cooking
        leftFist: Vector;        // Fist position during cook
        rightFist: Vector;
    }
    throw: {
        leftFist: Vector;        // Fist position during throw animation
        rightFist: Vector;
    }
};
```

**Frag Grenade example:**
```typescript
animation: {
    pinImage: "proj_frag_pin",           // Initial pin-in sprite
    liveImage: "proj_frag",              // Flying grenade sprite
    leverImage: "proj_frag_lever",       // Lever held sprite
    cook: {
        cookingImage: "proj_frag_nopin", // Pin removed while cooking
        leftFist: Vec(2.5, 0),
        rightFist: Vec(-0.5, 2.15)
    },
    throw: {
        leftFist: Vec(1.9, -1.75),
        rightFist: Vec(4, 2.15)
    }
}
```

### Detonation Particles

Particle effects spawn on detonation (shared particles subsystem):

```typescript
const effectiveParticles = (halloweenSkin && spookyParticles) 
    ? spookyParticles 
    : particles;
if (effectiveParticles !== undefined) {
    game.addSyncedParticles(effectiveParticles, position, layer);
}
```

**Types:**
- **Smoke Grenade:** `smoke_grenade_particle` (normal) + `plumpkin_smoke_grenade_particle` (Halloween)
- **Frag Grenade:** Explosion visual only
- **Flare:** No particles (visual flicker only)

### Screen Shake

Server sends explosion definition with camera shake parameters:

```typescript
cameraShake: {
    duration: number;   // ms to shake
    intensity: number;  // Magnitude of shake
}
```

Client applies shake via:
```
offsetX = (Math.random() - 0.5) * intensity * (1 - elapsedTime / duration)
offsetY = (Math.random() - 0.5) * intensity * (1 - elapsedTime / duration)
```

## Network Sync

### Throwable Creation Packet

When projectile spawned, server sends to all clients:

```typescript
// UpdatePacket includes:
{
    type: ObjectCategory.Projectile,
    id: projectile.id,
    position: position,
    rotation: rotation,
    layer: layer,
    full: {
        definition: throwable.definition,  // Serialized as single-byte index
        halloweenSkin: boolean,            // Halloween mode flag
        activated: boolean,                // C4 activation state
        c4: {
            throwerTeamID: number,         // For team colors
            tintIndex: number
        }
    }
}
```

Definitions serialized as binary index (1 byte) into `Throwables` registry (O(1) lookup).

### Position Updates

Each frame, dirty projectiles send partial updates:

```typescript
override get data(): FullData<ObjectCategory.Projectile> {
    return {
        position: this.position,  // Current (X, Y)
        rotation: this.rotation,  // Spin angle
        layer: this.layer,        // Z layer
        height: this._height,     // Z elevation
        // full data omitted (only sent on creation)
    };
}
```

### Detonation Event

When projectile detonates, it's removed from game and all clients:

```typescript
this.game.removeProjectile(this);  // Cleanup
// UpdatePacket sent to all clients with projectile.dead = true
```

Explosion visual effects handled by client decoding `ExplosionDefinition`.

## Known Gotchas

### 1. Throw Direction Ambiguity

**Problem:** Grenade throw direction depends on player rotation at throw time, not mouse position.
```typescript
velocity: Vec.add(
    Vec.rotate(Vec(speed, 0), owner.rotation),  // Uses owner.rotation, not mouse angle
    owner.velocity
)
```

**Gotcha:** If player rotates while holding throw key, grenade throws in latest rotation direction (not the direction they aimed).

**Prevention:** Players should avoid rotating after releasing throw key; client enforces this with animation locks.

### 2. Bounce Accumulation

**Problem:** Each bounce retains 30% velocity. If grenade bounces many times, it can travel very far.
```typescript
this._velocity = Vec.scale(newDir, length * 0.3);  // 30% retention
```

**Example:** Grenade with 100 speed
- Bounce 1: 30 speed → Bounce 2: 9 speed → Bounce 3: 2.7 speed → ...
- Stops effectively after 5–6 bounces

**Gotcha:** If grenade bounces at exact 90° angle to ceiling, it can hop repeatedly in narrow spaces. **Mitigation:** High drag on ground/water reduces bounce distance quickly.

### 3. Fuse Never Escapes Soft-Throw Window

**Problem:** If `fuseTime < cookTime`, fuse may expire while player holding grenade.
```typescript
// cookTime = 150, fuseTime = 750 (hypothetical)
// After 750 ms, fuse expires while player still cooking (soft throw)
```

**Mitigation:** ThrowableDefinition constraints ensure `fuseTime > cookTime` (or `!cookable`).

### 4. Impact Damage Requires In-Air Status

**Problem:** Impact damage only applies if projectile is in air when collision detected.
```typescript
if (!this.inAir) continue;

if ((object.isObstacle || object.isPlayer) && this.definition.impactDamage) {
    // Apply damage
}
```

**Gotcha:** Grenade sliding on ground (fully landed) doesn't apply impact damage to subsequent obstacles. Only impacts during flight count.

### 5. C4 Height Check Prevents Mid-Air Activation

**Problem:** C4 can only be activated on ground.
```typescript
activateC4(): boolean {
    if (this.inAir) return false;  // Prevent mid-air activation
    // ...
}
```

**Gotcha:** If player throws C4 and tries to detonate before landing, activation fails silently (returns false). Must land first.

### 6. Obstacle Below Becomes Immune to Explosion

**Problem:** Obstacles directly below grenade (in `_obstaclesBelow` set) are passed to explosion as `objectsToIgnore`.
```typescript
game.addExplosion(
    explosion,
    position,
    owner,
    layer,
    source,
    damageMod,
    this._obstaclesBelow  // <- These ignore damage
);
```

**Gotcha:** If grenade lands on barrel and explodes, the barrel doesn't take damage (intentional to prevent double-damage).

### 7. Halloween Spooky Particles vs Normal

**Problem:** Halloween mode sometimes swaps particle types.
```typescript
const effectiveParticles = (this.halloweenSkin && spookyParticles) 
    ? spookyParticles 
    : particles;
```

**Gotcha:** If `spookyParticles` not defined, falls back to `particles` (no error, silent fallback).

### 8. Decal Rotation Freezes at Detonation

**Problem:** Decal (scorch mark) inherits grenade rotation at detonation time.
```typescript
game.addDecal(decal_, this.position, this.rotation, this.layer);
```

**Gotcha:** Fast-spinning grenade creates visually rotated decal (minor visual glitch, not gameplay impact).

## Complex Functions

### `Projectile.update()` — @file server/src/objects/projectile.ts:100
**Purpose:** Core physics loop (gravity, drag, collision, fuse countdown).
**Implicit behavior:**
- Runs every frame (40 TPS = 25 ms per frame)
- Modifies `_height`, `_velocity`, `_velocityZ`, `_angularVelocity` in place
- Marks object dirty if any property changed
- Auto-detonates if `_fuseTime < 0`

**Called by:** `Game.update()` → per-projectile loop

### `Explosion.explode()` — @file server/src/objects/explosion.ts:28
**Purpose:** Ray-trace damage calculation from epicenter.
**Implicit behavior:**
- Shoots rays in circular pattern (angle step based on `explosionRayDistance`)
- Sorts collisions by distance (prevents through-wall damage)
- Only first object per ray takes damage
- Applies falloff, multipliers, perk modifiers
- Spawns shrapnel bullets

**Called by:** `Projectile._detonate()`, barrel/stove destruction, other explosive events

### `ThrowableItem._throw()` — @file server/src/inventory/throwableItem.ts:70
**Purpose:** Calculate throw velocity, spawn projectile, decrement ammo.
**Implicit behavior:**
- Speed based on `distance-to-mouse` + player perk scaling
- Initial velocity = throw direction + player movement (additive)
- Fuse countdown = time held (elapsed cooking time)
- Decrements count; removes item if count == 0
- Creates new `Projectile` object in game state

**Called by:** User release of throw key (E) or timeout

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Throwable System overview
- **Tier 1:** [../../../../docs/architecture.md](../../../../docs/architecture.md) — System architecture
- **Tier 2:** [../../projectiles-ballistics/README.md](../../projectiles-ballistics/README.md) — Shrapnel bullet mechanics
- **Tier 2:** [../../explosion-system/README.md](../../explosion-system/README.md) — Client-side explosion rendering
- **Tier 2:** [../../inventory/README.md](../../inventory/README.md) — Inventory item lifecycle
- **Tier 2:** [../../particle-system/README.md](../../particle-system/README.md) — Synced particle effects
- **Tier 2:** [../../game-objects-server/README.md](../../game-objects-server/README.md) — Game object model
- **Patterns:** [../patterns.md](../patterns.md) — Throwable system patterns (if exists)
