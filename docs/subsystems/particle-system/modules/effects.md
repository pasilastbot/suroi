# Particle System — Special Effects & Animations

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/particle-system/README.md -->
<!-- @source: client/src/scripts/managers/particleManager.ts -->

## Purpose

Defines the core mechanics of individual particle lifecycle, animation, and rendering within the Particle System. This module covers how particles are created, updated per frame, animated (scale/alpha/rotation), sorted for rendering, and cleaned up when dead.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/particleManager.ts` | `Particle` class, `ParticleManager` singleton, `ParticleEmitter`, `ParticleProperty` type | Medium |
| `common/src/utils/math.ts` | `Numeric.lerp()` for animation interpolation, easing functions | Low |
| `common/src/utils/vector.ts` | `Vec.add()`, `Vec.scale()` for position physics | Low |
| `common/src/definitions/obstacles.ts` | `TintedParticles` registry mapping particle frame names to sprite + tint | Low |
| `common/src/constants.ts` | `Layer` enum (render layers) | Low |
| `client/src/scripts/utils/pixi.ts` | `SuroiSprite`, `toPixiCoords()` for sprite rendering | Low |
| `client/src/scripts/managers/cameraManager.ts` | `getContainer(layer)` for render hierarchy | Low |

## Business Rules

- **Particle Creation:** `ParticleManager.spawnParticle(options)` creates exactly one `Particle` instance, adds to active set, and adds its sprite to the render container.
- **Batch Spawning:** `ParticleManager.spawnParticles(count, optionsFn)` spawns N particles, calling `optionsFn()` for each to allow variance (position, rotation, etc.) without duplicating config.
- **Frame Randomization:** If `ParticleOptions.frames` is an array, one frame is randomly selected at spawn time (not per-update).
- **Tint Precedence:** Custom `tint` in options overrides baked tint from `TintedParticles` registry. If no custom tint and frame is in registry, use registry tint; otherwise default to white (0xffffff).
- **Animation Interpolation:** Scale, alpha, rotation are interpolated linearly from `start` to `end` over particle lifetime using `interpFactor` (0→1). Optional easing function transforms `interpFactor` before lerp.
- **Lifetime Marking:** Particle is marked `dead = true` when `Date.now() >= _deathTime`. On next `ParticleManager.update()`, dead particles are deleted from the set and sprite is destroyed.
- **Layer Assignment:** Particles render on a PixiJS container determined by `ParticleOptions.layer` (default: `Layer.Ground`). Z-order within layer is set via `zIndex`.
- **Emitter Activation:** Emitters spawn particles only while `active = true`. Toggling `active` pauses/resumes spawning without destroying the emitter.
- **Cleanup Responsibility:** Caller must explicitly call `emitter.destroy()` or `particle.kill()` to immediately clean up; relying on timeout is acceptable for short-lived effects.

## Data Lineage

```
ParticleManager.spawnParticle(options)
  ↓
Particle constructor:
  • Generate frame: if array, random index; else use string
  • Lookup TintedParticles[frame] → base sprite, baked tint
  • Create SuroiSprite(base sprite)
  • Apply tint: custom > baked > 0xffffff
  • Initialize animation properties: scale, alpha, rotation (start values)
  • Set spawn time, death time = spawn time + lifetime
  ↓
ParticleManager.update(delta):
  • For each particle:
    • position += speed * (delta / 1000) [linear motion]
    • interpFactor = (now - spawnTime) / lifetime [clamped 0→1]
    • If animation property is object:
      • ease function: interpFactor → eased value (0→1)
      • lerp(start, end, eased) → new property value
    • _updateImage(): sync sprite to position, scale, rotation, alpha, zIndex
    • Check if dead; if yes, delete from set, destroy sprite, call onDeath callback
  • For each emitter:
    • If active & (now - lastSpawn >= delay):
      • Call spawnOptions() → ParticleOptions
      • Spawn particle
      • Update lastSpawn = now
      ↓
PixiJS Rendering:
  • SuroiSprite positioned at toPixiCoords(position)
  • Layer container renders particles back-to-front by zIndex
```

### Animation Property Mechanics

```typescript
// ParticleProperty: Fixed value or animated
type ParticleProperty = number | {
  readonly start: number
  readonly end: number
  readonly ease?: (x: number) => number  // Optional easing 0→1
}

// During particle update:
// 1. Compute progress: interpFactor = (now - spawnTime) / lifetime, clamped [0, 1]
// 2. Apply easing: eased = ease ? ease(interpFactor) : interpFactor
// 3. Linear interpolation: value = lerp(start, end, eased)
// 4. Assign to particle property (scale, alpha, rotation)
```

## Particle Lifecycle — Creation, Updating, Cleanup

### Phase 1: Creation

Called when event triggers (e.g., gun fires, player takes damage, obstacle destroyed):

```typescript
// Single particle
ParticleManager.spawnParticle({
  frames: "blood_particle",          // Frame ID or array of IDs
  position: playerPos,                // World coordinates
  speed: Vec(vx, vy),                 // Velocity in units/second
  lifetime: 2000,                     // Milliseconds
  zIndex: ZIndexes.Players + 1,       // Sort order within layer
  layer: Layer.Ground,                // (optional) render layer
  scale: { start: 1, end: 0 },        // (optional) animation
  alpha: { start: 1, end: 0 },        // (optional) animation
  rotation: { start: 0, end: Math.PI * 2 }, // (optional)
  tint: 0xff0000,                     // (optional) red tint
  onDeath: (p) => console.log("dead") // (optional) callback
});

// Batch spawn with variation
ParticleManager.spawnParticles(10, () => ({
  frames: "shell_casing",
  position: gunPos,
  speed: randomPointInsideCircle(Vec(0, 0), 5), // Vary velocity
  lifetime: 1500,
  zIndex: ZIndexes.Bullets - 1,
  rotation: { start: randomRotation(), end: randomRotation() },
  // ... other options shared by all
}));
```

**Internals:**

```typescript
constructor(options: ParticleOptions) {
  // 1. Select frame
  const frames = options.frames;
  const frame = typeof frames === "string" 
    ? frames 
    : frames[random(0, frames.length - 1)];
  
  // 2. Lookup tint in registry
  const tintedParticle = TintedParticles[frame];
  
  // 3. Create PixiJS sprite
  this.image = new SuroiSprite(tintedParticle?.base ?? frame);
  this.image.tint = options.tint ?? tintedParticle?.tint ?? 0xffffff;
  
  // 4. Initialize animation properties
  this.scale = typeof options.scale === "number" 
    ? options.scale 
    : options.scale?.start ?? 1;
  this.alpha = typeof options.alpha === "number" 
    ? options.alpha 
    : options.alpha?.start ?? 1;
  this.rotation = typeof options.rotation === "number" 
    ? options.rotation 
    : options.rotation?.start ?? randomRotation();
  
  // 5. Set lifetime
  this._deathTime = this._spawnTime + options.lifetime;
  
  // 6. Sync rendering
  this._updateImage();
}
```

### Phase 2: Updating (Per-Frame)

Runs every game tick (40 TPS, ~25 ms/tick) via `ParticleManager.update(delta)`:

```typescript
update(delta: number): void {
  // 1. Update position (linear physics)
  this.position = Vec.add(
    this.position, 
    Vec.scale(this.options.speed, delta / 1000) // delta in ms → seconds
  );
  
  // 2. Calculate animation progress
  const now = Date.now();
  let interpFactor: number;
  if (now >= this._deathTime) {
    this._dead = true;
    interpFactor = 1; // Clamp to end of animation
  } else {
    interpFactor = (now - this._spawnTime) / this.options.lifetime;
  }
  
  // 3. Interpolate scale
  if (typeof options.scale === "object") {
    this.scale = Numeric.lerp(
      options.scale.start,
      options.scale.end,
      (options.scale.ease ?? (t => t))(interpFactor)
    );
  }
  
  // 4. Interpolate alpha
  if (typeof options.alpha === "object") {
    this.alpha = Numeric.lerp(
      options.alpha.start,
      options.alpha.end,
      (options.alpha.ease ?? (t => t))(interpFactor)
    );
  }
  
  // 5. Interpolate rotation
  if (typeof options.rotation === "object") {
    this.rotation = Numeric.lerp(
      options.rotation.start,
      options.rotation.end,
      (options.rotation.ease ?? (t => t))(interpFactor)
    );
  }
  
  // 6. Sync PixiJS sprite
  this._updateImage();
}

protected _updateImage(): void {
  this.image
    .setZIndex(this.options.zIndex)
    .setVPos(toPixiCoords(this.position))
    .setScale(this.scale)
    .setRotation(this.rotation)
    .setAlpha(this.alpha);
}
```

### Phase 3: Cleanup

Triggered when `particle.dead = true`, or manually via `particle.kill()`:

```typescript
// Automatic cleanup (at end of ParticleManager.update)
if (particle.dead) {
  this.particles.delete(particle);      // Remove from active set
  particle.image.destroy();             // Destroy PixiJS sprite
  particle.options.onDeath?.(particle); // Call callback if defined
}

// Manual cleanup (immediate)
kill(): void {
  this._dead = true;
  ParticleManager.particles.delete(this);
  this.image.destroy();
}
```

**Garbage Collection:**

- Particles are cleaned up on the frame they expire (or manually). No active gc; dead particles are not held in memory.
- `SuroiSprite.destroy()` releases PixiJS textures and removes from render tree.
- No explicit pooling; particles are created on-demand and GC'd. Acceptable for typical particle counts (10-100 concurrent).

## Particle Types — Real-World Examples

Particles are spawned in response to gameplay events. Here are actual use cases from the codebase:

### Blood Splatter (Damage Feedback)

**When:** Player takes damage and has `cv_blood_splatter` enabled.
**File:** `client/src/scripts/objects/player.ts` (line ~616)

```typescript
ParticleManager.spawnParticles(random(15, 30), () => ({
  frames: "blood_particle",
  lifetime: random(1000, 3000),      // Vary lifetime
  position: this.position,
  layer: this.layer,
  alpha: {
    start: 1,
    end: 0                            // Fade out
  },
  scale: {
    start: randomFloat(0.8, 1.6),
    end: 0                            // Shrink to nothing
  },
  speed: randomPointInsideCircle(Vec(0, 0), 4),  // Radial spread
  zIndex: ZIndexes.Players + 1
}));
```

**Visual Effect:** 15-30 red blood particles burst in random directions, fade out over 1-3 seconds, shrink as they fade.

### Shell Casings (Gun Fire Feedback)

**When:** Gun fires; casings are ejected from weapon.
**File:** `client/src/scripts/objects/player.ts` (line ~354)

```typescript
ParticleManager.spawnParticles(casingSpec.count ?? 1, () => {
  const spinAmount = randomFloat(Math.PI / 2, Math.PI);
  const displacement = Vec.scale(
    randomVector(
      casingVelX?.min ?? 2,
      casingVelX?.max ?? -5,
      casingVelY?.min ?? 10,
      casingVelY?.max ?? 15
    ),
    this.sizeMod
  );
  
  if (casingVelX?.randomSign) {
    displacement.x *= randomSign();
  }
  
  return {
    frames: casingSpec.frame,
    position: gunMuzzlePos,
    speed: displacement,
    lifetime: 1500,
    zIndex: ZIndexes.Particles,
    padding: 10,
    scale: casingScale,
    rotation: {
      start: randomRotation(),
      end: randomRotation() + spinAmount  // Spin animation
    }
  };
});
```

**Visual Effect:** 1+ shell casings eject from gun at random angles, rotate while falling, disappear after 1.5 seconds.

### Gun Backblast (Recoil Smoke)

**When:** Gun fires with backblast definition.
**File:** `client/src/scripts/objects/player.ts` (line ~2000)

```typescript
ParticleManager.spawnParticles(
  backblast.particlesAmount,
  () => ({
    frames: trail.frame,
    speed: Vec.fromPolar(
      randomRotation(),
      randomFloat(backblast.min, backblast.max)
    ),
    position: gunBreechPos,  // Behind the gun
    lifetime: backblast.duration,
    zIndex: ZIndexes.Bullets - 1,
    scale: randomFloat(backblast.scale.min, backblast.scale.max),
    alpha: {
      start: randomFloat(trail.alpha.min, trail.alpha.max),
      end: 0
    },
    layer: this.layer,
    tint: pickRandomInArray([0x8a8a8a, 0x3d3d3d, 0x858585])  // Gray tints
  })
);
```

**Visual Effect:** Smoke/dust particles shoot out behind gun, fade to transparent.

### Explosion Rings (Large Impact)

**When:** Player hit by explosion or large impact.
**File:** `client/src/scripts/objects/player.ts` (line ~516)

```typescript
const options = {
  frames: "explosion_ring",
  position: this.position,
  lifetime: 1000,
  layer: this.layer,
  speed: Vec(0, 0)  // Stationary
};

// outer ring
ParticleManager.spawnParticle({
  ...options,
  scale: {
    start: randomFloat(0.45, 0.55),
    end: randomFloat(2.95, 3.05)    // Scale UP (expanding ring)
  },
  alpha: {
    start: randomFloat(0.55, 0.65),
    end: 0
  }
});

// inner ring (smaller, brighter)
ParticleManager.spawnParticle({
  ...options,
  scale: {
    start: randomFloat(0.15, 0.35),
    end: randomFloat(1.45, 1.55)
  },
  alpha: {
    start: randomFloat(0.25, 0.35),
    end: 0
  }
});
```

**Visual Effect:** Two concentric rings scale outward and fade, creating explosion effect. No motion; rings are centered on impact point.

### Airdrop Landing Smoke

**When:** Airdrop obstacle lands on map.
**File:** `client/src/scripts/objects/obstacle.ts` (line ~272)

```typescript
ParticleManager.spawnParticles(
  this.definition.airdrop?.particleAmount ?? 2,
  () => ({
    frames: `${this.definition.airdrop.particle}_1`,
    position: this.position,
    ...options(8, 18),              // Speed range
    rotation: { start: 0, end: randomFloat(Math.PI / 2, Math.PI * 2) }
  })
);
```

**Visual Effect:** 2-5 smoke particles launch from airdrop, spin while rising, fade out.

## Particle Acceleration & Velocity

### Linear Motion (No Gravity)

Particles move in a straight line at constant velocity. No gravity, drag, or acceleration:

```typescript
this.position = Vec.add(
  this.position,
  Vec.scale(this.options.speed, delta / 1000)
);
```

**Implications:**
- Velocity is global (world space), not relative to spawner.
- Particles with `speed: Vec(0, 0)` remain stationary (e.g., explosion rings).
- No decay; particles maintain velocity throughout lifetime.
- If velocity curve is desired, use a custom `onDeath` callback or emit new particles at different velocities.

### Velocity Specification

Velocity is a `Vector` (x, y) in units/second. Creation patterns:

```typescript
// Fixed direction
speed: Vec(5, 0)  // 5 units/sec rightward

// Random point in circle
speed: randomPointInsideCircle(Vec(0, 0), 4)  // 0-4 units/sec in any direction

// Velocity from polar coordinates
speed: Vec.fromPolar(randomRotation(), randomFloat(1, 5))  // Random angle, 1-5 units/sec

// Component-wise randomization
speed: randomVector(minX, maxX, minY, maxY)
```

### No Drag or Decay

Velocity is constant. To simulate deceleration:
- Use fixed lifetime (particle disappears before deceleration becomes visible).
- Or emit multiple sequential particles at different velocities.

## Particle Lifetime — Fade-Out Duration, Scaling Animation, Color Transitions

### Lifetime Property

`lifetime: number` is milliseconds from spawn to death:

```typescript
ParticleManager.spawnParticle({
  lifetime: 2000  // Particle lives 2 seconds
});
```

### Fade-Out Animation (Alpha)

Transition from opaque to transparent:

```typescript
alpha: {
  start: 1,     // Fully opaque
  end: 0        // Fully transparent
}
```

### Scaling Animation (Scale)

Grow/shrink over lifetime:

```typescript
// Shrink to nothing
scale: {
  start: 1.0,
  end: 0.0
}

// Expand (smoke cloud)
scale: {
  start: 0.45,
  end: 2.95
}

// No scaling
scale: 1.0  // Fixed, no animation
```

### Rotation Animation (Rotation)

Spin over lifetime:

```typescript
// Clockwise spin
rotation: {
  start: 0,
  end: Math.PI * 2  // 360°
}

// Random spin start, random spin end
rotation: {
  start: randomRotation(),
  end: randomRotation()
}
```

### Color Transitions

Colors are NOT animated. Tint is set at spawn and fixed for lifetime:

```typescript
tint: 0xff0000  // Red (fixed)
```

To simulate color fade:
- Create particles with different tints at different times.
- Use `alpha` animation to fade, which darkens the color via transparency.

### Easing Functions

Optional easing function transforms interpolation curve (0→1):

```typescript
alpha: {
  start: 1,
  end: 0,
  ease: EaseFunctions.quadIn   // Slow at start, fast at end
}

// Common easing functions (from math.ts):
// - EaseFunctions.linear (default)
// - EaseFunctions.quadIn / quadOut / quadInOut
// - EaseFunctions.cubicIn / cubicOut / cubicInOut
// - EaseFunctions.quarticIn / quarticOut
// - EaseFunctions.expoIn / expoOut
// - EaseFunctions.backIn / backOut
```

Example: Blood particles fade quickly at first, then linger:

```typescript
alpha: {
  start: 1,
  end: 0,
  ease: EaseFunctions.quadOut   // Fast fade at start
}
```

## Emitter System — Spawn Rate, Burst Emissions, Radiation Patterns

### ParticleEmitter Class

Spawns particles at a regular interval (`delay`):

```typescript
export class ParticleEmitter {
  delay: number;                            // Milliseconds between spawns
  active: boolean;                          // If false, no spawning
  spawnOptions: () => ParticleOptions;      // Function returning particle config
  
  destroy(): void {
    this._dead = true;  // Marked for cleanup on next update
  }
}
```

### Creating Emitters

```typescript
// Muzzle flash emitter (spawns every 100ms while firing)
const emitter = ParticleManager.addEmitter({
  delay: 100,
  active: true,
  spawnOptions: () => ({
    frames: "muzzle_flash",
    position: gunMuzzlePos,
    speed: Vec.fromPolar(gunDirection, 0),
    lifetime: 300,
    zIndex: ZIndexes.Bullets,
    scale: { start: 0.5, end: 0 }
  })
});

// Later: stop spawning
emitter.active = false;

// Later: clean up emitter
emitter.destroy();
```

### Spawn Rate Control

`delay` is milliseconds between spawns:

```typescript
delay: 50    // Spawn every 50ms = 20 particles/second
delay: 500   // Spawn every 500ms = 2 particles/second
```

### Burst Emissions

To spawn many particles at once, use `spawnParticles` (not emitters):

```typescript
// Burst: 30 particles at once
ParticleManager.spawnParticles(30, () => ({
  frames: "blood_particle",
  position: hitPos,
  speed: randomPointInsideCircle(Vec(0, 0), 5),
  lifetime: 2000,
  zIndex: ZIndexes.Players + 1
}));
```

### Radiation Patterns

Particles can burst in all directions by randomizing velocity:

```typescript
// Radial burst (all directions)
speed: randomPointInsideCircle(Vec(0, 0), maxSpeed)

// Directional cone
speed: Vec.fromPolar(
  centerAngle + randomFloat(-Math.PI / 4, Math.PI / 4),  // ±45° cone
  randomFloat(minSpeed, maxSpeed)
)

// One-way stream
speed: Vec.fromPolar(fixedAngle, randomFloat(minSpeed, maxSpeed))
```

## Sprite Sheets & Animation — Texture Selection, Frame Sequencing, Looping

### Frame Selection (Spritesheet Lookup)

Particles reference spritesheet frames by string ID:

```typescript
frames: "blood_particle"  // Single frame
// or
frames: ["explosion_ring", "explosion_ring_2", "explosion_ring_3"]  // Random frame
```

### Frame Randomization

If `frames` is an array, one frame is randomly selected at spawn (not per-update):

```typescript
export class Particle {
  constructor(options: ParticleOptions) {
    const frames = options.frames;
    const frame = typeof frames === "string"
      ? frames
      : frames[random(0, frames.length - 1)];  // Pick once
    
    const tintedParticle = TintedParticles[frame];
    this.image = new SuroiSprite(tintedParticle?.base ?? frame);
  }
}
```

**Use Case:** Vary appearance of blood particles without needing separate spawns:

```typescript
frames: [
  "blood_particle",
  "blood_particle_variant_1",
  "blood_particle_variant_2"
]
```

### Texture Tinting (TintedParticles Registry)

The `TintedParticles` registry maps frame IDs to base sprite + color tint:

```typescript
// From common/src/definitions/obstacles.ts
export const TintedParticles: Record<string, { 
  readonly base: string, 
  readonly tint: number, 
  readonly variants?: number 
}> = {
  cabin_wall_particle: { base: "wood_particle", tint: 0x5d4622 },
  metal_particle: { base: "metal_particle_1", tint: 0x5f5f5f },
  blood_particle: { base: "blood_particle", tint: 0xffffff },  // no tint
  // ... hundreds more
};
```

**Lookup Logic:**

```typescript
const tintedParticle = TintedParticles[frame];

// Tint precedence (first match wins):
// 1. options.tint (custom override)
// 2. TintedParticles[frame].tint (baked in registry)
// 3. 0xffffff (white, no tint)

this.image.tint = options.tint ?? tintedParticle?.tint ?? 0xffffff;
```

### No Per-Frame Animation

Particles do NOT animate through multiple frames of a spritesheet. Each particle holds a single static frame throughout its lifetime. Animation is achieved via:
- **Scale:** Grow/shrink the single sprite.
- **Alpha:** Fade in/out.
- **Rotation:** Spin the sprite.
- **Position:** Move the sprite.

### No Looping

If continuous frame-by-frame animation is needed, spawn sequential particles with different frames or use a `SyncedParticle` (server-synced) which supports frame sequences.

## Sorting & Rendering — Z-Index for Particles, Layer Management, Rendering Order

### Layer System

Particles render on PixiJS containers determined by `Layer` enum:

```typescript
export enum Layer {
  Ground = 0,      // Particles on ground
  UnderBuildings = 1,
  Buildings = 2,
  Furniture = 3,
  Death = 4,
  Equipment = 5,
  // ... etc
}
```

**Specification:**

```typescript
ParticleManager.spawnParticle({
  layer: Layer.Ground  // Render on ground layer container
});
```

**Internal:**

```typescript
spawnParticle(options: ParticleOptions): Particle {
  const particle = new Particle(options);
  this.particles.add(particle);
  
  // Add sprite to the appropriate layer container
  CameraManager.getContainer(options.layer ?? Layer.Ground).addChild(particle.image);
  
  return particle;
}
```

### Z-Index Within Layer

The `zIndex` property is a PixiJS z-order within a layer:

```typescript
ParticleManager.spawnParticle({
  zIndex: ZIndexes.Players + 1  // Render in front of players
});
```

**Common Z-Index Values:**

| Constant | Use |
|----------|-----|
| `ZIndexes.Bullets` | Projectiles |
| `ZIndexes.Bullets - 1` | Backblast smoke (behind bullets) |
| `ZIndexes.Players` | Players |
| `ZIndexes.Players + 1` | Blood, damage feedback, explosions |
| `ZIndexes.Particles` | General visual effects |

### Rendering Order

```
PixiJS Rendering Pipeline:
├─ Layer.Ground
│  ├─ zIndex: 0
│  ├─ zIndex: 1
│  └─ zIndex: ... (sorted)
├─ Layer.Buildings
│  ├─ zIndex: 0
│  ├─ zIndex: 1
│  └─ zIndex: ...
└─ Layer.... (deeper layers render later)
```

Particles on the same layer with different zIndex render back-to-front by zIndex. Particles on different layers render in layer order (ground first, ui last).

## Network Events — Server-Triggered Particles (Explosions, Impacts, Status Effects)

Server-triggered particles are **synced across all clients** via the network. They are `SyncedParticle` objects, not `Particle`:

### SyncedParticle (Network Integration)

`SyncedParticle` is a GameObject that renders particles defined by the server:

**File:** `client/src/scripts/objects/syncedParticle.ts`
**Definition Registry:** `common/src/definitions/syncedParticles.ts`

**Lifecycle:**

```
Server Event (e.g., grenade explodes)
  ↓
Server creates Smoke/TearGas obstacle
  ↓
Server serializes: SyncedParticle { pos, def, lifetime, ... }
  ↓
UpdatePacket sent to all clients
  ↓
Client Game.objects[ObjectCategory.SyncedParticle] receives update
  ↓
SyncedParticle.updateFromData() parses network data, starts animations
  ↓
SyncedParticle.update() per-frame, interpolates position/scale/alpha
  ↓
When lifetime expires, server removes object, clients remove locally
```

### Synced Particle Types

**File:** `common/src/definitions/syncedParticles.ts`

| Type | Trigger | Visual | Duration |
|------|---------|--------|----------|
| `smoke_grenade_particle` | Smoke grenade detonation | Drifting white clouds | 19-21 seconds |
| `tear_gas_particle` | Tear gas grenade | Light blue smoke | 19-21 seconds |
| `shrouded_particle` | Shrouded status item | Gray semi-transparent | 19-21 seconds |
| `ash_particle` | Ash urn obstacle | Dark gray ash clouds | 19-21 seconds |
| `airdrop_smoke_particle` | Airdrop landing | Fast dissipating smoke | 1.5-2.5 seconds |

**Key Properties:**

- **Scale Animation:** Most scale up over time (1.5-2 → 1.75-2.25), creating expanding smoke shape.
- **Alpha Animation:** Fade out with `expoIn` easing (slow at start, fast at end).
- **Position Animation:** May drift upward or sideways over lifetime (e.g., smoke rising).
- **Scope Zoom:** Smoke/tear gas particles trigger `snapScopeTo` (zooms scope to 1x when near player).

### Client-Only Particles vs. Synced Particles

| Property | Client-Only (`Particle`) | Synced (`SyncedParticle`) |
|----------|-------------------------|--------------------------|
| Lifetime | Milliseconds (short) | Server-controlled (long) |
| Network | No | Yes; all clients see same position |
| Position | Local position only | Interpolated from network updates |
| Scope Zoom | No | Yes (smoke/tear gas only) |
| Examples | Blood, casings, shell flash | Smoke grenades, tear gas, airdrops |

**Use Each For:**

- **Client-only particles:** Immediate visual feedback (gun fire, impact effects). Low latency.
- **Synced particles:** Gameplay-affecting effects (smoke obscures view, tear gas damages adrenaline). All clients must see same effect.

## Performance — Particle Pooling, Max Particle Limits, Update Optimization

### Pooling Strategy

**No explicit pre-allocated object pool.** Particles are created on-demand and garbage collected:

```typescript
// Creation
spawnParticle(options: ParticleOptions): Particle {
  const particle = new Particle(options);
  this.particles.add(particle);
  return particle;
}

// Cleanup (GC)
if (particle.dead) {
  this.particles.delete(particle);
  particle.image.destroy();
  // particle is now eligible for garbage collection
}
```

**Why acceptable:**
- Typical particle counts: 10-100 concurrent particles.
- Short lifetimes: 0.5-3 seconds.
- 40 TPS game loop: GC pauses impact 1 frame per ~100 particles, which is unnoticeable.

**Alternative if GC pressure becomes an issue:**
- Pre-allocate a `Particle` pool (e.g., 500 particles).
- Reuse particles via `reset()` method instead of creating new.
- This trades GC pressure for memory overhead. Not currently needed.

### Data Structure Efficiency

Active particles and emitters are stored in `Set<>` for O(1) add/remove:

```typescript
readonly particles = new Set<Particle>();
readonly emitters = new Set<ParticleEmitter>();
```

**Update loop is O(N):** Where N = number of active particles. Each particle update is O(1):
- Position physics: `Vec.add()`, `Vec.scale()` (vector ops)
- Property interpolation: `Numeric.lerp()`, easing function (O(1))
- PixiJS sync: `setZIndex()`, `setVPos()`, `setScale()`, etc. (O(1) per call)

### Typical Particle Budgets

Real-world examples from codebase:

```typescript
// Burst events (one-time)
ParticleManager.spawnParticles(random(15, 30), ...)  // Blood (15-30 particles)
ParticleManager.spawnParticles(10, ...)               // Shield destroy (10 particles)

// Emitter events (sustained)
ParticleManager.addEmitter({ delay: 100, ... })      // ~10 particles/second

// Network events (server-controlled)
// Smoke/tear gas: 10 particles per spawner, ~20 spawners per grenade
// = 200 particles total over lifetime
```

### Frame Budget Analysis

Game tick: 40 TPS → ~25 ms per tick

**Per-particle cost (typical):**
```
Position update: Vec.add() + Vec.scale() ≈ 0.05 ms
Property interpolation: 3× Numeric.lerp() ≈ 0.1 ms
PixiJS sync: 5 setters ≈ 0.2 ms
Total per particle: ~0.35 ms
```

**Scaling:**
```
100 particles × 0.35 ms = 35 ms
→ Exceeds 25 ms tick budget (impacts frame time)

50 particles × 0.35 ms = 17.5 ms (safe)
→ Leaves ~7.5 ms for other logic

Recommended max: ~100 concurrent particles for smooth 40 TPS
```

### Limits (Implicit, Not Enforced)

No hard cap on particle count. Limits are implicit:
- **Per-spawner:** Controlled by spawn rate (e.g., shell casing count, delay).
- **Per-event:** Limited by design (e.g., blood burst: max 30 particles).
- **Global:** Scales with concurrent events. No global limiter.

**If needed, could add:**
```typescript
const MAX_PARTICLES = 200;
if (this.particles.size < MAX_PARTICLES) {
  this.spawnParticle(options);
}
```

### Off-Screen Culling

**Not implemented.** Particles continue updating even when off-camera:

```typescript
// All particles update, regardless of camera position
for (const particle of this.particles) {
  particle.update(delta);  // No visibility check
}
```

**Impact:**
- Smoke/gas grenades spawn far off-screen may continue updating off-screen.
- CPU cost: O(N) even for off-screen particles.
- **Acceptable for Suroi scale:** Gun effective range ~200 units; typical map 1000×1000; off-screen particle count is small.

**Mitigation (if needed):**
- Check particle position against camera bounds: `if (pos.distance(cameraPos) > cullingRadius) continue;`
- Implement `particle.onOffScreen()` callback: destroy early if too far.

## Known Gotchas — Memory Leaks, Floating Particles, Z-Order Clipping

### 1. Emitter Memory Leak If Not Destroyed

**Context:** Long-running emitters that are never destroyed.
**Symptom:** Emitter continues spawning particles long after event ends (e.g., gun firing emitter kept alive).
**Root Cause:** Emitter marked `dead = true` only when `destroy()` is called. If caller forgets, active emitter runs forever.
**Prevention:**

```typescript
// BAD: Emitter kept alive
const emitter = ParticleManager.addEmitter({ ... });
// ... later, caller forgets to call emitter.destroy()

// GOOD: Explicitly destroy when done
const emitter = ParticleManager.addEmitter({ ... });
// ... when event ends:
emitter.destroy();

// GOOD: Use callback to self-destruct
const emitter = ParticleManager.addEmitter({
  delay: 100,
  active: true,
  spawnOptions: () => ({
    ...
    onDeath: (p) => {
      if (ParticleManager.particles.size === 0) {
        emitter.destroy();  // Self-destruct when all particles gone
      }
    }
  })
});
```

### 2. Floating Particles Off-Screen

**Context:** Particles with high velocity may fly off-map and continue updating.
**Symptom:** Particle position becomes very large (e.g., 10000, 10000), continuing to update off-screen.
**Root Cause:** No world bounds check; particles are not culled by position.
**Impact:** Minor CPU cost; acceptable for typical velocities (~10 units/sec, lifetime ~3 sec = max distance ~30 units).
**Prevention:**

```typescript
// Optional: bound-check in particle update
update(delta: number): void {
  this.position = Vec.add(this.position, Vec.scale(this.options.speed, delta / 1000));
  
  // Optionally kill if too far
  const MAX_DISTANCE = 1000;
  if (this.position.length > MAX_DISTANCE) {
    this.kill();
    return;
  }
  
  // ... rest of update
}
```

### 3. Z-Order Clipping w/ Multiple Layers

**Context:** Particles on a lower layer may render behind particles on an upper layer, breaking visual hierarchy.
**Symptom:** Blood particle (layer: Ground, zIndex: 10) renders behind smoke (layer: Buildings, zIndex: 5).
**Root Cause:** Layer enum determines rendering order; zIndex only sorts within a layer.
**Prevention:**

```typescript
// Blood on ground layer renders BEFORE buildings layer
// So specify blood on Buildings or higher layer if it should appear on top
ParticleManager.spawnParticle({
  layer: Layer.Buildings,  // Higher layer → renders later
  zIndex: 5,
  // ...
});
```

### 4. Tint Override Silent Failure

**Context:** Baked tint from `TintedParticles` overrides custom tint if not careful.
**Symptom:** Particle specified with `tint: 0xff0000` (red) but renders with registry tint (e.g., 0x5f5f5f gray).
**Root Cause:** Frame is in `TintedParticles` but `options.tint` was not passed.
**Prevention:**

```typescript
// If frame is in TintedParticles, be explicit about tint
ParticleManager.spawnParticle({
  frames: "metal_particle",  // In TintedParticles with tint 0x5f5f5f
  tint: 0xff0000,            // Explicit override → rendered red
  // ...
});
```

### 5. Animation Property Mutation

**Context:** Mutating `ParticleProperty` after spawn changes animation.
**Symptom:** Particle's scale doesn't animate as expected because options were reused.
**Root Cause:** `ParticleOptions` stored by reference in `Particle.options`. Mutations affect live particles.
**Prevention:**

```typescript
// BAD: Reusing options object
const baseOptions = { scale: { start: 1, end: 0 }, ... };
ParticleManager.spawnParticle({ ...baseOptions });
baseOptions.scale.end = 2;  // Oops, affects previous particle
ParticleManager.spawnParticle({ ...baseOptions });

// GOOD: Create fresh options for each particle
ParticleManager.spawnParticles(10, () => ({
  scale: { start: 1, end: 0 },  // New object each call
  // ...
}));
```

## Related Documents

### Tier 2
- [Particle System — Overview](../README.md) — Architecture, emitter/particle classes, synced particles, dependencies

### Tier 1
- [System Architecture](../../architecture.md) — Rendering pipeline, PixiJS integration
- [Data Model](../../datamodel.md) — ObjectDefinitions, SyncedParticles registry

### Tier 2 Subsystems
- [Object Definitions](../object-definitions/) — `SyncedParticles` registry, definition lookup, tinting
- [Client Rendering](../client-rendering/) — PixiJS `SuroiSprite`, layer rendering, z-order
- [Camera Management](../camera-management/) — render container hierarchy, viewport
- [Client Managers](../client-managers/) — `ParticleManager` as client-side singleton
- [Networking](../networking/) — `SyncedParticle` serialization in `UpdatePacket`

### Code References
- `client/src/scripts/managers/particleManager.ts` — `Particle`, `ParticleEmitter`, `ParticleManager` classes
- `client/src/scripts/objects/syncedParticle.ts` — `SyncedParticle` GameObject
- `common/src/definitions/syncedParticles.ts` — `SyncedParticles` registry
- `common/src/definitions/obstacles.ts` — `TintedParticles` registry
- `common/src/utils/math.ts` — `Numeric.lerp()`, `EaseFunctions`
- `common/src/constants.ts` — `Layer` enum, `ZIndexes`
