# Particle System — Emitter Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/particle-system/README.md -->
<!-- @source: client/src/scripts/managers/particleManager.ts -->

## Purpose

Manages continuous particle emission: emitter creation, spawn rate scheduling, particle pooling, lifetime management, and emitter cleanup.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/particleManager.ts` | ParticleManager singleton, ParticleEmitter class, Particle class | High |
| `client/src/scripts/utils/pixi.ts` | SuroiSprite rendering helper | Medium |
| `client/src/scripts/managers/cameraManager.ts` | Layer-based container assignment | Medium |

## Business Rules

- **Emitter Lifecycle:** Active emitters spawn particles at fixed intervals until manually destroyed
  - Active: `emitter.active = true` → spawning occurs
  - Inactive: `emitter.active = false` → no spawn, but emitter persists
  - Dead: `emitter.dead = true` → marked for removal on next update
  
- **Spawn Rate:** Particles spawn when `now - emitter.lastSpawn >= emitter.delay`
  - `delay` in milliseconds (e.g., 50 ms = 20 particles/second)
  - Checked every frame; can spawn multiple particles if frame drops
  
- **Per-Particle Spawn:** Each spawn call executes `emitter.spawnOptions()` callback
  - Returns `ParticleOptions` with position, velocity, lifetime, frames
  - Callback called per-spawn (not pre-generated list) → dynamic positioning
  
- **Particle Lifetime:** Each particle expires after `options.lifetime` milliseconds
  - Expiry checked against `Date.now()`
  - Dead particles removed from manager; image destroyed (prevents memory leaks)

- **Emitter Cleanup:** Dead emitters removed from manager set on next update
  - No manual cleanup needed by caller (automatic)
  
- **Layer Routing:** Particles added to camera container based on `options.layer ?? Layer.Ground`

## Data Lineage

```
ParticleManager.update(delta)
│
├─ Particle update loop:
│  └─ for (particle of this.particles)
│     ├─ particle.update(delta)
│     │  ├─ Advance position by velocity * delta
│     │  ├─ Calculate lifetime interpolation (0 to 1)
│     │  ├─ Apply scale/alpha/rotation easing
│     │  └─ Update sprite (zIndex, position, scale, rotation, alpha)
│     └─ if (particle.dead):
│        ├─ Remove from particles set
│        ├─ Destroy image (cleanup)
│        └─ Call options.onDeath?.() callback
│
└─ Emitter update loop:
   └─ for (emitter of this.emitters)
      ├─ if (emitter.dead):
      │  └─ Remove from emitters set
      │
      └─ if (emitter.active && now - lastSpawn >= delay):
         ├─ spawnOptions = emitter.spawnOptions()
         ├─ Spawn particle with options
         ├─ emitter.lastSpawn = Date.now()
         └─ [Loop continues next frame]
```

## Complex Functions

### `ParticleManager.addEmitter(options)` — @file client/src/scripts/managers/particleManager.ts:50

**Purpose:** Create and register an emitter for the manager's update loop.

**Parameters:**
```typescript
interface EmitterOptions {
    readonly delay: number                    // Milliseconds between spawns
    readonly active: boolean                  // Start active or inactive
    readonly spawnOptions: () => ParticleOptions  // Spawn callback
}
```

**Returns:** `ParticleEmitter` instance (for later control: activate/deactivate/destroy)

**Implicit Behavior:**
- Emitter added to `this.emitters` Set immediately
- `lastSpawn` initialized to 0 (first particle spawns on next update)
- Callback stored as-is (called every spawn; no lazy evaluation)

### `ParticleEmitter.spawnOptions()` Callback Pattern

**Called by:** Manager update loop when delay condition met

**Typical Usage Example (explosion particles):**
```typescript
{
    delay: 50,  // One particle every 50 ms
    active: true,
    spawnOptions: () => ({
        frames: "explosion_particle",
        position: explosionPosition,
        speed: Vec.fromPolar(randomRotation(), randomFloat(100, 300)),
        lifetime: 1000,  // 1 second lifespan
        zIndex: 10,
        layer: Layer.Ground,
        scale: { start: 1.0, end: 0.2, ease: easeOutQuad },
        alpha: { start: 1.0, end: 0.0, ease: easeLinear },
        rotation: { start: randomRotation(), end: randomRotation() }
    })
}
```

**Dynamic Positioning:**
- Callback re-evaluated each spawn
- Allows particles offset from fixed emitter position
- Example: Spread each particle away from source center

### `Particle.update(delta)` — @file client/src/scripts/managers/particleManager.ts:140

**Purpose:** Advance particle lifecycle: position, scale, alpha, rotation interpolation.

**Interpolation Logic:**
```typescript
const interpFactor = (now - spawnTime) / lifetime;  // 0 to 1
// For each property:
value = lerp(start, end, ease(interpFactor));
```

**Properties Interpolated:**
- `scale`: If defined as object, lerp from `start` to `end` with optional easing
- `alpha` (opacity): 0 (invisible) to 1 (opaque)
- `rotation`: Angular rotation in radians

**Easing Functions:** Optional `ease(t)` callback (default: identity `t => t`)
- Example: `easeOutQuad = t => 1 - (1 - t) ** 2` (deceleration)
- Easing determines pacing: linear (constant speed), ease-out (slowing), etc.

### Particle Sprite Rendering — @file client/src/scripts/managers/particleManager.ts:180–190

**Code:**
```typescript
protected _updateImage(): void {
    this.image
        .setZIndex(this.options.zIndex)
        .setVPos(toPixiCoords(this.position))
        .setScale(this.scale)
        .setRotation(this.rotation)
        .setAlpha(this.alpha);
}
```

**Method Chaining:** SuroiSprite builder pattern for fluent API

**Coordinate System:**
- `toPixiCoords()` converts game world coords to Pixi canvas coords
- Allows particles to follow world position (camera panning moves particles)

### Emitter Cleanup — @file client/src/scripts/managers/particleManager.ts:30–45

**Automatic Removal:**
```typescript
for (const emitter of this.emitters) {
    if (emitter.dead) {
        this.emitters.delete(emitter);
        continue;  // Skip spawn check for dead emitters
    }
    
    if (emitter.active && emitter.lastSpawn + emitter.delay < Date.now()) {
        this.spawnParticle(emitter.spawnOptions());
        emitter.lastSpawn = Date.now();
    }
}
```

**Pattern:**
- No explicit cleanup call needed
- Caller sets `emitter.destroy()` to mark dead
- Manager removes dead emitters on next update
- Prevents accumulation of inactive emitters

## Dependencies

**Internal:**
- [Camera Management](../../camera-management/) — Layer container routing
- [Math utilities](../../core-math-physics/) — Vec operations, easing functions
- [Rendering Utils](../../../client/src/scripts/utils/) — SuroiSprite wrapper

**External:**
- PixiJS Display Hierarchy (containers per layer)
- Date.now() for timing

## Configuration

| Setting | Effect | Type | Default |
|---------|--------|------|---------|
| `EmitterOptions.delay` | Milliseconds between spawns | number | Required |
| `EmitterOptions.active` | Start emitter active or paused | boolean | Required |
| `ParticleOptions.lifetime` | Particle duration (ms) | number | Required |
| `ParticleOptions.speed` | Velocity vector (mm/s) | Vector | Required |
| `ParticleOptions.scale` | Static or animated scale | number \| ParticleProperty | Required |
| `ParticleOptions.alpha` | Static or animated opacity | number \| ParticleProperty | Required |
| `ParticleOptions.rotation` | Static or animated rotation (rad) | number \| ParticleProperty | Required |
| `ParticleOptions.tint` | Hex color (0xRRGGBB) | number | 0xffffff (white) |
| `ParticleOptions.onDeath` | Callback when particle expires | function | undefined |

## Performance Considerations

- **Emitter Count:** No hard limit; each emitter costs one callback invocation per spawn
- **Particle Count:** Scales with (emitter count × spawn rate). Typical: 100–1000 particles during combat
- **Frame Rate Impact:** Particle updates are O(n) in particle count; expect 10–20 µs per particle on modern hardware
- **Memory:** Each particle image held in Pixi display tree; destruction via `.destroy()` releases GPU memory
- **Garbage Collection:** Frequent particle creation/destruction can trigger GC pauses; consider object pooling for very high-frequency emitters (e.g., bullet tracer particles)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Particle System subsystem overview
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Camera:** [../../camera-management/README.md](../../camera-management/README.md) — Layer-based rendering
- **Math:** [../../core-math-physics/README.md](../../core-math-physics/README.md) — Vector, easing utilities
- **Synced Particles:** [../../particle-system/modules/synced-particles.md](synced-particles.md) — Server-synchronized particle effects
