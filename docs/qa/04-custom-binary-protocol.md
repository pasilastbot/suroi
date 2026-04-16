# Q4: Why does the project use a custom binary protocol over WebSocket instead of JSON or Protocol Buffers?

**Short answer:** The custom `SuroiByteStream` protocol provides 3–5× better bandwidth efficiency than JSON while remaining simpler and more game-optimized than Protocol Buffers, enabling full game state updates for 100+ objects every 25ms (40 TPS) within strict frame budgets.

---

## Performance Rationale

### 1. **Extreme Real-Time Constraint: 25ms Tick Budget**

The game runs at **40 ticks per second** with a **25ms delta** per tick. Every tick sends an `UpdatePacket` from server to each client via WebSocket. At this frequency:

- **One 1-second game = 40 packets per player**
- **Bandwidth scales linearly with tick rate × map occupancy**

Reducing frame size by 3–5× directly reduces downstream bandwidth and latency jitter.

### 2. **Domain-Specific Quantization: Compact Game Values**

Instead of generic compression, `SuroiByteStream` encodes game values with **known ranges and precision loss** that's imperceptible in gameplay:

| Value | JSON | Binary | Savings |
|-------|------|--------|---------|
| Position (x, y) | `"x":962.123,"y":1000.456` = ~24 bytes | `writePosition()` = 4 bytes (quantized uint16×2) | **6× smaller** |
| Rotation (angle) | `"r":1.5708` = ~8 bytes | `writeRotation()` = 1 byte (quantized) | **8× smaller** |
| Scale | `"s":0.8` = ~5 bytes | `writeScale()` = 1 byte | **5× smaller** |
| Boolean group | `"dead":true,"downed":false,...` = 8+ bytes each | `writeBooleanGroup()` = 1 byte (8 flags) | **8× smaller** |

**Actual UpdatePacket sizes:**

- **PlayerData (typical):** 30–80 bytes binary vs. ~250–400 bytes JSON
- **Player partial update:** 7–10 bytes binary vs. ~50+ bytes JSON
- **Full object create:** 25–50 bytes binary vs. ~100–200 bytes JSON

### 3. **Multi-Object State Delivery**

A 25ms frame typically contains:
- **Server→Client:** 1 `UpdatePacket` with 5–50 visible objects (full or partial serialization)
- **Client→Server:** 1 `InputPacket` with player controls

**JSON overhead per object:**

```json
{
  "id": 42,
  "type": 0,
  "x": 962.0,
  "y": 1000.0,
  "rotation": 1.5708,
  "animation": 5,
  "action": null,
  "scale": 0.8,
  "layer": 3
}
```

vs. **Binary:**

```
[2-byte id][1-byte type][4-byte position][1-byte rotation][1-byte anim/action][1-byte scale][1-byte layer]
= 11 bytes + type-specific data
```

At 40 visible objects × 40 ticks/sec = **1,600 object updates per second**, the 3–5× reduction is **massive**.

---

## Why Not Protocol Buffers?

While Protocol Buffers would provide similar binary compression, there are practical reasons Suroi uses a custom binary protocol:

1. **Game-specific constraints are paramount**
   - Positions must fit the map bounds [0, 1924] → quantize to 2 bytes
   - Rotations wrap [−π, π] → quantize to 1 byte
   - These aren't generic protobuf optimizations; they require domain knowledge

2. **Protocol Buffers add complexity without benefit**
   - Requires code generation (proto files, compilation step)
   - No better compression than hand-crafted quantization
   - Runtime: varint encoding is not always smaller for numeric ranges

3. **Versionability is built-in**
   - `SuroiByteStream` version is tied to `GameConstants.protocolVersion` (currently **73**)
   - Client disconnects immediately if versions mismatch
   - Protobuf versionability adds field tags; custom binary has explicit format control

---

## Implementation Details: How Compact Encoding Works

### Two-Layer Stack

```
┌─────────────────────────────────────┐
│ Domain Layer (SuroiByteStream)      │
│ ├─ writePosition(x, y)       → 4 B   │
│ ├─ writeRotation(angle)     → 1 B   │
│ ├─ writeScale(scale)        → 1 B   │
│ ├─ writeBooleanGroup(8×bool)→ 1 B  │
│ └─ writeVector(x,y,bounds)  → 1–4 B │
├─────────────────────────────────────┤
│ Primitive Layer (ByteStream)        │
│ ├─ writeUint8/16/32/64       → 1–8 B│
│ ├─ writeFloat32/64           → 4–8 B│
│ └─ writeString(utf-8)        → N B  │
├─────────────────────────────────────┤
│ Wire Format (ArrayBuffer)            │
│ └─ Binary data via WebSocket         │
└─────────────────────────────────────┘
```

### Position Quantization Example

**From `common/src/utils/suroiByteStream.ts`:**

```typescript
writePosition(vector: Vector): this {
  this.writeUint16((vector.x / maxPosition) * 65535 + 0.5);
  this.writeUint16((vector.y / maxPosition) * 65535 + 0.5);
  return this;
}
```

- Maps [0, 1924] → [0, 65535] (2 bytes per axis)
- Precision: ±0.015 pixels (imperceptible at game speed)
- **Cost: 4 bytes vs. 16+ bytes in JSON**

---

## Packet Multiplexing Advantage

One WebSocket frame can carry **multiple packets** via `PacketStream`:

```
[1-byte type | payload...] [1-byte type | payload...] ...
```

**Benefits:**
- All updates sent atomically (single WebSocket message ≈ one network RTT)
- Tight packing of types: `PacketType - 1` = wire type (saves 1 byte overhead per packet)
- No JSON array wrapper overhead

---

## Visibility Culling + Binary = Efficient Scaling

The binary protocol works **synergistically** with visibility culling:

- Server only sends objects visible to each player (5–50 objects, not 200+)
- Each object: 7–50 bytes binary vs. 50–200+ bytes JSON
- **Result:** Bandwidth scales with view distance, not server population

---

## Trade-Off: Byte Efficiency vs. Debuggability

**Cost of custom binary:**
- Not human-readable in packet captures (use dev tools to decode)
- Protocol version **must** increment on **any** serialization format change

**Benefit:**
- 3–5× bandwidth reduction = **affordable scale** for a browser-based game

---

## Summary

Suroi uses a custom binary protocol because:

1. **Real-time constraint:** 40 TPS with 25ms frame buffer demands compact serialization
2. **Game-specific optimization:** Position/rotation/scale quantization beats generic compression
3. **Scale:** Multiplayer games with 100+ objects × 40 Hz require every byte to count
4. **Simplicity over generality:** Hand-tuned domain layer beats generic Protocol Buffers
5. **Bandwidth constraint:** Consumer broadband can't afford 200+ bytes/tick when 50 bytes suffices

---

## References

- **Tier 1:** `docs/api-reference.md` — Binary WebSocket protocol overview
- **Tier 2:** `docs/subsystems/networking/README.md` — Networking architecture & UpdatePacket flags
- **Tier 2:** `docs/subsystems/serialization-system/README.md` — SuroiByteStream compression strategies
- **Source:** `common/src/utils/suroiByteStream.ts` — Position, rotation, scale quantization implementation
- **Source:** `common/src/constants.ts` — `GameConstants.protocolVersion`, `maxPosition`, quantization bounds
