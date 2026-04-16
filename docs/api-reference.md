# API Reference — WebSocket Packet Protocol

<!-- @tier: 1 -->
<!-- @see-also: docs/subsystems/networking/ -->

## Overview

Suroi uses a **binary WebSocket protocol** for all game communication.
Encoding is performed by `SuroiByteStream`, a custom compact encoding layer that
extends a base `ByteStream` class. Multiple packets are multiplexed per
WebSocket frame via `PacketStream`.

- **Protocol version:** `73` (from `GameConstants.protocolVersion` in [common/src/constants.ts](../common/src/constants.ts))
- **Disconnect** is signalled by closing the WebSocket, not by a packet.

---

## Transport

| Property | Value |
|----------|-------|
| Protocol | WebSocket (binary frames) |
| Encoding | Custom binary via `SuroiByteStream extends ByteStream` |
| Multiplexing | `PacketStream` writes / reads multiple `Packet` instances per frame |
| Server runtime | Bun native WebSockets (no ws / socket.io) |

### HTTP + WebSocket Endpoints

Found in [server/src/server.ts](../server/src/server.ts):

| Path | Protocol | Direction | Purpose |
|------|----------|-----------|---------|
| `GET /api/serverInfo` | HTTP | Server→Client | Returns protocol version, player count, mode info, punishment status |
| `GET /api/getGame` | HTTP | Server→Client | Returns a game ID for the client to connect to |
| `WS /team` | WebSocket (JSON) | Bidirectional | Custom team management lobby; messages are JSON (`CustomTeamMessage`), **not** binary |

The **game play** WebSocket is served by per-game worker processes managed by
`GameManager`. Its path/port is not part of the primary Bun server seen in
`server.ts`.

---

## `SuroiByteStream` Encoding

`@file common/src/utils/suroiByteStream.ts`

`SuroiByteStream` adds game-specific encode/decode methods on top of the
generic `ByteStream` primitives.

### Custom Game-Specific Methods

| Method pair | Bytes | Description |
|-------------|-------|-------------|
| `writeVector(v, minX,minY,maxX,maxY, bytes)` / `readVector(...)` | `2×bytes` | 2D vector; each axis mapped to `bytes`-byte integer over `[min,max]`. `bytes` ∈ {1,2,3,4} |
| `writePosition(v)` / `readPosition()` | 4 (2+2) | Position clamped to `[0, GameConstants.maxPosition=1924]`; each axis is a `uint16` |
| `writeObjectType(t)` / `readObjectType()` | 1 | `ObjectCategory` enum value as `uint8` |
| `writeObjectId(id)` / `readObjectId()` | 2 | Object network ID as `uint16` |
| `writeObstacleRotation(v, mode)` / `readObstacleRotation(mode)` | 1 or 2 | `RotationMode.Full` → 2-byte `writeRotation`; `Limited` / `Binary` → 1-byte `uint8` |
| `writeScale(s)` / `readScale()` | 1 | Scale mapped from `[0.15, 3]` (objectMinScale/objectMaxScale) into a `uint8` |
| `writeLayer(l)` / `readLayer()` | 1 | Layer as `int8` |
| `writePlayerName(n)` / `readPlayerName()` | ≤16 | Null-terminated UTF-8; max `GameConstants.player.nameMaxLength = 16` bytes |

### Inherited `ByteStream` Primitives (used throughout all packets)

| Method | Bytes | Notes |
|--------|-------|-------|
| `writeUint8` / `readUint8` | 1 | Unsigned byte |
| `writeInt8` / `readInt8` | 1 | Signed byte |
| `writeUint16` / `readUint16` | 2 | |
| `writeUint24` / `readUint24` | 3 | Used for 24-bit name colour |
| `writeUint32` / `readUint32` | 4 | |
| `writeFloat(v, min, max, bytes)` / `readFloat(...)` | `bytes` | Maps a float in `[min,max]` to an integer of `bytes` bytes |
| `writeFloat32` / `readFloat32` | 4 | IEEE 754 single-precision |
| `writeRotation` / `readRotation` | 2 | Full-range rotation as 2-byte float |
| `writeRotation2` / `readRotation2` | 1 | Compact rotation as 1-byte float |
| `writeBooleanGroup(...bools)` / `readBooleanGroup()` | 1 | Up to 8 booleans packed into 1 byte |
| `writeBooleanGroup2(...bools)` / `readBooleanGroup2()` | 2 | Up to 16 booleans packed into 2 bytes |
| `writeArray(arr, fn, sizeBits?)` / `readArray(fn, sizeBits?)` | varies | Length-prefixed array; `sizeBits` defaults to 1-byte length |
| `writeSet(set, fn, sizeBits?)` | varies | Like `writeArray` but from a `Set` |
| `writeStream(other)` | varies | Appends another byte stream inline |
| `writeString(maxLen, str)` / `readString(maxLen)` | `maxLen` | Fixed-max-length string |

---

## Packet Architecture

`@file common/src/packets/packet.ts`

```typescript
class Packet<DataIn extends BasePacketData, DataOut = DataIn> {
    readonly type: PacketType;
    serialize(stream: SuroiByteStream, data: DataIn): void;
    deserialize(stream: SuroiByteStream, splits?: DataSplit): DataOut;
    create(data?: Partial<DataIn>): DataIn;
}
```

- `DataIn` is the **server-side** (or write-side) type; `DataOut` is the
  **client-side** (or read-side) type. They differ meaningfully only for
  `UpdatePacket` where the server sends pre-cached byte streams (`ServerOnly`)
  and the client receives deserialised objects (`ClientOnly`).
- The `splits` parameter (type `DataSplit`) is an optional byte-budget tracker
  filled in during deserialization (see [DataSplit Tracking](#datasplit-tracking)).

### `PacketType` Enum

`@file common/src/packets/packet.ts`

| Name | Value |
|------|-------|
| `Disconnect` | 0 |
| `GameOver` | 1 |
| `Input` | 2 |
| `Joined` | 3 |
| `Join` | 4 |
| `Kill` | 5 |
| `Map` | 6 |
| `Pickup` | 7 |
| `Report` | 8 |
| `Spectate` | 9 |
| `Update` | 10 |
| `Debug` | 11 |

> **Note:** `Disconnect` (0) is **not** serialized via `PacketStream`; the
> disconnect event is handled by the WebSocket `close` callback.

---

## Packet Framing

`@file common/src/packets/packetStream.ts`

```
For each packet in a WebSocket frame:
  [1 byte]   wire type = PacketType - 1   (zero-based index into Packets[])
  [N bytes]  packet body
```

`PacketStream.serialize(data)` writes `data.type - 1` as a `uint8`, then calls
the corresponding `Packet.serialize()` method. `PacketStream.deserialize()`
reads until the buffer is exhausted, returning one packet per call.

The `Packets` registry (in source order = wire-type order):

| Wire byte | Packet | `PacketType` value |
|-----------|--------|-------------------|
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

---

## Packet Types

---

### `JoinPacket` — `@file common/src/packets/joinPacket.ts`

**Direction:** Client → Server  
**Purpose:** Sent by the client immediately after opening the game WebSocket.
Carries the protocol version, player cosmetics, and emote loadout.

**Serialized Fields:**

| Field | Encoding | Notes |
|-------|----------|-------|
| Flags: `isMobile`, `hasBadge`, `emote[0..7]` present | `writeBooleanGroup2` | 2 bytes; 10 bits used |
| `protocolVersion` | `writeUint16` | Must equal `GameConstants.protocolVersion` (73) |
| `name` | `writePlayerName` | Null-terminated UTF-8, max 16 bytes; HTML tags stripped server-side |
| `skin` | `Loots.writeToStream` | 1-byte definition index |
| `badge` _(conditional: hasBadge)_ | `Badges.writeToStream` | 1-byte definition index |
| `emotes[i]` _(conditional: emote[i] present, i=0..7)_ | `Emotes.writeToStream` | 1-byte definition index each |

---

### `JoinedPacket` — `@file common/src/packets/joinedPacket.ts`

**Direction:** Server → Client  
**Purpose:** Server acknowledgement that the player has joined. Carries team
assignment and the server-validated emote loadout.

**Serialized Fields:**

| Field | Encoding | Notes |
|-------|----------|-------|
| `teamMode` | `writeUint8` | `TeamMode` enum value |
| `teamID` _(conditional: teamMode ≠ Solo)_ | `writeUint8` | Team identifier |
| Emote presence flags `[0..7]` | `writeBooleanGroup` | 1 byte |
| `emotes[i]` _(conditional: emote[i] present)_ | `Emotes.writeToStream` | 1 byte each |

---

### `InputPacket` — `@file common/src/packets/inputPacket.ts`

**Direction:** Client → Server  
**Purpose:** Per-frame player input: movement, aiming, attacking, and discrete
actions (item use, emotes, map pings, etc.).

**Serialized Fields:**

| Field | Encoding | Notes |
|-------|----------|-------|
| `pingSeq` | `writeUint8` | Sequence number; if bit 7 is set (`pingSeq & 128 !== 0`) **only this byte is written** (ping-only frame) |
| Movement flags + state flags | `writeBooleanGroup` | 1 byte: `up, down, left, right, isMobile, mobile.moving, turning, attacking` |
| `mobile.angle` _(conditional: isMobile)_ | `writeRotation2` | 1 byte |
| `rotation` _(conditional: turning)_ | `writeRotation2` | 1 byte |
| `distanceToMouse` _(conditional: turning)_ | `writeFloat(0, 256, 2)` | 2 bytes |
| `actions` array | `writeArray` | Variable-length; see action sub-fields below |

**Action sub-fields (inside `actions` array):**

| Action Type | Additional Fields | Encoding |
|-------------|-------------------|----------|
| `EquipItem`, `DropWeapon`, `LockSlot`, `UnlockSlot`, `ToggleSlotLock` | `slot` packed into bits [7:6] of the type byte | `uint8` (type + slot<<6) |
| `DropItem` | `item` | `Loots.writeToStream` (1 byte) |
| `UseItem` | `item` | `Loots.writeToStream` (1 byte) |
| `Emote` | `emote` | `Emotes.writeToStream` (1 byte) |
| `MapPing` | `ping` + `position` | `MapPings.writeToStream` (1 byte) + `writePosition` (4 bytes) |
| All other `SimpleInputActions` | _(none)_ | type byte only |

---

### `MapPacket` — `@file common/src/packets/mapPacket.ts`

**Direction:** Server → Client  
**Purpose:** Sent once on join. Describes the entire static map layout —
dimensions, rivers, all obstacle/building placements, and named locations.

**Serialized Fields:**

| Field | Encoding | Notes |
|-------|----------|-------|
| `seed` | `writeUint32` | 4 bytes; deterministic map seed |
| `width` | `writeUint16` | Map width in units |
| `height` | `writeUint16` | Map height in units |
| `oceanSize` | `writeUint16` | Ocean border size |
| `beachSize` | `writeUint16` | Beach border size |
| `rivers` array | `writeArray` (2-byte count) | See river sub-fields |
| `objects` array | `writeArray` (2-byte count) | See object sub-fields |
| `places` array | `writeArray` | See place sub-fields |

**River sub-fields:**

| Field | Encoding |
|-------|----------|
| `width` | `writeUint8` |
| `points` array | `writeArray` of `writePosition` (4 bytes each) |
| `isTrail` | `writeUint8` (`-1` if trail, `0` otherwise) |

**Object sub-fields (`ObjectCategory.Obstacle`):**

| Field | Encoding | Notes |
|-------|----------|-------|
| `type` | `writeObjectType` | 1 byte (`ObjectCategory.Obstacle`) |
| `position` | `writePosition` | 4 bytes |
| `definition` | `Obstacles.writeToStream` | 1 byte |
| `obstacleData` | `writeUint8` | Variation bits packed into MSBs |
| `rotation` _(conditional: RotationMode.Full)_ | `writeRotation` | 2 bytes; written after obstacleData |
| _(RotationMode.Limited / Binary)_ : rotation packed into LSBs of `obstacleData` | _(no extra write)_ | |

**Object sub-fields (`ObjectCategory.Building`):**

| Field | Encoding |
|-------|----------|
| `type` | `writeObjectType` (1 byte) |
| `position` | `writePosition` (4 bytes) |
| `definition` | `Buildings.writeToStream` (1 byte) |
| `orientation` | `writeObstacleRotation(RotationMode.Limited)` (1 byte) |
| `layer` | `writeLayer` (1 byte) |

**Place sub-fields:**

| Field | Encoding |
|-------|----------|
| `name` | `writeString(24)` |
| `position` | `writePosition` (4 bytes) |

---

### `UpdatePacket` — `@file common/src/packets/updatePacket.ts`

**Direction:** Server → Client  
**Purpose:** The main per-tick game state update. Every section is gated by a
16-bit flags field — only non-empty sections are written.

#### `UpdateFlags` Bitmask

| Flag | Bit | Section |
|------|-----|---------|
| `PlayerData` | 0 | Local player stats |
| `DeletedObjects` | 1 | Removed object IDs |
| `FullObjects` | 2 | New/full-update objects |
| `PartialObjects` | 3 | Position/rotation-only updates |
| `Bullets` | 4 | Bullet spawns |
| `Explosions` | 5 | Explosion events |
| `Emotes` | 6 | Emote triggers |
| `Gas` | 7 | Gas zone state change |
| `GasPercentage` | 8 | Gas transition progress |
| `NewPlayers` | 9 | Players entering view |
| `DeletedPlayers` | 10 | Players leaving view |
| `AliveCount` | 11 | Remaining player count |
| `Planes` | 12 | Airdrop plane positions |
| `MapPings` | 13 | Map ping events |
| `MapIndicators` | 14 | Map marker updates |
| `KillLeader` | 15 | Kill leader update |

#### Frame Layout

```
[uint16] flags
[conditional] PlayerData section
[conditional] DeletedObjects section
[conditional] FullObjects section
[conditional] PartialObjects section
[conditional] Bullets section
[conditional] Explosions section
[conditional] Emotes section
[conditional] Gas section
[conditional] GasPercentage section
[conditional] NewPlayers section
[conditional] DeletedPlayers section
[conditional] AliveCount section
[conditional] Planes section
[conditional] MapPings section
[conditional] MapIndicators section
[conditional] KillLeader section
```

#### PlayerData Section

Written via `serializePlayerData()`. Uses a 2-byte + 1-byte presence bitfield
(`writeBooleanGroup2` + `writeBooleanGroup`) to indicate which optional fields
follow.

| Sub-field | Encoding | Condition |
|-----------|----------|-----------|
| Presence flags | `writeBooleanGroup2` (2 bytes) + `writeBooleanGroup` (1 byte) | Always |
| `pingSeq` | `writeUint8` | Always |
| `minMax.maxHealth`, `minAdrenaline`, `maxAdrenaline` | 3 × `writeFloat32` | `hasMinMax` |
| `health` | `writeFloat(0,1,2)` | `hasHealth` |
| `adrenaline` | `writeFloat(0,1,2)` | `hasAdrenaline` |
| `shield` | `writeFloat(0,1,2)` | `hasShield` |
| `infection` | `writeFloat(0,1,2)` | `hasInfection` |
| `zoom` | `writeUint8` | `hasZoom` |
| `layer` | `writeLayer` | `hasLayer` |
| `id.spectating` + `id.id` | `writeUint8(-1 or 0)` + `writeObjectId` | `hasId` |
| `teammates` array | array of (status byte + id + position + normalizedHealth + colorIndex) | `hasTeammates` |
| `highlightedPlayers` array | array of (id + normalizedHealth float 0-1 1B) | `hasHighlightedPlayers` |
| Inventory: `activeWeaponIndex` + weapon slots | `writeUint8` + `writeBooleanGroup` + per-slot (definition + optional count/kills) | `hasInventory` |
| `lockedSlots` | `writeUint8` | `hasLockedSlots` |
| Items bitfield + non-zero counts + `scope` | bitfield chunks + `writeUint16` per count + `Scopes.writeToStream` | `hasItems` |
| `activeC4s` | `writeUint8(-1 or 0)` | `hasActiveC4s` |
| `perks` array | array of `Perks.writeToStream` | `hasPerks` |
| `teamID` | `writeUint8` | `hasTeamID` |
| `blockEmoting` | packed into `writeBooleanGroup2` | Always |

#### DeletedObjects Section

Array (2-byte count) of `writeObjectId` values.

#### FullObjects / PartialObjects Sections — Delta Object System

`fullDirtyObjects` (FullObjects flag) and `partialDirtyObjects` (PartialObjects
flag) implement the server's bandwidth optimisation:

- **FullObjects** — sent when an object enters a client's view or has a
  definition-level change. Each entry: `writeObjectId` + `writeObjectType` +
  `serializePartial(...)` + `serializeFull(...)` (2-byte count).
- **PartialObjects** — sent every tick for objects whose mutable state changed
  (position, rotation, animation). Each entry: `writeObjectId` +
  `writeObjectType` + `serializePartial(...)` only (2-byte count).

The per-category serialization is defined in
`ObjectSerializations` — see [Object Serializations](#object-serializations).

#### Bullets Section

Array (1-byte count) of `BaseBullet.serialize(stream)` calls.

#### Explosions Section

Array (1-byte count): each entry = `Explosions.writeToStream` + `writePosition`
+ `writeLayer`.

#### Emotes Section

Array (1-byte count): each entry = `Emotes.writeToStream` + `writeObjectId`
(player ID).

#### Gas Section

| Field | Encoding |
|-------|----------|
| `state` | `writeUint8` (`GasState` enum) |
| `currentDuration` | `writeUint8` |
| `oldPosition` | `writePosition` (4 bytes) |
| `newPosition` | `writePosition` (4 bytes) |
| `oldRadius` | `writeFloat(0, 2048, 2)` |
| `newRadius` | `writeFloat(0, 2048, 2)` |
| `finalStage` | `writeBooleanGroup` (1 byte) |

#### GasPercentage Section

Single `writeFloat(0, 1, 2)` — progress through the current gas transition.

#### NewPlayers Section

Array (1-byte count): each entry:

| Field | Encoding |
|-------|----------|
| `id` | `writeObjectId` (2 bytes) |
| `name` | `writePlayerName` (≤16 bytes) |
| Decoration flags | `writeUint8`: bit 1 = `hasColor`, bit 0 = `hasBadge` |
| `nameColor` _(conditional: hasColor)_ | `writeUint24` (3 bytes) |
| `badge` _(conditional: hasBadge)_ | `Badges.writeToStream` (1 byte) |

#### DeletedPlayers Section

Array (1-byte count) of `writeObjectId` values.

#### AliveCount Section

Single `writeUint8`.

#### Planes Section

Array (1-byte count): each entry:

| Field | Encoding |
|-------|----------|
| `position` | `writeVector(pos, -1924,-1924,3848,3848, 3)` (6 bytes) |
| `direction` | `writeRotation2` (1 byte) |

#### MapPings Section

Array (default count): each entry = `MapPings.writeToStream` + `writePosition`
+ `writeObjectId(playerId)` if `definition.isPlayerPing`.

#### MapIndicators Section

Array (default count): each entry:

| Field | Encoding |
|-------|----------|
| `id` | `writeUint8` |
| Dirty flags: `positionDirty`, `definitionDirty`, `dead` | `writeBooleanGroup` (1 byte) |
| `position` _(conditional: positionDirty)_ | `writePosition` (4 bytes) |
| `definition` _(conditional: definitionDirty)_ | `MapIndicators.writeToStream` (1 byte) |

#### KillLeader Section

| Field | Encoding |
|-------|----------|
| `id` | `writeObjectId` (2 bytes) |
| `kills` | `writeUint8` |

---

### `KillPacket` — `@file common/src/packets/killPacket.ts`

**Direction:** Server → Client  
**Purpose:** Kill/down event for the killfeed. Also conveys who performed the
kill and with what weapon.

**Serialized Fields:**

`kfData` is a single `uint16` packing multiple flags:

```
bit [4:0]  damageSource  (DamageSources enum, max 32 values)
bit [5]    killed
bit [6]    downed
bit [8:7]  credited state  (00=none, 01=victim===credited, 10=credited differs)
bit [9]    hasAttackerId
```

| Field | Encoding | Condition |
|-------|----------|-----------|
| `kfData` | `writeUint16` | Always |
| `victimId` | `writeObjectId` | Always |
| `attackerId` | `writeObjectId` | `hasAttackerId` |
| `creditedId` | `writeObjectId` | `hasCreditedId && not victim` |
| `kills` | `writeUint8` | `hasAttackerId || hasCreditedId` |
| `weaponUsed` (Gun) | `Guns.writeToStream` | `damageSource === Gun` |
| `weaponUsed` (Melee) | `Melees.writeToStream` | `damageSource === Melee` |
| `weaponUsed` (Throwable) | `Throwables.writeToStream` | `damageSource === Throwable` |
| `weaponUsed` (Explosion) | `Explosions.writeToStream` | `damageSource === Explosion` |
| `weaponUsed` (Obstacle) | `Obstacles.writeToStream` | `damageSource === Obstacle` |
| `killstreak` | `writeUint8` | weapon has `killstreak` property |

---

## Known Issues & Gotchas

- **Protocol version is strict:** `GameConstants.protocolVersion: 73` is baked into every client and server at build time. Client connects send their protocol version in `JoinPacket`; server immediately disconnects if mismatch. Incrementing protocol requires new client build **and** all active clients must reconnect. No fallback exists.
- **UpdatePacket growth is unbounded:** `UpdatePacket` serialization dynamically includes `fullDirtyObjects` and `partialDirtyObjects` based on what changed that tick. If many objects have dirty state (new: 300 bytes; partial: ~20 bytes), frame size can exceed network MTU (~1500 bytes), forcing IP fragmentation. No flow control exists for slow clients.
- **SuroiByteStream float encoding is lossy:** Floats are encoded as fixed-size integers over `[min, max]` ranges (e.g., position as 2×uint16 over `[0, 1924]`). Precision is limited: a position float maps to ~1 world unit resolution. Rounding errors accumulate over many ticks if client interpolation is not careful.
- **BooleanGroup max 8/16 bits:** `writeBooleanGroup` packs up to 8 bools into 1 byte; `writeBooleanGroup2` up to 16 into 2 bytes. Adding more boolean flags to `UpdatePacket.PlayerData` will overflow the bitfield and require adding a third byte, increasing every frame by 1 byte.
- **PartialDirtyObjects is per-category:** Partial updates only serialize position/rotation, not definition changes. If an object's definition changes locally (weapon swap, equipment equip), the server must send a `fullDirtyObject` update, not a partial. Clients that see only partial updates for definition changes will render stale data.
- **Disconnect is WebSocket close, not a packet:** The `Disconnect` PacketType (value 0) is **never serialized**. Client/server detect disconnection via WebSocket `close` event. Graceful shutdown must close the socket, not send a packet.

## Dependencies on This Document

This document defines the **binary protocol contract** that all communication subsystems implement:

- [Networking](docs/subsystems/networking/) — Implements PacketStream and individual packet serialization per this spec
- [Game Loop](docs/subsystems/game-loop/) — Generates `UpdatePacket` data every tick based on dirty object tracking
- [Object Definitions](docs/subsystems/object-definitions/) — Objects serialize via `ObjectSerializations` using types defined in this protocol
- [Client Rendering](docs/subsystems/client-rendering/) — Deserializes packets and updates ObjectPool state
- [Inventory](docs/subsystems/inventory/) — Inventory state transmitted via `PlayerData` section of `UpdatePacket`
- [Spatial Grid](docs/subsystems/spatial-grid/) — Position/rotation updates serialized in `PartialObjects` section
- [Gas System](docs/subsystems/gas/) — Gas state changes sent via `Gas` and `GasPercentage` sections

## Related Documents

### Tier 1 — Architecture & High-Level
- [Project Description](description.md) — Business domain and features
- [System Architecture](architecture.md) — WebSocket endpoints and transport paths
- [Data Model](datamodel.md) — Enumerations and GameConstants referenced in protocol
- [Development Guide](development.md) — Testing and validation tools

### Tier 2 — Subsystems
- [Networking](docs/subsystems/networking/) — Packet transport and multiplexing implementation
- [Game Loop](docs/subsystems/game-loop/) — UpdatePacket generation and dirty tracking
- [Object Definitions](docs/subsystems/object-definitions/) — Object serialization schemas
- [Client Rendering](docs/subsystems/client-rendering/) — Packet deserialization and rendering
- [Inventory](docs/subsystems/inventory/) — Inventory serialization in PlayerData
- [Gas System](docs/subsystems/gas/) — Gas Updates serialization
- [Spatial Grid](docs/subsystems/spatial-grid/) — Position updates in PartialObjects

### Tier 3 — Key Modules
- [SuroiByteStream](docs/subsystems/networking/modules/stream.md) — Encoding implementation
- [UpdatePacket Module](docs/subsystems/networking/modules/update-packet.md) — Delta encoding logic
- [Object Serializations](docs/subsystems/object-definitions/modules/serialization.md) — Per-category serialization formats

#### `DamageSources` Enum

| Value | Name |
|-------|------|
| 0 | `Gun` |
| 1 | `Melee` |
| 2 | `Throwable` |
| 3 | `Explosion` |
| 4 | `Gas` |
| 5 | `Obstacle` |
| 6 | `BleedOut` |
| 7 | `FinallyKilled` |
| 8 | `Disconnect` |

---

### `GameOverPacket` — `@file common/src/packets/gameOverPacket.ts`

**Direction:** Server → Client  
**Purpose:** Sent when the local player (and teammates) finish the game.
Carries final placement and per-player stats.

**Serialized Fields:**

| Field | Encoding |
|-------|----------|
| `rank` | `writeUint8` |
| `teammates` array | `writeArray` (default count); see below |

**Teammate sub-fields:**

| Field | Encoding |
|-------|----------|
| `playerID` | `writeObjectId` (2 bytes) |
| `kills` | `writeUint8` |
| `damageDone` | `writeUint16` |
| `damageTaken` | `writeUint16` |
| `timeAlive` | `writeUint16` |
| `alive` | `writeUint8` (1 = alive) |

---

### `PickupPacket` — `@file common/src/packets/pickupPacket.ts`

**Direction:** Server → Client  
**Purpose:** Notifies the client that a pickup succeeded or shows an inventory
message (e.g. "inventory full").

**Serialized Fields:**

`pickupData` byte layout:
```
bit [7]    hasItem
bit [6]    hasMessage
bits [2:0] message value (only when !hasItem && hasMessage)
```

| Field | Encoding | Condition |
|-------|----------|-----------|
| `pickupData` | `writeUint8` | Always |
| `item` | `Loots.writeToStream` | `hasItem` |

---

### `SpectatePacket` — `@file common/src/packets/spectatePacket.ts`

**Direction:** Client → Server  
**Purpose:** Requests a spectate state change (next player, previous player,
spectate a specific player by ID, etc.).

**Serialized Fields:**

| Field | Encoding | Condition |
|-------|----------|-----------|
| `spectateAction` | `writeUint8` | Always |
| `playerID` | `writeObjectId` | `spectateAction === SpectateActions.SpectateSpecific` |

---

### `ReportPacket` — `@file common/src/packets/reportPacket.ts`

**Direction:** Client → Server  
**Purpose:** Player report submission.

**Serialized Fields:**

| Field | Encoding |
|-------|----------|
| `playerID` | `writeObjectId` (2 bytes) |
| `reportID` | `writeString(8)` (8 bytes) |

---

### `DebugPacket` — `@file common/src/packets/debugPacket.ts`

**Direction:** Client → Server  
**Purpose:** Developer/debug controls (speed override, noclip, invulnerability,
spawn loot, spawn dummies). Not available in production.

**Serialized Fields:**

| Field | Encoding | Condition |
|-------|----------|-----------|
| Flags: `overrideZoom`, `noClip`, `invulnerable`, `spawnLoot`, `spawnDummy`, `dummyHasVest`, `dummyHasHelmet` | `writeBooleanGroup` (1 byte) | Always |
| `speed` | `writeFloat32` (4 bytes) | Always |
| `layerOffset` | `writeInt8` (1 byte) | Always |
| `zoom` | `writeUint8` | `overrideZoom` |
| `spawnLootType` | `Loots.writeToStream` | `spawnLoot` |
| `dummyVest` | `Armors.writeToStream` | `spawnDummy && dummyHasVest` |
| `dummyHelmet` | `Armors.writeToStream` | `spawnDummy && dummyHasHelmet` |

---

## Object Serializations

`@file common/src/utils/objectsSerializations.ts`

Each `ObjectCategory` has four methods:

| Method | Purpose |
|--------|---------|
| `serializePartial(stream, data)` | Mutable per-tick state (position, rotation, animation, alive/dead) |
| `serializeFull(stream, data)` | Immutable or first-seen data (definition, layer) |
| `deserializePartial(stream)` | Inverse of serializePartial |
| `deserializeFull(stream)` | Inverse of serializeFull |

`serializeFull` is only called together with `serializePartial` (in
`FullObjects` frames). `serializePartial` alone is used in `PartialObjects`
frames.

### `ObjectCategory.Player`

**Partial fields:**

| Field | Encoding |
|-------|----------|
| `position` | `writePosition` (4 bytes) |
| `rotation` | `writeRotation2` (1 byte) |
| Animation + action | `writeUint8` — packed bitfield: `Nnnn nccC` where N=animationDirty, n=animation (4 bits), c=action (2 bits), C=actionDirty |
| `action.item` _(conditional: actionDirty && item present)_ | `Loots.writeToStream` (1 byte) |

**Full fields:**

| Field | Encoding |
|-------|----------|
| `layer` | `writeLayer` (1 byte) |
| 16 boolean flags | `writeBooleanGroup2` (2 bytes): `dead, downed, beingRevived, invulnerable, hasReloadMod, halloweenThrowableSkin, hasHelmet, hasVest, hasDisguise, infected, hasBackEquippedMelee, hasBubble, activeOverdrive, hasMagneticField, isCycling, emitLowHealthParticles` |
| `teamID` | `writeUint8` |
| `activeItem` | `Loots.writeToStream` (1 byte) |
| `sizeMod` | `writeFloat(0, 4, 1)` |
| `reloadMod` _(conditional)_ | `writeFloat(0, 4, 1)` |
| `skin` | `Skins.writeToStream` (1 byte) |
| `helmet` _(conditional)_ | `Armors.writeToStream` (1 byte) |
| `vest` _(conditional)_ | `Armors.writeToStream` (1 byte) |
| `backpack` | `Backpacks.writeToStream` (1 byte) |
| `activeDisguise` _(conditional)_ | `Obstacles.writeToStream` (1 byte) |
| `backEquippedMelee` _(conditional)_ | `Melees.writeToStream` (1 byte) |

### `ObjectCategory.Obstacle`

**Partial fields:**

| Field | Encoding |
|-------|----------|
| Flags: `dead, playMaterialDestroyedSound, waterOverlay, powered` | `writeBooleanGroup` (1 byte) |
| `scale` | `writeScale` (1 byte) |

**Full fields:**

| Field | Encoding | Notes |
|-------|----------|-------|
| `definition` | `Obstacles.writeToStream` (1 byte) | |
| `position` | `writePosition` (4 bytes) | |
| `layer` | `writeLayer` (1 byte) | |
| `obstacleData` | `writeUint8` | Packs variation (MSBs), door offset+locked or activated (LSBs) |
| `rotation` _(conditional: RotationMode.Full)_ | `writeRotation` (2 bytes) | After obstacleData |
| _(RotationMode.Limited/Binary)_ | packed into `obstacleData` | No extra write |

### `ObjectCategory.Loot`

**Partial fields:** `writePosition` (4 bytes) + `writeLayer` (1 byte)

**Full fields:**

| Field | Encoding |
|-------|----------|
| `definition` | `Loots.writeToStream` (1 byte) |
| `isNew` + `count` | `writeUint16` — bit 15 = isNew, bits [14:0] = count |

### `ObjectCategory.DeathMarker`

**Partial fields only** (no full serialization):

| Field | Encoding |
|-------|----------|
| `position` | `writePosition` (4 bytes) |
| `layer` | `writeLayer` (1 byte) |
| `isNew` | `writeUint8` (-1 or 0) |
| `playerID` | `writeObjectId` (2 bytes) |

### `ObjectCategory.Building`

**Partial fields:**

| Field | Encoding |
|-------|----------|
| Flags: `dead, hasPuzzle, puzzle.solved, puzzle.errorSeq` | `writeBooleanGroup` (1 byte) |
| `layer` | `writeLayer` (1 byte) |

**Full fields:**

| Field | Encoding |
|-------|----------|
| `definition` | `Buildings.writeToStream` (1 byte) |
| `position` | `writePosition` (4 bytes) |
| `orientation` | `writeUint8` |

### `ObjectCategory.Decal`

**Partial fields only** (no full serialization):

| Field | Encoding |
|-------|----------|
| `definition` | `Decals.writeToStream` (1 byte) |
| `position` | `writePosition` (4 bytes) |
| `rotation` | `writeObstacleRotation(definition.rotationMode)` |
| `layer` | `writeLayer` (1 byte) |

### `ObjectCategory.Parachute`

**Partial:** `writeFloat(height, 0, 1, 1)` (1 byte)  
**Full:** `writePosition` (4 bytes)

### `ObjectCategory.Projectile`

**Partial fields:**

| Field | Encoding |
|-------|----------|
| `position` | `writePosition` (4 bytes) |
| `rotation` | `writeRotation2` (1 byte) |
| `layer` | `writeLayer` (1 byte) |
| `height` | `writeFloat(0, GameConstants.projectiles.maxHeight, 1)` (1 byte) |

**Full fields:**

| Field | Encoding | Condition |
|-------|----------|-----------|
| `definition` | `Throwables.writeToStream` (1 byte) | Always |
| `halloweenSkin` + `activated` | `writeBooleanGroup` (1 byte) | Always |
| `c4.throwerTeamID` | `writeUint8` | `definition.c4` |
| `c4.tintIndex` | `writeUint8` | `definition.c4` |

### `ObjectCategory.SyncedParticle`

**Partial fields only** (no full serialization):

| Field | Encoding | Condition |
|-------|----------|-----------|
| `definition` | `SyncedParticles.writeToStream` (1 byte) | Always |
| `startPosition` | `writePosition` (4 bytes) | Always |
| `endPosition` | `writePosition` (4 bytes) | Always |
| `layer` | `writeLayer` (1 byte) | Always |
| `age` | `writeFloat(0, 1, 1)` (1 byte) | Always |
| `lifetime` | `writeFloat(min, max, 1)` | `definition.lifetime` is a range object |
| `angularVelocity` | `writeFloat(min, max, 1)` | `definition.angularVelocity` is a range object |
| `scale.start` + `scale.end` | `writeScale` × 2 | `definition.scale` defined |
| `alpha.start` + `alpha.end` | `writeFloat(0,1,1)` × 2 | `definition.alpha` defined |
| `variant` | `writeUint8` | `definition.variations` defined |
| `creatorID` | `writeObjectId` | `definition.hasCreatorID` |

---

## DataSplit Tracking

`@file common/src/packets/packet.ts`

`DataSplit` is a `Record<DataSplitTypes, number>` accumulator. During
deserialization the callbacks `saveIndex()` / `recordTo(target)` capture byte
counts per category — used for bandwidth profiling.

| `DataSplitTypes` | Includes |
|------------------|----------|
| `PlayerData` | PlayerData section, JoinedPacket |
| `Players` | `ObjectCategory.Player` object data |
| `Obstacles` | `ObjectCategory.Obstacle` object data |
| `Loots` | `ObjectCategory.Loot` object data |
| `SyncedParticles` | `ObjectCategory.SyncedParticle` object data |
| `GameObjects` | All other categories + map + explosions + bullet + new/deleted players |
| `Killfeed` | `KillPacket` data |

`getSplitTypeForCategory(category)` returns the matching `DataSplitTypes` for
a given `ObjectCategory`.

---

## Related Documents

- **Tier 1:** [docs/architecture.md](architecture.md) — System overview
- **Tier 2:** [docs/subsystems/networking/](subsystems/networking/) — Networking subsystem details
- **Tier 2:** [docs/subsystems/object-definitions/](subsystems/object-definitions/) — Object definition registry
