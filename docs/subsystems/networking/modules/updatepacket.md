# Networking — UpdatePacket Structure & Serialization

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/networking/README.md -->
<!-- @source: common/src/packets/updatePacket.ts -->

## Purpose

Binary protocol specification for **UpdatePacket** — game state delta sent every **40 TPS (25ms)** from server to each client. Contains object creates (full serialization), updates (partial serialization), deletes, player data, bullets, explosions, gas state, and UI information.

## UpdatePacket Overview

**Sent from server to each player every 25ms** (40 TPS = `GameConstants.tps`):

```
UpdatePacket {
  flags: uint16,           // which fields present (UpdateFlags)
  playerData: PlayerData,  // if PlayerData flag set
  deletedObjects: uint16[],
  fullDirtyObjects: {
    id: uint16,
    type: uint8,
    partialData: { ... },
    fullData: { ... }
  }[],
  partialDirtyObjects: {
    id: uint16,
    type: uint8,
    partialData: { ... }
  }[],
  bullets: Bullet[],
  explosions: Explosion[],
  emotes: Emote[],
  gas: GasState,
  newPlayers: Player[],
  deletedPlayers: uint16[],
  // ... etc
}
```

### UpdateFlags Bit Structure

Flags field (`uint16`) indicates which sections follow:

```typescript
enum UpdateFlags {
    PlayerData       = 1 << 0,  // 0x0001 — this client's player data
    DeletedObjects   = 1 << 1,  // 0x0002 — objects destroyed
    FullObjects      = 1 << 2,  // 0x0004 — new objects (full serialization)
    PartialObjects   = 1 << 3,  // 0x0008 — changed objects (partial only)
    Bullets          = 1 << 4,  // 0x0010 — projectiles fired this tick
    Explosions       = 1 << 5,  // 0x0020 — explosions
    Emotes           = 1 << 6,  // 0x0040 — player emotes
    Gas              = 1 << 7,  // 0x0080 — gas state change
    GasPercentage    = 1 << 8,  // 0x0100 — gas progress bar
    NewPlayers       = 1 << 9,  // 0x0200 — new players joined
    DeletedPlayers   = 1 << 10, // 0x0400 — players left
    AliveCount       = 1 << 11, // 0x0800 — alive player count
    Planes           = 1 << 12, // 0x1000 — plane positions
    MapPings         = 1 << 13, // 0x2000 — map pings
    MapIndicators    = 1 << 14, // 0x4000 — map UI indicators
    KillLeader       = 1 << 15  // 0x8000 — kill leader info
}
```

## Player Data Serialization

Per-client state sent in PlayerData section (if flag is set):

### Conditional Fields (Optional)

Player data uses optional fields with a **boolean group** header (2 bytes):

```typescript
writeBooleanGroup2(
    hasMinMax,             // max health, min/max adrenaline
    hasHealth,             // current health
    hasAdrenaline,         // current adrenaline
    hasShield,             // shield
    hasZoom,               // scope magnification
    hasLayer,              // map layer (depth)
    hasId,                 // spectate target ID
    hasTeammates,          // teammate positions & health
    hasHighlightedPlayers, // enemy callout list
    hasInventory,          // weapon slots & ammo counts
    hasLockedSlots,        // locked inventory slots
    hasItems,              // healing items, ammo counts
    hasActiveC4s,          // C4 placement state
    hasPerks,              // active perks
    hasTeamID,             // team ID
    blockEmoting           // emote punishment flag
);

writeBooleanGroup(
    hasInfection           // poison/infection status
);
```

Only fields with `true` flag are serialized.

### Byte Costs (Example Typical State)

**Packet header + flags:** 2 bytes

**PlayerData section (if hasPlayerData):**
- Boolean headers: 2 bytes
- minMax (if has): 12 bytes (3× float32 — max health, min/max adrenaline)
- health (if has): 2 bytes (float quantized to [0,1] precision 0.01)
- adrenaline (if has): 2 bytes
- shield (if has): 2 bytes
- infection (if has): 2 bytes
- zoom (if has): 1 byte (scope level)
- layer (if has): 1 byte (vector layers)
- spectate (if has): 3 bytes (status byte + uint16 target ID)
- teammates (if has): 3 bytes header + 10 bytes per teammate (id + pos + health + color)
- inventory (if has): 1 byte (active slot) + 1 byte (bitmask) + per-gun definition index (1-2 bytes)
- items (if has): variable bitmap + uint16 per unique item

**Typical PlayerData cost: 30–80 bytes** per player update.

## Object Serialization: Full vs. Partial

### Object Types & Categories

```
ObjectCategory {
    Player (0),           // living/dead players
    Obstacle (1),         // buildings, walls, doors
    Loot (2),             // weapons, healing, scopes
    Building (3),         // building shells (deprecated)
    Decal (4),            // blood, water stains, bullet holes
    SyncedParticle (5),   // visual effects synced with server
    Projectile (6),       // bullets, grenades in flight
    Parachute (7),        // player parachutes
    DeathMarker (8)       // death skulls on ground
}
```

### Full Serialization (ObjectCategory.Player)

Sent when object is **first visible** to client (enters view):

```typescript
serializeFull(stream, {
  full: {
    layer,              // map layer
    dead,               // boolean
    downed,             // boolean
    beingRevived,       // boolean
    teamID,             // uint8
    invulnerable,       // boolean
    activeItem,         // weapon/melee definition
    sizeMod,            // float [0,4]
    reloadMod,          // float [0,4] or undefined
    skin,               // cosmetic definition
    helmet,             // armor definition or undefined
    vest,               // armor definition or undefined
    backpack,           // backpack definition
    activeDisguise,     // obstacle definition or undefined
    backEquippedMelee,  // melee definition or undefined
    // ... other state
  }
}) {
  stream.writeLayer(layer);
  stream.writeBooleanGroup2(...);  // 2 bytes for conditional flags
  stream.writeUint8(teamID);
  Loots.writeToStream(stream, activeItem);  // 1-2 bytes
  stream.writeFloat(sizeMod, 0, 4, 1);      // 1 byte quantized
  if (hasReloadMod) {
    stream.writeFloat(reloadMod, 0, 4, 1);  // 1 byte if present
  }
  Skins.writeToStream(stream, skin);        // 1-2 bytes
  if (hasHelmet) Armors.writeToStream(...); // 1-2 bytes if present
  if (hasVest) Armors.writeToStream(...);   // 1-2 bytes if present
  // ... etc
}
```

**Full cost: 25–50 bytes per player create** (variable by cosmetics).

### Partial Serialization (ObjectCategory.Player)

Sent when object **already known** to client and some fields changed:

```typescript
serializePartial(stream, {
  position,    // Vector quantized
  rotation,    // angle in radians
  animation,   // uint8 (0-15) or undefined
  action       // action type or undefined
}) {
  stream.writePosition(position);   // 4 bytes
  stream.writeRotation2(rotation);  // 2 bytes
  
  // Dirty bits packed into 1 byte:
  // Format: Nnnn nccC
  // N: animation dirty, n: animation (4 bits), c: action (2 bits), C: action dirty
  const animationDirty = animation !== undefined;
  const actionDirty = action !== undefined;
  
  let byte = (animationDirty ? 128 : 0) + (actionDirty ? 1 : 0);
  if (animationDirty) byte += animation << 3;
  if (actionDirty) byte += action << 1;
  
  stream.writeUint8(byte);          // 1 byte
  
  if (actionDirty && action.item) {
    Loots.writeToStream(stream, action.item);  // 1-2 bytes if UseItem action
  }
}
```

**Partial cost: 7–10 bytes per player update** (typical).

## Object Creation vs. Update Decision

Server maintains two sets per tick:

```typescript
readonly fullDirtyObjects = new Set<BaseGameObject>();  // newly visible
readonly partialDirtyObjects = new Set<BaseGameObject>(); // changed
```

**During update phase (game.ts ~line 469):**

```
for each partialObject in partialDirtyObjects:
  if not in fullDirtyObjects:  // avoid double-send
    add to UpdatePacket.partialDirtyObjects

for each fullObject in fullDirtyObjects:
  add to UpdatePacket.fullDirtyObjects (full + partial data)
```

### When Object Transitions Full → Partial

1. Object created or becomes visible → added to `fullDirtyObjects`
2. Next tick, if still dirty but not new → moved to `partialDirtyObjects`
3. Fields that change → marked dirty on serialization (implicit)

## Object Delete Format

```
deletedObjects: uint16[]  // array of NetworkIDs
```

**Cost: 2 bytes per deletion** (just ID).

**Header format:**
- Array count written as uint16 (big deletions rare)
- Each entry: uint16 ID

**Client processing:**
```
for id in deletedObjects:
  object = objectPool.getById(id)
  objectPool.returnToPool(object)  // recycle renderer
```

## Visibility Culling Integration

UpdatePacket is **per-client** — only includes visible objects:

**Server-side (player.secondUpdate):**

```
visibleObjects = grid.getVisibleObjects(player.position, player.viewRadius)

creates: visibleObjects.filter(obj => obj.isNew && wasNotPreviouslyVisible())
updates: visibleObjects.filter(obj => isDirty(obj) && isInView())
deletes: previouslyVisible.filter(obj => !visibleObjects.includes(obj))

updatePacket = {
  fullDirtyObjects: creates,
  partialDirtyObjects: updates,
  deletedObjects: deletes
}
sendToClient(player, updatePacket)
```

Clients typically see **5–50 objects at a time** (vs. 200+ on server).

## Array Serialization Format

Arrays use **count headers**:

```
writeArray(array, writeFn, countBytes=2) {
  stream.writeUintN(array.length, countBytes);  // 1 or 2 bytes
  for item in array:
    writeFn(item)
}
```

For DeletedObjects (usually few):
- Count header: **2 bytes** (supports up to 65K deletes, unlikely)
- Each ID: **2 bytes**

For FullObjects (usually few):
- Count header: **2 bytes**
- Each full object: **ID (2) + Category (1) + Partial (~7) + Full (~25) = ~35 bytes**

For PartialObjects (common, many):
- Count header: **2 bytes**
- Each update: **ID (2) + Category (1) + Partial data (~7) = ~10 bytes**

For Bullets (typically few per tick):
- Count header: **1 byte** (supports up to 255)
- Each bullet: **~10–20 bytes** (position, velocity, owner, definition)

## Position Encoding (Quantization)

Position is **quantized to map bounds** (not float32):

```typescript
writePosition(vector: Vector) {
  // Map max position: 1924 units (GameConstants.maxPosition)
  // Quantize to uint16 [0, 65535]
  x_quant = (vector.x / 1924) * 65535
  y_quant = (vector.y / 1924) * 65535
  
  stream.writeUint16(x_quant)  // 2 bytes
  stream.writeUint16(y_quant)  // 2 bytes
  // Total: 4 bytes (not 8)
}

readPosition(): Vector {
  x_quant = stream.readUint16()
  y_quant = stream.readUint16()
  
  x = (x_quant / 65535) * 1924
  y = (y_quant / 65535) * 1924
  
  return {x, y}
}

// Precision: 1924 / 65535 ≈ 0.029 units
// Imperceptible at 1924×1924 map scale
```

## Rotation Encoding

Rotation is **quantized to uint16**:

```typescript
writeRotation2(angle: number) {  // angle in radians (-π to π)
  quant = ((angle + π) / (2π)) * 65535
  stream.writeUint16(quant)  // 2 bytes
  // Precision: 2π / 65535 ≈ 0.000096 radians ≈ 0.0055°
}
```

## Float Quantization

Floats normalized to range [min, max] are **packed to 1–4 bytes**:

```typescript
writeFloat(value: number, min: number, max: number, bytes: 1|2|3|4) {
  // Normalize to [0, 1]
  normalized = (value - min) / (max - min)
  
  // Quantize to N bytes
  if bytes === 1:
    quant = normalized * 255  // uint8
  else if bytes === 2:
    quant = normalized * 65535  // uint16
  else if bytes === 3 or 4: ... // larger ranges
  
  stream.writeUintN(quant, bytes)
}
```

**Example — health [0, 200]:**

```
Patient health: 150
normalized = 150 / 200 = 0.75
quant_2byte = 0.75 * 65535 ≈ 49,151
stream.writeUint16(49151)  // 2 bytes

// Client side
quant_2byte = stream.readUint16()
normalized = quant_2byte / 65535 = 0.75
health = 0.75 * 200 = 150  ✓
```

## Data Lineage: Server Build → Network → Client Deserialize

```
┌─ Server Tick T ─────────────────────────┐
│                                          │
│  Player.position = {500, 400}           │
│  Player.health = 150                    │
│  Obstacle.destroyed = true              │
│  Weapon.ammoInMag = 24                  │
│                                          │
│  Mark dirty:                            │
│  partialDirtyObjects.add(player)        │
│  fullDirtyObjects.add(newWeapon)        │
└──────────────────────────────────────────┘
                    ↓
┌─ Serialize this Tick ────────────────────┐
│                                          │
│  for player in partialDirtyObjects:     │
│    playerPartialStream.writePosition()   │
│    playerPartialStream.writeRotation()   │
│    playerPartialStream.writeAnimation()  │
│                                          │
│  for weapon in fullDirtyObjects:        │
│    weaponPartialStream.writePosition()   │
│    weaponFullStream.writeAmmoInMag()     │
└──────────────────────────────────────────┘
                    ↓
┌─ UpdatePacket Construction ──────────────┐
│                                          │
│  flags = UpdateFlags.PartialObjects |   │
│          UpdateFlags.FullObjects |   │
│          UpdateFlags.PlayerData        │
│  stream.writeUint16(flags)     // 2B   │
│  serializePlayerData()         // 50B  │
│  partialObjectsCache:          // 10B  │
│    {id: 1, type: Player, ...}         │
│  fullObjectsCache:             // 50B  │
│    {id: 2, type: Loot, ...}           │
│                                          │
│  Total: ~112 bytes                     │
└──────────────────────────────────────────┘
                    ↓
┌─ Network Transmission ───────────────────┐
│                                          │
│  WebSocket frame over TCP               │
│  Latency ~20–100ms (typical)           │
│                                          │
└──────────────────────────────────────────┘
                    ↓
┌─ Client Deserialization (game tick T+1) ┘
│                                          │
│  flags = stream.readUint16()           │
│  if (flags & PlayerData):              │
│    playerData = deserializePlayerData() │
│    updateUI(playerData)                │
│                                          │
│  if (flags & PartialObjects):          │
│    foreach stream.readUint16() as id:  │
│      object = objectPool[id]           │
│      object.position = readPosition()  │
│      game.updateObject(object)         │
│                                          │
│  if (flags & FullObjects):             │
│      object.createFromFull()           │
│      objectPool.register(object)       │
│                                          │
│  updateRendering()                     │
└──────────────────────────────────────────┘
```

## Typical Packet Size Breakdown

**Example 40-player game, player sees 30 objects (~8 players + 22 props/weapons):**

```
Packet header:
  flags (uint16)                          2 bytes

PlayerData (this client):
  headers + inventory + health            50 bytes

PartialObjects (8 players × 10 bytes):
  8 players moving                        80 bytes

FullObjects (2 new weapons):
  headers + positions + definitions       50 bytes

Bullets (3 projectiles × 15 bytes):
  header + 3 bullets                      20 bytes

DeletedObjects (1 weapon despawned):
  header + 1 id                           4 bytes

Gas (if state changed):
  state + positions + radii               15 bytes

UI (NewPlayers, etc):
  (conditional, less frequent)            10 bytes

────────────────────────────────────────
TOTAL:  ~230 bytes per client per tick
────────────────────────────────────────

Network throughput (40 TPS):
  230 bytes/tick × 40 ticks/sec = 9.2 KB/sec downstream per player
  4 players × 9.2 KB = 36.8 KB/sec server → all clients
```

## Frequency & Timing

```
Server tick loop (40 TPS = 25ms per tick):

  t=0ms      Tick 0: execute movement, collisions, actions
             → construct UpdatePacket for all players
             → serialize dirty objects
             → send over WebSocket

  t=25ms     Tick 1: execute movement, collisions, actions
             → construct UpdatePacket
             → send

  t=50ms     Tick 2: ...
  ...

Client receives:
  packet @ t≈25ms (network latency 20–100ms)
  process packet @ t≈30ms
  render @ t≈31ms
  user UI updates @ t≈32ms
  
User perceives:
  Server state is 1–3 ticks behind (25–75ms of latency)
```

## Known Issues & Gotchas

### 1. **Invisible Updates Before Visibility**
Objects are marked dirty **before** visibility check. If object enters view same tick it's created:
- Full packet sent correctly
- But if object exits view before next tick → never reaches partial stage

### 2. **Position Precision Loss**
Quantization loses ~0.029 units of precision. At map corners (x=1900):
```
quantized_x = (1900 / 1924) * 65535 ≈ 64,487
recovered_x = (64487 / 65535) * 1924 ≈ 1899.97
error ≈ 0.03 units (imperceptible)
```

### 3. **Partial-Only Objects Without Full**
If client never saw an object's full state, partial data may be incomplete. Clients must handle:
```
// Position/rotation might be missing
if (objectData.position === undefined) {
  object.position = lastKnownPosition  // fallback
}
```

### 4. **Limited to 65K Object IDs**
NetworkID is uint16:
```
if (objectCount > 65535) {
  // ID allocation wraps
  newObjectId = nextObjectId % 65536  // wraps to 0
  // Risk: ID collision if old object not deleted
}
```

**Unlikely in practice** (65K simultaneous objects ≈ 2KB unique IDs at 2 bytes each, needing ~2GB map).

### 5. **Sync Loss on Packet Loss**
UDP doesn't guarantee delivery (if used for objects vs. TCP for critical packets):
```
Lost packet: UpdatePacket with creates
→ Client never sees object spawn
→ Later partial update for unseen object
→ Client object pool error
```

Suroi uses **TCP WebSocket** (reliable), so this is mitigated.

### 6. **PlayerData Too Large**
If inventory/items/perks over-serializes, PlayerData can exceed expected cost:
```
// Problematic:
- 4 weapons all with stats → +8 bytes
- 100 item types with non-zero counts → +200 bytes
- 10 active perks → +50 bytes
// PlayerData could exceed 400 bytes alone
```

### 7. **Dirty Flag Contamination**
Object marked dirty, then removed from visible set:
```
// Bad:
fullDirtyObjects.add(obstacle)
obstacle.health = 0  // destroyed
visibleObjects = calculateVisible()  // obstacle no longer visible

// obstacle still in fullDirtyObjects
// Gets serialized and sent to client
// Client spawns destroyed object
```

Requires cleanup in postPacket() phase.

## Related Documents

### Tier 2
- [Networking](../README.md) — binary protocol overview, packet types, packet stream
- [Spatial Grid & Visibility](../../spatial-grid/) — calculating visible objects per client
- [Serialization System](../../serialization-system/) — SuroiByteStream implementation

### Tier 3
- [game-loop/modules/update-phase.md](../../game-loop/modules/update-phase.md) — when UpdatePacket is constructed per tick
- [game-loop/modules/second-update-phase.md](../../game-loop/modules/second-update-phase.md) — visibility culling & per-player packet construction

### Source Files
- `@file common/src/packets/updatePacket.ts` — UpdatePacket class, serialization logic
- `@file common/src/utils/suroiByteStream.ts` — binary encoding methods (writePosition, writeFloat, etc.)
- `@file server/src/game.ts` — dirty object tracking, update loop
- `@file common/src/utils/objectsSerializations.ts` — per-category full/partial serialization
