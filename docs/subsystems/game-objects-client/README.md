# Client-Side Game Objects

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/game-objects-client/modules/ -->
<!-- @source: client/src/scripts/objects/ -->

## Purpose

Client-side renderable representations of server-side entities. This subsystem manages all visual game objects that the player sees on screen — from players and obstacles to projectiles and loot. State arrives via `UpdatePacket` delta messages; this subsystem syncs that state to PixiJS sprites, handles interpolation between updates, and manages object pooling for efficient memory use.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/game.ts` | Game singleton; `ObjectPool<ObjectMapping>` holds active objects; `processUpdate()` creates/updates/deletes objects from UpdatePacket |
| `client/src/scripts/objects/gameObject.ts` | Abstract base class for all renderable objects; position/rotation interpolation; container lifecycle |
| `client/src/scripts/objects/player.ts` | Player sprite, animations, armor indicator, damage feedback, team indicator |
| `client/src/scripts/objects/obstacle.ts` | Obstacle sprite, destruction animation, door animation, smoke particles |
| `client/src/scripts/objects/building.ts` | Building sprite, ceiling toggle (raycast-based), door animation, puzzle state |
| `client/src/scripts/objects/bullet.ts` | Tracer sprite for bullet trails; client-side only (not in ObjectPool) |
| `client/src/scripts/objects/projectile.ts` | Throwable item sprite, flicker animation, height-based z-index |
| `client/src/scripts/objects/loot.ts` | Loot sprite with item background, bobbing animation, count display |
| `client/src/scripts/objects/plane.ts` | Airdrop plane sprite, flight path animation |
| `client/src/scripts/objects/deathMarker.ts` | Tombstone sprite at player death location |
| `client/src/scripts/objects/decal.ts` | Persistent visual decal (blood splatter, bullet hole) |
| `client/src/scripts/objects/parachute.ts` | Parachute sprite during player landing |
| `client/src/scripts/objects/syncedParticle.ts` | Network-synced particle system (e.g., explosion smoke) |
| `client/src/scripts/utils/pixi.ts` | `SuroiSprite` helper; texture management; spritesheet loading (hi-res 4096×4096 or lo-res 2048×2048) |
| `client/src/scripts/managers/cameraManager.ts` | Layer containers (basement, ground, upstairs); depth sorting; layer transitions |

## Architecture

### Object Pool System

Client maintains an `ObjectPool<ObjectMapping>` in the Game singleton to efficiently reuse object allocations:

```typescript
// Game.ts line 558
readonly objects = new ObjectPool<ObjectMapping>();

// ObjectPool methods
objects.get(id: number): GameObject | undefined        // retrieve by ID
objects.add(object: GameObject): void                   // add/register
objects.delete(object: GameObject): void                // remove/release
objects.getCategory(category: ObjectCategory): GameObject[]  // query by type
```

The ObjectPool holds active game object instances. When a new object is needed, it's either reused from a freed slot or allocated fresh. This prevents allocation churn in long matches.

### Object Lifecycle: Create → Update → Delete

#### 1. Creation (UpdatePacket with fullDirtyObjects)

```
UpdatePacket received
  ↓
Game.processUpdate(packet.fullDirtyObjects)
  ↓
for each {id, type, data}:
  if object doesn't exist:
    ├─ new ObjectClassMapping[type](id, data)
    ├─ this.objects.add(object)
    └─ object.updateLayer() → add container to stage
  else:
    └─ object.updateFromData(data, false)
```

**ObjectClassMapping** is a typed registry that maps `ObjectCategory` → Type:

```typescript
// Game.ts lines 67-90
const ObjectClassMapping = {
    [ObjectCategory.Player]: Player,
    [ObjectCategory.Obstacle]: Obstacle,
    [ObjectCategory.DeathMarker]: DeathMarker,
    [ObjectCategory.Loot]: Loot,
    [ObjectCategory.Building]: Building,
    [ObjectCategory.Decal]: Decal,
    [ObjectCategory.Parachute]: Parachute,
    [ObjectCategory.Projectile]: Projectile,
    [ObjectCategory.SyncedParticle]: SyncedParticle
} satisfies Record<ObjectCategory, constructor>
```

All 9 object categories are created this way. Each constructor receives `(id: number, data: ObjectsNetData[Category])` and calls `updateFromData(data, true)` internally.

#### 2. Partial Update (UpdatePacket with partialDirtyObjects)

```
UpdatePacket received
  ↓
for each {id, data}:
  object = objects.get(id)
  object.updateFromData(data, false)
```

Partial updates apply deltas (position, rotation, health, etc.) without resetting the full state.

#### 3. Deletion (UpdatePacket with deletedObjects)

```
UpdatePacket received
  ↓
for each deletedId:
  object = objects.get(deletedId)
  object.destroy()
    ├─ kill all timeouts
    ├─ destroy container (removes from stage)
    └─ cleanup sprite, graphics, audio
  objects.delete(object)
```

### Rendering Pipeline: Stages → Containers → Layers

Objects are PixiJS Sprites/Graphics in a depth-sorted layer system:

```
PixiJS Application (WebGL)
  ↓
CameraManager.container
  ├─ layerContainers[0] (Basement — Layer.Basement)
  ├─ layerContainers[1] (Ground — Layer.Ground)
  ├─ layerContainers[2] (Upstairs — Layer.Upstairs)
  └─ tempLayerContainer (transient during layer changes)
```

Each object's `container` (a PixiJS Container) is added to the appropriate layer based on its `layer` property:

```typescript
// gameObject.ts
updateLayer(forceUpdate = false): void {
    // Determine target container for this layer
    const newContainer = CameraManager.getContainer(this.layer, this.layerContainerIndex);
    
    // Handle layer transitions (fade in/out ceiling when moving upstairs)
    if (oldContainer !== newContainer) {
        oldContainer?.removeChild(this.container);
        newContainer.addChild(this.container);
    }
}
```

**Layer transitions** (e.g., entering a building's upstairs) trigger a ~800ms fade effect to smoothly blend the container visibility.

### Position & Rotation Interpolation

UpdatePacket arrives ~25–40× per second (one every 25–40 ms). PixiJS renders unconstrained by tick rate. To prevent jittery visual movement, client interpolates between the last known position and the new one:

```typescript
// gameObject.ts lines 24-49
set position(position: Vector) {
    if (this._positionManuallySet) {
        this._oldPosition = Vec.clone(this._position);  // save previous for lerp
    }
    this._lastPositionChange = Date.now();
    this._position = position;
}

updateContainerPosition(): void {
    if (!this._oldPosition) return;
    
    // Lerp from old to new, clamped to [0, 1]
    const t = Math.min((Date.now() - this._lastPositionChange) / Game.serverDt, 1);
    this.container.position = toPixiCoords(
        Vec.lerp(this._oldPosition, this._position, t)
    );
}
```

**Game.serverDt** is the measured time between consecutive UpdatePackets (typically ~25 ms). Interpolation runs continuously during `Game.update()` beforePixiJS renders.

Rotation interpolation uses `Angle.minimize()` to avoid spinning the long way around.

### Sprite Management: SuroiSprite & Textures

All sprites are instances of `SuroiSprite` (extends PixiJS Sprite):

```typescript
// pixi.ts lines 68+
export class SuroiSprite extends Sprite {
    static getTexture(frame: string): Texture {
        if (!Assets.cache.has(frame)) {
            console.warn(`Texture not found: "${frame}"`);
            frame = "_missing_texture";
        }
        return Texture.from(frame);
    }

    constructor(frame?: string) {
        super(frame ? SuroiSprite.getTexture(frame) : undefined);
        this.anchor.set(0.5);  // pivot at center
    }
}
```

**Texture loading:**
- Spritesheets are loaded on game start (`loadSpritesheets()`)
- **High resolution**: 4096×4096 spritesheets (loaded if device supports MAX_TEXTURE_SIZE >= 4096)
- **Low resolution**: 2048×2048 spritesheets (fallback for older hardware)
- Textures are registered in PixiJS Assets cache keyed by frame name (e.g., `"player_base"`, `"ak47"`)

### Object Type Catalog

Each object type is a subclass of `GameObject` created via the `.derive(ObjectCategory.X)` mixin pattern:

```typescript
// player.ts
export class Player extends GameObject.derive(ObjectCategory.Player) {
    // → creates Player AND sets this.type = ObjectCategory.Player
    // → adds this.isPlayer = true
}
```

This ensures type safety for the ObjectPool and enables type-specific behavior:

| Category | Class | Purpose |
|----------|-------|---------|
| `Player` | `Player` | 35 props (position, rotation, health, armor, activeItem, animation state); emote rendering |
| `Obstacle` | `Obstacle` | destruction state, door animation, leaf/mount particles |
| `Loot` | `Loot` | item definition, count (ammo stacks), bobbing tween |
| `Building` | `Building` | ceiling visibility toggle, puzzle state, door/wall graphics |
| `Projectile` | `Projectile` | throwable item in flight; flicker; height-based z-index |
| `DeathMarker` | `DeathMarker` | static tombstone at death location |
| `Decal` | `Decal` | blood splatter, bullet hole; fade-out animation |
| `Parachute` | `Parachute` | visual during landing; deleted when landed |
| `SyncedParticle` | `SyncedParticle` | explosion smoke, muzzle flash; position/scale/alpha animations |

**Not in ObjectPool:**
- **Bullets** (`Game.bullets: Set<Bullet>`) — tracer line rendering; client-side only; not synced via UpdatePacket
- **Planes** (`Game.planes: Set<Plane>`) — airdrop plane sprite; special lifecycle outside ObjectPool

## Interfaces & Contracts

### GameObject Base Interface

Every object type must implement:

```typescript
abstract class GameObject<Cat extends ObjectCategory = ObjectCategory> {
    readonly id: number;
    readonly type: Cat;
    destroyed: boolean;
    
    position: Vector;    // with interpolation support
    rotation: number;    // with interpolation support
    layer: Layer;
    
    readonly container: Container;  // PixiJS container for this object's sprites/graphics
    
    abstract updateFromData(data: ObjectsNetData[Cat], isNew: boolean): void;
    abstract update(): void;
    abstract updateInterpolation(): void;  // called each PixiJS frame to interpolate position/rotation
    abstract updateDebugGraphics(): void;  // for hitbox visualization
    
    destroy(): void;
    playSound(name: string, options?: SoundOptions): GameSound;
    updateLayer(forceUpdate?: boolean): void;
}
```

### updateFromData(data, isNew)

Called when the object receives a state update from the server:

```typescript
override updateFromData(data: ObjectsNetData[ObjectCategory.Player], isNew = false): void {
    if (data.full) {
        // Apply full state (position, rotation, health, inventory, etc.)
        // Typically only on creation (isNew === true)
        this.position = data.full.position;
        this.rotation = data.full.rotation;
        this.health = data.full.health;
        // ...and update sprites
    }
    if (data.partial) {
        // Apply deltas only
        if (data.partial.position !== undefined) this.position = data.partial.position;
        if (data.partial.rotation !== undefined) this.rotation = data.partial.rotation;
        // ...
    }
}
```

## Dependencies

### Depends on:

- **[Networking](../networking/)** — `UpdatePacket` structure, binary deserialization, ObjectsNetData union types
- **[Object Definitions](../object-definitions/)** — texture lookups via ObjectDefinitions registry, definition data (BuildingDefinition, ObstacleDefinition, etc.)
- **[Client Rendering](../client-rendering/)** — PixiJS Application, stage, layer management via CameraManager
- **Math & Utils** — position interpolation (Vec.lerp), rotation minimize (Angle.minimize), hitbox collision (CircleHitbox, etc.)

### Depended on by:

- **[Input Management](../input-management/)** — objects queried for hover/click detection
- **[Client Rendering](../client-rendering/)** — objects' containers rendered each frame; layer transitions managed
- **[Sound Management](../sound-management/)** — position-based sounds from objects
- **[Particle System](../particle-system/)** — particle emitters attached to objects (obstacle destruction, damage pops)

## Known Issues & Gotchas

1. **No client-side state lag compensation** — visual state lags behind server by 1 UpdatePacket interval (25–40 ms). Interpolation smooths this but cannot predict server physics. Quick server actions (explosion, gunfire) always arrive after visual indication.

2. **Sprite popping on layer transitions** — no smooth fade when an object moves between indoor/outdoor layers; container visibility changes instantly. Layering uses `tempLayerContainer` + fade tween as a workaround but can still appear jarring at edges.

3. **Object pool allocation stalls** — garbage collector can stall the render loop when reusing/allocating hundreds of objects (common in large matches with many players). No preallocated object pool; allocations happen on-demand.

4. **Ghost objects if DeletePacket lost** — if `deletedObjects` is lost in network transmission, object lingers in ObjectPool until player disconnect or match end. No cleanup timeout.

5. **Stale textures mid-game** — if a definition changes server-side (e.g., weapon skin swap), client still shows old texture from spritesheets cache. Requires reconnect to load updated assets.

6. **Position interpolation breaks on layer change** — when an object moves to a different layer (e.g., player goes upstairs), interpolation may jump because layer transition logic doesn't preserve velocity. Object can briefly teleport.

7. **No occlusion culling** — all objects render even if off-screen or blocked by buildings. On large maps with many entities, this causes GPU overhead.

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — PixiJS rendering pipeline, client-server update loop
- [Data Model](../../datamodel.md) — UpdatePacket structure, ObjectsNetData serialization format

### Tier 2 — Other Subsystems
- [Networking](../networking/) — UpdatePacket encoding/decoding, packet types
- [Client Rendering](../client-rendering/) — PixiJS initialization, layer/container management, stage structure
- [Object Definitions](../object-definitions/) — sprite texture registry, definition lookups
- [Input Management](../input-management/) — object hover/click detection
- [Sound Management](../sound-management/) — position-based audio from objects
- [Particle System](../particle-system/) — particle emitters attached to objects

### Tier 3 (Modules)
- [modules/player.md](modules/player.md) — player state sync, animations, damage feedback
- [modules/buildings.md](modules/buildings.md) — building ceiling/wall visibility logic
- [modules/obstacles.md](modules/obstacles.md) — obstacle destruction, door mechanics
- [modules/interpolation.md](modules/interpolation.md) — position/rotation interpolation algorithm

### Patterns
- [patterns.md](patterns.md) — Object pool allocation, interpolation, sprite helpers
