# Input Pipeline — Input Processing & Packet Queueing

<!-- @tier: 3 -->
<!-- @parent: /docs/subsystems/input-management/README.md -->
<!-- @source: client/src/scripts/managers/inputManager.ts, server/src/server.ts, server/src/game.ts, server/src/objects/player.ts, common/src/packets/inputPacket.ts -->

## Purpose

Defines how hardware input (keyboard, mouse, gamepad) flows from client event listeners → buffered state → network serialization → server processing. This module bridges the **I/O layer** (DOM events) with the **game action layer** (EquipItem, UseItem, etc.).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/inputManager.ts` | Input state polling, event listeners, InputPacket assembly, action queuing | High |
| `common/src/packets/inputPacket.ts` | InputPacket serialization/deserialization, binary encoding scheme | High |
| `server/src/game.ts` | Packet reception (PacketStream deserialization), route to player | Medium |
| `server/src/objects/player.ts` (processInputs) | Movement/attacking state application, action processing dispatch | High |
| `common/src/constants.ts` | InputActions enum, GameConstants.player properties | Low |
| `client/src/scripts/console/variables.ts` | Key binding system, defaultBinds | Low |

---

## Input Sources

Real hardware events are captured by `InputManager` event listeners (not polling):

| Event | Source | Fields Affected |
|-------|--------|-----------------|
| `keydown` / `keyup` | `window` | movement (up/down/left/right), actions from console bindings |
| `pointerdown` | `#game` container | attacking ← true, mouse position |
| `pointerup` | `#game` container | attacking ← false |
| `pointermove` | `#game` container | mousePosition, rotation angle (desktop) OR movementAngle (mobile) |
| `wheel` | `#game` container | distanceToMouse (scroll zoom affects perceived distance) |
| Gamepad API | Polling loop (navigator.getGamepads) | **Not in current code** — mobile only uses touch |

**Mobile/Touch Handling:**
- Nipplejs joystick library provides virtual joystick
- `moving` + `angle` replace keyboard movement
- No turning (mouse aim unavailable on touch)
- Touch follows pointer (not separate multi-touch)

**Example (keyboard):**
```typescript
// @file client/src/scripts/managers/inputManager.ts
addEventListener("keydown", e => {
    const actions = mapper.getActionsBoundToInput(e.code);
    for (const action of actions) {
        if (action is movement → movement[direction] = true)
        else → addAction(action)
    }
});
```

---

## InputPacket Structure

The complete InputData type transmitted in each InputPacket:

```typescript
// @file common/src/packets/inputPacket.ts
type InputData = {
    readonly type: PacketType.Input,
    readonly movement: {
        readonly up: boolean,
        readonly down: boolean,
        readonly left: boolean,
        readonly right: boolean
    },
    readonly attacking: boolean,
    readonly actions: readonly InputAction[],
    readonly pingSeq: number
} & MobileMixin & TurningMixin;

// Mobile-specific (mutually exclusive with desktop turning)
type MobileMixin = 
    { readonly isMobile: false; readonly mobile?: undefined }
    | { readonly isMobile: true; readonly mobile: { readonly moving: boolean; readonly angle: number } };

// Desktop mouse aiming (mutually exclusive with mobile)
type TurningMixin = 
    { readonly turning: false; readonly rotation?: undefined }
    | { readonly turning: true; readonly rotation: number; readonly distanceToMouse: number };

// Actions are parameterized by type
type InputAction = 
    | { type: InputActions.EquipItem; slot: number }
    | { type: InputActions.DropWeapon; slot: number }
    | { type: InputActions.DropItem; item: HealingItemDefinition | ScopeDefinition | ... }
    | { type: InputActions.UseItem; item: HealingItemDefinition | ScopeDefinition }
    | { type: InputActions.Emote; emote: EmoteDefinition }
    | { type: InputActions.MapPing; ping: PlayerPing; position: Vector }
    | { type: InputActions.LockSlot | UnlockSlot | ... };  // no extra data
```

**Archive Encoding (actual binary wire format):**
```
[uint8] pingSeq
  │ (if pingSeq & 128) return early — "ping only" (ignored on server)
  │
[1 byte bitmask] up, down, left, right, isMobile, moving (if mobile), turning, attacking
  │
[conditional] if mobile: [uint8] angle (rotation2 encoding)
[conditional] if turning: [uint8] rotation + [2 bytes] distanceToMouse
[variable] actions[] array:
  └─ for each action:
     [uint8] type + (slot << 6) [if slot-based action]
     [conditional extra data by type]:
        - DropItem/UseItem: [Loots.writeToStream] item
        - Emote: [Emotes.writeToStream] emote
        - MapPing: [MapPings.writeToStream] ping + [position] vector
```

**Serialization Magic:**
- `writeBooleanGroup()` packs 8 booleans into 1 byte
- Action encoding combines type (6 bits) + slot (2 bits, for EquipItem/DropWeapon/LockSlot)
- Rotation encoded as 2-byte [`writeRotation2`] — quantized 0..2π to uint16
- Distance encoded as 2-byte float with known range [0, player.maxMouseDist]

---

## Frame Pacing — Client Send Rate

The client **does NOT send input every frame**. Instead, `InputManager.update()` checks for **delta** before sending:

**Pacing Algorithm (every client frame):**
```typescript
// @file client/src/scripts/managers/inputManager.ts:278+
update(): void {
    const packet = assembleInputData(movement, attacking, turning, actions, pingSeq);
    
    // Increment timer
    this._inputPacketTimer += Game.serverDt;  // serverDt = client frame time (e.g., 16ms @ 60fps)
    
    // Send if:
    // 1. State changed (areDifferent checks fields)
    // 2. OR timer exceeded 100ms
    if (
        this._lastInputPacket === undefined
        || areDifferent(this._lastInputPacket, packet)
        || this._inputPacketTimer >= 100
    ) {
        Game.sendPacket(InputPacket.create(packet));
        this._lastInputPacket = packet;
        this._inputPacketTimer = 0;
    }
    
    // Clear action queue (consumable per-packet)
    this.actions.length = 0;
}
```

**Change Detection (`areDifferent`)**:
```typescript
// @file common/src/packets/inputPacket.ts
export function areDifferent(newPacket: InputData, oldPacket: InputData): boolean {
    // Always resend if actions present
    if (newPacket.actions.length > 0) return true;
    
    // Check movement (object deep compare)
    for (const key of ["up", "down", "left", "right"] as const)
        if (newPacket.movement[key] !== oldPacket.movement[key]) return true;
    
    // Check mobile flag + angle
    if (newPacket.isMobile !== oldPacket.isMobile) return true;
    if (newPacket.isMobile)
        if (newPacket.mobile.moving !== oldPacket.mobile.moving
            || newPacket.mobile.angle !== oldPacket.mobile.angle) return true;
    
    // Check attacking, turning, rotation, distance
    for (const key of ["attacking", "turning", "rotation", "distanceToMouse"] as const)
        if (newPacket[key] !== oldPacket[key]) return true;
    
    return false;
}
```

**Timing Details:**
- **100ms threshold**: ~4 packets per second minimum (at 40 TPS server, = 1 packet per 2.5 ticks)
- **Client frame rate**: Up to 60 FPS (16.67ms per frame)
- **Server frame rate**: Fixed 40 TPS (25ms per tick = GameConstants.tps = 40)
- **Expected packets/sec**: 10–60 depending on how quiet input is

---

## Server Reception & Packet Deserialization

**Entry Point:** WebSocket message arrives at server

```typescript
// @file server/src/server.ts (Bun.serve routes)
// WebSocket established via /api/gameConnection route
// → forwards messages to game instance

// @file server/src/game.ts:281
onMessage(player: Player | undefined, message: ArrayBuffer): void {
    if (!player) return;
    
    const packetStream = new PacketStream(new SuroiByteStream(message));
    while (true) {
        const packet = packetStream.deserialize();  // Reads multi-packet stream
        if (packet === undefined) break;
        
        switch (packet.type) {
            case PacketType.Input:
                // Ignore if player hasn't joined, is dead, or game over
                if (!player.joined || player.dead || this.over) break;
                
                player.processInputs(packet);  // ← Apply state
                break;
            // ... other packet types (Join, Spectate, Debug)
        }
    }
}
```

**Deserialization is automatic** via `Packet<T>.deserialize()` on the `InputPacket` type. The inverse of the serialize encoding (see InputPacket Structure above).

---

## Input Buffer & Action Queuing

**Key insight:** The "buffer" is **not a true queue** — it's a consumable per-packet array.

| Aspect | Detail |
|--------|--------|
| **Storage** | `actions: InputAction[]` — mutable array in InputManager |
| **Capacity** | Max 7 items per packet (client-side check: `if (this.actions.length > 7) return;`) |
| **Consumption** | Cleared every time packet is sent: `this.actions.length = 0` |
| **Lifecycle** | action added → stored in array → sent in next packet → cleared → forgot |
| **No persistence** | Actions cannot "queue up" across multiple packets; only current packet's actions are sent |

**Action Queue Example:**
```typescript
// Player presses 'Q' (mapped to UseItem)
addAction({ type: InputActions.UseItem, item: HealthPotion });  // actions = [UseItem]

// Player presses 'E' (mapped to Emote)
addAction({ type: InputActions.Emote, emote: WaveEmote });  // actions = [UseItem, Emote]

// Frame update → packet sent
if (packet changed) {
    Game.sendPacket(InputPacket.create({
        // ...
        actions: [UseItem, Emote]  // both sent
    }));
    this.actions.length = 0;  // CLEAR (buffer is emptied)
}

// Next frame, if no new actions added
addAction(nothing);  // actions = []
// → areDifferent returns false if nothing else changed
// → no packet sent (until 100ms timer fires)
```

---

## Key/Button State Tracking

Movement and attack state are **stateful booleans**—not consumable like actions:

| State | Type | Source | Modified By | Transmitted |
|-------|------|--------|-------------|-------------|
| `movement.up` | boolean | keyboard (W) | keydown/keyup | every InputPacket |
| `movement.down` | boolean | keyboard (S) | keydown/keyup | every InputPacket |
| `movement.left` | boolean | keyboard (A) | keydown/keyup | every InputPacket |
| `movement.right` | boolean | keyboard (D) | keydown/keyup | every InputPacket |
| `attacking` | boolean | mouse/gamepad | pointerdown/up, gamepad trigger | every InputPacket |
| `turning` | boolean | mouse movement | pointermove (and `turning` flag set) | if true |
| `rotation` | number (0..2π) | mouse position | pointermove calculates angle to cursor | if turning |
| `distanceToMouse` | number | wheel zoom | wheel event or lerp to target | if turning |

**Important Subtlety:** Server tracks **edge detection**:
```typescript
// @file server/src/objects/player.ts:3372
processInputs(packet: InputData): void {
    const wasAttacking = this.attacking;
    const isAttacking = packet.attacking;
    
    this.attacking = isAttacking;
    this.startedAttacking ||= !wasAttacking && isAttacking;  // ← transition detect
    this.stoppedAttacking ||= wasAttacking && !isAttacking;  // ← transition detect
    
    // Later in tick(): check startedAttacking/stoppedAttacking for weapon firing
}
```

The server **remembers whether attacking started or stopped** because the gun fires **on the frame attacking becomes true**, not every frame. This prevents semi-automatic weapons from firing every tick.

---

## Network Transmission — Binary Format & Optimization

**Packet Encoding Rationale:**

| Field | Bytes | Rationale | Cost |
|-------|-------|-----------|------|
| pingSeq | 1 | Ping tracking for latency (bit 7 = "ping only" flag) | 1 B |
| Movement + flags | 1 | Packed as bitmask (8 booleans = 1 byte) | 1 B |
| Mobile angle | 1 | Only if isMobile=true (rotation2 quantization) | 0–1 B |
| Turning (rotate + dist) | 3 | Only if turning=true (rotation2 + float16) | 0–3 B |
| Actions | 1–∞ | Array length + action data (variable per action type) | 1–∞ B |
| **Minimum (no input)** | **2** | pingSeq + bitmask | – |
| **Average (idle)** | **2–4** | Sent every 100ms, tiny packets | – |
| **With turning** | **5–8** | ~5 bytes for movement+aiming | – |
| **With 5 actions** | **10–20** | Typical combat packet (gun swap, equipped, shot angle) | – |

**Why not send every frame?**
- 60 FPS client = 60 packets/sec × 10 bytes avg = 600 bytes/sec per player
- With 100 ms throttle: 10 packets/sec × 10 bytes = 100 bytes/sec per player
- **6× bandwidth savings**, and movement doesn't need 60 Hz precision (40 TPS server can't perceive it)

**Packet Loss Handling:**
- **No retransmission** — packets are transient state snapshots
- Out-of-order packets are ignored (server only processes latest)
- Drop 1 packet → only lose 1 update; next packet resends current state
- Actions **can be lost** if packet drops (player presses 'use item' but packet lost = item not used)
- No ack/retry mechanism (best-effort, acceptable for action games)

---

## Input Lag — Latency Sources & Perception

**Sources of perceived input lag** (milliseconds of delay):

| Source | Est. Latency | Why |
|--------|--------------|-----|
| **OS input latching** | 1–3 ms | Kernel event multiplexing |
| **Browser event dispatch** | 1–5 ms | JS engine events queued + executed |
| **Frame pacing** | 0–16 ms | InputManager.update() waits for next client frame to send |
| **Network RTT** | 20–200 ms | Internet latency, geography |
| **Server processing** | 0–25 ms | Game.tick() cycle (1/40 TPS = 25 ms max) |
| **Server → client update** | 20–200 ms | Update packet return trip |
| **Client rendering** | 0–16 ms | PixiJS render time, V-sync |
| **Display** | 1–10 ms | Monitor panel response time |
| **Total (typical)** | **60–450 ms** | Local: 30ms, Network: 200ms, Local: 30ms |

**Mitigations visible in code:**
- ✅ Client sends input aggressively (every change, min 100ms)
- ✅ Server applies inputs immediately in `processInputs()` (no queue delay)
- ✅ Server echoes player position in UpdatePacket (client sees self react instantly)
- ⚠️ No client-side prediction (player waits for server before visual feedback)
- ⚠️ No input interpolation (discrete snapshots, not smooth curves)

**Perception Tips:**
- Aiming (rotation) is **sent every frame** if mouse moves → low latency
- Movement (WASD) is sent **every change** → low latency
- Gun fire (attacking) is sent **every frame** (attacking=true state) → low latency
- Item actions (UseItem, EquipItem) sent **next packet** (at most 100ms delay)

---

## Input Processing Flow (Server-Side)

**When packet arrives, server applies inputs in this order:**

```typescript
// @file server/src/objects/player.ts:3369
processInputs(packet: InputData): void {
    // 1. Update movement from packet
    this.movement = {
        ...packet.movement,
        ...(packet.isMobile ? packet.mobile : { moving: false, angle: 0 })
    };
    
    // 2. Track ping (for latency diagnostics)
    this._pingSeq = packet.pingSeq;
    
    // 3. Update attacking state (with edge detection for firing)
    const wasAttacking = this.attacking;
    const isAttacking = packet.attacking;
    this.attacking = isAttacking;
    this.startedAttacking ||= !wasAttacking && isAttacking;  // fire trigger
    this.stoppedAttacking ||= wasAttacking && !isAttacking;
    
    // 4. Update aiming (turning)
    if (this.turning = packet.turning) {
        this.rotation = packet.rotation;
        this.distanceToMouse = packet.distanceToMouse;
    }
    
    // 5. Process action array
    const inventory = this.inventory;
    for (const action of packet.actions) {
        // dispatch by type (EquipItem, DropWeapon, UseItem, etc.)
        // → calls inventory.useItem(), .setActiveWeaponIndex(), etc.
        // → may trigger reload, drop, equip animations
    }
}
```

The actions are **processed sequentially** in order received. No deduplication or queueing—just iteration.

---

## Gotchas

### 1. **Actions Are Consumable Per-Packet**
Myth: "I sent UseItem, but the server never processed it."
Reality: The action was sent in packet N, but if packet N was lost to network, server never sees it.
- **Fix:** Client should NOT rely on single input per second. Resend if critical (e.g., hold spacebar to reload).

### 2. **Simultaneous Opposite Inputs**
Scenario: Player holds W (up=true) + S (down=true) at same time (say, key stuck or rapid toggle).
Expected: Both flags sent in InputPacket.movement = { up:true, down:true, left:false, right:false }
Server: Applies both. Movement calculation might average to zero or use priority (depends on game loop).
- **Real code:** Movement is applied directly; opposite directions cancel out in game loop velocity calc.

### 3. **Mouse Aim While Dead**
Server checks `if (!player.joined || player.dead || this.over) break;` — **ignores input packets from dead players**.
Client still sends packets, but server silently drops them. No waiting for respawn.

### 4. **100ms Timeout Always Fires**
Even if no input changes, a packet is sent every 100ms (to confirm player is still there; server uses this for timeout detection).
- **Why:** If client fully stops sending after idle, server can't distinguish "player disconnected" vs "player idle with no input".
- **Implication:** Bandwidth is nonzero even when idle.

### 5. **Actions Array Limit**
Client enforces max 7 actions per packet: `if (this.actions.length > 7) return;`
If player buttons spam faster than packets send, 8th action is dropped.
- **Typical case:** Not an issue (humans press ~5 buttons/sec max).
- **Bots/cheaters:** Could spam actions faster than 7/packet.

### 6. **PingSeq Bit 7 Means "Ignore Everything Else"**
```typescript
if ((pingSeq & 128) !== 0) return;  // deserialize early exit
```
If top bit set, the packet only contains pingSeq, nothing else. Likely used for **ping-only messages** to reduce packet size for latency measurements.
- **Implication:** Server doesn't process any movement/actions if bit 7 set.

### 7. **No Client-Side Prediction of Others**
Client does NOT guess where enemy will be based on their last input. All object positions come from server UpdatePacket.
- **Symptom:** Brief "teleport" when server corrects client-stale position.
- **Reason:** Server is source of truth; client doesn't simulate game physics for remote players.

### 8. **Rotation Not Sent Inside Bitmask**
The `turning` flag **is in the bitmask**, but actual rotation values (rotation + distanceToMouse) are sent **separately after the bitmask** if turning=true.
- Allows compact encoding when turning=false (no data wasted).

### 9. **Mobile vs Turning Are Mutually Exclusive**
A client **cannot be both mobile and desktop at once**. One InputData is either isMobile=true (with mobile.angle) OR isMobile=false (with turning/rotation) if the player is aiming.
- **Edge case:** Tablet could claim both; code assumes one or the other.

---

## Cross-Tier References

### Tier 2 (Parent Subsystem)
- [Input Management](../README.md) — Subsystem overview, event listeners, key binding system
- [Input Management Patterns](../patterns.md) — Reusable patterns for custom input modes

### Tier 1 (Architecture)
- [Architecture Overview](../../../architecture.md) — Client/server topology, networking layer
- [API Reference](../../../api-reference.md) — Binary protocol, PacketType.Input specification
- [Data Model](../../../datamodel.md) — Movement state, inventory state

### Related Tier 3 Modules
- [Serialization System — SuroiByteStream](../../../subsystems/serialization-system/modules/suroi-byte-stream.md) — Binary encoding (writeBooleanGroup, writeRotation2)
- [Networking — Packet Types](../../../subsystems/networking/modules/packet-types.md) — PacketStream, PacketType enum
- [Game Loop — Tick Module](../../../subsystems/game-loop/modules/tick.md) — Game.tick() frequency (40 TPS), serverDt

### External Concepts
- **BooleanGroup Encoding:** Bit-packing 8 booleans into 1 byte
- **Rotation2 Encoding:** 2-byte quantized rotation (0..2π → uint16)
- **nipplejs:** Virtual joystick library for mobile touch input
- **Bun WebSockets:** Low-level socket implementation used by server

---

## Summary

The **input pipeline** is the critical bridge from user intent to server state:

1. **Client collects** hardware input (keyboard, mouse) into state variables
2. **Client buffers** state and actions until change detected or 100ms timeout
3. **Client sends** binary InputPacket over WebSocket
4. **Server deserializes** packet and applies inputs immediately
5. **Server processes** actions (equip, drop, use) from action array
6. **Server broadcasts** result in UpdatePackets (all clients see player move/attack)

The design prioritizes **bandwidth efficiency** (merge 100 frames into 1 packet) and **responsiveness** (send immediately on change), achieving ~10–60 packets/sec with <10 bytes per idle packet and <20 bytes for combat.

