# Q: How do I add a new player stat or attribute that the client needs to display?

<!-- @tags: player, server, UpdatePacket, serialization, protocol -->
<!-- @related: docs/subsystems/objects/modules/server-objects.md, docs/subsystems/packets/modules/update-packet.md -->

## Overview

Player stats flow through a specific pipeline:

```
Player (server) → PlayerData serialization → UpdatePacket → Client UIManager
```

You need to touch 4 files: `player.ts` (server), `updatePacket.ts` (common),
`uiManager.ts` (client), and bump the protocol version.

## 1. Add the field to the Player server object

```typescript
// server/src/objects/player.ts
class Player extends ... {
    myNewStat = 0;   // add field

    // When it changes, call setDirty():
    set myNewStat(value: number) {
        this._myNewStat = value;
        this.setDirty();   // ← triggers UpdatePacket PlayerData
    }
}
```

## 2. Add to the PlayerData interface

```typescript
// common/src/packets/updatePacket.ts
export interface PlayerData {
    // ...existing fields...
    readonly myNewStat?: number    // optional — only sent when changed
}
```

## 3. Add serialization to serializePlayerData

```typescript
// common/src/packets/updatePacket.ts — serializePlayerData()

// 3a. Add presence flag (inside writeBooleanGroup2 call):
const hasMyNewStat = myNewStat !== undefined;
// Add to the writeBooleanGroup2(...) call alongside other has* flags

// 3b. Serialize the value:
if (hasMyNewStat) {
    strm.writeFloat(myNewStat, 0, 100, 2);  // range [0,100], 2 bytes
    // or: strm.writeUint8(myNewStat);       // 0-255 integer
}
```

**Important:** The `writeBooleanGroup2` call packs up to 16 booleans into 2 bytes.
If there are already 16 presence flags, you need to add a new `writeBooleanGroup`
call. **Always add the flag in the same position in both `serialize` and
`deserialize` — they must be perfectly symmetric.**

## 4. Add deserialization to deserializePlayerData

```typescript
// common/src/packets/updatePacket.ts — deserializePlayerData()

// 4a. Read the presence flag from readBooleanGroup2:
// Match the exact position you added in step 3a.

// 4b. Deserialize the value:
if (hasMyNewStat) {
    data.myNewStat = strm.readFloat(0, 100, 2);
}
```

## 5. Serialize from Player on the server

```typescript
// server/src/objects/player.ts — in the PlayerData serialization:
// (where other stats like health and adrenaline are written)
if (this.fullDirty || this.myNewStatDirty) {
    data.myNewStat = this.myNewStat / this.maxMyNewStat; // normalize to [0,1]
}
```

## 6. Receive on the client

```typescript
// client/src/scripts/managers/uiManager.ts — updateUI(data: PlayerData)
if (data.myNewStat !== undefined) {
    this.myNewStat = data.myNewStat;
    this.updateMyNewStatDisplay();
}

private updateMyNewStatDisplay(): void {
    // Update DOM elements:
    $("#my-new-stat-bar").css("width", `${this.myNewStat * 100}%`);
    $("#my-new-stat-text").text(Math.round(this.myNewStat * 100));
}
```

## 7. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

PlayerData format change = protocol bump required.

## 8. Add HTML for the new stat

```html
<!-- client/src/index.html — in the HUD section -->
<div id="my-new-stat-container">
    <span id="my-new-stat-text">0</span>
    <div id="my-new-stat-bar-bg">
        <div id="my-new-stat-bar"></div>
    </div>
</div>
```

## 9. Validate

```bash
bun validateDefinitions
bun dev
```

## Key Serialization Rules

- `serializePlayerData` and `deserializePlayerData` must be **exact inverses**
- Presence flags (`has*`) use `writeBooleanGroup` / `readBooleanGroup` — they
  pack booleans into bytes. Count carefully.
- Use `writeFloat(value, min, max, bytes)` for floats in a known range
- Use `writeUint8` for small integers, `writeUint16` for larger

## Related

- [Protocol Reference](../protocol.md) — SuroiByteStream encoding methods
- [UpdatePacket](../subsystems/packets/modules/update-packet.md) — PlayerData structure
- [Tick & Serialization](../subsystems/game-loop/modules/tick-serialization.md) — dirty tracking
- [HUD Module](../subsystems/ui/modules/hud.md) — existing HUD elements
- [Protocol Version](04-protocol-version-bump.md) — when to bump
