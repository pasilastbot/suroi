# Q22: How do I add a completely new weapon type (e.g., a deployable turret)?

**Answer:**

Suroi's current architecture assumes weapons fall into one of three categories: **Gun** (ammo-consuming firearms), **Melee** (close-range attacks), or **Throwable** (cook-and-throw explosives). A deployable turret is fundamentally different and requires architectural changes.

## Current Weapon Architecture

**Current types** are defined in `DefinitionType` enum:

```typescript
export enum DefinitionType {
    Gun,        // Ammo-consuming projectile launcher
    Melee,      // Hitbox-based close-range attack
    Throwable   // Cooked grenade / explosives
}
```

These are **inventory item categories**, stored in defined slots:

```typescript
[DefinitionType.Gun, DefinitionType.Gun, DefinitionType.Melee, DefinitionType.Throwable]
//       Slot 0              Slot 1            Slot 2             Slot 3
```

**Key assumption:** Weapons are **always player-held**, active on demand, and consume ammo or have cooldown timers.

## Why Deployables Break This Model

A **deployable turret** breaks the assumption because:
- ❌ Spawns a **server-side game object** instead of firing from player
- ❌ Persists **independently** after deployment
- ❌ Has its own **health, targeting, fire loop**
- ❌ Doesn't fit the "player-held weapon" model

## Step-by-Step Process

### Phase A: Architecture Decisions

Before coding, decide:

1. **Is it active or passive?**
   - Active: Player places/deactivates manually
   - Passive: Place-and-forget; turret handles targeting

2. **How many can a player deploy?**
   - Single per player (replaces previous)
   - Multiple (limited by count)
   - Unlimited (cost is time/ammo)

3. **Inventory interaction?**
   - Option A: Hold in slot 3 (like Throwable) → `useItem()` to place
   - Option B: Separate deployer tool
   - Option C: Not in inventory (passive ability via perk)

4. **Server scope:**
   - New `ObjectCategory`?
   - or specialized `Projectile` variant?

### Phase B: Define the Deployable Item Class

For **Option A** (simplest: hold as Throwable-like), create:

```typescript
// In server/src/inventory/deployableItem.ts

export class DeployableItem extends InventoryItemBase {
    readonly category = DefinitionType.Deployable;
    
    constructor(owner: Player, definition: DeployableDefinition) {
        super(owner, definition);
    }
    
    _useItemNoDelayCheck(skipAttackCheck: boolean): void {
        if (!skipAttackCheck || this.owner.dead || this.owner.downed) return;
        
        const placement = this.getValidPlacement();
        if (!placement) return;
        
        // Create turret object on server
        const turret = new Turret(this.owner.game, this.definition, placement);
        this.owner.game.grid.addObject(turret);
        turret.setDirty();
        
        if (this.definition.maxPerPlayer === 1) {
            if (this.owner.turret) {
                this.owner.turret.destroy();
            }
            this.owner.turret = turret;
        }
    }
    
    getValidPlacement(): Vector | null {
        // Ray-cast from player towards cursor
        // Check: no obstacle collision, no building overlap, ground layer
        // Return placement position or null
    }
}
```

### Phase C: Define DeployableDefinition

```typescript
export interface DeployableDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Deployable
    readonly deploymentRange: number       // Max placement distance
    readonly placementRadius: number       // Hitbox size when placed
    readonly health: number
    readonly maxPerPlayer?: number         // 1 = single active per player
    readonly fireMode: FireMode
    readonly fireDelay: number
    readonly bulletCount: number
    readonly targetingRange: number        // Max fire distance
    readonly rotationSpeed: number         // degrees/second
}
```

## Code Changes Required

| Layer | File(s) | Changes |
|-------|---------|---------|
| **Definitions** | `common/src/definitions/items/deployables.ts` (NEW) | Create `Deployables` registry |
| **DefinitionType** | `common/src/utils/objectDefinitions.ts` | Add `Deployable` enum |
| **InventoryItem** | `server/src/inventory/inventoryItem.ts` | Mark `DeployableItem` as derived |
| **Deployable class** | `server/src/inventory/deployableItem.ts` (NEW) | Implement placement, targeting |
| **Turret object** | `server/src/objects/turret.ts` (NEW) | Game object, update, collision |
| **ObjectCategory** | `common/src/constants.ts` | Add `Turret` to enum |
| **Serialization** | `common/src/utils/objectsSerializations.ts` | Add `TurretData` schema |
| **Client renderer** | `client/src/scripts/objects/turret.ts` (NEW) | Render sprite, health bar |
| **Loots** | `common/src/definitions/loots.ts` | Add Deployables to loot drops |

## ObjectDefinitions Registration

Create the registry in `common/src/definitions/items/deployables.ts`:

```typescript
export const Deployables = new ObjectDefinitions<DeployableDefinition>([
    {
        idString: "sentry_turret",
        name: "Sentry Turret",
        defType: DefinitionType.Deployable,
        deploymentRange: 50,
        placementRadius: 2.5,
        health: 100,
        maxPerPlayer: 1,
        fireMode: FireMode.Auto,
        fireDelay: 75,
        bulletCount: 1,
        targetingRange: 100,
        rotationSpeed: 180,
    },
    {
        idString: "flame_turret",
        name: "Mounted Flamethrower",
        // ... similar structure
    }
]);
```

Register in loots:

```typescript
export const Loots = new ObjectDefinitions<LootDefinition>([
    ...Guns.definitions,
    ...Melees.definitions,
    ...Throwables.definitions,
    ...Deployables.definitions,  // NEW
]);
```

## Client-Side Rendering

Create `client/src/scripts/objects/turret.ts`:

```typescript
export class Turret extends GameObject implements SuroiObjectWithHealth {
    private _sprite: Sprite;
    private _healthBar: Sprite;
    
    readonly health = 100;
    readonly maxHealth = 100;
    
    constructor(id: number) {
        super(id);
        this._sprite = new Sprite(TextureManager.getTexture("sentry_turret"));
        this._sprite.anchor.set(0.5);
        this.container.addChild(this._sprite);
    }
    
    updateFromNetworkData(data: DeployableObjectData): void {
        this.position = data.position;
        if (data.rotation !== undefined) {
            this._sprite.rotation = data.rotation;
        }
        if (data.health !== undefined) {
            this.health = data.health;
            this._updateHealthBar();
        }
    }
    
    interpolate(nextData: DeployableObjectData, alpha: number): void {
        // Lerp position between frames
        this.position.x = this.position.x + (nextData.position.x - this.position.x) * alpha;
        // Slerp rotation for smooth barrel tracking
        const angleDelta = nextData.rotation - this._sprite.rotation;
        this._sprite.rotation += angleDelta * alpha;
    }
}
```

## Network Synchronization

Define serialization in `common/src/utils/objectsSerializations.ts`:

```typescript
[ObjectCategory.Turret]: {
    serialize(obj: Turret, stream: SuroiByteStream, isFullUpdate: boolean): void {
        if (isFullUpdate) {
            stream.writeObjectId(obj.id);
            stream.writePosition(obj.position);
            stream.writeRotation(obj.rotation);
            stream.writeUint8(obj.health);
        } else {
            // Partial update: only changed fields
            if (obj.dirty.position) stream.writePosition(obj.position);
            if (obj.dirty.rotation) stream.writeRotation(obj.rotation);
            if (obj.dirty.health) stream.writeUint8(obj.health);
        }
    },
    
    deserialize(stream: SuroiByteStream, isFullUpdate: boolean): TurretData {
        if (isFullUpdate) {
            return {
                id: stream.readObjectId(),
                position: stream.readPosition(),
                rotation: stream.readRotation(),
                health: stream.readUint8(),
            };
        }
        // ... partial deserialize
    }
}
```

Turrets serialize as part of `FullObjects` or `PartialObjects` in UpdatePacket, just like any other game object.

## Gameplay Flow

1. **PICKUP:** Player finds turret deployer item
2. **DEPLOYMENT:** Player places at valid location (within range, no overlap)
3. **NETWORK SYNC:** Server sends UpdatePacket with new Turret in fullDirtyObjects
4. **CLIENT SPAWN:** Client deserializes, creates Turret object, renders sprite
5. **ACTIVE:** Server runs Turret.update() every tick (find target, rotate, fire)
6. **PARTIAL UPDATES:** Client receives position/rotation/health changes
7. **DESTRUCTION:** Player deactivates or turret dies; server marks dirty, client removes

## References

- **Tier 2:** [docs/subsystems/inventory/README.md](docs/subsystems/inventory/README.md) — Weapon architecture
- **Tier 2:** [docs/subsystems/object-definitions/README.md](docs/subsystems/object-definitions/README.md) — Definition registry
- **Tier 2:** [docs/subsystems/game-objects-server/README.md](docs/subsystems/game-objects-server/README.md) — Object lifecycle
- **Tier 2:** [docs/subsystems/networking/README.md](docs/subsystems/networking/README.md) — Serialization
- **Source:** [common/src/definitions/items/guns.ts](common/src/definitions/items/guns.ts) — Example weapon definitions
