# Q9: What happens when a packet is lost or arrives out of order — is there any compensation or does WebSocket TCP handle it?

**Answer:**

**WebSocket TCP handles it — Suroi has NO explicit packet loss/out-of-order compensation at the protocol layer.**

Instead, Suroi **relies entirely on TCP's reliability guarantees**. WebSocket is built on TCP, which guarantees:
- **In-order delivery** — packets arrive in the same order they were sent
- **No loss** — TCP retransmits any dropped packets automatically

The game design then exploits this by using **idempotent state updates** that safely tolerate duplicate packets (should they ever arrive).

---

## Design Details

### 1. **No Sequence Numbers or Acknowledgments**

The Suroi protocol has **zero packet sequencing or acknowledgment logic**:

- `InputPacket` has a `pingSeq` field for measuring latency, but it's **not for retransmission** — it's simply a probe number the client increments to measure round-trip time
- `UpdatePacket` has no ACK mechanism  
- There are **no retransmission timers** or lost-packet detection at the application layer

### 2. **Idempotency (Applying Same Update Twice ≈ Applying Once)**

The packet design is stateless, meaning applying the same packet twice has nearly the same effect as applying once:

**UpdatePacket — idempotent by design:**
- Carries a **delta state** (only changed fields), not a command
- Applying update twice: position → [128, 256] twice = same result as once
- Example: `{ objectId: 7, position: [128, 256], rotation: 1.57 }` is safe to receive twice

**InputPacket — aggregated & discarded:**
- Client sends input every tick (~25 ms)
- Server processes and **discards old input after 25ms** — even if a duplicate arrives late, it's thrown away
- Only the latest input state matters

**KillPacket — instantaneous & idempotent:**
- Killing a player twice has no effect (already dead)
- Reapplying the same kill doesn't change anything

### 3. **Heartbeat Monitoring (Detects Dead Connections, Not Lost Packets)**

Both client and server monitor **silence** to detect dropped connections, but this is for **timeout detection**, not packet loss recovery:

**Server → Client timeout:**
```typescript
// Client marks when last UPDATE arrived
let lastUpdateTime = Date.now();

// If silent for ~5 seconds, client assumes server dead
if (Date.now() - lastUpdateTime > TIMEOUT_MS) {
    this.disconnect("Server timeout");
}
```

**Client → Server timeout:**
```typescript
// Server marks when last INPUT arrived
if (Date.now() - player.lastInputTime > HEARTBEAT_TIMEOUT_MS) {  // ~10-15 sec
    player.socket.close(1000, "Heartbeat timeout");
}
```

This timeout detects **network outages** (complete silence), not individual packet loss.

### 4. **Practical Impact: Packet Loss is Extremely Rare**

Since WebSocket runs over TCP:
- **Lost packets are retransmitted by the OS kernel** — transparent to the application
- **Out-of-order packets are reordered by the OS** — application never sees them out of order
- **Duplicate packets at the OS level** are filtered by TCP's sequence number tracking

The only way a packet could be lost **end-to-end** is if the entire TCP connection dies (network cable unplugged, etc.), at which point the heartbeat timeout kicks in and the player is disconnected.

---

## Summary

| Scenario | Suroi Behavior |
|----------|---|
| **Packet dropped by network** | OS/TCP layer retransmits automatically; application never sees it |
| **Packet arrives out of order** | TCP reorders it; application receives in-order stream |
| **Duplicate packet (impossible at TCP layer but if app-logic duped)** | Idempotent design means applying twice ≈ once |
| **Entire connection dies (network down)** | Heartbeat timeout (~5-15 sec) triggers disconnect; player removed from game |
| **Explicit application-layer recovery** | None — relies entirely on TCP guarantees |

The design is elegant: **by using TCP + stateless delta updates, Suroi avoids the complexity of custom reliability layers** (like what you'd need with UDP). The trade-off is slightly higher latency compared to optimized UDP games, but for a browser-based game, the simplicity win is worth it.

---

## References

- **Tier 1:** `docs/api-reference.md` — Transport layer (WebSocket binary protocol)
- **Tier 2:** `docs/subsystems/networking/README.md` — Networking overview
- **Tier 3:** `docs/subsystems/networking/modules/protocol.md` — Detailed protocol, error handling, heartbeat
- **Tier 3:** `docs/subsystems/networking/modules/updatepacket.md` — UpdatePacket design, idempotency, TCP reliability
