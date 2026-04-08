# Q: How do I add a new packet type for client-server communication?

<!-- @tags: packets, protocol, serialization -->
<!-- @related: docs/subsystems/packets/README.md, docs/protocol.md -->

## When to Add a New Packet Type

Most communication already fits into existing packets:
- New game state fields → extend `UpdatePacket` PlayerData or object serialization
- New discrete events → extend `KillPacket` or `GameOverPacket`
- New player inputs → add to `InputPacket` `InputActions` enum

Only add a new packet type when the communication doesn't fit any existing packet:
a fundamentally new client↔server exchange with its own lifecycle.

## Step-by-Step

### 1. Add to the PacketType enum

```typescript
// common/src/packets/packet.ts
export enum PacketType {
    Disconnect,
    GameOver,
    Input,
    Joined,
    Join,
    Kill,
    Map,
    Pickup,
    Report,
    Spectate,
    Update,
    Debug,
    MyPacket,   // ← append here
}
```

**Important:** Append at the end. The `Packets` array is indexed by
`PacketType - 1`. Inserting in the middle breaks the index mapping.

### 2. Create the packet file

```typescript
// common/src/packets/myPacket.ts
import { SuroiByteStream } from "../utils/suroiByteStream";
import { Packet, PacketType } from "./packet";

interface MyPacketData {
    readonly type: PacketType.MyPacket
    readonly someField: number
    readonly anotherField: string
}

export const MyPacket = new Packet<MyPacketData>(
    PacketType.MyPacket,
    // serialize (sender → stream)
    (stream, data) => {
        stream.writeUint16(data.someField);
        stream.writeString(data.anotherField, 32);
    },
    // deserialize (stream → receiver)
    (stream) => ({
        type: PacketType.MyPacket,
        someField: stream.readUint16(),
        anotherField: stream.readString(32),
    })
);
```

### 3. Register in packetStream.ts

```typescript
// common/src/packets/packetStream.ts
import { MyPacket } from "./myPacket";

export const Packets = [
    GameOverPacket,
    InputPacket,
    JoinedPacket,
    JoinPacket,
    KillPacket,
    MapPacket,
    PickupPacket,
    ReportPacket,
    SpectatePacket,
    UpdatePacket,
    DebugPacket,
    MyPacket,       // ← append last
] as const;
```

The position in this array must match `PacketType.MyPacket - 1` (0-based).

### 4. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

### 5. Send from server

```typescript
// server/src/game.ts or appropriate server file
import { PacketStream } from "@common/packets/packetStream";
import { PacketType } from "@common/packets/packet";

// In the appropriate handler:
const stream = new PacketStream(new SuroiByteStream(new ArrayBuffer(16)));
stream.serialize({
    type: PacketType.MyPacket,
    someField: 42,
    anotherField: "hello",
});
player.sendPacket(stream.getBuffer());
```

### 6. Handle on client

```typescript
// client/src/scripts/game.ts — in the message handler / packet dispatch loop
case PacketType.MyPacket: {
    const data = MyPacket.deserialize(stream);
    // handle data.someField, data.anotherField
    break;
}
```

Or if the client sends to the server, add the send call in the appropriate
input handler and handle the incoming packet in `server/src/game.ts`.

### 7. Validate

```bash
bun validateDefinitions
bun dev
```

## Design Checklist

- [ ] Only add a new type if no existing packet fits
- [ ] Append to `PacketType` enum and `Packets` array — never insert in the middle
- [ ] Bump `protocolVersion`
- [ ] Add `serialize` and `deserialize` that are exact inverses
- [ ] Use `writeString(s, maxLen)` for all strings — ensure same `maxLen` on both sides
- [ ] Handle the packet on the receiving side

## Related

- [Protocol Reference](../protocol.md) — binary format, PacketType enum, Packets array
- [Packets Subsystem](../subsystems/packets/README.md) — existing packet reference
- [Protocol Version](04-protocol-version-bump.md) — bumping the version
