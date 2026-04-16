# Object Lifecycle

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/objects/gameObject.ts, server/src/game.ts -->

## Purpose

Documents the lifecycle of server-side game objects from creation through network serialization to final destruction. Covers object registration, network ID allocation, dirty flag tracking, and cleanup.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/gameObject.ts` | `BaseGameObject` abstract class hierarchy | High |
| `server/src/game.ts` | Object creation methods (`addObject`, `deleteObject`, tick loop) | High |
| `server/src/utils/grid.ts` | Spatial grid registration, position updates | Medium |
| `common/src/utils/objectsSerializations.ts` | Serialization schemas per category | High |

## Object Creation & Registration

### Step 1: Object Instantiation

All server game objects inherit from `BaseGameObject`:

```typescript
// server/src/objects/gameObject.ts
export abstract class BaseGameObject<Cat extends ObjectCategory = ObjectCategory> {
    declare readonly abstract type: Cat;
    
    readonly id: number;              // Unique network ID (uint16)
    readonly game: Game;
    
    abstract readonly fullAllocBytes: number;   // Max bytes for full serialization
    abstract readonly partialAllocBytes: number; // Max bytes for delta serialization
    
    _position: Vector;
    _rotation: number;
    damageable: boolean;
    dead: boolean;
}
```

**Concrete types:**

```
BaseGameObject
├── Player       (extends to add health, inventory, animation)
├── Obstacle     (building, wall, static props)
├── Building     (door, constructible, interactable)
├── Loot         (weapon, ammo, healing item on ground)
├── Projectile   (grenade, bullet after initial creation)
├── Parachute    (falling loot from airdrop)
├── DeathMarker  (tombstone at death location)
├── Decal        (blood splash, scorch mark, footprint)
└── SyncedParticle (effect visible to all players)
```

```typescript
// Example: Creating a player
const player = new Player(
    game,
    { x: 100, y: 200 },  // position
    gameData                // socket, name, loadout, etc.
);
```

### Step 2: Register in Game

```typescript
// server/src/game.ts addObject()
addObject(object: BaseGameObject): void {
    // Allocate network ID (uint16, incremented)
    object.id = this.idAllocator.next();
    
    // Register in Grid for spatial queries
    this.grid.addObject(object);
    
    // Queue for network broadcast
    this.fullDirtyObjects.add(object);  // Mark for full serialization
    
    // Track by category if needed
    if (object.type === ObjectCategory.Player) {
        this.livingPlayers.add(object as Player);
    }
    if (object.type === ObjectCategory.Projectile) {
        this.bullets.add(object as Bullet);
    }
    
    // Plugin hook
    this.pluginManager.emit("object_added", { object });
}
```

### Network ID Allocation

**File:** `server/src/utils/idAllocator.ts` (inferred)

```typescript
class IDAllocator {
    private _nextId = 0;
    
    next(): number {
        return this._nextId++;  // uint16, wraps at 2^16
    }
    
    free(id: number): void {
        // Optional: recycle IDs (most implementations don't)
    }
}
```

IDs are **never reused** in a single game session (sequential, monotonically increasing).

## Category-Based Object Types

Objects are typed at the packet layer via `ObjectCategory` enum:

```typescript
// common/src/constants.ts
export enum ObjectCategory {
    Player = 0,
    Obstacle = 1,
    DeathMarker = 2,
    Loot = 3,
    Building = 4,
    Decal = 5,
    Parachute = 6,
    Projectile = 7,
    SyncedParticle = 8
}
```

Each category has its own serialization schema:

```typescript
// common/src/utils/objectsSerializations.ts
const ObjectSerializations = {
    [ObjectCategory.Player]: { /* position, rotation, health, animation, etc */ },
    [ObjectCategory.Obstacle]: { /* position, rotation, scale */ },
    [ObjectCategory.Loot]: { /* position, rotation, item type */ },
    [ObjectCategory.Building]: { /* position, rotation, health, variation */ },
    // ... etc
};
```

## Dirty Flag Tracking

The server tracks which objects changed since last UPDATE packet to minimize bandwidth:

### Mutable vs Immutable

```typescript
// server/src/game.ts
public readonly partialDirtyObjects = new Set<BaseGameObject>();
public readonly fullDirtyObjects = new Set<BaseGameObject>();

tick(): void {
    // Clear dirty flags from previous tick
    this.partialDirtyObjects.clear();
    this.fullDirtyObjects.clear();
    
    // ... simulate
    
    // Objects modified this tick are added during simulation:
    player.position = newPos;  // Setter marks as partialDirtyObjects
    player.health = newHealth; // Setter marks as partialDirtyObjects
    
    //... build UPDATE packet from dirty sets
}
```

### Partial vs Full Serialization

**Partial dirty:** Object only has minor changes (position, rotation, animation):

```typescript
// ~2-10 bytes
{
    objectId: 42,
    position: [newX, newY],     // 4 bytes (compressed Vector)
    rotation: newRot,            // 2 bytes (angle)
    // Omit: health, inventory, skin, etc (unchanged)
}
```

**Full dirty:** Object is new or has major state change:

```typescript
// ~40-100 bytes
{
    objectId: 42,
    position: [newX, newY],
    rotation: newRot,
    health: 100,
    skin: "hazel_jumpsuit",
    animation: "running",
    inventory: [ { item: "M16A4", ammo: 120 }, ... ],
    // All fields serialized
}
```

### When Objects are Marked

```typescript
// server/src/objects/player.ts (example)
set position(pos: Vector) {
    this._position = pos;
    this.game.grid.updateObject(this);
    this.game.partialDirtyObjects.add(this);  // ← Mark dirty
}

health = 100;
takeDamage(amount: number): void {
    this.health -= amount;
    this.game.partialDirtyObjects.add(this); // ← Mark dirty
    
    if(this.health <= 0) {
        this.kill();  // Calls deleteObject
    }
}
```

## Object Serialization

### Full Serialization Buffers

Each object pre-allocates buffers for both full & partial serialization:

```typescript
// server/src/objects/gameObject.ts
abstract class BaseGameObject {
    readonly abstract fullAllocBytes: number;
    readonly abstract partialAllocBytes: number;
    
    private _fullStream?: SuroiByteStream;
    get fullStream(): SuroiByteStream {
        return this._fullStream ??= new SuroiByteStream(
            new ArrayBuffer(this.fullAllocBytes)
        );
    }
    
    private _partialStream?: SuroiByteStream;
    get partialStream(): SuroiByteStream {
        return this._partialStream ??= new SuroiByteStream(
            new ArrayBuffer(this.partialAllocBytes)
        );
    }
}
```

Example allocations:

| Category | Full Bytes | Partial Bytes |
|----------|-----------|--------------|
| Player | 256 | 32 |
| Obstacle | 64 | 16 |
| Loot | 48 | 12 |
| Building | 96 | 24 |
| Projectile | 48 | 12 |

### Serialization in UpdatePacket

```typescript
// common/src/packets/updatePacket.ts

// For each dirty object in fullDirtyObjects:
for (const obj of game.fullDirtyObjects) {
    const stream = obj.fullStream;
    stream.index = 0;  // Reset write position
    
    // Write: id, category, then category-specific data
    stream.writeObjectId(obj.id);
    stream.writeObjectType(obj.type);
    
    // Category-specific serialization
    ObjectSerializations[obj.type].serialize(stream, obj);
    
    // Compress to UPDATE packet
    fullObjectsBuffer.append(stream.buffer.slice(0, stream.index));
}

// For each dirty object in partialDirtyObjects:
for (const obj of game.partialDirtyObjects) {
    const stream = obj.partialStream;
    stream.index = 0;
    
    stream.writeObjectId(obj.id);
    ObjectSerializations[obj.type].serializePartial(stream, obj);
    
    partialObjectsBuffer.append(stream.buffer.slice(0, stream.index));
}
```

### Per-Category Serialization

Player:

```typescript
ObjectCategory.Player: {
    serialize: (stream, player) => {
        stream.writeVector(player.position, 0, 0, 1924, 1924, 2);
        stream.writeRotation(player.rotation);
        stream.writeFloat32(player.health);
        stream.writeInt8(player.zoom);
        stream.writeString(player.name);
        stream.writeUint32(player.skin);
        // ... animation state, affected debuffs, etc
    },
    serializePartial: (stream, player) => {
        // Only position, rotation if changed
        stream.writeVector(player.position, 0, 0, 1924, 1924, 2);
        stream.writeRotation(player.rotation);
    }
}
```

Obstacle:

```typescript
ObjectCategory.Obstacle: {
    serialize: (stream, obs) => {
        stream.writeVector(obs.position, 0, 0, 1924, 1924, 2);
        stream.writeRotation(obs.rotation);
        stream.writeFloat32(obs.scale);
        stream.writeUint8(obs.variation);  // Differ by material, rotation, etc
    },
    serializePartial: (stream, obs) => {
        // Obstacles rarely move; only position/rotation if needed
        stream.writeVector(obs.position, 0, 0, 1924, 1924, 2);
    }
}
```

## Object Destruction & Cleanup

### Step 1: Mark for Deletion

```typescript
// server/src/game.ts
deleteObject(object: BaseGameObject): void {
    if (object.dead) {
        this.warn(`[Game] Tried to delete already-dead object: ${ObjectCategory[object.type]}`);
        return;
    }
    
    object.dead = true;
    this.deletedObjects.push(object.id);  // Queue deletion packet
    this.grid.removeObject(object);       // Remove from spatial grid
    
    // Category-specific cleanup
    if (object.type === ObjectCategory.Player) {
        const player = object as Player;
        this.livingPlayers.delete(player);
        this.connectedPlayers.delete(player);
        if (player.socket) player.socket.close();
    }
    
    if (object.type === ObjectCategory.Projectile) {
        this.bullets.delete(object as Bullet);
    }
    
    // Plugin hook
    this.pluginManager.emit("object_deleted", { object });
}
```

### Step 2: Broadcast Deletion

In next UPDATE packet, add to `deletedObjects` array:

```typescript
const updatePacket = {
    type: PacketType.Update,
    // ... playerData, globalData, dirty objects ...
    deletedObjects: [42, 87, 123]  // Remove these object IDs client-side
};
```

Client-side:

```typescript
// client/src/scripts/game.ts
case PacketType.Update:
    for (const objectId of update.deletedObjects) {
        this.objects.delete(objectId);  // Remove from ObjectPool
        const obj = this.objects.get(objectId);
        if (obj) obj.destroy();  // Clean up Pixi container, sounds, etc
    }
    break;
```

### Step 3: Memory Reclamation

```typescript
// server/src/game.ts/Grid.removeObject()
removeObject(object: GameObject): void {
    this._removeFromGrid(object);
    this.pool.delete(object);  // Remove from ObjectPool
    // Object is now eligible for garbage collection (no refs held)
}
```

Java/TypeScript GC collects `BaseGameObject` once all references freed:

1. Removed from `Grid.pool`
2. Removed from category-specific set (`livingPlayers`, `bullets`, etc.)
3. No dangling references in tick loop

## Complex Functions

### `grid.updateObject(object)` — @file server/src/utils/grid.ts:53

**Purpose:** Refresh object's grid cell occupancy after position change.

**Algorithm:**
1. Remove object from old cells
2. Calculate new cells object occupies based on hitbox
3. Add object to new cells
4. Cache list of cell positions for fast removal

**Why needed:** Objects move every tick. Grid must track new position to enable O(1) spatial queries.

### `game.addObject(object)` — @file server/src/game.ts

**Purpose:** Register object, allocate network ID, track for updates.

**Why needed:** Server must track every object for simulation updates and network broadcasts.

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Game objects subsystem overview
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — System architecture
- **Tier 3:** [../../spatial-grid/modules/query-optimization.md](../../spatial-grid/modules/query-optimization.md) — Grid management
- **Networking:** [../../networking/modules/packet-types.md](../../networking/modules/packet-types.md) — Serialization formats
- **Game Loop:** [../../game-loop/README.md](../../game-loop/README.md) — Tick timing and update frequency (40 TPS)
