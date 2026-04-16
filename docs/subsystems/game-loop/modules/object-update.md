# Game Loop — Object Update Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/game.ts -->

## Purpose
Encapsulates the core object update loop that iterates all active game objects each tick and calls their `update()` methods, coordinating movement, ballistics, collision, and state changes across all object categories.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/game.ts` | Main game tick; update loop orchestration | High |
| `server/src/objects/` | All game object classes (Player, Gun, Projectile, Obstacle, etc.) | High |
| `server/src/utils/grid.ts` | Spatial grid: `ObjectCategory` pooling and queries | Medium |
| `common/src/utils/objectDefinitions.ts` | Object definition registry and interfaces | Medium |

## Update Loop Architecture

The game tick is organized as **three sequential passes** over all players. The second pass includes updating non-player objects:

```
Tick Start (every 25ms at 40 TPS)
  │
  ├─── Pass 1: Players Update
  │    │ for each player:
  │    │   player.update()        // state machine, input processing
  │    │
  │    └─ End: dirty flags set on player
  │
  ├─── Pass 2: Non-Player Objects + Network
  │    │ for each loot:
  │    │   loot.update()
  │    │
  │    │ for each projectile:
  │    │   projectile.update()
  │    │   projectile.checkDamage()
  │    │
  │    │ for each bullet:
  │    │   bullet.update()         // trajectory + collision
  │    │   collect damage records
  │    │
  │    │ for each player:
  │    │   player.secondUpdate()   // serialize to packet
  │    │
  │    └─ End: packets constructed, dirty objects buffered
  │
  └─── Tick End
       Return to next tick
```

**Key principle:** Objects update in category order, not random order. This prevents frame desync.

## Object Update Interface

All game objects extend `BaseGameObject` and implement `update()`:

```typescript
abstract class BaseGameObject {
  abstract update(): void;  // Called once per tick
  // ... other methods
}
```

### Update Responsibilities by Category

| Category | Update Logic | Damage Check |
|----------|--------------|--------------|
| **Player** | Movement, input, status effects | Yes (taken, not dealt) |
| **Loot** | Despawn timeout, item decay | No |
| **Projectile** | Lifetime countdown, detonation logic | No |
| **Parachute** | Descent velocity, ground collision | No |
| **SyncedParticle** | Animation frame, fadeout | No |
| **Bullet** | Trajectory + grid traversal, collision detection | Yes (collision → damage records) |
| **Obstacle** | Door state, destructible HP, fires | N/A (stateless objects) |
| **Gun** | Reload countdown, fire cooldown | N/A (part of Player inventory) |

## Per-Category Update Details

### Loot Update — @file server/src/objects/loot.ts
```typescript
loot.update()
  ├─ Check despawn timeout
  │   └─ If expired, mark dead and remove from world
  └─ Check layer transition (if on water, apply buoyancy)
```

**Implicit behavior:** Loot does NOT move on its own; only despawned by timeout.

### Projectile Update — @file server/src/objects/projectile.ts
```typescript
projectile.update()
  ├─ Decrement lifetime counter
  ├─ Apply physics (gravity if not instant)
  ├─ Check detonation triggers:
  │   ├─ Timeout reached
  │   ├─ Collision with obstacle/player
  │   └─ Custom logic per type (grenade bounce vs. flash explosion)
  └─ If detonated, queue explosion
```

**Output:** Explosions queued in `game.explosions` for damage processing

### Bullet Update (Complex) — @file server/src/objects/bullet.ts

Bullets are the most complex objects. Update trajectory and test collision:

```typescript
bullet.update()
  ├─ Step 1: Advance spatial position
  │   ├─ Current position + velocity × dt
  │   ├─ Check grid cells crossed this frame
  │   └─ Broad-phase: query grid for objects in crossed cells
  │
  ├─ Step 2: Narrow-phase collision test
  │   ├─ For each potential obstacle/player:
  │   │   ├─ Test ray-circle hitbox intersection
  │   │   ├─ If hit:
  │   │   │   ├─ Store damage record { object, damage, source }
  │   │   │   ├─ Apply penetration logic (bullets can pierce)
  │   │   │   └─ Mark bullet dead if penetration exhausted
  │   │   │
  │   │   └─ If no penetration left, stop trajectory
  │   │
  │   └─ Return array of damage records
  │
  └─ Step 3: Reflected bullet handling
      └─ If reflected by geometry, update velocity and continue
```

**DamageRecord collection:** All damage from all bullets collected this tick, then applied together (for shotgun desyncing, see next section)

### SyncedParticle Update
```typescript
particle.update()
  ├─ Decrement lifetime
  ├─ Advance animation frame
  └─ If expired, mark dead
```

**Synced particles:** Server-driven animations (smoke clouds, blood, explosions) sent to clients for rendering

## Bullet Damage Application (Deferred)

**CRITICAL:** Damage is NOT applied immediately when a bullet hits. Instead:

1. **Tick N:** All bullets update, all collisions tested, damage records collected
2. **After all bullets updated:** Iterate damage records and apply damage

**Why?** Prevents desyncing on shotgun pellets. Imagine a shotgun hits a destructible crate:
- Without deferral: Pellet 1 hits, crate destroyed, pellets 2–8 pass through
- With deferral: All 8 pellets test collision against the UN-destroyed crate, consistent with client

**Code location:** @file server/src/game.ts:360–400

```typescript
// Collect all damage records from all bullets
let records: DamageRecord[] = [];
for (const bullet of this.bullets) {
  records = records.concat(bullet.update());
  // ... lifetime and removal logic
}

// AFTER all bullets updated, apply damage
for (const { object, damage, source, weapon, position } of records) {
  object.damage({
    amount: damage,
    source,
    weaponUsed: weapon,
    position: position
  });
  // ... on-hit effects (explosions, etc.)
}
```

## Update Order Importance

**Players MUST update before non-player objects** because:
1. Player input is processed in Pass 1 (sets `pendingAction`)
2. Pendable actions affect visibility calculations in `secondUpdate()`
3. Network packet must reflect player's current state

**Within Pass 2, order is fixed:**
1. Loots (stateless, low impact)
2. Parachutes (independent)
3. Projectiles (before bullets, spawn explosions)
4. SyncedParticles (animations, low impact)
5. **Bullets (must update last, generate damage records)**

Non-player objects update before bullet damage is applied to ensure consistent state.

## Spatial Grid Integration

Objects are organized in a spatial grid for efficient collision detection:

```typescript
// ObjectPool indexed by ObjectCategory
this.grid.pool.getCategory(ObjectCategory.Player)   // → Player[]
this.grid.pool.getCategory(ObjectCategory.Loot)     // → Loot[]
this.grid.pool.getCategory(ObjectCategory.Projectile) // → Projectile[]
```

**Update pattern:**
```typescript
for (const projectile of this.grid.pool.getCategory(ObjectCategory.Projectile)) {
  projectile.update();
}
```

**Why:** Object pool is a typed array-like container; fast iteration, no allocation per frame.

## Network Dirty Object Tracking

As objects update, they set dirty flags:

```typescript
object.velocity.x = 5;  // Changed
object.dirty.position = true;  // Flag for network sync
```

After all updates, dirty objects are collected for serialization:

```typescript
this.partialDirtyObjects  // Minor changes (position, rotation only)
this.fullDirtyObjects     // Major changes (health, inventory, etc.)
```

These sets are read in `secondUpdate()` to construct update packets.

## Timeout System

Long-running timers (healing cooldown, reload time) are managed via `addTimeout()`:

```typescript
this.game.addTimeout(() => {
  gun.reload();
}, 1500);  // Callback after 1500ms
```

Timeouts are ticked each frame and executed when `this.game.now >= timeout.end`.

Location: @file server/src/objects/gameObject.ts and @file server/src/game.ts:310–325

## Complex Functions

### `game.tick()` — @file server/src/game.ts:313+
**Purpose:** Main game loop orchestration
**Complexity:** ~200 lines
**Called by:** Game worker process (Node.js event loop every ~25ms)
**Implicit behavior:**
- Updates all timeout callbacks
- Iterates all objects by category
- Defers damage application after bullet updates
- Generates explosions from projectile detonations

### `bullet.update()` — @file server/src/objects/bullet.ts
**Purpose:** Trajectory calculation and collision detection
**Complexity:** ~150 lines (broad-phase + narrow-phase)
**Implicit behavior:**
- Traces ray through grid cells
- Clips hitbox against obstacles (not perfect circle, axis-aligned boxes)
- Handles penetration (shotgun pellets can pass through obstacles)
- Returns array of `DamageRecord` (not applied yet)

### `object.damage()` — @file server/src/objects/gameObject.ts
**Purpose:** Apply damage and trigger on-hit logic
**Complexity:** ~50 lines
**Called by:** After all bullets updated
**Implicit behavior:**
- Subtracts from `health` pool
- Checks death condition
- Triggers on-hit effects (explosions, knockback)
- Marks object dirty for network sync

## Configuration & Tuning

**Game tick rate:** @file common/src/constants.ts
```typescript
GameConstants.tps = 40;  // ticks per second (25ms per tick)
```

**Grid cell size:** @file server/src/utils/grid.ts
```typescript
GRID_SIZE = 32;  // px per cell for spatial hashing
```

**Max bullets per frame:** No hard limit; depends on gun fire rate and ammo

**Penetration rules:** Defined per gun/bullet definition
```typescript
BulletDefinition {
  penetrationLevel?: number;  // How many objects it can pass through
}
```

## Related Documents
- **Tier 2:** [Game Loop](../README.md) — subsystem overview
- **Tier 2:** [Spatial Grid](../../spatial-grid/README.md) — efficient queries
- **Tier 3:** [Player Update](player-update.md) — sibling module (Pass 1)
- **Tier 3:** [Network Tick](network-tick.md) — packet construction (Pass 2 part 2)
- **Tier 3:** [Game State](game-state.md) — game phases and conditions
- **Patterns:** [../patterns.md](../patterns.md) — subsystem patterns
