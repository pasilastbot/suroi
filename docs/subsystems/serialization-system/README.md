# Serialization System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/serialization-system/modules/ -->
<!-- @source: common/src/utils/suroiByteStream.ts -->

## Purpose

Two-layer binary encoding/decoding stack for compact WebSocket frame transmission and deterministic object state reconstruction. Enables the protocol to fit full game updates (potentially 100+ objects) in single frames at 40 TPS game tick rate. Used by both server (serializing updates) and client (deserializing updates).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/utils/byteStream.ts` | Base ByteStream class: primitive read/write (uint8/16/32/64, int8/16/32/64, float32/64, strings) |
| `common/src/utils/suroiByteStream.ts` | Domain-specific extensions: position, rotation, scale, vectors, obstacle rotation, player names, boolean packing |
| `common/src/utils/objectsSerializations.ts` | Per-ObjectCategory full/partial serialization schemas; defines what fields serialize for Player, Obstacle, Loot, etc. |
| `common/src/utils/objectPool.ts` | Type-safe object container/registry; organizes GameObjects by ObjectCategory into Sets for fast lookups |
| `common/src/utils/baseBullet.ts` | Bullet definition type schema (not serialization code) |
| `common/src/utils/terrain.ts` | Terrain floor type definitions (not serialization code) |

## Architecture

**Two-layer serialization stack:**

```
┌─────────────────────────────────────────────────────┐
│ Domain Layer (SuroiByteStream)                       │
│ ├─ writePosition(x, y) → 4 bytes                     │
│ ├─ writeRotation(angle) → 1 byte                     │
│ ├─ writeScale(scale) → 1 byte                        │
│ ├─ writeBooleanGroup(b0, b1, ...) → 1 byte          │
│ ├─ writeBooleanGroup2(b0, ..., b15) → 2 bytes       │
│ └─ writeVector(x, y, minX, maxX, minY, maxY, bytes) │
├─────────────────────────────────────────────────────┤
│ Primitive Layer (ByteStream)                         │
│ ├─ writeUint8(v) → 1 byte                            │
│ ├─ writeUint16(v) → 2 bytes                          │
│ ├─ writeUint32(v) → 4 bytes                          │
│ ├─ writeInt8/16/32 → signed variants                │
│ ├─ writeFloat32/64 → IEEE 754 floats                │
│ ├─ writeString(bytes, str) → UTF-8 encoded          │
│ └─ ensureSize() / index tracking                     │
├─────────────────────────────────────────────────────┤
│ Wire Format (ArrayBuffer)                             │
│ └─ Binary data transmitted via WebSocket frame       │
└─────────────────────────────────────────────────────┘
```

## Data Flow

```
Server game state changes
  → identify dirty objects
  → UpdatePacket created with ObjectCategory, object IDs
  → for each dirty object:
    ├─ serialize full state (if first update or reset flag set)
    │  └─ write all fields via ObjectSerializations[Category].serializeFull()
    └─ serialize partial state (delta update)
       └─ write only changed fields via ObjectSerializations[Category].serializePartial()
  → PacketStream multiplexes all packets into single binary message
  → transmit ArrayBuffer over WebSocket
  
  ↓

Client receives ArrayBuffer
  → PacketStream demultiplexes packets
  → UpdatePacket deserialized:
    ├─ read ObjectCategory + object ID
    ├─ read partial fields via ObjectSerializations[Category].deserializePartial()
    └─ (if full flag) read full fields via ObjectSerializations[Category].deserializeFull()
  → ObjectPool updated with deserialized state
```

## SuroiByteStream API

### High-Level Write Methods

| Method | Bytes | Purpose | Range / Encoding |
|--------|-------|---------|-------------------|
| `writePosition(vector)` | 4 | 2D position clamped to map bounds | x, y as uint16 each, normalized to [0, maxPosition] |
| `writeRotation2(angle)` | 2 | Angle in radians, higher precision | uint16 mapped from [-π, π] → [0, 65535] |
| `writeScale(scale)` | 1 | Object scale factor | uint8 mapped from [objectMinScale, objectMaxScale] |
| `writeObstacleRotation(value, mode)` | 1–2 | Obstacle rotation by mode | Full: 2 bytes; Limited/Binary: 1 byte |
| `writeVector(vector, minX, minY, maxX, maxY, bytes)` | 1–4 | 2D vector with custom bounds | Constrained float encoding |
| `writeObjectType(type)` | 1 | Object category enum | uint8, 0–13 (Player, Obstacle, Loot, etc.) |
| `writeObjectId(id)` | 2 | Unique object identifier | uint16, 0–65535 |
| `writeLayer(layer)` | 1 | Z-layer/depth | int8, typically -128 to 127 |
| `writePlayerName(name)` | 1–16 | UTF-8 encoded player name | Fixed 16-byte buffer, null-terminated |

### Low-Level Write Methods (inherited from ByteStream)

| Method | Bytes | Purpose |
|--------|-------|---------|
| `writeUint8(v)` | 1 | 0–255 |
| `writeUint16(v)` | 2 | 0–65,535 |
| `writeUint24(v)` | 3 | 0–16,777,215 |
| `writeUint32(v)` | 4 | 0–4,294,967,295 |
| `writeUint64(v)` | 8 | BigInt, 0–18,446,744,073,709,551,615 |
| `writeInt8(v)` | 1 | -128–127 |
| `writeFloat32(v)` | 4 | 32-bit IEEE 754 float |
| `writeFloat64(v)` | 8 | 64-bit IEEE 754 double |
| `writeFloat(v, min, max, bytes)` | 1–4 | Quantized float in [min, max] with precision = range / (2^(8*bytes) - 1) |
| `writeRotation(v)` | 1 | Angle in [-π, π], 1-byte precision |
| `writeBooleanGroup(...b0–b7)` | 1 | Pack 8 booleans into 1 byte as bit flags |
| `writeBooleanGroup2(...b0–bF)` | 2 | Pack 16 booleans into 2 bytes as bit flags |

### Read Methods (mirror of write)

All read methods reverse their write counterparts:
- `readPosition()` → Vector from 2 uint16
- `readRotation2()` → angle from uint16
- `readScale()` → scale from uint8
- `readVector(minX, minY, maxX, maxY, bytes)` → Vector
- `readObjectType()` → ObjectCategory
- `readObjectId()` → number
- `readLayer()` → Layer
- `readPlayerName()` → string
- `readUint8/16/24/32/64()`, `readFloat()`, `readRotation()`, `readBooleanGroup()`, `readBooleanGroup2()`

**Throw on underflow:** Read methods throw if buffer exhausted (no graceful fallback).

## ObjectSerializations Pattern

Per-ObjectCategory, define **what fields are serialized and how**:

```typescript
ObjectSerializations[ObjectCategory.Player] = {
  serializePartial(stream, { position, rotation, animation, action }) {
    stream.writePosition(position);       // 4 bytes
    stream.writeRotation2(rotation);      // 2 bytes
    
    // Pack dirty flags + animation + action into 1 byte
    const animationDirty = animation !== undefined;
    const actionDirty = action !== undefined;
    let byte = (animationDirty ? 128 : 0) + (actionDirty ? 1 : 0);
    if (animationDirty) byte += animation << 3;
    if (actionDirty) byte += action.type << 1;
    
    stream.writeUint8(byte);              // 1 byte
    if (actionDirty && action.item) {
      Loots.writeToStream(stream, action.item);  // Variable
    }
  },
  
  serializeFull(stream, { full: { layer, dead, downed, ... } }) {
    stream.writeLayer(layer);             // 1 byte
    
    // Pack 16 booleans into 2 bytes
    stream.writeBooleanGroup2(
      dead, downed, beingRevived, invulnerable, // ... 16 flags total
    );                                    // 2 bytes
    
    stream.writeUint8(teamID);            // 1 byte
    Loots.writeToStream(stream, activeItem);  // Variable
    // ... more fields
  },
  
  deserializePartial(stream) {
    // Reverse of serializePartial
    const data = {
      position: stream.readPosition(),
      rotation: stream.readRotation2(),
    };
    const byte = stream.readUint8();
    // ... unpack flags
    return data;
  },
  
  deserializeFull(stream) {
    // Reverse of serializeFull
    const layer = stream.readLayer();
    const [dead, downed, ...] = stream.readBooleanGroup2();
    // ... more fields
  }
}
```

**Partial vs. Full:**
- **Partial**: Delta update with only changed fields (position, rotation, animation state, action)
- **Full**: Initial state with all fields (inventory, armor, skin, team, infection status, perks, etc.)

Server generates `UpdatePacket` with partial updates for most objects, full updates for new objects.

## ObjectPool Pattern

Container for game objects organized by category and ID:

```typescript
interface GameObject {
  readonly type: ObjectCategory;
  readonly id: number;
}

const objects = new ObjectPool();
objects.add(player);                    // Add by category
const cat = objects.getCategory(ObjectCategory.Player);  // Set<Player>
const obj = objects.get(id);            // Fast ID lookup
objects.delete(player);                 // Remove + cleanup
```

**Used by:**
- Server `GameManager` to track all entities (players, obstacles, loot, death markers, etc.)
- Client to maintain renderable object state during updates

**Not true memory pooling:** Objects are not pre-allocated and reused; the pool is a named container. GC pressure comes from object allocation on spawn, not from serialization.

## ObjectDefinitions Registry

Enables O(1) lookup of definitions by `idString` and compact binary encoding:

```typescript
const Weapons = new ObjectDefinitions([
  { idString: "ak-47", damage: 15, ... },
  { idString: "m16", damage: 16, ... },
  // ... 200+ weapons
]);

// Encode
if (Weapons.overLength) {
  stream.writeUint16(Weapons.idStringToNumber["ak-47"]);  // 2 bytes
} else {
  stream.writeUint8(Weapons.idStringToNumber["ak-47"]);   // 1 byte
}

// Decode
const idx = Weapons.overLength ? stream.readUint16() : stream.readUint8();
const weapon = Weapons.definitions[idx];  // O(1)
```

**overLength flag:** Set to true if count > 255, requiring 2-byte indices instead of 1-byte. Examples:
- Weapons, ammo types: < 256 → 1 byte
- Obstacles, building types: < 256 → 1 byte
- Scopes, armors: < 256 → 1 byte

## Compression Strategies

### Position Encoding
Map bounds: [0, `maxPosition`] → [0, 65535] (2 bytes per coordinate)

```
writePosition(vector):
  x_uint16 = (vector.x / maxPosition) * 65535 + 0.5
  y_uint16 = (vector.y / maxPosition) * 65535 + 0.5
```

Loss: Sub-pixel precision (maxPosition ≈ 32,000, so ~0.49 unit per LSB).

### Scale Encoding
Range: [`objectMinScale`, `objectMaxScale`] → [0, 255] (1 byte)

```
writeScale(scale):
  uint8 = ((scale - MIN_SCALE) / (MAX_SCALE - MIN_SCALE)) * 255 + 0.5
```

Precision: ~0.004 scale units per LSB (typical for obstacles).

### Rotation Encoding

**1-byte (writeRotation):**
- Range: [-π, π] → [0, 255]
- Precision: ~0.025 radians (±1.4°)
- Used for obstacles with limited rotation modes

**2-byte (writeRotation2):**
- Range: [-π, π] → [0, 65535]
- Precision: ~0.0000958 radians (±0.0055°)
- Used for players and full-rotation obstacles

```
writeRotation2(angle):
  uint16 = (angle / τ + 0.5) * 65535 + 0.5
  where τ = 2π

readRotation2():
  angle = τ * (uint16 / 65535 - 0.5)
```

### Float Quantization
Generic constrained float in [min, max]:

```
writeFloat(value, min, max, bytes):
  range = 2^(8 * bytes) - 1
  uint_n = ((value - min) / (max - min)) * range + 0.5
```

**Typical uses:**
- Health: [0, 1] in 2 bytes → 0.0000153 per LSB
- Adrenaline: [0, 1] in 2 bytes
- Shield: [0, 1] in 2 bytes
- Size modifier: [0, 4] in 1 byte (obstacles)

### Boolean Packing
**writeBooleanGroup (8 booleans, 1 byte):**
```
byte = (b0 ? 1 : 0) + (b1 ? 2 : 0) + ... + (b7 ? 128 : 0)
```

**writeBooleanGroup2 (16 booleans, 2 bytes):**
```
uint16 = (b0 ? 1 : 0) + (b1 ? 2 : 0) + ... + (bF ? 32768 : 0)
```

Examples in player state:
- Partial: animation dirty (1 bit) + action dirty (1 bit) packed with enum values
- Full: 16 flags (dead, downed, invulnerable, has armor, has helmet, Halloween skin, etc.) in 2 bytes

## Dependencies

### Depends on:
- **[Core Math & Physics](../core-math-physics/)** — Vector, Angle, math constants (π, τ, halfπ), coordinate bounds
- **[Object Definitions](../object-definitions/)** — Definition registries `ObjectDefinitions<T>` for compact indexing
- **[Constants](../../constants.md)** — `ObjectCategory`, `Layer`, `RotationMode`, `GameConstants.maxPosition`, scale min/max, animation types

### Depended on by:
- **[Networking](../networking/)** — All packet classes use `SuroiByteStream`; `UpdatePacket` extensively uses `ObjectSerializations`
- **[Game Objects Server](../game-objects-server/)** — Server objects serialize state to `UpdatePacket`
- **[Game Objects Client](../game-objects-client/)** — Client objects deserialize from `UpdatePacket` updates
- **[Game Loop](../game-loop/)** — Game loop frame sends packed updates via `UpdatePacket` every tick (40 TPS = 25 ms)

## Performance Characteristics

| Factor | Characteristic |
|--------|-----------------|
| **Allocation** | Allocation-free: reuses ObjectPool container for object storage |
| **GC Pressure** | No new allocations in hot path (serialization doesn't allocate); objects allocated on spawn are pre-serialized |
| **Time Complexity** | O(n) where n = number of bytes serialized/deserialized; linear scan through fields |
| **UpdatePacket Size** | Typical ~200–500 bytes per frame for 100 players + map objects; delta encoding (partial updates) reduces vs. full state |
| **Quantization Error** | Position: ~0.49 units; Rotation2: ~0.0056°; Health: ~0.0015; Accept as acceptable for real-time IO game |

## Known Issues & Gotchas

1. **No error recovery on malformed packets** — Deserialization throws `RangeError` if buffer underflows; no graceful fallback. Malformed UpdatePacket from corrupted WebSocket frame will crash object updates.

2. **Float precision loss** — 32-bit floats in animations and projectile state lose sub-pixel precision; quantization errors accumulate over time but are imperceptible in 40 TPS gameplay.

3. **Rotation encoding truncation** — uint16 quantization of angles loses ~0.0056° per tick; accumulated error is negligible (< 0.1° over 60 seconds).

4. **Boolean packing max 16** — `writeBooleanGroup2` caps at 16 packed booleans. Exceeding 16 requires multiple writes or custom bit-packing.

5. **ObjectPool size fixed at initialization** — Pool preallocates category sets but does not dynamically resize; if game creates more objects than expected, only affects memory, not correctness.

6. **Definition index collision not detected** — If two definitions have same `idString`, constructor throws. But at runtime, if definitions array is mutated externally, `readFromStream` may retrieve wrong definition.

7. **ObstacleRotation mode mismatch** — Server and client must agree on `RotationMode` per obstacle type. Mismatch causes deserialization to read wrong number of bytes, corrupting packet stream downstream.

8. **Player name UTF-8 overflow** — `writePlayerName` truncates to 16 bytes after UTF-8 encoding; multi-byte characters near boundary may be cut mid-character, causing invalid UTF-8.

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — System composition and data flow
- [API Reference](../../api-reference.md) — Binary protocol wire format spec

### Tier 2 — Subsystems
- [Networking](../networking/) — Packet framing, `UpdatePacket` structure
- [Object Definitions](../object-definitions/) — Definition registry patterns
- [Core Math & Physics](../core-math-physics/) — Vector, angle types, constants
- [Game Objects Server](../game-objects-server/) — How server objects serialize
- [Game Objects Client](../game-objects-client/) — How client objects deserialize

### Tier 3 — Modules
- [Object Definitions — Registry](../object-definitions/modules/registry.md) — `ObjectDefinitions<T>` implementation
