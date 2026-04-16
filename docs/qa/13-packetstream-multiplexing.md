# Q13: What is the PacketStream multiplexing model, and can multiple packet types be batched into a single WebSocket frame?

**Answer:**

The **PacketStream multiplexing model** enables multiple packets of different types to be **batched into a single WebSocket binary frame**, significantly reducing network overhead. This is critical for Suroi's 40 TPS game loop where bandwidth efficiency is essential.

---

## Architecture

### Three-Layer Model

```
┌─────────────────────────────────────────┐
│ Layer 3: WebSocket Transport             │
│ socket.send(ArrayBuffer) ← binary frame  │
├─────────────────────────────────────────┤
│ Layer 2: PacketStream Multiplexer        │
│ Serializes/deserializes multiple packets │
│ into/from a single ArrayBuffer           │
├─────────────────────────────────────────┤
│ Layer 1: Packet<DataIn, DataOut> Classes │
│ Each type serializes its payload         │
└─────────────────────────────────────────┘
```

### Packet Framing Format

Each WebSocket frame contains **zero or more packets**, each formatted as:

```
[packet 0]           [packet 1]           [packet N]
┌──────────────────┬──────────────────┬──────────────────┐
│ 1-byte type      │ 1-byte type      │ 1-byte type      │
│ payload bytes    │ payload bytes    │ payload bytes    │
└──────────────────┴──────────────────┴──────────────────┘
```

**Wire format:** `[type:u8 | payload...] [type:u8 | payload...] ...`

---

## Key Implementation Details

### PacketStream Class

`PacketStream` is the multiplexer:

```typescript
export class PacketStream {
    readonly stream: SuroiByteStream;

    serialize(data: MutablePacketDataIn): void {
        const type = data.type - 1;  // Convert enum to 0-based index
        this.stream.writeUint8(type); // Write 1-byte type marker
        Packets[type].serialize(this.stream, data); // Append payload
    }

    deserialize(splits?: DataSplit): PacketDataOut | undefined {
        if (this.stream.buffer.byteLength <= this.stream.index) return;
        
        const type = this.stream.readUint8(); // Read type marker
        return Packets[type].deserialize(this.stream, splits);
    }

    getBuffer(): ArrayBuffer {
        return this.stream.buffer.slice(0, this.stream.index); // Trim to used bytes
    }
}
```

**Key pattern:** 
- `serialize()` writes `type - 1` as a single byte, then the packet payload
- `deserialize()` reads that byte, looks up the packet class in the `Packets` registry, and delegates deserialization
- `getBuffer()` returns only the used portion of the pre-allocated buffer

### Packet Registry

The wire-type ordering maps 0-based bytes to packet types:

| Wire byte | Packet | `PacketType` | 
|-----------|--------|------------|
| 0 | `GameOverPacket` | 1 |
| 1 | `InputPacket` | 2 |
| 2 | `JoinedPacket` | 3 |
| 3 | `JoinPacket` | 4 |
| 4 | `KillPacket` | 5 |
| 5 | `MapPacket` | 6 |
| 6 | `PickupPacket` | 7 |
| 7 | `ReportPacket` | 8 |
| 8 | `SpectatePacket` | 9 |
| 9 | `UpdatePacket` | 10 |
| 10 | `DebugPacket` | 11 |

**Note:** `PacketType.Disconnect` (0) is **not** multiplexed; disconnect is signaled by the WebSocket `close` event.

---

## Server-Side Batching (Per-Tick)

`Player.secondUpdate()` batches multiple packets:

```typescript
// Reset stream buffer for reuse (pre-allocated 64 KB)
this._packetStream.stream.index = 0;

// Batch all queued packets for THIS player
for (const packet of this._packets) {
    this._packetStream.serialize(packet);  // Append to stream
}

// Batch all globally-broadcast packets (same for all players)
for (const packet of this.game.packets) {
    this._packetStream.serialize(packet);  // Append to stream
}

// Single WebSocket transmission
this.sendData(this._packetStream.getBuffer());
```

**Buffer management:**
- Each player has a **dedicated 64 KB pre-allocated `ArrayBuffer`** (`1 << 16 = 65536 bytes`)
- Buffer is **reused every tick** (index reset, not reallocated)
- Multiple packets are serialized sequentially into the same buffer
- Only **used bytes are sent** (via `getBuffer().slice(0, index)`)

**Example batching (typical frame):**
- 1× `UpdatePacket` (largest, ~1-10 KB)
- 1× `KillPacket` (if a player died, ~20 bytes)
- 1× other control packets as needed

---

## Client-Side Demultiplexing

`Game.onmessage()` deserializes all packets in a frame:

```typescript
this._socket.onmessage = (message: MessageEvent<ArrayBuffer>): void => {
    const stream = new PacketStream(message.data);
    const splits = [0, 0, 0, 0, 0, 0, 0] satisfies DataSplit;
    
    while (true) {
        // Keep deserializing until buffer exhausted
        const packet = stream.deserialize(splits);
        if (packet === undefined) break;  // EOF
        
        this.onPacket(packet);  // Dispatch each packet
    }
};
```

**Dispatch logic:** Each packet type is routed to its handler:
- `PacketType.Update` → `game.processUpdate()`
- `PacketType.Kill` → `UIManager.processKillPacket()`
- `PacketType.GameOver` → `UIManager.showGameOverScreen()`
- etc.

---

## Multiplexing Benefits

| Benefit | Impact |
|---------|--------|
| **Single frame header** | TCP/IP + WebSocket overhead amortized across N packets |
| **Buffer reuse** | No GC pressure; 64 KB allocated once per player |
| **Bandwidth efficiency** | 1-byte type marker per packet vs. full packet headers |
| **Deterministic ordering** | Packets deserialized in order sent; no coordination overhead |
| **Tick-aligned batching** | All state changes for a tick sent atomically |

---

## Example: Typical 40 TPS Frame

**Serialization (server):**
```
0x0A (UpdatePacket)  ← type byte (10 - 1 = 9)
[5000 bytes] UpdatePacket payload
0x04 (KillPacket)    ← type byte (5 - 1 = 4)
[20 bytes] KillPacket payload
0x06 (PickupPacket)  ← type byte (7 - 1 = 6)
[8 bytes] PickupPacket payload
```
**Total: 1 WebSocket frame, 5028 bytes, 3 packets, 0 fragmentation**

**Deserialization (client):**
```
Read @ offset 0: type=0x0A → UpdatePacket.deserialize()
Read @ offset 5001: type=0x04 → KillPacket.deserialize()
Read @ offset 5029: type=0x06 → PickupPacket.deserialize()
Read @ offset 5037: EOF → stop deserialization
```

---

## References

- **Tier 1:** `docs/api-reference.md` — Packet framing format and `SuroiByteStream` encoding
- **Tier 2:** `docs/subsystems/networking/README.md` — Overview, packet inventory, SuroiByteStream methods
- **Tier 2:** `docs/subsystems/serialization-system/README.md` — Two-layer encoding stack, ObjectSerializations pattern
- **Tier 3:** `docs/subsystems/networking/modules/protocol.md` — Connection lifecycle and frame structure
- **Source:** `common/src/packets/packetStream.ts` — Multiplexer implementation
- **Source:** `server/src/objects/player.ts` — Server-side batching
- **Source:** `client/src/scripts/game.ts` — Client-side demultiplexing
