# Game Loop — Network Tick Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/game.ts, server/src/objects/player.ts -->

## Purpose
Encapsulates per-tick packet construction, visibility culling per player, binary serialization into compact `UpdatePacket` format, and broadcasting to all players while maintaining per-tick network budget constraints.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/game.ts` | Packet broadcasting and dirty object collection | Medium |
| `server/src/objects/player.ts` | `secondUpdate()` and per-player packet generation | Medium |
| `common/src/packets/updatePacket.ts` | UpdatePacket structure and serialization schema | High |
| `common/src/utils/suroiByteStream.ts` | Binary stream (compact encoding/decoding) | High |

## Packet Construction Pipeline

After all objects update in a tick, packets are constructed and sent to players:

```
Post-Update State
  │
  ├─ Collect dirty objects
  │  ├─ fullDirtyObjects (major state change)
  │  └─ partialDirtyObjects (minor changes)
  │
  ├─ For each player:
  │  │
  │  ├─ Compute visible objects (visibility culling)
  │  │   └─ Query what this player can see (LOS, distance, layer)
  │  │
  │  ├─ Build UpdatePacket
  │  │   ├─ Serialize self state (position, rotation, health)
  │  │   ├─ Serialize visible full-dirty objects
  │  │   ├─ Serialize visible partial-dirty objects
  │  │   └─ Serialize new/deleted objects
  │  │
  │  ├─ Apply per-tick network budget
  │  │   └─ If packet too large, trim lower-priority objects
  │  │
  │  └─ Send to player's WebSocket
  │
  └─ Clear dirty object sets for next tick
```

## Visibility Culling Per Player

**Goal:** Each player only receives information about objects they can see.

**Culling criteria:**
1. **Distance:** Object within `scope.zoomLevel` render distance
2. **Layer:** Object on same layer or adjacent layer (nearby floor/ceiling)
3. **Line-of-Sight (LOS):** No occluding walls between player and object
4. **Scope effect:** Equipped scope expands visibility hitbox radius

**Implementation approach:**
- Player's `update()` method calculates visible objects (see [Player Update Module](player-update.md))
- Stores bitmask or set of visible object IDs
- In `secondUpdate()`, filter objects against this visibility set

**LOS calculation:** @file common/src/utils/math.ts
```typescript
// Ray-trace from player to object center
// Check intersection with obstacle walls
const canSee = !Geometry.getIntersection(
  playerPos,
  objectPos,
  obstacleSegments
);
```

**Scope zoom adjustment:** 
```
Render distance = baseDistance × (zoomLevel / DEFAULT_SCOPE.zoomLevel)
// e.g., 8x scope = 2× render distance
```

## UpdatePacket Structure

The binary `UpdatePacket` is the primary message sent each tick:

```typescript
UpdatePacket {
  // Player self-state (always sent if dirty)
  myPosition: Vector,
  myRotation: number,
  myHealth: number,
  myAdrenaline: number,
  // ... other self fields

  // Objects in world
  newPlayer: Player[],      // Players spawned this tick
  deletedObjects: ObjectID[],
  
  // Full update objects (major state change)
  fullUpdate: {
    objectID: number,
    ObjectData (position, rotation, state, etc.)
    // ... per object
  }[],
  
  // Partial update objects (minor changes)
  partialUpdate: {
    objectID: number,
    PartialData (only position/rotation)
    // ... per object
  }[],
  
  // Projectiles, emotes, map pings, explosions
  projectiles: Projectile[],
  emotes: Emote[],
  mapPings: MapPing[],
  explosions: Explosion[],
  
  // Game state (alive count, etc.)
  aliveCount: number,
  killedBy?: {
    killer: string,
    weapon: string
  }
}
```

**Binary serialization:** Uses `SuroiByteStream` (single-byte tags + variable-length fields)

## Serialization Format (Compact Binary)

The protocol uses a **tag-based variable-length encoding** to minimize bytes:

```
UpdatePacket binary layout:
┌─────────────────────────────────────────────────────────────┐
│ Packet Type (u8)           │ 0x00 = UPDATE                  │
├─────────────────────────────────────────────────────────────┤
│ My Position (tag + float32 x 2)  │ if dirty.position        │
├─────────────────────────────────────────────────────────────┤
│ My Rotation (tag + float32)      │ if dirty.rotation        │
├─────────────────────────────────────────────────────────────┤
│ My Health (tag + float32)        │ if dirty.health          │
├─────────────────────────────────────────────────────────────┤
│ Full Update Count (u16)          │ Number of full updates    │
│ [Object ID (u16) + Data ...]     │ Repeated per object       │
├─────────────────────────────────────────────────────────────┤
│ Partial Update Count (u16)       │ Number of partial updates │
│ [Object ID (u16) + Pos only...]  │ Repeated per object       │
├─────────────────────────────────────────────────────────────┤
│ New Objects Count (u16)          │ Players/loot spawned      │
│ [...New object data...]          │ Full serialization        │
├─────────────────────────────────────────────────────────────┤
│ Deleted Objects Count (u16)      │ Objects removed           │
│ [Object ID (u16) ...]            │ Repeated per ID           │
├─────────────────────────────────────────────────────────────┤
│ Projectiles, Emotes, Pings, etc. │ If any new this tick      │
└─────────────────────────────────────────────────────────────┘
```

**Example size for one player full update:**
- Position: 1 (tag) + 8 bytes = 9 bytes
- Rotation: 1 + 4 bytes = 5 bytes
- Health: 1 + 4 bytes = 5 bytes
- **Total: ~16 bytes per player per tick** (matches `fullAllocBytes`)

**Partial update (position + rotation only):**
- **~12 bytes** (matches `partialAllocBytes`)

## Per-Tick Network Budget

Network bandwidth is constrained. Each tick has a maximum **packet size limit**:

**Default:** 65,536 bytes per tick (64 KB)
**Calculation:** 60 players × ~1000 bytes per player per tick

**Budget enforcement:** @file server/src/game.ts
```typescript
const maxPacketSize = 65536;  // bytes per tick
let currentSize = 0;

for (const visibleObject of visibleObjects) {
  const objectSize = visibleObject.fullAllocBytes; // or partialAllocBytes
  
  if (currentSize + objectSize > maxPacketSize) {
    // Skip this object (deprioritized)
    continue;
  }
  
  currentSize += objectSize;
  // Add to packet
}
```

**Deprioritization when budget exceeded:**
1. **Keep:** Self updates (always sent)
2. **Keep:** Living players (high priority)
3. **Keep:** Projectiles (dangerous, must sync)
4. **Drop:** Loot (low priority, can catch next tick)
5. **Drop:** Obstacles (static for most players)

## Object Dirty Flag System

Objects track what changed since last tick:

```typescript
// BaseGameObject
dirty = {
  position: false,      // Moved this tick
  rotation: false,      // Rotated
  health: false,        // HP changed
  animation: false,     // Frame advanced
  inventory: false,     // Item picked up / dropped
  teammates: boolean,   // Team state changed (team mode)
  // ... more flags
};
```

**Usage:**
- `secondUpdate()` checks flags before serializing
- Only serialize changed fields → smaller packets

**Flag reset:** After serialization, clear dirty flags
```typescript
this.dirty.position = false;
this.dirty.rotation = false;
// ... etc
```

## Shared Data Stream Optimization

Multiple players may share **identical object data** (e.g., obstacle positions never change):

**Optimization:**
- First player to see an object: Full serialization
- Other players seeing same object: Can reference cached serialization

**Implementation:** `@common/packets/updatePacket.ts` uses `UpdateDataCommon` struct for shared data

```typescript
// Shared data reused across players
UpdateDataCommon {
  position: Vector,
  rotation: number,
  animation: number,
}

// Per-player differences
PlayerData extends UpdateDataCommon {
  health: number,         // Only player has this
  inventory: Item[],      // Only player has this
}
```

## Broadcasting Pattern

After packet construction, broadcast to all players:

```typescript
// Phase 2 Pass 2: Construct packets per player
for (const player of this.connectedPlayers) {
  if (player.joined && !player.dead) {
    const packet = new UpdatePacket(player);
    packet.serialize(player.visibility);
    player.sendPacket(packet);
  }
}

// Spectators (if team mode)
for (const spectator of this.spectators) {
  const packet = new UpdatePacket(spectator);
  packet.serialize(spectator.teamVisibility);
  spectator.sendPacket(packet);
}
```

**Packet sending:** Uses WebSocket `send()` method (non-blocking, buffered by OS)

## Dirty Object Collection

At tick start, dirty objects are collected:

```typescript
// During player.secondUpdate()
if (this.dirty.position) {
  this.game.partialDirtyObjects.add(this);
} else if (majorStateChange) {
  this.game.fullDirtyObjects.add(this);
}

// During obstacle.secondUpdate() (if it exists)
// Static obstacles rarely dirty
```

**Sets are cleared at end of tick:**
```typescript
this.partialDirtyObjects.clear();
this.fullDirtyObjects.clear();
```

## Complex Functions

### `player.secondUpdate()` — @file server/src/objects/player.ts:1799
**Purpose:** Serialize player state into UpdatePacket
**Complexity:** ~100 lines
**Precondition:** `update()` already ran; dirty flags set
**Implicit behavior:**
- Reads `dirty` flags
- Constructs `PlayerData` (full or partial)
- Adds to appropriate `game.fullDirtyObjects` or `game.partialDirtyObjects`
- Sends custom `UpdatePacket` to this player (with their visibility applied)
- Resets dirty flags for next tick

**Key snippet:**
```typescript
let packet = new UpdatePacket();

// Self updates (always sent if dirty)
if (this.dirty.position) {
  packet.myPosition = this.position;
  packet.myRotation = this.rotation;
}

// Visible objects
for (const visible of this.visibleObjects) {
  if (this.game.fullDirtyObjects.has(visible)) {
    packet.addFullUpdate(visible);
  } else if (this.game.partialDirtyObjects.has(visible)) {
    packet.addPartialUpdate(visible);
  }
}

this.sendPacket(packet);
```

### `UpdatePacket.serialize()` — @file common/src/packets/updatePacket.ts
**Purpose:** Encode packet into binary bytes
**Complexity:** ~200 lines (variable-length encoding)
**Precondition:** All packet data populated
**Return:** `Uint8Array` (binary buffer)
**Implicit behavior:**
- Uses `SuroiByteStream` for compact encoding
- Writes tags + values for each field
- Optimizes for small numbers (varint encoding)
- Returns buffer ready for network transmission

### `player.sendPacket()` — @file server/src/objects/player.ts
**Purpose:** Send serialized packet over WebSocket
**Parameters:** `UpdatePacket` or other packet type
**Implicit behavior:**
- Serializes packet to bytes
- Sends over WebSocket connection
- No buffering (OS handles TCP buffering)
- If connection closed, quietly drops packet

## Configuration & Tuning

Max packet size: @file server/src/config.example.json
```javascript
"maxPacketSize": 65536,  // bytes
```

Dirty object budget per object category:
```typescript
// Estimated from definition
const byteBudget = {
  [ObjectCategory.Player]: 16,      // fullAllocBytes
  [ObjectCategory.Loot]: 4,         // Small
  [ObjectCategory.Obstacle]: 8,     // Static, rarely changes
  // ... per category
};
```

## Performance Considerations

**Bandwidth:** 60 players × 1KB per player per tick × 40 TPS = 2.4 MB/s ingress
**Memory:** All dirty objects stored in sets; cleared each tick

**Optimization techniques:**
1. **Partial updates:** Send only changed fields (12 bytes vs. 16 bytes)
2. **Visibility culling:** Skip objects outside player's field of view
3. **Network budget:** Prioritize critical objects (players, projectiles)
4. **Dirty flag caching:** Avoid repeated serialization of unchanged objects

## Related Documents
- **Tier 2:** [Game Loop](../README.md) — subsystem overview
- **Tier 2:** [Networking](../../networking/README.md) — packet protocol details
- **Tier 2:** [Serialization System](../../serialization-system/README.md) — binary encoding
- **Tier 3:** [Player Update](player-update.md) — visibility calculation
- **Tier 3:** [Object Update](object-update.md) — dirty flag setting
- **Tier 3:** [Game State](game-state.md) — game phases
- **Patterns:** [../patterns.md](../patterns.md) — subsystem patterns
