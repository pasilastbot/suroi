# Q10: How does the dirty flag system decide what to include in UpdatePacket?

**Answer:**

The dirty flag system categorizes game object state changes into two sizes: `fullDirtyObjects` (major state changes ~25-50 bytes) and `partialDirtyObjects` (incremental updates ~7-10 bytes), enabling delta encoding to minimize bandwidth.

## FullDirtyObjects vs. PartialDirtyObjects

**Two per-tick sets on the server** that track object state changes:

```typescript
Game.ts:
readonly partialDirtyObjects = new Set<BaseGameObject>();  // incremental
readonly fullDirtyObjects = new Set<BaseGameObject>();     // major changes
```

**Distinction:**
- **`fullDirtyObjects`:** Objects in **major structural change** (newly created, became visible, significant reset). Requires sending **complete state**.
- **`partialDirtyObjects`:** Objects that changed **incrementally** (moved, rotated, animation, action). Requires sending only **changed fields**.

## How Objects Get Marked Dirty

[BaseGameObject](server/src/objects/gameObject.ts):

```typescript
setDirty(): void {
    this.game.fullDirtyObjects.add(this);  // major change
}

setPartialDirty(): void {
    this.game.partialDirtyObjects.add(this);  // incremental change
}
```

**When `setDirty()` (full serialization):**
- Object is **newly created** (spawned loot, new player)
- Object **becomes visible** to client
- Object's **definition changes** (weapon upgrade, armor equip)
- Object's **layer changes** (player goes upstairs)
- **Structural reset** occurs (down→revived, respawn)

**When `setPartialDirty()` (partial serialization):**
- Position changes
- Rotation changes
- Animation state changes
- Action changes (using healing, firing)

## UpdatePacket Composition: Partial vs. Full Fields

### Player Object Example

**Partial Serialization** (7-10 bytes):

```
serializePartial(stream):
  writePosition(position)           // 4 bytes
  writeRotation(rotation)           // 2 bytes
  
  byte = (animationDirty ? 128 : 0) + animation bits + action bits
  writeUint8(byte)                  // 1 byte
  
  if (action === UseItem)
    Loots.writeToStream(item)       // 1-2 bytes
```

**Total: 7-10 bytes**

**Full Serialization** (25-50 bytes):

```typescript
serializeFull(stream, { layer, dead, downed, teamID, 
                        activeItem, skin, helmet, vest, ... }):
  writeLayer(layer)                    // 1 byte
  writeBooleanGroup2(16 flags)         // 2 bytes — dead, downed, beingRevived, etc.
  writeUint8(teamID)                   // 1 byte
  Loots.writeToStream(activeItem)      // 1-2 bytes (weapon)
  if (hasHelmet)
    Armors.writeToStream(helmet)       // 1-2 bytes
  if (hasVest)
    Armors.writeToStream(vest)         // 1-2 bytes
  Backpacks.writeToStream(backpack)    // 1-2 bytes
  // ... more equipment
```

**Total: 25-50 bytes**

### Obstacle Object Example

**Partial Serialization** (2 bytes):

```typescript
serializePartial(stream, { dead, playSound, waterOverlay, powered, scale }):
  writeBooleanGroup(dead, playSound, waterOverlay, powered)  // 1 byte
  writeScale(scale)                                          // 1 byte
```

**Total: 2 bytes**

**Full Serialization** (8-15 bytes):

```typescript
serializeFull(stream, { position, definition, rotation, layer, variation }):
  Obstacles.writeToStream(definition)  // 1-2 bytes
  writePosition(position)              // 4 bytes
  writeLayer(layer)                    // 1 byte
  obstacleData = (variation + rotation + door/activated)  // 1 byte
```

**Total: 8-15 bytes**

## Delta Encoding Strategy

### Conditional Field Serialization

Player state uses `writeBooleanGroup2` (16 bits) header:

```
hasHealth | hasAdrenaline | hasShield | hasZoom | hasLayer | hasTeammates | ...
```

Only fields marked "true" are serialized next.

**Example:** Stationary player not healing:
- Header: health dirty only
- Writes: 2 bytes (header) + 2 bytes (health value)
- Saves: ~16 bytes vs. shipping all fields

### Bit-Packing Dirty Flags

Player partial uses 1 byte for animation + action:

```
Nnnn nccC
N = animation dirty bit
n = animation index (4 bits, 0-15)
c = action type (2 bits, 0-3)
C = action dirty bit
```

Saves 4 bytes vs. writing 4 separate fields.

### Ranged Float Quantization

Instead of 4-byte float:

```typescript
writeFloat(value, minVal, maxVal, numBytes)
```

Maps range [min, max] to integer [0, 2^(8*numBytes)-1]

**Example:** playerSize [0, 4] in 1 byte:
- Value: 2.5
- Quantized: (2.5/4) × 255 = 159
- Wire: 1 byte (saves 3 bytes per float)

## When Full vs. Partial is Sent

**Server decides per tick based on object state:**

```typescript
for (player in visiblePlayers):
    visibleObjects = grid.getVisibleObjects(player.position)
    
    for (object in visibleObjects):
        if (newly visible):
            packet.fullDirtyObjects.add(object)
        else if (partialDirtyObjects.has(object)):
            packet.partialDirtyObjects.add(object)
        else if (fullDirtyObjects.has(object)):
            packet.fullDirtyObjects.add(object)  ← override
    
    for (object in previouslyVisibleObjects):
        if (!visibleObjects.includes(object)):
            packet.deletedObjects.add(object.id)
```

**Critical rule:**
```typescript
if (this.fullDirtyObjects.has(partialObject)) continue;
// Do not send partial AND full for same object in same tick
// Full already includes partial data
```

## Examples with Specific Game Objects

### Player Moving and Firing

| Tick | Event | Method Called | Packet Contents | Bytes |
|------|-------|---------------|-----------------|-------|
| 0 | Player spawns | `setDirty()` | Full: team, skin, armor, backpack | 35-50 |
| 1 | Player moves | `setPartialDirty()` | Partial: position, rotation | 7 |
| 2 | Fires weapon | `setPartialDirty()` | Partial: position, rotation, action | 10 |
| 3 | Still firing | `setPartialDirty()` | Partial: position, rotation, animation | 9 |
| 4 | Health changed | PlayerData section | PlayerData header + health value | 5 |
| 5 | Takes helmet off | `setDirty()` | Full state with hasHelmet=false | 35-45 |

### Loot Item

| Tick | Event | Method Called | Packet Contents | Bytes |
|------|-------|---------------|-----------------|-------|
| 0 | Item drops | `setDirty()` | Full: type, position, definition | 8-15 |
| 1-5 | Item idle | — | Not in packet (not dirty) | 0 |
| 6 | Player picks up | `delete()` | DeletedObjects array | 2 |

### Obstacle (Door/Wall)

| Tick | Event | Method Called | Packet Contents | Bytes |
|------|-------|---------------|-----------------|-------|
| 0 | Map spawns | `setDirty()` | Full: definition, position, rotation | 12-18 |
| 1 | Door locked | `setPartialDirty()` | Partial: state flags + scale | 2 |
| 2-10 | No change | — | Not in packet | 0 |
| 11 | Destroyed | `setPartialDirty()` | Partial: dead = true | 2 |

## Bandwidth Summary

**40 TPS = 25ms per tick**

**Typical 60-player scenario:**

```
Per tick:
  ~5-20 players visible per client
  ~30-100 obstacles in view
  ~0-5 projectiles
  ~0-2 explosions

UpdatePacket per client:
  PlayerData (self): 2-50 bytes
  Full objects: 5-10 creates × 25-50 bytes = 125-500 bytes
  Partial objects: 20-80 objects × 7 bytes = 140-560 bytes
  Deleted objects: 0-5 × 2 bytes = 0-10 bytes
  Bullets/explosions/gas: 20-100 bytes
  
  Total: 300-1200 bytes/tick
  
  Bandwidth: 300-1200 × 40 ticks/sec = 12-48 KB/s per player
```

## References

- **Tier 2:** [docs/subsystems/serialization-system/](docs/subsystems/serialization-system/README.md) — Two-layer encoding
- **Tier 2:** [docs/subsystems/networking/](docs/subsystems/networking/README.md) — UpdatePacket flags
- **Tier 3:** [docs/subsystems/networking/modules/updatepacket.md](docs/subsystems/networking/modules/updatepacket.md)
- **Source:** [server/src/game.ts:66-67, 469-499](server/src/game.ts) — Dirty object sets
- **Source:** [common/src/utils/objectsSerializations.ts](common/src/utils/objectsSerializations.ts) — Serialization schema
