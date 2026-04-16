# Binary Encoding & SuroiByteStream

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/serialization-system/README.md -->
<!-- @source: common/src/utils/byteStream.ts, common/src/utils/suroiByteStream.ts -->

## Purpose

Two-class encoding system for compact binary representation of game state. `ByteStream` provides primitive types (integers, floats, strings); `SuroiByteStream` adds domain-specific methods (position, rotation, scale, vectors) optimized for game object serialization. Uses IEEE-754 floats with quantization and range-constrained encoding to fit 100+ object updates per 25ms tick (40 TPS).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/utils/byteStream.ts` | Base ByteStream class: primitive read/write, boolean packing, rotation quantization | Medium |
| `common/src/utils/suroiByteStream.ts` | SuroiByteStream domain layer: position, scale, obstacle rotation, player names, vectors | Medium |
| `common/src/constants.ts` | Game bounds: `maxPosition=1924`, `nameMaxLength=16`, `objectMinScale`/`objectMaxScale` | Low |

## SuroiByteStream API

### Domain-Specific Write Methods

All methods return `this` for method chaining.

| Method Signature | Bytes | Description |
|------------------|-------|-------------|
| `writePosition(vector: Vector)` | 4 | Map-space position quantized to 16-bit x/y |
| `writeVector(vector: Vector, minX, minY, maxX, maxY, bytes: 1\|2\|3\|4)` | 1–4 | Arbitrary vector in constrained range |
| `writeRotation(value: number)` | 1 | Angle quantized to [−π, π] in 1 byte |
| `writeScale(scale: number)` | 1 | Object scale quantized to [MIN, MAX] in 1 byte |
| `writeObjectType(type: ObjectCategory)` | 1 | Alias for `writeUint8(type)` |
| `writeObjectId(id: number)` | 2 | Alias for `writeUint16(id)` |
| `writeObstacleRotation(value: number, mode: RotationMode)` | 1–2 | Rotation in Full/Limited/Binary mode; not space-efficient |
| `writeLayer(layer: Layer)` | 1 | Signed int8 layer value |
| `writePlayerName(name: string)` | ≤16 | UTF-8 player name, null-terminated, max 16 bytes |

### Domain-Specific Read Methods

| Method Signature | Bytes | Description |
|------------------|-------|-------------|
| `readPosition(): Vector` | 4 | Read position from quantized 16-bit x/y |
| `readVector(minX, minY, maxX, maxY, bytes: 1\|2\|3\|4): Vector` | 1–4 | Read arbitrary vector from constrained range |
| `readRotation(): number` | 1 | Read angle from [−π, π], 1 byte |
| `readScale(): number` | 1 | Read scale from [MIN, MAX], 1 byte |
| `readObjectType(): ObjectCategory` | 1 | Alias for `readUint8()` |
| `readObjectId(): number` | 2 | Alias for `readUint16()` |
| `readObstacleRotation(mode: RotationMode): { rotation: number, orientation: Orientation }` | 1–2 | Read rotation in specified mode |
| `readLayer(): Layer` | 1 | Read signed int8 layer value |
| `readPlayerName(): string` | ≤16 | Read UTF-8 name, null-terminated |

## Primitive Types

### Base ByteStream Encoding

All primitive operations advance `.index` pointer and use little-endian byte order.

#### Unsigned Integers

| Type | Range | Bytes | Write Method | Read Method |
|------|-------|-------|--------------|-------------|
| uint8 | [0, 256) | 1 | `writeUint8(v)` | `readUint8()` |
| uint16 | [0, 65536) | 2 | `writeUint16(v)` | `readUint16()` |
| uint24 | [0, 16777216) | 3 | `writeUint24(v)` [non-native] | `readUint24()` [non-native] |
| uint32 | [0, 4294967296) | 4 | `writeUint32(v)` | `readUint32()` |
| uint64 | [0, 2^64) | 8 | `writeUint64(bi)` | `readUint64()` → **bigint** |

#### Signed Integers

| Type | Range | Bytes | Write Method | Read Method |
|------|-------|-------|--------------|-------------|
| int8 | [−128, 128) | 1 | `writeInt8(v)` | `readInt8()` |
| int16 | [−32768, 32768) | 2 | `writeInt16(v)` | `readInt16()` |
| int32 | [−2^31, 2^31) | 4 | `writeInt32(v)` | `readInt32()` |
| int64 | [−2^63, 2^63) | 8 | `writeInt64(bi)` | `readInt64()` → **bigint** |

#### IEEE-754 Floats

| Type | Precision | Bytes | Write Method | Read Method |
|------|-----------|-------|--------------|-------------|
| float32 | ~7 decimal digits | 4 | `writeFloat32(v)` | `readFloat32()` |
| float64 | ~16 decimal digits | 8 | `writeFloat64(v)` | `readFloat64()` |

### Endianness

All multi-byte primitives use **little-endian** byte order (LSB first), matching JavaScript's native `DataView` methods (`getUint16`, `getUint32`, etc.).

**Example:**  
```typescript
const stream = new ByteStream(new ArrayBuffer(2));
stream.writeUint16(0x1234);  // Bytes: 0x34, 0x12 (little-endian)
// For big-endian systems, extra swapping logic would be needed
```

## String Encoding

### Standard UTF-8 Strings

```typescript
writeString(bytes: number, string: string): this
readString(bytes: number): string
```

**Behavior:**
- Encodes/decodes UTF-8 via [TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder) and [TextDecoder](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder)
- **Write:** Null-terminates on first `\0` byte or fills remaining slots with `\0` padding
- **Read:** Reads up to `bytes` count or stops at first `\0`, whichever comes first
- **Truncation:** Excess bytes silently dropped, UTF-8 encoding may be split mid-character

**Example:**
```typescript
const stream = new ByteStream(new ArrayBuffer(10));
stream.writeString(10, "Hello");  // Writes: 'H','e','l','l','o',0,0,0,0,0
stream.index = 0;
console.log(stream.readString(10));  // "Hello"
```

### Player Names (Fixed-Length, Null-Terminated)

```typescript
writePlayerName(name: string): this  // Max 16 bytes
readPlayerName(): string              // Max 16 bytes
```

**Constants (from `constant.ts`):**
- `maxLength = 16` bytes
- Name is UTF-8 encoded, null-terminated within the 16-byte window
- Used by `UpdatePacket` for compact player identity

**Specification:**
- Encodes name as UTF-8 bytes via `TextEncoder`
- Writes up to 16 uint8 values
- Stops encoding on first `\0` padding byte
- **Read:** Decodes uint8 array stopping at `\0`

**Example:**
```typescript
stream.writePlayerName("Alice");      // Bytes: 'A','l','i','c','e',0,0,...
stream.index = 0;
console.log(stream.readPlayerName()); // "Alice"
```

## Vector Encoding

### Position Quantization

Positions in the map are encoded as 16-bit x/y pairs within the game bounds.

```typescript
writePosition(vector: Vector): this
readPosition(): Vector
```

**Quantization formula:**
- Input bounds: [0, `maxPosition`] where `maxPosition = 1924`
- Output range: [0, 65535] (16-bit unsigned)
- **Write:** `uint16 = ((x / 1924) * 65535 + 0.5)`
- **Read:** `float = 1924 * (uint16 / 65535)`

**Byte cost:** 4 bytes (2 for x, 2 for y)

**Precision loss:**
- Map width = 1924 pixels
- Quantization cells ≈ 1924 / 65535 ≈ 0.029 pixels
- **Precision:** ±0.015 pixels (excellent for 1924×1924 map)

**Example:**
```typescript
const pos = { x: 962, y: 1000 };  // Center-ish
stream.writePosition(pos);
stream.index = 0;
const decoded = stream.readPosition();
// decoded.x ≈ 962.0, decoded.y ≈ 1000.0 (within quantization error)
```

### Arbitrary Vector Encoding (Range-Constrained)

```typescript
writeVector(vector: Vector, minX, minY, maxX, maxY, bytes: 1|2|3|4): this
readVector(minX, minY, maxX, maxY, bytes: 1|2|3|4): Vector
```

**Quantization formula:**
- Range: [min, max] is inclusive
- Precision: `2^(8*bytes) - 1` levels
- **Write:** `uint = ((value - min) / (max - min)) * maxQuantum + 0.5`
- **Read:** `float = min + (max - min) * uint / maxQuantum`

| Bytes | Max Quantum | Precision | Use Case |
|-------|-------------|-----------|----------|
| 1 | 255 | 1/256th of range | Small fields (knock vectors, velocity components) |
| 2 | 65535 | 1/65536th of range | Positions, velocities |
| 3 | 16777215 | 1/16M th of range | High-precision trajectories |
| 4 | 4294967295 | full 32-bit | Float32-equivalent precision |

**Example:**
```typescript
// Velocity in range [-100, 100] m/s, quantized to 1 byte
const vel = { x: 50, y: -25 };
stream.writeVector(vel, -100, -100, 100, 100, 1);
stream.index = 0;
const decoded = stream.readVector(-100, -100, 100, 100, 1);
// decoded.x ≈ 50 ± (200/255) ≈ ±0.78
```

### Rotation Quantization

Two variants: full-circle and half-circle precision.

```typescript
writeRotation(value: number): this            // 1 byte, ±π
writeRotation2(value: number): this           // 2 bytes, ±π
readRotation(): number                        // 1 byte, ±π
readRotation2(): number                       // 2 bytes, ±π
```

**Quantization formula:**
- Input: angle in radians, range [−π, π]
- **rotationWrite 1-byte:** `uint8 = ((angle / τ + 0.5) * 255 + 0.5)`
- **Read 1-byte:** `angle = τ * (uint8 / 255 - 0.5)`
- **Write 2-byte:** `uint16 = ((angle / τ + 0.5) * 65535 + 0.5)`
- **Read 2-byte:** `angle = τ * (uint16 / 65535 - 0.5)`

**Precision:**
- 1 byte: 256 levels → ≈ 0.0245 radians ≈ 1.4° per level
- 2 bytes: 65536 levels → ≈ 0.000096 radians ≈ 0.0055° per level

**Example:**
```typescript
const angle = Math.PI / 4;  // 45 degrees
stream.writeRotation(angle);
stream.index = 0;
const decoded = stream.readRotation();
// decoded ≈ 0.785 (within ±1.4°)
```

### Scale Quantization

Object scale encoded in 1 byte within [MIN_OBJECT_SCALE, MAX_OBJECT_SCALE].

```typescript
writeScale(scale: number): this
readScale(): number
```

**Constants (from code):**
- `objectMinScale` and `objectMaxScale` define the range
- `objectScaleRange = objectMaxScale - objectMinScale`

**Quantization:**
- **Write:** `uint8 = ((scale - min) / range * 255 + 0.5)`
- **Read:** `scale = min + range * (uint8 / 255)`

**Byte cost:** 1 byte

**Example:**
```typescript
const scale = 0.8;
stream.writeScale(scale);
stream.index = 0;
const decoded = stream.readScale();
// decoded ≈ 0.8 (within ±range/510)
```

## Boolean & Flags

### 8-Bit Boolean Groups

Pack up to 8 booleans into 1 byte using bitwise operations.

```typescript
writeBooleanGroup(b0, b1?, b2?, b3?, b4?, b5?, b6?, b7?): this
readBooleanGroup(): boolean[] & { length: 8 }
```

**Bit mapping (LSB first):**
- Bit 0 → `b0` (value 1)
- Bit 1 → `b1` (value 2)
- Bit 2 → `b2` (value 4)
- ...
- Bit 7 → `b7` (value 128)

**Omitted arguments** are treated as `false` but always written as `0`.

**Byte cost:** 1 byte = 8 booleans

**Example:**
```typescript
stream.writeBooleanGroup(true, false, true);  // Byte value = 5 (bits 0,2 set)
stream.index = 0;
const [b0, b1, b2, b3, b4, b5, b6, b7] = stream.readBooleanGroup();
console.log(b0, b1, b2, b3);  // true, false, true, false
```

### 16-Bit Boolean Groups

Pack up to 16 booleans into 2 bytes.

```typescript
writeBooleanGroup2(b0, b1, ..., bF): this
readBooleanGroup2(): boolean[] & { length: 16 }
```

**Bit mapping:** Same LSB-first pattern as 8-bit variant, extended to 16 bits.

**Byte cost:** 2 bytes = 16 booleans

**Use case:** Packing large state flags (e.g., armor status, healing status, aiming mode, is-dead, etc.)

## Array Serialization

Suroi does not define generic array encoding in a single method. Arrays are serialized via:
1. **Length prefix:** uint8 or uint16 (or fixed length known at decode time)
2. **Element loop:** Manual iteration, calling appropriate write/read for each element

**Pattern (from UpdatePacket and object serialization):**

```typescript
// Write array with length prefix
const items = [item1, item2, item3];
stream.writeUint8(items.length);
for (const item of items) {
    stream.writeUint16(item.id);
    stream.writeUint8(item.count);
}

// Read array with length prefix
const count = stream.readUint8();
const decoded = [];
for (let i = 0; i < count; i++) {
    decoded.push({
        id: stream.readUint16(),
        count: stream.readUint8()
    });
}
```

**Constraints:**
- **No length suffix:** Must know length at decode time or read length prefix
- **No delimiter:** Elements must all decode consistently (no variable-width elements without manual length tracking)
- **Manual indexing:** Developer responsible for byte order, alignment, and format consistency

## Definition Index Encoding

Object definitions (weapons, items, obstacles, etc.) are stored in `ObjectDefinitions<T>` registries with **O(1) lookup by idString**. On the wire, they serialize as **single-byte index** if ≤255 definitions, else **two-byte index**.

```typescript
// From common/src/utils/objectDefinitions.ts (conceptual)
ObjectDefinitions.get("gun_ak47") → index 42
writeUint8(42)  // Compact wire format
```

**In UpdatePacket:**
- Player's equipped weapon is serialized as 1 byte (weapon index, not full definition)
- Projectile type is 1 byte (bullet index)
- Obstacle type is 1 byte (obstacle index)

**Byte cost:**
- 1 byte for ≤255 definitions (current game state)
- Would expand to 2 bytes if definitions exceed 255

**Validation:**
```typescript
const maxCategorySize = Math.max(
    Weapons.length, Items.length, Obstacles.length, ...
);
if (maxCategorySize > 255) {
    throw new Error("Definition index overflow: need 2-byte encoding");
}
```

## Variable-Length Integers (VarInt)

**Not currently used in Suroi's encoding.** All integers are fixed-width. If added in future:

**Potential implementation (example only):**
```typescript
// Write: small ints use 1 byte, large ints use multi-byte with continuation bit
// Read: check continuation bit to determine how many bytes follow
```

**Rationale:** Fixed-width simpler to reason about and maintain. Varints add complexity for marginal savings (most integers are small, but predictability matters for frame budgeting).

## Performance Patterns

### Avoiding Allocations

**Zero-copy reading:**
- `ByteStream.buffer` is **not copied**; it's a view over the original `ArrayBuffer`
- Read operations extract raw bytes without allocating intermediate arrays
- **String decoding only:** TextDecoder allocates JS strings (unavoidable)

**Example:**
```typescript
const socket = receive(binaryFrame);  // ArrayBuffer from WebSocket
const stream = new ByteStream(socket); // No copy, just wraps with DataView
const pos = stream.readPosition();    // Reads 4 bytes, no intermediate array
```

### Index Advancement

The `.index` pointer is manually advanced after each operation. Stream does not auto-advance:
```typescript
stream.writeUint8(42);     // index: 0 → 1
stream.writeUint8(43);     // index: 1 → 2
stream.index;              // Now 2
```

**Pattern:** Always check `.index` after reading to ensure no overread; use `.buffer.byteLength` for bounds.

### Method Chaining

All write methods return `this`, enabling chaining:
```typescript
stream
    .writeUint8(PACKET_TYPE)
    .writeUint16(objectId)
    .writePosition(pos)
    .writeBooleanGroup(isAlive, isDowned);
```

### Quantization for Bandwidth

Quantized encoding (position, rotation, scale, vectors) trades precision for bandwidth:
- **Position:** 4 bytes per object (vs 16 for float64 x/y)
- **Rotation:** 1 byte (vs 8 for float64)
- **Scale:** 1 byte (vs 8 for float64)
- **Savings:** ≈75% reduction per object on spatial fields

**Trade-off:** Precision loss is <0.03 pixels for position (imperceptible at 1924×1924 map).

## Error Handling

### Buffer Overflow / Underflow

Suroi's ByteStream **does not validate bounds**. Undefined behavior if:
- **Write beyond buffer:** `setUint8` will throw `RangeError` (from DataView)
- **Read beyond buffer:** `getUint8` will throw `RangeError` (from DataView)
- **Negative index:** `RangeError` from DataView

**Prevention:** Caller must ensure buffer size matches serialized data:

```typescript
const expectedBytes = 
    1 +                    // packet type
    2 +                    // object count
    (objectCount * 20);    // rough estimate per object
const buffer = new ArrayBuffer(expectedBytes);
const stream = new ByteStream(buffer);
// Use stream...
```

### NaN / Infinity Handling

**Floats (writeFloat32, writeFloat64):**
- NaN and ±Infinity encode normally (IEEE-754 special values)
- Decode correctly back to NaN/Infinity
- **However:** Range-constrained floats (writeFloat, writeVector) have undefined behavior with NaN/Infinity

**Integers (writeUint8, writeInt8, etc.):**
- NaN → undefined behavior (depends on DataView implementation)
- Infinity → undefined behavior
- Non-integers → truncated (_not_ rounded)

**Best practice:** Validate non-NaN finitude before quantized encoding:
```typescript
if (!Number.isFinite(angle)) {
    angle = 0;  // Fallback
}
stream.writeRotation(angle);
```

### String Truncation

UTF-8 strings are silently truncated if encoding exceeds `bytes` limit:

```typescript
stream.writeString(3, "Hello");  // Writes only first 3 bytes: 'H','e','l'
stream.index = 0;
console.log(stream.readString(3));  // "Hel" (no error thrown)
```

**Gotcha:** Multi-byte UTF-8 characters may be split. Reading the truncated string may produce mojibake or decoded garbage.

```typescript
stream.writeString(3, "你好");  // "你" = 3 UTF-8 bytes; "好" truncated
```

## Known Gotchas

### Vector Quantization Precision Loss

**Issue:** Small velocity components lose precision when quantized.

**Example:**
```typescript
const vel = { x: 0.1, y: 0 };  // Very small velocity
stream.writeVector(vel, -100, -100, 100, 100, 1);
stream.index = 0;
const decoded = stream.readVector(-100, -100, 100, 100, 1);
// decoded.x ≈ 0.78 (rounded to nearest 200/255 ≈ 0.78)
```

**Mitigation:** Use 2-byte quantization (bytes=2) for high-precision vectors (missiles, knockback).

### Rotation Precision (1 byte)

**Issue:** 1-degree precision may be insufficient for high-friction movement or precise aiming.

**Example:**
```typescript
const angle = 0.01;  // ~0.57° offset
stream.writeRotation(angle);
stream.index = 0;
const decoded = stream.readRotation();
// decoded ≈ 0 (rounded to nearest level ≈ 1.4°)
```

**Mitigation:** Use `writeRotation2()` (2 bytes) for gameplay requiring sub-degree precision.

### String Null-Termination Semantics

**Issue:** Null bytes in the middle terminate encoding prematurely.

```typescript
const name = "Alice\x00Bob";  // Embedded null
stream.writePlayerName(name);  // Encodes only "Alice"
stream.index = 0;
console.log(stream.readPlayerName()); // "Alice"
```

**Mitigation:** Validate player names for null bytes before encoding, or strip them.

### Integer Overflow

**Issue:** Writing integers outside range produces truncation, not overflow error.

```typescript
stream.writeUint8(256);  // 0x100, LSB only written
stream.index = 0;
console.log(stream.readUint8());  // 0 (LSB of 256)
```

**Mitigation:** Assert bounds before writing:
```typescript
console.assert(id >= 0 && id <= 65535, "ID out of range");
stream.writeUint16(id);
```

### Position Bounds Assumption

**Issue:** `writePosition`/`readPosition` assume position is in [0, maxPosition]. Values outside cause undefined behavior:

```typescript
const pos = { x: -100, y: 2000 };  // Negative x
stream.writePosition(pos);
stream.index = 0;
const decoded = stream.readPosition();
// decoded.x will be incorrect (quantization assumes [0, 1924])
```

**Mitigation:** Clamp position before encoding:
```typescript
pos.x = Math.max(0, Math.min(pos.x, GameConstants.maxPosition));
```

### ObstacleRotation Mode Inconsistency

**Issue:** `writeObstacleRotation` uses different encoding depending on RotationMode, but the mode enum value is **not** encoded in the wire format.

```typescript
stream.writeObstacleRotation(45, RotationMode.Full);   // 1 byte
stream.writeObstacleRotation(45, RotationMode.Binary); // 1 byte
// Decoder must **know in advance** which mode was used
```

**Mitigation:** Send the mode/metadata separately, or always uses a fixed mode per obstacle type.

### Buffer Reuse Across Packets

**Issue:** Only `.index` pointer resets, not the buffer contents. Reading stale data from unwritten regions:

```typescript
const stream = new ByteStream(buffer);
stream.writeUint8(42);
stream.index = 0;
const v1 = stream.readUint8();  // 42
const v2 = stream.readUint8();  // Whatever was in buffer[1] before (undefined behavior)
```

**Mitigation:** Always reset `.index = 0` before reading, and ensure written data length is known.

## Related Documents

**Tier 2:**
- [Serialization System](../README.md) — Two-layer encoding architecture overview
- [Networking Subsystem](../../networking/README.md) — How binary packets are transmitted via WebSocket
- [Object Definitions](../../object-definitions/README.md) — Registry pattern for weapons, items, obstacles (1-byte index encoding)

**Tier 3:**
- [UpdatePacket Module](../../networking/modules/updatepacket.md) — Packet structure using SuroiByteStream (position, rotation, scale fields)
- [Game Objects - Server](../../game-objects-server/) — Per-object serialization using quantized encoding

**Tier 1:**
- [API Reference](../../../api-reference.md) — Binary WebSocket protocol specification
- [Data Model](../../../datamodel.md) — Game entities and their serialized representations
- [Architecture](../../../architecture.md) — System-wide data flow and networking layer
