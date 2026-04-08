# Protocol Reference

<!-- @tier: 1 -->
<!-- @see-also: docs/subsystems/packets/, docs/datamodel.md -->
<!-- @source: common/src/packets/, common/src/utils/byteStream.ts, common/src/utils/suroiByteStream.ts -->
<!-- @updated: 2026-03-04 -->

## Overview

Suroi uses a **custom binary protocol** over WebSockets.

**Documentation index:** See [content-plan.md](content-plan.md) for the full documentation index and status. All game data is packed into dense `ArrayBuffer` frames using a custom serialization layer called `SuroiByteStream`. There is no JSON — every byte is hand-serialized to minimize bandwidth at 40 TPS.

**Current protocol version: 73** (`GameConstants.protocolVersion` in `common/src/constants.ts`)

## Binary Serialization Layer

### ByteStream (base class)

`common/src/utils/byteStream.ts` — Base read/write primitives over a raw `ArrayBuffer` backed by a `DataView`.

| Method | Size | Description |
|--------|------|-------------|
| `readUint8` / `writeUint8` | 1 byte | Unsigned 8-bit integer (0–255) |
| `readInt8` / `writeInt8` | 1 byte | Signed 8-bit integer (-128–127) |
| `readUint16` / `writeUint16` | 2 bytes | Unsigned 16-bit integer (0–65535) |
| `readInt16` / `writeInt16` | 2 bytes | Signed 16-bit integer |
| `readUint24` / `writeUint24` | 3 bytes | Unsigned 24-bit integer (0–16777215) |
| `readUint32` / `writeUint32` | 4 bytes | Unsigned 32-bit integer |
| `readFloat16` / `writeFloat16` | 2 bytes | Half-precision float |
| `readFloat32` / `writeFloat32` | 4 bytes | Full-precision float |
| `readFloat` / `writeFloat` | N bytes | Float mapped to [min, max] range in N bytes |
| `readBoolean` / `writeBoolean` | 1 byte | Boolean (0 or 1) |
| `readBooleanGroup` | 1 byte | Packs up to 8 booleans into 1 byte |
| `readBooleanGroup2` | 2 bytes | Packs up to 16 booleans into 2 bytes |
| `readRotation` / `writeRotation` | 2 bytes | Rotation angle as half-precision float |
| `readString` / `writeString` | N+1 bytes | Null-terminated UTF-8 string up to N chars |

`readBooleanGroup` / `readBooleanGroup2` are used extensively to pack presence flags, avoiding per-field type bytes.

### SuroiByteStream (game-specific extension)

`common/src/utils/suroiByteStream.ts` — Extends `ByteStream` with game-domain methods:

| Method | Encoding | Description |
|--------|----------|-------------|
| `writePosition` / `readPosition` | 4 bytes (2×uint16) | World position mapped to `[0, 1924]` |
| `writeVector` / `readVector` | N×2 bytes | 2D vector with configurable precision |
| `writeObjectType` / `readObjectType` | 1 byte | `ObjectCategory` enum value |
| `writeObjectId` / `readObjectId` | 2 bytes | Object instance ID (uint16) |
| `writeScale` / `readScale` | 1 byte | Scale mapped to `[0.15, 3.0]` |
| `writeObstacleRotation` / `readObstacleRotation` | 1–2 bytes | Rotation given a `RotationMode` |
| `writeLayer` / `readLayer` | 1 byte (int8) | Layer enum value |
| `writePlayerName` / `readPlayerName` | ≤17 bytes | Null-terminated, max 16 UTF-8 chars |

**Position encoding:** Each coordinate maps a float in `[0, 1924]` to a uint16 in `[0, 65535]` — 2 bytes per axis, 4 bytes per position, ~0.03 unit precision.

**Scale encoding:** 1 byte maps `[0.15, 3.0]` → `[0, 255]`.

**Definition index encoding:** If a definition list has ≤255 entries: 1 byte. If >255: 2 bytes.

## Packet System

### PacketStream

`common/src/packets/packetStream.ts` — Wraps `SuroiByteStream` and handles framing.

Each WebSocket message is one `PacketStream` buffer. A buffer can contain **one or more packets** read sequentially until the buffer is exhausted.

```
Buffer layout:
┌──────────┬──────────────────────────────────────────┐
│  type    │  packet payload                           │
│  1 byte  │  (variable length, packet-specific)       │
├──────────┼──────────────────────────────────────────┤
│  type    │  next packet payload                      │
│  ...     │  ...                                      │
└──────────┴──────────────────────────────────────────┘
```

The `type` byte is `PacketType - 1` (0-indexed).

### Packet Class

```typescript
class Packet<DataIn, DataOut = DataIn> {
    readonly type: PacketType
    serialize: (stream: SuroiByteStream, data: DataIn) => void
    deserialize: (stream: SuroiByteStream, splits?: DataSplit) => DataOut
    create(data?: Partial<DataIn>): DataIn
}
```

Each packet has a `serialize` function (called by the sender) and a `deserialize` function (called by the receiver). The `DataSplit` mechanism records byte offsets for each data category in the `UpdatePacket` to enable efficient partial updates.

## Packet Reference

### PacketType Enum

```typescript
export enum PacketType {
    Disconnect,  // 0
    GameOver,    // 1
    Input,       // 2
    Joined,      // 3
    Join,        // 4
    Kill,        // 5
    Map,         // 6
    Pickup,      // 7
    Report,      // 8
    Spectate,    // 9
    Update,      // 10
    Debug        // 11
}
```

### JoinPacket (Client → Server)

Sent once immediately after the WebSocket connection is established.

| Field | Type | Description |
|-------|------|-------------|
| `protocolVersion` | uint16 | Must match `GameConstants.protocolVersion` |
| `name` | string (≤16 chars) | Player display name |
| `isMobile` | boolean | Whether the client is on mobile |
| `skin` | definition index | Selected skin |
| `badge` | definition index (optional) | Selected badge |
| `emotes[0..7]` | definition indices (optional) | Up to 8 equipped emotes |

The server disconnects the client immediately if `protocolVersion` doesn't match.

### JoinedPacket (Server → Client)

Sent in response to a valid `JoinPacket`. Contains initial game state.

Fields include the player's assigned ID, team information, and initial map data.

### MapPacket (Server → Client)

Describes the current map: dimensions, terrain, rivers, buildings, obstacles, and other static world data. Sent once on join.

### InputPacket (Client → Server)

Sent every frame with the current player inputs.

| Field | Type | Description |
|-------|------|-------------|
| `moving` | boolean | Whether player is moving |
| `moveAngle` | rotation | Direction of movement |
| `attacking` | boolean | Whether firing/attacking |
| `turning` | boolean | Whether aiming direction changed |
| `rotation` | rotation | Player aim direction |
| `actions` | `InputActions[]` | Discrete actions this frame (equip, reload, interact, etc.) |

`InputActions` enum:

| Action | Description |
|--------|-------------|
| `EquipItem` | Equip a weapon slot |
| `EquipLastItem` | Re-equip previously active item |
| `EquipOtherWeapon` | Switch between gun slots |
| `DropWeapon` | Drop current weapon |
| `DropItem` | Drop a non-weapon item |
| `SwapGunSlots` | Swap primary and secondary |
| `LockSlot` / `UnlockSlot` / `ToggleSlotLock` | Lock a weapon slot |
| `Interact` | Interact with world object |
| `Reload` | Reload current gun |
| `Cancel` | Cancel current action |
| `UseItem` | Use a healing item |
| `Emote` | Play an emote |
| `MapPing` | Place a map ping |
| `Loot` | Pick up nearby loot |
| `ExplodeC4` | Detonate placed C4 |

### UpdatePacket (Server → Client)

The primary server→client packet, sent every tick (40 Hz). Contains **delta-compressed** game state: only fields that changed since the last tick are included.

The payload is divided into named **data splits** (tracked via `DataSplit`) to allow the client to measure bandwidth by category:

| Split | Category |
|-------|----------|
| `PlayerData` | The local player's stats (health, adrenaline, inventory, etc.) |
| `Players` | All visible players' full/partial update data |
| `Obstacles` | Obstacles (create/update/destroy) |
| `Loots` | Loot items (create/update/destroy) |
| `SyncedParticles` | Server-side particles |
| `GameObjects` | Buildings, decals, parachutes, projectiles, death markers |
| `Killfeed` | Kill feed events |

Each object in an update comes in one of two forms:
- **Full data** — sent when the object first enters the player's view
- **Partial data** — delta update with only changed fields

### KillPacket (Server → Client)

Sent when any player in the game gets a kill. Contains killer, victim, weapon used, and kill streak info.

### GameOverPacket (Server → Client)

Sent to a player when the game ends for them (died or won). Contains final stats: kills, damage dealt, time survived, placement.

### PickupPacket (Server → Client)

Sent when the local player picks up an item. Contains the item definition and the `InventoryMessages` result code:

| Code | Meaning |
|------|---------|
| `NotEnoughSpace` | Backpack full |
| `ItemAlreadyEquipped` | Duplicate equipped item |
| `BetterItemEquipped` | Already have a better version |
| `CannotUseFlare` | Special case for flare items |

### SpectatePacket (Client → Server)

Sent after the player dies to spectate another player.

`SpectateActions`:

| Action | Description |
|--------|-------------|
| `BeginSpectating` | Start spectating (after death) |
| `SpectatePrevious` | Switch to previous player |
| `SpectateNext` | Switch to next player |
| `SpectateSpecific` | Spectate a specific player by ID |
| `SpectateKillLeader` | Spectate the kill leader |
| `Report` | Report the spectated player |

### ReportPacket (Client → Server)

Sent when a player reports another player.

### DebugPacket (Server → Client)

Development-only packet for debug visualizations. Not used in production.

## Object Serialization

### ObjectSerializations

`common/src/utils/objectsSerializations.ts` defines per-category full and partial serialization. Each category has:
- **Full data** — complete object state (sent when object first becomes visible)
- **Partial data** (on some categories) — delta fields (sent when something changes)

Object update structure in `UpdatePacket`:

```
For each created/updated object:
┌──────────┬──────────────────────────────┐
│  Object  │  Full flag (1 bit)           │
│  ID      │  + full or partial data      │
│  2 bytes │  (category-specific layout)  │
└──────────┴──────────────────────────────┘
```

## Version Management

The protocol version (`GameConstants.protocolVersion`) must be incremented any time the wire format changes:

- Adding or removing fields in any packet
- Adding or removing items in any `ObjectDefinitions` list (changes indices)
- Changing encoding of any field (byte width, range, etc.)

The client sends its version in `JoinPacket`. The server rejects connections with a mismatched version, preventing deserialization errors.

**Current version: 73**

## Subsystem References

- [Packets](subsystems/packets/) — Tier 2 documentation for individual packet implementations
- [Objects](subsystems/objects/) — How object categories map to serialization formats

## Known Issues / Tech Debt

- **No backward compatibility:** Protocol version mismatch means immediate disconnect. No graceful degradation.
- **Disconnect packet:** `PacketType.Disconnect` exists in the enum but is not in the `Packets` array — it is handled at the WebSocket level (close reason), not as a serialized packet.

## Related Documents

- **Tier 1:** [architecture.md](architecture.md) — System design and WebSocket connection lifecycle
- **Tier 1:** [datamodel.md](datamodel.md) — Game entities that are serialized in packets
- **Tier 1:** [development.md](development.md) — When and how to bump the protocol version
