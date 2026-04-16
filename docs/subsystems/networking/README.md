# Networking Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/networking/modules/ -->
<!-- @source: common/src/packets/, common/src/utils/suroiByteStream.ts -->

## Purpose

The networking subsystem implements the binary WebSocket transport layer: every byte sent between
the Bun server and browser clients passes through this subsystem. It provides compact binary
serialization via `SuroiByteStream`, per-packet type classes, and a `PacketStream` multiplexer
that packs or reads multiple packets from a single WebSocket frame.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/packets/packet.ts` | `Packet<DataIn, DataOut>` base class, `PacketType` enum, `DataSplitTypes`, `UpdateFlags` |
| `common/src/packets/packetStream.ts` | `PacketStream` — multiplexes many packets into/out of one `ArrayBuffer` |
| `common/src/utils/suroiByteStream.ts` | `SuroiByteStream` — compact domain-specific binary encoding on top of `ByteStream` |
| `common/src/utils/objectsSerializations.ts` | Per-`ObjectCategory` partial/full serializers (`ObjectSerializations` map) |
| `common/src/packets/updatePacket.ts` | `UpdatePacket` — the main server→client state-sync packet (largest, most complex) |
| `common/src/packets/inputPacket.ts` | `InputPacket` — client→server player controls each tick |
| `common/src/packets/joinPacket.ts` | `JoinPacket` — client→server join request |
| `common/src/packets/joinedPacket.ts` | `JoinedPacket` — server→client join acknowledgment |
| `common/src/packets/mapPacket.ts` | `MapPacket` — server→client full map layout (sent once on join) |
| `common/src/packets/gameOverPacket.ts` | `GameOverPacket` — server→client end-of-game summary |
| `common/src/packets/killPacket.ts` | `KillPacket` — server→client kill/down event for killfeed |
| `common/src/packets/pickupPacket.ts` | `PickupPacket` — server→client item pickup confirmation or inventory message |
| `common/src/packets/spectatePacket.ts` | `SpectatePacket` — client→server spectate action |
| `common/src/packets/reportPacket.ts` | `ReportPacket` — bidirectional player report |
| `common/src/packets/debugPacket.ts` | `DebugPacket` — client→server developer debug controls (dev mode only) |

## Architecture

The subsystem is organized in three layers:

```
┌──────────────────────────────────────────────────────────┐
│  Layer 3: WebSocket transport                             │
│  server: player.sendData(buffer)  →  socket.send(buffer)  │
│  client: socket.onmessage (ArrayBuffer)                   │
├──────────────────────────────────────────────────────────┤
│  Layer 2: PacketStream multiplexer                        │
│  serialize(packetData) — prepend 1-byte type + payload    │
│  deserialize()         — read 1-byte type, dispatch       │
├──────────────────────────────────────────────────────────┤
│  Layer 1: Packet<DataIn, DataOut> classes                 │
│  Each packet: serialize(stream, data) / deserialize(stream)│
│  Encoding via SuroiByteStream domain methods              │
└──────────────────────────────────────────────────────────┘
```

**Framing format (per WebSocket message):**

```
[ type:u8 | payload... ] [ type:u8 | payload... ] ...
```

`PacketStream.serialize` writes the type byte as `packetType - 1` (the `PacketType` enum starts at 0,
but `Disconnect` is 0 and never serialized, so the wire index is `type - 1`).
`PacketStream.deserialize` reads that byte and looks up `Packets[index]`.

**`Packets` registry (`packetStream.ts`):**

```typescript
export const Packets = [
    GameOverPacket,   // index 0 → PacketType.GameOver (1)
    InputPacket,      // index 1 → PacketType.Input (2)
    JoinedPacket,     // index 2 → PacketType.Joined (3)
    JoinPacket,       // index 3 → PacketType.Join (4)
    KillPacket,       // index 4 → PacketType.Kill (5)
    MapPacket,        // index 5 → PacketType.Map (6)
    PickupPacket,     // index 6 → PacketType.Pickup (7)
    ReportPacket,     // index 7 → PacketType.Report (8)
    SpectatePacket,   // index 8 → PacketType.Spectate (9)
    UpdatePacket,     // index 9 → PacketType.Update (10)
    DebugPacket,      // index 10 → PacketType.Debug (11)
] as const;
```

Note: `PacketType.Disconnect` (value `0`) exists in the enum but is not present in the `Packets`
array. Disconnection is handled by the WebSocket `close` event, not a packet.

## Packet Inventory

| Packet | Direction | Purpose |
|--------|-----------|---------|
| `JoinPacket` | Client → Server | Initial join: protocol version, player name, skin, badge, emote slots |
| `JoinedPacket` | Server → Client | Join acknowledged: team mode, team ID, confirmed emote slots |
| `MapPacket` | Server → Client | Full map layout sent once: seed, dimensions, rivers, static obstacles, place names |
| `UpdatePacket` | Server → Client | Per-tick state delta: player data, dirty objects (full/partial), bullets, explosions, gas, killfeed, etc. |
| `InputPacket` | Client → Server | Per-tick player inputs: WASD movement, rotation, mouse distance, action list, ping sequence |
| `KillPacket` | Server → Client | Kill/down event: victim, attacker, weapon, damage source, killfeed bits |
| `GameOverPacket` | Server → Client | End-of-game: rank, per-teammate kills/damage/time-alive stats |
| `PickupPacket` | Server → Client | Item pickup confirmation or inventory-full/locked-slot message |
| `SpectatePacket` | Client → Server | Spectate action: next/previous/specific player (by object ID) |
| `ReportPacket` | Client ↔ Server | Player report: target player ID + report ID string |
| `DebugPacket` | Client → Server | Dev-only: speed override, noclip, invulnerability, spawn loot/dummy, layer offset |

## SuroiByteStream Encoding

`SuroiByteStream` extends the generic `ByteStream` base class with domain-specific methods
that encode game values more compactly than JSON or naïve binary.

The key insight is **ranged float quantization**: instead of 4-byte IEEE 754 floats, values are
written as N-byte unsigned integers that linearly span a known `[min, max]` range. For example,
a world position is written as two `uint16` values (4 bytes total) covering `[0, GameConstants.maxPosition]`
rather than two `float32` values (8 bytes).

### Key Encoding Methods

| Method pair | Encodes | Wire cost |
|-------------|---------|-----------|
| `writePosition(v)` / `readPosition()` | World XY vector, range `[0, maxPosition]` | 4 bytes (2×uint16) |
| `writeVector(v, minX, minY, maxX, maxY, bytes)` / `readVector(...)` | Arbitrary ranged XY vector | `2×bytes` bytes |
| `writeRotation(r)` / `readRotation()` | Full-precision angle (from `ByteStream`) | 1 byte (quantized −π…π) |
| `writeRotation2(r)` / `readRotation2()` | Lower-precision rotation (from `ByteStream`) | 1 byte |
| `writeObstacleRotation(v, mode)` / `readObstacleRotation(mode)` | Rotation packed by `RotationMode` (`Full`=1B float, `Limited`/`Binary`=1B uint) | 1–2 bytes |
| `writeScale(s)` / `readScale()` | Object scale, range `[objectMinScale, objectMaxScale]` | 1 byte (uint8) |
| `writeObjectType(t)` / `readObjectType()` | `ObjectCategory` enum | 1 byte (uint8) |
| `writeObjectId(id)` / `readObjectId()` | Runtime object ID | 2 bytes (uint16) |
| `writeLayer(l)` / `readLayer()` | Map layer (`Layer` enum, signed) | 1 byte (int8) |
| `writePlayerName(n)` / `readPlayerName()` | UTF-8 player name, max `nameMaxLength` bytes, null-terminated | ≤16 bytes |
| `writeBooleanGroup(b…)` / `readBooleanGroup()` | Up to 8 booleans packed into 1 byte | 1 byte |
| `writeBooleanGroup2(b…)` / `readBooleanGroup2()` | Up to 16 booleans packed into 2 bytes | 2 bytes |

Individual `ObjectDefinition` references are serialized by their registry's own `writeToStream` /
`readFromStream` pair (e.g. `Loots.writeToStream(stream, item)`), which writes the definition's
index in its `ObjectDefinitions` registry as a compact integer. See
[Object Definitions](../object-definitions/) for details.

## Data Flow

### Client → Server (input)

```
Keyboard/mouse events
  → InputManager captures state
  → InputPacket.create({ movement, turning, rotation, actions, ... })
  → PacketStream.serialize(inputPacket)      ← 1-byte type + payload
  → socket.send(buffer)                      ← binary WebSocket message

  ── network ──

server socket.onmessage(message: ArrayBuffer)
  → game.onMessage(player, message)
  → PacketStream.deserialize()               ← reads type byte, dispatches
  → packet.type === PacketType.Input
  → player.processInputs(packet)
```

### Server → Client (update, per tick)

```
game.tick() — 40 TPS
  → partialDirtyObjects → object.serializePartial()
  → fullDirtyObjects    → object.serializeFull()
  → player.secondUpdate()
       → builds UpdatePacket (flags, playerData, dirty caches, bullets, ...)
       → sendPacket(updatePacket)  ← queues UpdatePacket
  → player.secondUpdate() finishes
  → _packetStream.stream.index = 0
  → for each queued packet: _packetStream.serialize(packet)
  → sendData(_packetStream.getBuffer())       ← socket.send(buffer)

  ── network ──

client socket.onmessage(message: ArrayBuffer)
  → new PacketStream(message.data)
  → while stream.deserialize(splits) → onPacket(packet)
       → PacketType.Update   → game.processUpdate(packet)
       → PacketType.Kill     → UIManager.processKillPacket(packet)
       → PacketType.GameOver → UIManager.showGameOverScreen(packet)
       → PacketType.Joined   → game.startGame(packet)
       → PacketType.Map      → MapManager.updateFromPacket(packet)
       → PacketType.Pickup   → UIManager / autoPickup
       → PacketType.Report   → UIManager.processReportPacket(packet)
```

Each player has a dedicated `PacketStream` backed by a **64 KB pre-allocated buffer**
(`new ArrayBuffer(1 << 16)`). The stream index is reset to 0 at the start of `secondUpdate`,
allowing buffer reuse without allocation.

## UpdatePacket Delta System

`UpdatePacket` is the most bandwidth-intensive packet. It uses a **16-bit flags word** to make
every section optional; only sections with a non-zero flag bit are written to or read from the
stream.

### Flags (`UpdateFlags` const enum)

| Flag | Bit | Section |
|------|-----|---------|
| `PlayerData` | bit 0 | Local player stats (health, adrenaline, inventory, etc.) |
| `DeletedObjects` | bit 1 | Object IDs that left the player's view or were destroyed |
| `FullObjects` | bit 2 | Objects entering view for the first time (partial + full data) |
| `PartialObjects` | bit 3 | Objects already in view that changed partial state |
| `Bullets` | bit 4 | New bullet trajectories visible to this player |
| `Explosions` | bit 5 | Explosions within visible range |
| `Emotes` | bit 6 | Player emotes from visible players |
| `Gas` | bit 7 | Gas zone state change (position, radius, duration) |
| `GasPercentage` | bit 8 | Gas shrink progress within current stage (0–1, 2 bytes) |
| `NewPlayers` | bit 9 | Players that joined or entered view (name/badge/color) |
| `DeletedPlayers` | bit 10 | Players that left the game |
| `AliveCount` | bit 11 | Current living player count |
| `Planes` | bit 12 | Airdrop plane positions and directions |
| `MapPings` | bit 13 | Map pings (player and global) |
| `MapIndicators` | bit 14 | Map indicator updates (dirty position/definition/dead flags) |
| `KillLeader` | bit 15 | Kill leader player ID and kill count |

The flags are written as a placeholder `uint16(0)` at the start of serialization; after all
sections are written, the stream index is rewound to the flags position and the actual flags
value is written.

### Full vs Partial Objects

Every server-side game object (`BaseGameObject`) maintains two cached byte streams:
- **`partialStream`** — small, per-tick state (position, rotation, animation). Written when the
  object enters `game.partialDirtyObjects`.
- **`fullStream`** — complete state (definition reference, layer, variant, door state, etc.).
  Written when the object enters `game.fullDirtyObjects`, which happens on first visibility or
  after a structural change.

In `player.secondUpdate()`:
- **Objects entering view** → added to `fullObjects` set → serialized under `FullObjects` flag
  (writes `partialStream` + `fullStream` back-to-back).
- **Objects already in view that are dirty** → checked against `game.partialDirtyObjects` and
  `game.fullDirtyObjects`; full-dirty objects skip the partial-only path and are included under
  `FullObjects`.
- **Vanished objects** → IDs pushed to `packet.deletedObjects` → serialized under `DeletedObjects` flag.

Client deserializes via `ObjectSerializations[type].deserializePartial(stream)` /
`deserializeFull(stream)` from `common/src/utils/objectsSerializations.ts`.

### PlayerData Section

`PlayerData` is its own mini-delta: a leading `writeBooleanGroup2` (16 bits) + `writeBooleanGroup`
(8 bits) encodes which of the ~17 optional fields are present (health, adrenaline, shield,
infection, zoom, layer, inventory, items, perks, teammates, etc.). Only present fields are written.
This means on most ticks only 2–4 bytes of player state are transmitted.

### DataSplit Accounting

The `deserialize` callbacks optionally receive a `DataSplit` array and two helper closures
(`saveIndex`, `recordTo`). As each section is deserialized the number of bytes consumed is
accumulated into the appropriate `DataSplitTypes` bucket (PlayerData, Players, Obstacles, Loots,
SyncedParticles, GameObjects, Killfeed). The client uses this breakdown to render a network
traffic graph per category.

## Interfaces & Contracts

```typescript
// Packet base class (common/src/packets/packet.ts)
class Packet<DataIn, DataOut = DataIn> {
    readonly type: PacketType;
    serialize(stream: SuroiByteStream, data: DataIn): void;
    deserialize(stream: SuroiByteStream, splits?: DataSplit): DataOut;
    create(data?: SDeepPartial<DataIn>): SDeepMutable<DataIn>;
}

// PacketStream (common/src/packets/packetStream.ts)
class PacketStream {
    readonly stream: SuroiByteStream;
    constructor(source: SuroiByteStream | ArrayBuffer);
    serialize(data: MutablePacketDataIn): void;        // writes type byte + payload
    deserialize(splits?: DataSplit): PacketDataOut | undefined; // reads type byte, returns packet or undefined at end
    getBuffer(): ArrayBuffer;                           // slices buffer to current stream index
}
```

## Dependencies

- **Depends on:**
  - [Object Definitions](../object-definitions/) — `ObjectDefinitions<T>` registries provide
    `writeToStream` / `readFromStream` for every entity type; all definition references are
    serialized as 1-byte (or variable) indices into these registries.
  - `common/src/utils/objectsSerializations.ts` — per-`ObjectCategory` serializer map used by
    `UpdatePacket` for dirty object encoding.
  - `common/src/constants.ts` — `GameConstants` (maxPosition, nameMaxLength, protocolVersion,
    objectMinScale, objectMaxScale) referenced throughout encoding.

- **Depended on by:**
  - [Game Loop](../game-loop/) — `game.onMessage` dispatches inbound packets;
    `player.secondUpdate()` assembles and sends the `UpdatePacket` each tick.
  - [Client Rendering](../client-rendering/) — `game.onPacket` routes incoming packets to
    object managers and UI.
  - [Inventory Subsystem](../inventory/) — `PickupPacket` notifies client of pickup results;
    `InputPacket` carries `UseItem` / `DropItem` actions.

## Known Gotchas

- **`PacketType.Disconnect` is wire-absent.** It is value `0` in the enum but not in the `Packets`
  array. The wire type byte is `packetType - 1`, meaning `GameOverPacket` is wire byte `0`.
  If you add a new packet, its `PacketType` enum value must be consistent with its `Packets` array
  index + 1.
- **64 KB per-player buffer.** `PacketStream` is backed by `new ArrayBuffer(1 << 16)`. If a
  single tick's total serialized output exceeds 65 535 bytes for one player, the buffer silently
  overflows. This has not been a practical problem but is worth watching for in large team modes.
- **Ping-only InputPacket.** When `pingSeq & 128 !== 0` the serializer writes only the `pingSeq`
  byte and returns immediately — the packet carries no movement data. This is used as a pure RTT
  measurement probe.
- **HTML stripping in player names.** `JoinPacket.deserialize` runs
  `name.split(/<[^>]+>/g).join("").trim()` on the received name to strip HTML tags before use.
- **`DebugPacket` is dev-only.** The server only processes it when `process.env.NODE_ENV === "development"`.

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview and tech stack
- **Tier 1:** [docs/api-reference.md](../../api-reference.md) — Full packet field reference
- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — Core entity types referenced in packets
- **Tier 2:** [Object Definitions](../object-definitions/) — Definition registries used for entity encoding
- **Tier 2:** [Game Loop](../game-loop/) — Tick driver that populates dirty sets and calls `secondUpdate`
- **Tier 2:** [Client Rendering](../client-rendering/) — Consumes deserialized `UpdatePacket` data
- **Patterns:** [patterns.md](patterns.md) — Reusable networking patterns
