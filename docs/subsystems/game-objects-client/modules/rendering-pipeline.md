# Rendering Pipeline

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: client/src/scripts/objects/gameObject.ts, client/src/scripts/game.ts -->

## Purpose

Documents how the client transforms server game state into PixiJS render objects. Covers ObjectPool lifecycle, sprite creation, network data updates, deletion, and pooling for reuse.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/game.ts` | Game instance, ObjectPool, message routing | High |
| `client/src/scripts/objects/gameObject.ts` | `GameObject` base class and lifecycle | High |
| `client/src/scripts/objects/*.ts` (13 files) | Concrete renderable classes (Player, Obstacle, Loot, etc.) | Medium |
| `common/src/utils/objectPool.ts` | Generic object pool data structure | Medium |
| `client/src/scripts/managers/textureManager.ts` | Sprite/texture loading and caching | Medium |

## ObjectPool Architecture

The client maintains a `Map<number, GameObject>` indexed by network object ID:

```typescript
// client/src/scripts/game.ts
readonly objects = new ObjectPool<ObjectMapping>();

// ObjectMapping is the type that maps ObjectCategory → Renderable class:
interface ObjectMapping {
    [ObjectCategory.Player]: Player;
    [ObjectCategory.Obstacle]: Obstacle;
    [ObjectCategory.DeathMarker]: DeathMarker;
    [ObjectCategory.Loot]: Loot;
    [ObjectCategory.Building]: Building;
    [ObjectCategory.Decal]: Decal;
    [ObjectCategory.Parachute]: Parachute;
    [ObjectCategory.Projectile]: Projectile;
    [ObjectCategory.SyncedParticle]: SyncedParticle;
}
```

**Operations:**

```typescript
// Add new object from UPDATE packet
const obj = new Player(objectId);  // Construct
this.objects.set(objectId, obj);   // Register

// Retrieve and update
const player = this.objects.get(playerId);
player.updateFromNetworkData(data);

// Delete
const obs = this.objects.get(objectId);
obs.destroy();  // Cleanup Pixi/sounds
this.objects.delete(objectId);
```

**Why ObjectPool:** Avoid HashMap thrashing. UPDATE packets are frequent (40/sec, ~40-200 objects each). O(1) ID lookup is critical for 100+ concurrent objects.

## Sprite Creation & Initialization

### Type-Driven Instantiation

Each `ObjectCategory` maps to a concrete renderable class:

```typescript
// When UPDATE packet arrives with new object:
const objectsData = update.fullDirtyObjects[ObjectCategory.Player];

for (const netData of objectsData) {
    const objectId = netData.id;
    const category = ObjectCategory.Player;
    
    // Determine class from category
    const ObjectClass = objectClassByCategory[category];  // → Player class
    
    // Instantiate with network ID
    const gameObject = new ObjectClass(objectId);
    
    // Register and render
    this.objects.set(objectId, gameObject);
    this.stage.addChild(gameObject.container);  // PixiJS container
}
```

### Sprite Construction (Player Example)

```typescript
// client/src/scripts/objects/player.ts
export class Player extends GameObject {
    private _spriteContainer = new Container();
    
    constructor(id: number) {
        super(id);
        
        // Create main sprite
        this.sprite = new Sprite(TextureManager.getTexture("player_head"));
        this._spriteContainer.addChild(this.sprite);
        
        // Create children (name label, health bar, etc.)
        this.nameLabel = new Text("Alice", { fill: 0xFFFFFF });
        this._spriteContainer.addChild(this.nameLabel);
        
        // Layer in scene graph
        this.container.addChild(this._spriteContainer);
        this.layer = Layer.Ground;  // Z-index group
    }
}
```

### TextureManager Caching

```typescript
// client/src/scripts/managers/textureManager.ts
export class TextureManager {
    private static _textures = new Map<string, Texture>();
    
    static getTexture(key: string): Texture {
        if (!this._textures.has(key)) {
            // Load from spritesheet (Vite-packed 4096×4096 or 2048×2048)
            const tex = Texture.from(await import(`../../../assets/sprites/${key}.png`));
            this._textures.set(key, tex);
        }
        return this._textures.get(key)!;
    }
}
```

## Update from Network Data

### Delta Decoding

When UPDATE packet arrives with partial or full data:

```typescript
// client/src/scripts/game.ts
onMessage(message: MessageEvent<ArrayBuffer>): void {
    const stream = new PacketStream(message.data);
    const update = stream.deserialize();
    
    if (update.type === PacketType.Update) {
        // Update player state
        if (update.playerData) {
            this.updatePlayerData(update.playerData);
        }
        
        // Update full objects (new or major state change)
        for (const category of Object.keys(update.fullDirtyObjects)) {
            const objectsData = update.fullDirtyObjects[category];
            for (const netData of objectsData) {
                let obj = this.objects.get(netData.id);
                
                if (!obj) {
                    // Create new object
                    obj = this.createObject(netData.id, category);
                }
                
                // Apply full state
                obj.updateFromFullData(netData);
            }
        }
        
        // Update partial objects (delta only)
        for (const category of Object.keys(update.partialDirtyObjects)) {
            const objectsData = update.partialDirtyObjects[category];
            for (const netData of objectsData) {
                const obj = this.objects.get(netData.id);
                if (obj) {
                    // Apply only changed fields
                    obj.updateFromPartialData(netData);
                }
            }
        }
        
        // Delete removed objects
        for (const objectId of update.deletedObjects) {
            const obj = this.objects.get(objectId);
            if (obj) {
                obj.destroy();
                this.objects.delete(objectId);
            }
        }
    }
}
```

### Per-Category Update Handlers

**Player update:**

```typescript
// client/src/scripts/objects/player.ts
updateFromFullData(data: PlayerNetData): void {
    this.position = data.position;
    this.rotation = data.rotation;
    this.nameLabel.text = data.name;
    this.nameLabel.style.fill = data.nameColor;
    this.health = data.health;
    this.sprite.texture = TextureManager.getTexture(data.skin);
    this.updateHealth Bar();
}

updateFromPartialData(data: PlayerPartialNetData): void {
    this.position = data.position;  // Smooth interpolation
    this.rotation = data.rotation;
}
```

**Obstacle update:**

```typescript
updateFromFullData(data: ObstacleNetData): void {
    this.position = data.position;
    this.rotation = data.rotation;
    this.scale = data.scale;
    this.sprite.texture = TextureManager.getTexture(
        Obstacles.get(data.definition).texture
    );
}

updateFromPartialData(data: ObstaclePartialNetData): void {
    // Obstacles rarely move; partial update omitted usually
    this.position = data.position;
}
```

### Interpolation

Client-side smooth movement between server updates (40 TPS = 25 ms apart):

```typescript
// client/src/scripts/objects/gameObject.ts
private _oldPosition?: Vector;
private _lastPositionChange?: number;

set position(pos: Vector) {
    if (this._positionManuallySet) {
        this._oldPosition = Vec.clone(this._position);  // Save prior frame
    }
    this._lastPositionChange = Date.now();
    this._position = pos;
}

private updateContainerPosition(): void {
    if (!this._oldPosition) return;
    
    // Interpolate between old and new position
    const elapsed = Date.now() - this._lastPositionChange;
    const t = Math.min(elapsed / Game.serverDt, 1);  // 0 ≤ t ≤ 1
    
    this.container.position = toPixiCoords(
        Vec.lerp(this._oldPosition, this._position, t)
    );
}
```

**Result:** Smooth 60 FPS client animation despite 40 TPS server updates.

## Object Deletion & Pooling

### Destruction Sequence

When UPDATE packet includes object ID in `deletedObjects`:

```typescript
// client/src/scripts/objects/gameObject.ts
destroy(): void {
    this.destroyed = true;
    
    // Stop animations and timers
    for (const timeout of this.timeouts) {
        Game.clearTimeout(timeout);
    }
    this.timeouts.clear();
    
    // Remove event listeners
    this.off("*");  // Remove all listeners
    
    // Stop sounds
    if (this.deathSound) {
        SoundManager.stop(this.deathSound);
    }
    
    // Remove from PixiJS render tree
    if (this.container.parent) {
        this.container.parent.removeChild(this.container);
    }
    
    // Cleanup children
    this.container.destroy({ children: true });
    
    // Mark for pooling (not shown here, but could reuse container)
}
```

### Optional Pooling for Reuse

Production clients **may** reuse destroyed objects to avoid allocation overhead:

```typescript
// client/src/scripts/utils/gameObjectPool.ts (hypothetical)
export class RendererObjectPool {
    private _playerPool: Player[] = [];
    
    getPlayer(id: number): Player {
        let obj = this._playerPool.pop();
        if (!obj) {
            obj = new Player(id);
        } else {
            obj.id = id;
            obj.reset();  // Re-initialize
        }
        return obj;
    }
    
    releasePlayer(player: Player): void {
        player.destroy();
        this._playerPool.push(player);
    }
}
```

**Most implementations don't pool** — V8/SpiderMonkey GC is fast enough.

## Z-Index Assignment

Objects are rendered in **layers** (PixiJS Container grouping):

```typescript
// common/src/constants.ts
export enum Layer {
    Ground = 0,
    Ground1 = 1,
    Beach = 2,
    Water = 3,
    Shallow = 4   // Deeper water
}
```

```typescript
// client/src/scripts/utils/layer.ts
export const GameObjectLayerMap: Record<ObjectCategory, Layer> = {
    [ObjectCategory.Player]: Layer.Ground,
    [ObjectCategory.Obstacle]: Layer.Ground,
    [ObjectCategory.Loot]: Layer.Ground,
    [ObjectCategory.Building]: Layer.Ground1,
    [ObjectCategory.Decal]: Layer.Ground,
    [ObjectCategory.Projectile]: Layer.Ground,
    [ObjectCategory.SyncedParticle]: Layer.Ground,
    [ObjectCategory.Parachute]: Layer.Ground1,
    [ObjectCategory.DeathMarker]: Layer.Ground
};

// In game.ts:
const layerContainer = getLayerContainer(layer);  // Get PixiJS Container for layer
layerContainer.addChild(gameObject.container);     // Add to correct Z-order
```

**Result:** Objects in deeper layers always render behind shallower layers, regardless of rendering order.

## Dirty Flag Handling (Client-Side)

Client-side optimizations minimize re-renders:

```typescript
// client/src/scripts/objects/player.ts
private _healthDirty = false;

set health(value: number) {
    if (this._health === value) return;  // No change
    this._health = value;
    this._healthDirty = true;  // Mark for re-render
}

// In render loop (every frame):
private updateHealthBar(): void {
    if (!this._healthDirty) return;
    
    const ratio = this.health / GameConstants.player.defaultHealth;
    this.healthBar.width = ratio * 50;  // Update PixiJS graphic
    this._healthDirty = false;
}
```

Avoids redundant PixiJS calls (expensive: recomputing graphics, updating texture UV coords).

## Complex Functions

### `gameObject.updateFromFullData(netData)` — @file client/src/scripts/objects/gameObject.ts

**Purpose:** Apply complete state from network packet.

**Algorithm:**
1. Deserialize network-compressed data (vector, rotation, enum IDs)
2. Update all object fields (position, sprite, health, etc.)
3. Trigger visual updates (animations, color, size)
4. Play sounds if state changed substantially

**Why needed:** New objects or major state refreshes must set all properties at once.

### `gameObject.destroy()` — @file client/src/scripts/objects/gameObject.ts

**Purpose:** Clean up PixiJS containers, event handlers, timers before object is freed.

**Algorithm:**
1. Set `destroyed` flag to block further updates
2. Stop animations/timers
3. Remove event listeners
4. Stop sound playback
5. Remove PixiJS Container from parent
6. Recursively destroy all children

**Why needed:** PixiJS Container keeps references. Must explicitly remove from parent tree to enable GC.

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Client-side game objects subsystem
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — System architecture
- **Tier 3:** [../../networking/modules/packet-types.md](../../networking/modules/packet-types.md) — Update packet structure
- **Camera:** [../../camera-management/README.md](../../camera-management/README.md) — View culling and viewport management
- **Managers:** [../../client-managers/README.md](../../client-managers/README.md) — Texture, sound, input managers
