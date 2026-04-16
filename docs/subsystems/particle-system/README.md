# Particle System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: client/src/scripts/managers/particleManager.ts -->

## Purpose

Manages visual effects that provide gameplay and feedback: smoke clouds (visual obscuration), tear gas (gameplay mechanic), ash particles, airdrop smoke, and other cosmetic effects. Particles sync across all clients for major gameplay events via the `SyncedParticle` network object; client-only particles provide local visual feedback (muzzle flashes, impact effects).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/managers/particleManager.ts` | `ParticleManager` singleton — emitter lifecycle, particle pooling, per-frame update |
| `client/src/scripts/objects/syncedParticle.ts` | `SyncedParticle` — GameObject that renders network-synced particle effects |
| `common/src/definitions/syncedParticles.ts` | `SyncedParticles` registry — particle effect definitions and configuration |
| `client/src/scripts/game.ts` | Game owns `ParticleManager`, calls `update()` per tick, resets on game end |

## Architecture

Particles flow in two independent streams:

### Stream 1: Client-Only Particles (Visual Feedback Only)

Local visual feedback that does not network-sync. Used for immediate UI feedback before server acknowledgment.

```
Input/Event (e.g., gun fire prediction)
  → ParticleManager.addEmitter({ delay, active, spawnOptions })
  → Emitter spawns particles at interval (delay)
  → Each particle: position += velocity each frame
  → Scale/alpha/rotation animate via ParticleProperty (start→end with easing)
  → Particle marked dead when lifetime expires
  → ParticleManager.update() cleans up dead particles and destroys image
```

**Key Classes:**

- `ParticleEmitter` — manages spawn timing
  - `delay: number` — milliseconds between spawns
  - `active: boolean` — if false, emitter does not spawn
  - `spawnOptions: () => ParticleOptions` — function returning particle config for each spawn
  - `destroy()` — marks emitter dead; cleaned up on next update cycle

- `Particle` — visual particle instance
  - `position: Vector` — world position
  - `lifetime: number` — ms particle lives before marked dead
  - `scale` / `alpha` / `rotation` — interpolated per frame
  - `image: SuroiSprite` — rendered sprite (PixiJS)
  - `update(delta)` — position physics, interpolate animations, check lifetime
  - `kill()` — immediate cleanup

### Stream 2: Server-Synced Particles (Gameplay Events)

Particles that represent gameplay state (smoke grenades, tear gas, ash urns, airdrops). All clients receive identical particle state from server and render in sync.

```
Gameplay Event (e.g., grenade explodes)
  → Server creates Smoke/TearGas/Ash obstacle
  → Obstacle.serialize() → SyncedParticle in UpdatePacket
  → All clients receive packet
  → Game.objects[ObjectCategory.SyncedParticle] updated
  → SyncedParticle.updateFromData() → set position, definition, animations
  → SyncedParticle.update() → interpolate position, scale, alpha per network tick
  → When lifetime expires, SyncedParticle removed from objects
```

**Key Class:**

- `SyncedParticle extends GameObject<ObjectCategory.SyncedParticle>`
  - `definition: SyncedParticleDefinition` — config from registry
  - `_positionAnim?: InternalAnimation<Vector>` — interpolate start→end position over duration (e.g., smoke drift)
  - `_scaleAnim?: InternalAnimation<number>` — animate scale with easing
  - `_alphaAnim?: InternalAnimation<number>` — animate alpha with easing
  - `angularVelocity: number` — rotation speed
  - `_age: number` — normalized 0→1 through lifetime
  - `_alphaMult: number` — special multiplier for creator (e.g., creator sees shrouded particles dimmer)
  - `updateFromData(data)` — parse network data, set animations
  - `update()` — advance age, interpolate position/scale/alpha, rotate
  - Optional: hitbox (for rendering debug, scope zoom triggering), scope snap

## Particle Types (SyncedParticles Registry)

| Effect ID | Name | Trigger | Visual | Mechanics |
|-----------|------|---------|--------|-----------|
| `smoke_grenade_particle` | Smoke Grenade Particle | Smoke grenade detonation | Drifting white smoke clouds | 10 particles/spawner, 300ms stagger, zooms scope to 1x |
| `plumpkin_smoke_grenade_particle` | Plumpkin Smoke Grenade | Plumpkin weapon | Purple-tinted smoke | Same as smoke grenade; tint 0x854770 |
| `shrouded_particle` | Shrouded Particle | Shrouded status item | Gray semi-transparent smoke | Creator sees at 15% opacity; 50% opacity for others; hasCreatorID=true |
| `tear_gas_particle` | Tear Gas Particle | Tear gas grenade | Light blue smoke | 10 particles/spawner; depletes adrenaline 0.0055/ms for caught players; zooms scope |
| `ash_particle` | Urn Ash Particle | Ash urn obstacle | Dark gray ash clouds | Visual effect; no gameplay mechanic |
| `airdrop_smoke_particle` | Airdrop Smoke | Airdrop landing | Fast dissipating smoke | 5 particles/spawner; 100ms stagger; 1500-2500ms lifetime (shorter than smoke grenade) |

**Registration Pattern:** All `SyncedParticles` inherit from `smokeLike()` template, which sets default scale (1.5-2→1.75-2.25), alpha fade (1→0 with expoIn easing), lifetime (19-21s), and zIndex.

## ParticleManager API

### Main Methods

| Method | Signature | Purpose | Returns |
|--------|-----------|---------|---------|
| `spawnParticle` | `(options: ParticleOptions): Particle` | Create one particle; add to active set; add image to render container |  `Particle` |
| `spawnParticles` | `(count: number, options: () => ParticleOptions): void` | Spawn N particles; call options function for each to vary (position, rotation, etc.) | void |
| `addEmitter` | `(options: EmitterOptions): ParticleEmitter` | Create emitter that spawns particles at interval; add to active set | `ParticleEmitter` |
| `update` | `(delta: number): void` | [Per-frame] Update all particles (physics, animations, lifetime); update all emitters (spawn new particles if active); clean up dead particles/emitters | void |
| `reset` | `(): void` | Clear all particles and emitters (called on game end) | void |

### ParticleOptions Type

```typescript
interface ParticleOptions {
  readonly frames: string | readonly string[]      // Sprite frame ID or array for random selection
  readonly position: Vector                         // World position
  readonly speed: Vector                            // Velocity (units per second)
  readonly lifetime: number                         // Milliseconds before marked dead
  readonly zIndex: number                           // PixiJS z-order
  readonly layer?: Layer                            // Render layer (default: Ground)
  readonly scale?: ParticleProperty                 // Start/end scale with optional easing
  readonly alpha?: ParticleProperty                 // Start/end alpha with optional easing
  readonly rotation?: ParticleProperty              // Start/end rotation with optional easing
  readonly tint?: number                            // Color tint (hex, e.g., 0xff0000)
  readonly onDeath?: (particle: Particle) => void  // Callback when particle dies
}

// ParticleProperty: Fixed number OR animated
type ParticleProperty = number | {
  readonly start: number
  readonly end: number
  readonly ease?: (x: number) => number  // Easing function (0→1)
}
```

### EmitterOptions Type

```typescript
interface EmitterOptions {
  readonly delay: number                           // Milliseconds between spawns
  readonly active: boolean                         // If false, emitter inactive
  readonly spawnOptions: () => ParticleOptions    // Function returning particle config
}
```

## Performance Optimization

### Particle Pooling
- Particles stored in `Set<Particle>` for O(1) add/remove
- Dead particles deleted immediately; `image.destroy()` cleans PixiJS resources
- No pre-allocation pool; particles created on-demand (GC pressure acceptable for particle counts)

### Emitter Pooling
- Active emitters stored in `Set<ParticleEmitter>`; inactive emitters removed next update cycle
- Emitters can be toggled `active: boolean` to pause/resume spawning without destruction

### Limits (Implicit)
- **Per-emitter spawn rate:** controlled by `delay` (e.g., smoke grenades spawn every 300ms)
- **Global particle count:** unbounded; scales with number of active emitters
- **Typical per-spawner:** 10 particles for smoke/tear gas; 5 for airdrop
- **Spawner stagger:** `staggering.delay` (e.g., 300ms) spreads particle births over time, amortizing GPU cost

### Frame Budget
- Particle `update()` is O(N) where N = active particle count
- `ParticleManager.update(delta)` runs per-frame in game loop (40 TPS, ~25ms per tick)
- Position physics: `Vec.add(position, Vec.scale(speed, delta/1000))`
- Animation interpolation: O(1) per property (scale/alpha/rotation)

### Off-Screen Culling
Particles are not culled when off-camera; they continue updating even outside viewport. This is acceptable for smoke/ash (typically short-lived or slow-moving) but means off-screen effects still have CPU cost.

### Scope Zoom Mechanic
Synced particles can trigger scope zoom (snapScopeTo):
- Smoke/tear gas particles: `snapScopeTo: "1x_scope"` — zooms player's scope to 1x when particle is nearby
- Scope zooming is handled by `objectManager` (not `ParticleManager`); particles just carry the definition

## Dependencies

### Depends on:

- **[Object Definitions](../object-definitions/)** — `SyncedParticles` registry lookup by ID; definitions control animation, lifetime, sprites
- **[Constants](../../constants.ts)** — `Layer` enum (render layer), `ZIndexes` (z-order values)
- **[Math Utils](../../utils/math.ts)** — `Numeric.lerp()`, `EaseFunctions` (easing function library), `Angle.normalize()`
- **[Vector Utils](../../utils/vector.ts)** — `Vec.add()`, `Vec.scale()`, `Vec.lerp()` for position physics
- **[PixiJS Utilities](../utils/pixi.ts)** — `SuroiSprite`, `toPixiCoords()` for sprite rendering

### Depended on by:

- **[Game Loop](../game-loop/)** — calls `ParticleManager.update(delta)` every tick; calls `reset()` on game end
- **[Game Objects Client](../game-objects-client/)** — `SyncedParticle` is a client-side GameObject; integrated via `objectManager`
- **[Networking](../networking/)** — `SyncedParticle` is serialized in `UpdatePacket` as part of `ObjectsNetData[ObjectCategory.SyncedParticle]`
- **[Camera Management](../camera-management/)** — `ParticleManager` adds particles to `CameraManager.getContainer(layer)` for rendering

## Known Issues & Gotchas

### 1. Off-Screen Emitters Not Culled
**Context:** Client-only emitters (`ParticleManager`) continue updating even when far off-camera.
**Symptom:** Emitters spawned during combat may keep running after player leaves area.
**Impact:** Minor CPU cost; acceptable for typical game scales.
**Mitigation:** Manually call `emitter.destroy()` when emitter is no longer needed (e.g., gun stops firing).

### 2. Synced Particle Lifetime is Server-Driven
**Context:** Synced particles die when server removes them from network state.
**Symptom:** If client receives out-of-order packets, particle lifetime may desync between clients.
**Mitigation:** Lifetime is resent every UpdatePacket; one-frame desync is imperceptible.

### 3. Scope Zoom Lingers After Particle Dies
**Context:** Smoke particles trigger scope zoom; zoom reverts when particle dies. But smoke dissipates visually before lifetime expires.
**Symptom:** Player may see through smoke visually but scope remains zoomed.
**Mitigation:** `scopeOutPreMs` property (e.g., 3200ms) reverts zoom N ms before particle dies.

### 4. Alpha Multiplier for Creator
**Context:** Shrouded particles use `creatorMult` to make creator see effect dimmer (0.15 opacity).
**Symptom:** Creator of shrouded item sees very dim smoke; other players see normal opacity.
**Behavior:** Intentional; prevents creator from self-blocking.

### 5. Sprite Tinting via TintedParticles Registry
**Context:** Particle frames can have baked tint overrides in `TintedParticles` (from obstacles definitions).
**Symptom:** If frame is in `TintedParticles`, its tint is applied even if custom tint is passed.
**Precedence:** `options.tint` (custom) → `TintedParticles[frame].tint` (baked) → 0xffffff (white/no tint).

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Rendering pipeline, GameObject model, network architecture
- [Data Model](../../datamodel.md) — ObjectDefinitions registry pattern

### Tier 2 — Subsystems
- [Object Definitions](../object-definitions/) — `SyncedParticles` registry and Definition lookup
- [Networking](../networking/) — `SyncedParticle` serialization in `UpdatePacket`
- [Client Rendering](../client-rendering/) — PixiJS sprite rendering, layer management
- [Game Objects (Client)](../game-objects-client/) — GameObject lifecycle, `SyncedParticle` pool integration
- [Camera Management](../camera-management/) — render container hierarchy

### Tier 3 — Modules (if available)
- Client Particle Emitter module — detailed `ParticleEmitter` / `Particle` lifecycle
- Synced Particle Updates module — network deserialization and animation
- Particle Definition Registry module — spritesheet frame mapping, tint resolution
