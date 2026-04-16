# Packet Types

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: common/src/packets/ -->

## Purpose

Defines all packet types exchanged between client and server, their data schemas, serialization formats, and deserialization patterns. Packets are the fundamental unit of network communication in Suroi.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/packets/packet.ts` | Base `Packet<DataIn, DataOut>` class and `PacketType` enum | Medium |
| `common/src/packets/packetStream.ts` | Multiplexes multiple packets into a stream | Medium |
| `common/src/packets/*.ts` (11 files) | Individual packet definitions | Medium |
| `common/src/utils/suroiByteStream.ts` | Compact binary serialization layer | High |

## Packet Type Inventory

All 12 packet types defined in `PacketType` enum:

| Type | Direction | Purpose | Frequency |
|------|-----------|---------|-----------|
| **Disconnect** (0) | Server→Client | Tell client connection closed | Rare |
| **GameOver** (1) | Server→Client | Send end-of-game state (victory/loss) | Once per game |
| **Input** (2) | Client→Server | Player input (movement, actions) | Every frame (~40 TPS = every 25ms) |
| **Joined** (3) | Server→Client | Confirm player joined, send initial state | Once at join |
| **Join** (4) | Client→Server | Request to join game with loadout | Once per game |
| **Kill** (5) | Server→Client | Announce death event, update killfeed | Per death |
| **Map** (6) | Server→Client | Send map data, buildings, obstacles | Once at start |
| **Pickup** (7) | Server→Client | Tell client item was picked up | Per pickup |
| **Report** (8) | Client→Server | Debug/report commands | Rare |
| **Spectate** (9) | Client→Server | Switch spectate target | Per spectate action |
| **Update** (10) | Server→Client | Game state delta (player positions, health, objects) | Every frame |
| **Debug** (11) | Server→Client | Debug info (only in debug builds) | Rare |

## Packet Instantiation Patterns

Every packet is a singleton instance following this pattern (example: `KillPacket`):

```typescript
// From common/src/packets/killPacket.ts
export const KillPacket = new Packet<KillData>(PacketType.Kill, {
    serialize(stream, data) { /* ... */ },
    deserialize(stream, data, saveIndex, recordTo) { /* ... */ }
});
```

**How to instantiate packet data:**

```typescript
// Create empty packet data with type
const killData = KillPacket.create({
    victimId: 42,
    attackerId: 7,
    damageSource: DamageSources.Gun
});

// Serialize to wire format
const packet = new PacketStream();
packet.serialize(killData);
const buffer = packet.getBuffer();

// Deserialize from wire format
const incomingPacket = new PacketStream(buffer);
const receivedKill = incomingPacket.deserialize();
```

## Packet Inheritance Hierarchy

All packets extend the generic `Packet<DataIn, DataOut>` base class. Most packets have symmetric data (DataIn = DataOut):

```
Packet<DataIn, DataOut>
├── KillPacket       (KillData → KillData)
├── UpdatePacket     (UpdateDataIn → UpdateDataOut)
├── JoinPacket       (JoinData → JoinData)
├── JoinedPacket     (JoinedData → JoinedData)
├── InputPacket      (InputData → InputData)
├── MapPacket        (MapData → MapData)
├── PickupPacket     (PickupData → PickupData)
├── GameOverPacket   (GameOverData → GameOverData)
├── SpectatePacket   (SpectateData → SpectateData)
├── DisconnectPacket (DisconnectData → DisconnectData)
├── ReportPacket     (ReportData → ReportData)
└── DebugPacket      (DebugData → DebugData)
```

## Serialization Per Packet Type

Packets use `SuroiByteStream` for compact binary encoding to minimize bandwidth:

### Vector & Position Serialization

```typescript
// From common/src/utils/suroiByteStream.ts
writePosition(vector: Vector): void
    → Uses 4 bytes (2×uint16) for x, y ∈ [0, maxPosition]
    → maxPosition = 1924 (map bounds)
    → Maps [0, 1924] → [0, 65535]

writeVector(vector, minX, minY, maxX, maxY, bytes): void
    → Generic vector with custom range
    → bytes ∈ {1, 2, 3, 4} for bandwidth control
```

### Object ID & Type Serialization

```typescript
writeObjectId(id: number): void
    → Alias for writeUint16(id)
    → 2 bytes for network ID ∈ [0, 2^16-1]

writeObjectType(type: ObjectCategory): void
    → Alias for writeUint8(type)
    → 1 byte for category enum
```

### Boolean Groups (Bit-Packing)

Reduces bandwidth by packing multiple bools into one byte:

```typescript
// Kill packet header: 10 bits packed into 2 bytes
writeBooleanGroup(
    hasAttackerId,        // bit 9
    creditedState,        // bits 8-7 (2 bits for: none/victim/other)
    downed,               // bit 6
    killed,               // bit 5
    damageSource,         // bits 4-0 (5 bits for enum)
)
```

### Protocol-Specific Codecs

- **Player data delta:** Only changed fields serialized (health, adrenaline, zoom, position, layer, etc.)
- **Object category splitting:** `DataSplitTypes` enum tracks bandwidth per category (Players, Obstacles, Loots, etc.) for metrics
- **Dirty flag tracking:** Partial vs full serialization per object

## Deserialization Patterns

### Standard Deserialize Signature

```typescript
deserialize(
    stream: SuroiByteStream,
    splits?: DataSplit  // Optional metrics collector
): PacketDataOut
```

Key patterns:

1. **Sparse data:** Read presence flags first, then optional fields:
   ```typescript
   const hasHealth = stream.readBoolean();
   const health = hasHealth ? stream.readFloat(0, maxHealth, 2) : undefined;
   ```

2. **Conditional branches:** Different packet versions or modes:
   ```typescript
   const type = stream.readUint8();  // 1 byte type discriminator
   if (type === SomeType) {
       // read SomeType-specific data
   } else {
       // read other data
   }
   ```

3. **Metrics collection:** Track bandwidth per data category:
   ```typescript
   const saveIndex = () => (savedIndex = stream.index);
   const recordTo = (target: DataSplitTypes) => {
       splits[target] += stream.index - savedIndex;
   };
   // Use in deserialize: saveIndex(); ...read...; recordTo(DataSplitTypes.Players);
   ```

## Data Split Types

Deserialize calls track bandwidth per category for performance profiling:

```typescript
export enum DataSplitTypes {
    PlayerData,           // Player state (health, inventory, etc.)
    Players,              // Player objects (position, animation, etc.)
    Obstacles,            // Obstacle objects (buildings, walls, etc.)
    Loots,                // Loot objects (weapons, ammo, etc.)
    SyncedParticles,      // Particle effects
    GameObjects,          // Generic objects (parachutes, projectiles, etc.)
    Killfeed              // Kill notifications
}
```

Mapped from `ObjectCategory`:

```typescript
function getSplitTypeForCategory(category: ObjectCategory): DataSplitTypes {
    switch(category) {
        case ObjectCategory.Player:        return DataSplitTypes.Players;
        case ObjectCategory.Building:      return DataSplitTypes.GameObjects;
        case ObjectCategory.Loot:          return DataSplitTypes.Loots;
        // ... etc
    }
}
```

## Error Packets (Implicit)

The protocol doesn't have explicit error packets. Instead:
- **Connection errors** → WebSocket `onerror` / `onclose`
- **Heartbeat timeout** → Client detects no `UpdatePacket` for ~5s, assumes disconnected
- **Invalid packet** → Stream reads past buffer → implicit deserialization failure (no exception thrown, client just gets garbage or disconnects)

## Key Packets Deep Dive

### Update Packet — `@file common/src/packets/updatePacket.ts:1`

Most frequent packet (every 25ms at 40 TPS). Contains:

```typescript
interface UpdateDataIn {
    type: PacketType.Update;
    playerData?: PlayerData;              // Health, inventory, adrenaline, etc.
    globalData?: UpdateDataCommon;        // Map indicators, announcements
    killfeed?: Killfeed;                  // Latest kills
    partialDirtyObjects: ObjectsNetData;  // Delta updates (position, rotation)
    fullDirtyObjects: ObjectsNetData;     // Full state refresh for new/changed objects
    deletedObjects: number[];             // Object IDs to remove
}
```

Player data is split (via booleanGroup) to only serialize changed fields:

```typescript
// From updatePacket.ts:serializePlayerData()
const hasHealth = health !== undefined;
const hasAdrenaline = adrenaline !== undefined;
// ... 15 similar flags
strm.writeBooleanGroup(hasHealth, hasAdrenaline, ...);
if (hasHealth) strm.writeFloat(health, 0, 1, 2);  // 2 bytes
if (hasAdrenaline) strm.writeFloat(adrenaline, 0, 1, 2);
```

### Join Packet — `@file common/src/packets/joinPacket.ts:1`

Client→Server to join game:

```typescript
interface JoinData {
    type: PacketType.Join;
    name: string;                 // Player name (max 16 chars)
    loadout: { skin: string; badge?: string; emote?: string; };
    autoFill?: boolean;          // Auto-fill team slots
    gameMode?: string;           // e.g., "normal", "team", "deathmatch"
}
```

### Kill Packet — `@file common/src/packets/killPacket.ts:1`

Server→Client death announcement:

```typescript
interface KillData {
    type: PacketType.Kill;
    victimId: number;                    // Who died
    attackerId?: number;                 // Who killed them (undefined = gas/obstacle)
    creditedId?: number;                 // Who gets credit (may differ from attacker in team mode)
    kills?: number;                      // Attacker's total kills this game
    damageSource: DamageSources;         // Gun / Melee / Throwable / Explosion / Gas / ...
    weaponUsed?: ...Definition;          // Gun/melee/throwable/explosion/obstacle definition
    downed?: boolean;                    // Player was downed (not killed outright)
    killed?: boolean;                    // Player is fully dead
}
```

## Deserialization Error Handling

Packets do **not** throw errors on malformed data. Instead:

- **Partial stream:** Deserializer reads as far as buffer allows, ignores missing fields
- **Out-of-range values:** No range validation; client receives raw values (may cause visual glitches, not crashes)
- **Circular import prevention:** `Packets` array defined in `packetStream.ts` to avoid cyclic imports with individual packet files

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Networking subsystem overview
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — System architecture
- **Tier 3:** [protocol.md](protocol.md) — WebSocket connection and handshake flow
- **Source:** `common/src/packets/` — All packet implementations
- **Serialization:** [../../../../core-math-physics/modules/binary-encoding.md](../../../../core-math-physics/modules/binary-encoding.md) — Detailed SuroiByteStream reference
