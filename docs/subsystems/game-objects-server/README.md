# Server-Side Game Objects

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/game-objects-server/modules/ -->
<!-- @source: server/src/objects/ -->

## Purpose

Unified runtime entity system encompassing all server-side game objects. Each object inherits from [`BaseGameObject`] (or specialized base) and manages state, collision, damage, networking, and lifecycle. Objects flow through a single update loop in [`Game.tick()`] and synchronize to clients via dirty flags and serialization.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/objects/gameObject.ts` | Abstract [`BaseGameObject`] base class — id, position, rotation, collision, dirty flags, serialization |
| `server/src/objects/player.ts` | [`Player`] — player entity; movement, damage, inventory, actions; **1 per connected socket** |
| `server/src/objects/obstacle.ts` | [`Obstacle`] — map static/destructible obstacles; health, loot drops, doors, scales |
| `server/src/objects/building.ts` | [`Building`] — interactable structures; spawn hitbox, ceiling damage, puzzle logic |
| `server/src/objects/bullet.ts` | [`Bullet`] — ballistic projectiles from guns; **server-only, not in ObjectCategory** |
| `server/src/objects/projectile.ts` | [`Projectile`] — thrown objects (grenades); physics, bounce, fuse, damage on impact |
| `server/src/objects/loot.ts` | [`Loot`] — dropped items (guns, ammo, armor); collection logic, despawn |
| `server/src/objects/syncedParticle.ts` | [`SyncedParticle`] — networked visual effects; animations, lifetime, position easing |
| `server/src/objects/deathMarker.ts` | [`DeathMarker`] — tombstone on death; 30-second lifetime, tied to player |
| `server/src/objects/decal.ts` | [`Decal`] — persistent marks (blood, bullet holes); no collision, immutable |
| `server/src/objects/emote.ts` | [`Emote`] — overhead emoticon above player; **not a GameObject, just data** |
| `server/src/objects/parachute.ts` | [`Parachute`] — landing device during airdrop; falling animation, landing damage |
| `server/src/objects/explosion.ts` | [`Explosion`] — transient damage area; ray-cast visibility, knockback; **not a GameObject** |
| `server/src/objects/mapIndicator.ts` | [`MapIndicator`] — beacon on map (loot indicator, player highlight); **separate tracking** |

## Architecture

All game objects flow through a centralized update system coordinated by the [`Game`] instance:

```
Game.tick() (every 25ms @ 40 TPS)
  ├─ Handle timeouts
  ├─ Check airdrop spawn
  ├─ Update objects: loot, parachute, projectile, syncedParticle
  │   └─ Each object: hitbox, collisions, state mutations
  ├─ Update bullets (server-only)
  │   └─ Collision detection → DamageRecords
  ├─ Apply damage (all accumulated records)
  ├─ Update explosions (ray-cast damage)
  ├─ Update gas
  ├─ Update living players (movement, actions)
  ├─ Serialize dirty objects (full/partial)
  ├─ Calculate visibility & send UpdatePackets to clients
  └─ Clear dirty sets & reset state
```

Nearly all classes extend [`BaseGameObject`], providing:

- **Identity:** `id` (uint16, via [`IDAllocator`]), `type` (ObjectCategory enum)
- **Spatial:** `position` (Vector), `rotation` (angle), `layer` (Layer enum for depth sorting)
- **State:** `dead` (flag), `damageable` (flag), `hitbox` (optional Hitbox)
- **Network Sync:** `setDirty()` / `setPartialDirty()` — marks for serialization
- **Lifecycle:** `constructor(game, position)`, `update()`, `damage()`, `destroy()`
- **Serialization:** `serializeFull()` / `serializePartial()` — encode state to byte stream

## Object Categories (ObjectCategory Enum)

The `ObjectCategory` enum (from [`@common/constants.ts`](../../../../common/src/constants.ts)) defines 9 serializable object types. Game maintains a spatial grid with category-keyed object pools accessed via `grid.pool.getCategory(ObjectCategory.X)`:

| ID | Name | Class | Count | Synced | Purpose |
|----|------|-------|-------|--------|---------|
| 0 | Player | [`Player`] | ≤100 | ✅ Yes | Connected player entities |
| 1 | Obstacle | [`Obstacle`] | ≤10,000 | ✅ Yes | Map fixtures (trees, rocks, crates, etc.) |
| 2 | DeathMarker | [`DeathMarker`] | ≤100 | ✅ Yes | Tombstone on player death |
| 3 | Loot | [`Loot`] | ≤10,000 | ✅ Yes | Item drops (guns, ammo, armor, healing) |
| 4 | Building | [`Building`] | ≤1000 | ❌ No | Structures with doorways (not synced as objects) |
| 5 | Decal | [`Decal`] | ≤10,000 | ✅ Yes | Visual marks (blood splatters, bullet holes) |
| 6 | Parachute | [`Parachute`] | ≤10,000 | ✅ Yes | Airdrop falling devices |
| 7 | Projectile | [`Projectile`] | ≤10,000 | ✅ Yes | Grenades, throwables in flight |
| 8 | SyncedParticle | [`SyncedParticle`] | ≤100,000 | ✅ Yes | Network-synced visual effects |

**Non-ObjectCategory entities** (tracked separately, not in grid.pool):

- **Bullet** — Ballistic projectiles from guns; server-only, no network serialization; collision triggers `DamageRecord` entries
- **Explosion** — Damage/knockback effect manager; transient, ray-cast LOS; not an object, just a computation
- **Emote** — Floating emoticon; data holder, not a `BaseGameObject`; stored in `game.emotes[]`
- **MapIndicator** — Map beacon; separate ID allocator; tracks position/definition dirty flags

## Object Lifecycle

### Spawning

```typescript
// 1. Instantiate
const player = new Player(game, socket, position, layer, team);
const loot = new Loot(game, definition, position, layer, { count: 5 });
const bullet = new Bullet(game, source, shooter, options);

// 2. Register in grid (BaseGameObject only)
game.grid.addObject(player);      // → grid.pool allocates ID, adds to spatial hash
game.grid.addObject(loot);

// 3. Mark dirty (added automatically in constructor via setDirty())
player.setDirty();                // → added to game.fullDirtyObjects

// 4. Network sync
// Next game.tick() → serializeFull() → included in UpdatePacket
```

### Updating (Per-Tick)

```typescript
// In Game.tick() for each category:
for (const loot of game.grid.pool.getCategory(ObjectCategory.Loot)) {
    loot.update();                // Apply physics, check despawn timer, etc.
}

// Player updates happen separately:
for (const player of game.livingPlayers) {
    player.update();              // Movement, actions, damage, state mutations
}

// Bullet updates (server-only):
for (const bullet of game.bullets) {
    const damageRecords = bullet.update();  // Collision detection
    records.concat(damageRecords);
}

// Dirty flags (set during update):
object.setPartialDirty();         // → added to game.partialDirtyObjects
// or
object.setDirty();                // → added to game.fullDirtyObjects (full + partial)
```

Network sync happens after all updates:

```typescript
// Serialize dirty objects
for (const obj of game.partialDirtyObjects) {
    if (!game.fullDirtyObjects.has(obj)) {
        obj.serializePartial();   // Encode changes only
    }
}
for (const obj of game.fullDirtyObjects) {
    obj.serializeFull();          // Encode full state
}

// Clear dirty sets (next tick, new mutations mark dirty again)
game.fullDirtyObjects.clear();
game.partialDirtyObjects.clear();
```

### Destruction

```typescript
// 1. Mark dead
object.dead = true;

// 2. Remove from grid
game.removeObject(object);        // → grid.removeObject()
                                  // → spatial hash removal
                                  // → ID allocator recycles object.id

// 3. Cleanup dependencies
object.destroy?.();               // (if applicable)

// 4. Network sync
// Object marked dead, client receives UpdatePacket with delete flag
```

Example: **Loot despawn**

```typescript
// game/tick() watches loot timers
for (const loot of game.grid.pool.getCategory(ObjectCategory.Loot)) {
    loot.update();
    if (loot.despawnTimer.expired) {
        game.removeLoot(loot);      // → dead, removeObject(), removed from game
    }
}
```

## Key Patterns

### Damage Model

All damageable objects implement `damage(params: DamageParams)`:

```typescript
interface DamageParams {
    readonly amount: number
    readonly source?: GameObject | DamageSources.Gas | DamageSources.Obstacle | ...
    readonly weaponUsed?: GunItem | MeleeItem | ThrowableItem | Explosion | Obstacle
    readonly position?: Vector
}

object.damage({ amount: 50, source: bullet.shooter, weaponUsed: bullet.sourceGun });
```

**Damageable objects:**
- [`Player`] — `health + shield + adrenaline` system
- [`Obstacle`] — `health` → destruction → loot drops
- [`Building`] — ceiling damage via `damageCeiling()`
- [`Projectile`] — health for C4, grenades
- **Not damageable:** [`Loot`], [`Decal`], [`DeathMarker`]

### Dirty Flag Pattern

Objects track network state changes via two dirty sets:

```typescript
// Full dirty: entire state sent to clients
object.setDirty();
game.fullDirtyObjects.add(object);

// Partial dirty: only changed fields sent
object.setPartialDirty();
game.partialDirtyObjects.add(object);
```

**Budget constraints:** Each object has allocation budgets:

```typescript
class Loot extends BaseGameObject.derive(ObjectCategory.Loot) {
    override readonly fullAllocBytes = 4;      // Full state: 4 bytes
    override readonly partialAllocBytes = 8;   // Partial updates: 8 bytes
}

class Player extends BaseGameObject.derive(ObjectCategory.Player) {
    override readonly fullAllocBytes = 16;     // Join packet: 16 bytes
    override readonly partialAllocBytes = 12;  // Position update: 12 bytes
}
```

### Grid Spatial Hashing

Objects indexed by position for collision queries:

```typescript
// Find objects in a hitbox (early-exit)
const nearby = game.grid.intersectsHitbox(circleHitbox, layer);
for (const obj of nearby) {
    if (obj.hitbox?.collidesWith(circleHitbox)) {
        // Collision detected
    }
}
```

### Event System

Objects can emit events via [`PluginManager`]:

```typescript
game.pluginManager.emit("loot_will_generate", { definition, ... });
game.pluginManager.emit("loot_did_generate", { loot, ... });
game.pluginManager.emit("player_will_connect", ...);
game.pluginManager.emit("building_will_damage_ceiling", { building, damage });
```

## Interfaces & Contracts

### `BaseGameObject` (Abstract Base)

**Constructor:**
```typescript
constructor(game: Game, position: Vector)
```

**Public Properties:**
| Property | Type | Mutable | Purpose |
|----------|------|--------|---------|
| `id` | `uint16` | ✖️ No | Unique network ID (via IDAllocator) |
| `game` | `Game` | ✖️ No | Reference to parent game instance |
| `type` | `ObjectCategory` | ✖️ No | Object class identifier |
| `position` | `Vector` | ✅ Yes | X, Y coordinates |
| `rotation` | `number` | ✅ Yes | Angle in radians |
| `layer` | `Layer` | ✅ Yes | Vertical depth (Basement, Ground, Upstairs) |
| `hitbox` | `Hitbox?` | ✅ Yes | Collision shape (CircleHitbox, RectangleHitbox) |
| `damageable` | `boolean` | Read-only | Can this object take damage? |
| `dead` | `boolean` | Read-only | Is this object dead/destroyed? |
| `fullAllocBytes` | `number` | Read-only | Network buffer size for full state |
| `partialAllocBytes` | `number` | Read-only | Network buffer size for partial updates |

**Public Methods:**
```typescript
// Network sync marks
setDirty(): void                          // Mark for full serialization
setPartialDirty(): void                   // Mark for partial serialization

// Serialization (internal, called by Game)
serializeFull(): void                     // Encode full state to fullStream
serializePartial(): void                  // Encode partial state to partialStream

// Damage & lifecycle
abstract damage(params: DamageParams): void
abstract get data(): FullData<Cat>        // Serializable data object
```

### `Player` (Extends BaseGameObject)

**Key Methods:**
```typescript
addHealth(amount: number): void           // Heal player
damage(params: DamageParams): void        // Take damage, trigger death if health ≤ 0
takeAction(action: Action): void          // Start reload/heal/revive action
giveItem(item: InventoryItem): boolean    // Add to inventory, return success
dropItem(slot: number): void              // Drop weapon at current position
```

**Key Properties:**
```typescript
name: string                              // Player username
health: number                            // Current health (0–maxHealth)
adrenaline: number                        // Adrenaline (0–100)
shield: number                            // Shield (0–100)
inventory: Inventory                      // Items, weapons, consumables
```

### `Obstacle` (Extends BaseGameObject)

**Key Properties:**
```typescript
definition: ObstacleDefinition            // Definition lookup
health: number                            // Current health
maxHealth: number                         // Max health (from definition)
scale: number                             // Visual/collision scale
collidable: boolean                       // Can bullet/player collide?
isDoor: boolean                           // Is this a door?
loot: LootItem[]                          // Loot drops on destruction
```

**Key Methods:**
```typescript
damage(params: DamageParams): void        // Reduce health, spawn loot, destroy if health ≤ 0
```

### `Building` (Extends BaseGameObject)

**Key Properties:**
```typescript
definition: BuildingDefinition            // Definition lookup
collidable: boolean                       // Can players/objects collide?
damageable: boolean                       // Can ceiling be damaged?
hitbox?: Hitbox                           // Collision shape
scopeHitbox?: Hitbox                      // Scope blocking (ceiling effect)
interactableObstacles: Set<Obstacle>      // Doors, buttons, etc.
hasPuzzle: boolean                        // Does this building have a puzzle?
puzzle?: { ... }                          // Puzzle state (solved, inputOrder, etc.)
```

**Key Methods:**
```typescript
damageCeiling(damage?: number): void      // Reduce wall count, potentially destroy
```

### `Loot` (Extends BaseGameObject)

**Key Properties:**
```typescript
definition: LootDefinition                // Definition lookup (gun, ammo, armor, etc.)
count: number                             // Stack count (ammo, healing items)
itemData?: ItemData                       // Metadata (kills, damage for guns)
velocity: Vector                          // Physics velocity
hitbox: CircleHitbox                      // Collision (position tracked here)
mapIndicator?: MapIndicator               // Optional beacon (legendary loot)
```

**Key Methods:**
```typescript
push(angle: number, speed: number): void  // Apply physics velocity
```

### `Bullet` (Extends BaseBullet, NOT BaseGameObject)

**Key Properties:**
```typescript
game: Game                                // Parent game
sourceGun: GunItem | Explosion            // Source weapon
shooter: GameObject                       // Who fired this bullet
definition: BulletDefinition              // Ballistics, damage
clipDistance: number                      // Max range
finalPosition: Vector                     // End of trajectory
position: Vector                          // Current position
rotation: number                          // Angle
layer: number                             // Vertical layer
```

**Key Methods:**
```typescript
update(): DamageRecord[]                  // Trace projectile, return collisions
```

**Note:** Bullets are **server-only**. They do not appear in ObjectCategory and are not synced to clients directly. Damage is transmitted via [`KillPacket`] or indirectly through explosion effects.

### `Projectile` (Extends BaseGameObject)

**Key Properties:**
```typescript
definition: ThrowableDefinition           // Definition lookup
health: number                            // For C4 detonation
hitbox: CircleHitbox                      // Collision shape
owner: GameObject                         // Who threw this?
inAir: boolean                            // Currently flying?
activated: boolean                        // C4 armed?
```

**Key Methods:**
```typescript
update(): void                            // Update position, check fuse, apply physics
damage(params): void                      // Take damage (C4 detonation)
```

### `Explosion` (NOT a GameObject)

**Constructor:**
```typescript
constructor(
    game: Game,
    definition: ExplosionDefinition,
    position: Vector,
    source: GameObject,
    layer: Layer,
    weapon?: GunItem | MeleeItem | ThrowableItem,
    damageMod?: number,
    objectsToIgnore?: Set<GameObject>
)
```

**Key Methods:**
```typescript
explode(): void                           // Ray-cast damage to all nearby objects
```

**Note:** Explosions are **transient calculations**, not persistent objects. Created, immediately exploded, discarded in the same tick.

### `SyncedParticle` (Extends BaseGameObject)

**Key Properties:**
```typescript
definition: SyncedParticleDefinition      // Animation & behavior
creatorID?: number                        // Who created this particle?
scale: number                             // Visual scale (animated)
angularVelocity: number                   // Rotation speed
age: number                               // Current age in milliseconds
```

**Key Methods:**
```typescript
update(): void                            // Update age, animate position/scale/alpha
```

### `DeathMarker` (Extends BaseGameObject)

**Key Properties:**
```typescript
player: Player                            // Associated player
hitbox: RectangleHitbox                   // Collision (fixed 5×5)
isNew: boolean                            // Just created? (for animation state)
```

**Note:** Immutable after creation. Cannot be damaged.

### `Decal` (Extends BaseGameObject)

**Key Properties:**
```typescript
definition: DecalDefinition               // Type (blood, bullet hole, etc.)
position: Vector                          // Location
rotation: number                          // Visual rotation
layer: Layer                              // Depth
```

**Note:** No hitbox, no collision. Immutable visual marks.

### `Parachute` (Extends BaseGameObject)

**Key Properties:**
```typescript
height: number                            // 0–1, falling animation
hitbox: CircleHitbox                      // Collision with objects
endTime: number                           // Landing time (timestamp)
```

**Key Methods:**
```typescript
update(): void                            // Decrease height, trigger landing on height < 0
```

### `Emote` (NOT a BaseGameObject)

**Properties:**
```typescript
definition: EmoteDefinition               // Emoticon type
player: Player                            // Associated player
playerID: number                          // Player's network ID
```

**Note:** Data holder only. Stored in `game.emotes[]`, not in grid.pool.

### `MapIndicator` (NOT a BaseGameObject)

**Properties:**
```typescript
id: number                                // Unique per-game ID
definition: MapIndicatorDefinition        // Type (loot, player, airdrop)
position: Vector                          // Location (updated via dirty flag)
positionDirty: boolean                    // Position changed
definitionDirty: boolean                  // Definition changed
dead: boolean                             // Marked for removal
```

## Dependencies

### Depends on:

- **[Game Loop](../game-loop/)** — [`Game.tick()`] orchestrates all object updates; [`Game.addObject()`], [`Game.removeObject()`] manage lifecycle
- **[Spatial Grid](../spatial-grid/)** — [`Grid.addObject()`], [`Grid.removeObject()`], `grid.intersectsHitbox()` for collision queries
- **[Inventory](../inventory/)** — [`Player`] owns `inventory`; [`Loot`] contains item definitions
- **[Networking](../networking/)** — Objects serialize via `serializeFull()` / `serializePartial()`; transmitted in [`UpdatePacket`]
- **[Object Definitions](../object-definitions/)** — [`ObjectDefinitions`] registry for O(1) lookup by `idString`; definition reification
- **[Plugin System](../plugins/)** — [`PluginManager`] event bus; ~30 typed events (e.g., `"player_will_connect"`, `"loot_did_generate"`)

### Depended on by:

- **[Game Loop](../game-loop/)** — [`Game`] owns all objects; `tick()` drives updates
- **[Networking](../networking/)** — Objects serialize to [`UpdatePacket`]; clients deserialize and render
- **[Client Rendering](../client-rendering/)** — Client-side `ObjectPool` mirrors server objects; receives synced state

## Known Issues & Gotchas

### 1. **Bullet Is Server-Only**
`Bullet` extends `BaseBullet` (from `@common/utils/`), **not** `BaseGameObject`. Bullets are never in `ObjectCategory` or `grid.pool`. They exist only on the server. Collision data is transmitted indirectly via `DamageRecord` entries or explosion effects in `KillPacket`.

### 2. **NetworkID Wraps at 65,536**
`IDAllocator` uses 16-bit unsigned integers (uint16). After allocating 65,536 IDs, the allocator recycles freed IDs in a FIFO queue. **Risk:** If a client is holding a stale reference to a destroyed object when its ID is reused, the client may receive updates for the wrong object. Clients should always verify object type when receiving updates.

### 3. **Building Objects Don't Serialize**
`Building` is in `ObjectCategory` but **does not** transmit network state. Buildings are communicated to clients **once** in the [`GameMap`] buffer during initial join. Clients reconstruct buildings from the map definition. Door state is transmitted separately (interaction packets). See [Map Generation](../map/) for details.

### 4. **Collision Happens Every Frame**
No AABB cache or early-exit optimization. Every `update()` may perform `grid.intersectsHitbox()` queries. Objects with expensive hitbox calculations (rotated rectangles) can contribute to tick time. The 40 TPS limit (25ms per tick) is the hard boundary.

### 5. **Layer Changes During Update Can Cause Visual Popping**
When an object's `layer` is mutated during `update()`, the spatial grid position is not updated. The grid entry remains under the old layer until the object is removed/re-added. Clients will see the object jump between z-indices visually. Workaround: Only change layer during controlled transitions (e.g., entering/exiting buildings); avoid mid-tick layer mutations.

### 6. **Update Order Matters for Same-Tick Interactions**
In `Game.tick()`, objects are updated in strict order:
1. Loot, Parachute, Projectile, SyncedParticle
2. Bullets (collision detection)
3. Damage application
4. Players

If a bullet hits an object on the same tick that object's update() runs, the hit is recorded but damage is deferred until after all updates. This prevents sync issues where a destroyed object receives damage from bullets that passed through it.

### 7. **Emotes Are Not GameObjects**
`Emote` is a simple data holder, not a `BaseGameObject`. It has no hitbox, no grid entry, no dirty flags. Emotes are added to `game.emotes[]` and serialized directly in the player's data packet. They're cleaned up in `postPacket()`.

### 8. **IDAllocator Recycles Synchronously**
When an object is destroyed via `removeObject()`, its ID is **immediately** returned to the IDAllocator:

```typescript
game.removeObject(obj);
// obj.id is now available for reuse by the next object
```

If the client has not yet received the delete packet, reusing the ID can cause desync. This is an edge case that rarely occurs if update packets are sent reliably.

### 9. **Dirty Flags Cleared Every Tick**
`fullDirtyObjects` and `partialDirtyObjects` are cleared at the end of `Game.tick()`. If an object mutates after its dirty flag is cleared but before the next tick, the mutation is **NOT** synced. Objects must re-mark themselves dirty during their `update()` method. Any property changes after `update()` returns are lost.

### 10. **Map Bounds Enforcement for SyncedParticles**
`SyncedParticle` clamps position to `[0, GameConstants.maxPosition]` in the constructor. Particles that move outside the map bounds are clamped back in. This prevents out-of-bounds rendering but can cause jerky motion near edges.

### 11. **Explosion Raycast Has Ray Density Limits**
Explosions use a ray-cast algorithm with variable ray density based on `explosionRayDistance` constant. High-damage explosions with small radius may miss some objects due to sparse rays. The algorithm trades accuracy for CPU cost.

### 12. **Loot Despawn Timer Is Relative to Game Time**
Despawn timers are relative to `game.now` (milliseconds since game start), **not** wall-clock time. If the server's tick rate fluctuates, despawn timing may drift. In practice, with consistent 40 TPS, this is negligible.

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — System overview, tech stack, component map
- [Data Model](../../datamodel.md) — ObjectCategory enum, NetworkID type, serialization format

### Tier 2 — Subsystems
- [Game Loop](../game-loop/README.md) — Orchestrates object updates, timing
- [Spatial Grid](../spatial-grid/README.md) — Collision queries, object pooling
- [Inventory](../inventory/README.md) — Item management, player equipment
- [Networking](../networking/README.md) — Binary packet protocol, UpdatePacket serialization
- [Object Definitions](../object-definitions/README.md) — Definition registry, reification
- [Map Generation](../map/README.md) — Map structure, obstacle spawning, building placement
- [Plugin System](../plugins/README.md) — Event system, server extensibility
- [Client Rendering](../client-rendering/README.md) — Client-side object pool, visual state sync

### Tier 3 — Modules (To Be Created)
- `modules/player.md` — Player entity, movement, damage, inventory, actions
- `modules/obstacle.md` — Destructible obstacles, loot drops, health
- `modules/bullet.md` — Ballistic projectiles, collision, damage records
- `modules/projectile.md` — Grenades, physics, fuse, impact
- `modules/loot.md` — Item drops, collection, despawn
- `modules/synced-particles.md` — Networked effects, animations, lifetime

### Patterns
- (To Be Created) `patterns.md` — Common object patterns (damage flow, state mutations, serialization)
