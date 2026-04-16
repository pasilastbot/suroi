# Networking — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/networking/README.md -->

## Pattern: Packet Authoring

**When to use:** Adding a new packet type to the client↔server protocol.

**Implementation:**

1. Add a new value to the `PacketType` enum in `common/src/packets/packet.ts`.  
   The enum value must equal the desired `Packets` array index + 1 (because `Disconnect` occupies value `0` and is not in the array).

2. Create `common/src/packets/<name>Packet.ts`. Define a data interface (or type) that includes
   `readonly type: PacketType.<Name>`, then instantiate `Packet<DataIn, DataOut>`:

   ```typescript
   export const MyPacket = new Packet<MyData>(PacketType.My, {
       serialize(stream, data) {
           // write fields with SuroiByteStream methods
       },
       deserialize(stream, data, saveIndex, recordTo) {
           // read fields in the exact same order
           // call saveIndex() before reading, recordTo(DataSplitTypes.X) after
           // for bandwidth graph accounting
       }
   });
   ```

3. Add the new packet to the `Packets` array in `common/src/packets/packetStream.ts` at the
   correct index position. **Order is significant** — the array index is the wire type byte.

4. Handle the packet:
   - **Server receives:** add a `case PacketType.My:` branch in `game.ts` → `onMessage()`.
   - **Client receives:** add a `case PacketType.My:` branch in `client/src/scripts/game.ts`
     → `onPacket()`.
   - **Server sends:** call `player.sendPacket(MyPacket.create({ ... }))`.
   - **Client sends:** call `game.sendPacket(MyPacket.create({ ... }))`.

**Example file:** `@file common/src/packets/pickupPacket.ts` (minimal packet: 1-byte bitmask +
optional `Loots` definition index)

---

## Pattern: Delta-Encoded Optional Fields (bool-group header)

**When to use:** A packet section has many fields that are only sometimes present (e.g. `PlayerData`
inside `UpdatePacket`, or `JoinPacket`'s emote slots).

**Implementation:**

Pack N presence booleans into one or two bytes at the start of the section using
`writeBooleanGroup` (up to 8 booleans → 1 byte) or `writeBooleanGroup2` (up to 16 booleans → 2
bytes). Then write each field conditionally:

```typescript
// serialize
const hasHealth  = data.health !== undefined;
const hasZoom    = data.zoom   !== undefined;
stream.writeBooleanGroup2(hasHealth, hasZoom, /* … up to 16 */ );
if (hasHealth) stream.writeFloat(data.health, 0, 1, 2);
if (hasZoom)   stream.writeUint8(data.zoom);

// deserialize
const [hasHealth, hasZoom] = stream.readBooleanGroup2();
if (hasHealth) data.health = stream.readFloat(0, 1, 2);
if (hasZoom)   data.zoom   = stream.readUint8();
```

On ticks where only 1-2 fields are dirty the entire PlayerData section may be as small as 4 bytes.

**Example file:** `@file common/src/packets/updatePacket.ts` — `serializePlayerData` / `deserializePlayerData`

---

## Pattern: UpdateFlags Bitmask

**When to use:** A large packet (like `UpdatePacket`) contains many independent optional sections.

**Implementation:**

Reserve 2 bytes for flags at the start, write a placeholder, then set bits as sections are
written, and seek back to write the real flags value at the end:

```typescript
// serialize
const flagsIdx = strm.index;
strm.writeUint16(0);              // placeholder

let flags = 0;
if (data.gas) {
    // … write gas section …
    flags |= UpdateFlags.Gas;
}
// … other sections …

const end = strm.index;
strm.index = flagsIdx;
strm.writeUint16(flags);          // overwrite placeholder
strm.index = end;                 // restore position

// deserialize
const flags = stream.readUint16();
if (flags & UpdateFlags.Gas) {
    // … read gas section …
}
```

`UpdateFlags` is a `const enum` (`bit 0` through `bit 15`), so all flag checks compile to integer
literals with no runtime object lookups.

**Example file:** `@file common/src/packets/updatePacket.ts`

---

## Pattern: ObjectDefinition Reference Encoding

**When to use:** Serializing a reference to any typed game entity (weapon, skin, obstacle,
emote, badge, scope, perk, etc.) — anything defined in a `ObjectDefinitions<T>` registry.

**Implementation:**

Every `ObjectDefinitions<T>` instance exposes `writeToStream(stream, definition)` and
`readFromStream(stream)`. These encode the definition as its compact index in the registry (a
`uint8` for most registries, auto-sized to the minimum required bit width). This is far smaller
than sending the full `idString`.

```typescript
// serialize (server or client)
Loots.writeToStream(stream, item.definition);     // writes a small integer index
Emotes.writeToStream(stream, emote.definition);

// deserialize
const definition = Loots.readFromStream<WeaponDefinition>(stream);
const emote      = Emotes.readFromStream(stream);
```

**Constraint:** Both sides must have identical, identically-ordered registry arrays. The server
never sends `idString` names over the wire; it only sends indices.

**Example files:**
- `@file common/src/utils/suroiByteStream.ts`
- `@file common/src/packets/joinPacket.ts` (`Loots.writeToStream`, `Badges.writeToStream`, `Emotes.writeToStream`)

---

## Pattern: Packed Bitmask for Killfeed Metadata

**When to use:** A packet has a small number of enum/boolean fields that together fit in 1–2
bytes, avoiding multiple `writeUint8` calls.

**Implementation:**

Calculate a bitmask manually, write it as a single integer, and decode it on the other side using
bit-masks and shifts. Document the bit layout in a comment.

```typescript
// KillPacket example — 16-bit kfData:
//   a c c d k s s s s s r r r r r r
//   a = hasAttackerId
//   cc = credited state (00/01/10)
//   d = downed,  k = killed
//   s = damageSource (5 bits)
//   r = reserved

let kfData = data.damageSource & 0b1_1111;        // bits 4:0
if (data.killed)          kfData += 1 << 5;
if (data.downed)          kfData += 1 << 6;
if (victimIsCreditedId)   kfData += 0b01 << 7;
else if (hasCreditedId)   kfData += 0b10 << 7;
if (hasAttackerId)        kfData += 1 << 9;
stream.writeUint16(kfData);

// deserialize
const kfData      = stream.readUint16();
const damageSource = kfData & 0b1_1111;
const killed      = (kfData >> 5) & 1;
const downed      = (kfData >> 6) & 1;
const creditBits  = (kfData >> 7) & 0b11;
const hasAttacker = (kfData >> 9) & 1;
```

**Example file:** `@file common/src/packets/killPacket.ts`

---

## Pattern: Inventory Bitfield Encoding

**When to use:** Serializing a sparse inventory (many item types, but most quantities are zero).

**Implementation:**

Write a bitfield where each bit indicates whether the corresponding item has a non-zero count,
then write only the non-zero counts. Iterate items in chunks of 8, writing one `writeUint8`
bitfield byte per chunk, then write all non-zero `uint16` counts after.

```typescript
// serialize
const nonNullCounts: number[] = [];
let itemPresent = 0, idx = 0;
while (idx < itemKeysLength) {
    for (let i = 0; i < 8 && idx < itemKeysLength; i++, idx++) {
        const count = items[itemKeys[idx]];
        if (count > 0) { itemPresent += 2 ** i; nonNullCounts.push(count); }
    }
    stream.writeUint8(itemPresent);
    itemPresent = 0;
}
for (const count of nonNullCounts) stream.writeUint16(count);

// deserialize: read bitfield bytes → collect present indices → read counts for those indices
```

**Example file:** `@file common/src/packets/updatePacket.ts` — `serializePlayerData` items section

---

## Pattern: Obstruction Rotation Bit-Packing

**When to use:** Serializing obstacle rotation and variation together in a single byte for
`Limited` and `Binary` modes, while `Full` mode uses a separate float.

**Implementation:**

Within `MapPacket`, obstacle data is packed into an `obstacleData` byte: variation occupies the
MSBs (shifted left by `8 - variationBits`), and rotation (for `Limited` or `Binary` modes)
occupies the LSBs.

```typescript
let obstacleData = 0;
if (hasVariation) obstacleData = variation << (8 - variationBits);

switch (rotationMode) {
    case RotationMode.Full:
        stream.writeUint8(obstacleData);      // variation only
        stream.writeRotation(rotation);        // float follows
        break;
    case RotationMode.Limited:
    case RotationMode.Binary:
        stream.writeUint8(obstacleData + rotation);  // both packed
        break;
    case RotationMode.None:
        stream.writeUint8(obstacleData);      // variation only, no rotation
        break;
}
```

**Example file:** `@file common/src/packets/mapPacket.ts`

---

## Pattern: Pre-allocated Per-Player PacketStream Buffer

**When to use:** Avoiding allocation hot-paths in the per-tick send loop.

**Implementation:**

Each `Player` owns a single `PacketStream` backed by a fixed 64 KB `ArrayBuffer` allocated at
construction time:

```typescript
private readonly _packetStream = new PacketStream(new SuroiByteStream(new ArrayBuffer(1 << 16)));
```

In `secondUpdate()`, before serializing packets, the stream index is reset to `0`:

```typescript
this._packetStream.stream.index = 0;
for (const packet of this._packets) { this._packetStream.serialize(packet); }
for (const packet of this.game.packets) { this._packetStream.serialize(packet); }
this.sendData(this._packetStream.getBuffer());
```

`getBuffer()` slices the underlying `ArrayBuffer` to the current index, so only the written bytes
are sent. The buffer itself is reused every tick.

**Example file:** `@file server/src/objects/player.ts` — `secondUpdate()`
