# Q8: How large is a typical UpdatePacket in bytes, and what is the downstream bandwidth per player at 40 TPS?

**Answer:**

A **typical UpdatePacket is approximately 230 bytes**, with downstream bandwidth of **~9.2 KB/sec per player** at 40 TPS.

---

## Detailed Breakdown

### UpdatePacket Structure & Typical Size

| Component | Typical Size |
|-----------|---|
| Packet flags (uint16) | 2 bytes |
| PlayerData (health, adrenaline, inventory, teammates) | 30–80 bytes (typically ~50) |
| PartialObjects (8 visible players × ~10 bytes each) | 80 bytes |
| FullObjects (new objects entering view, 2 items) | 50 bytes |
| Bullets/Explosions/Emotes | 20 bytes |
| DeletedObjects | 4 bytes |
| Gas state (if changed) | 15 bytes |
| UI (NewPlayers, MapIndicators, etc.) | 10 bytes |
| **Total** | **~230 bytes** |

This is for a typical scenario: **40-player game where a client sees ~30 objects (8 other players + 22 props/items)**.

### Per-Component Costs

**PlayerData section** (30–80 bytes typical):
- Boolean headers: 3 bytes
- Health/adrenaline/shield: 2 bytes each
- Inventory + weapons: 2–5 bytes 
- Teammates (if squad): 3 header + 10 bytes per teammate
- **Range: 30–80 bytes** depending on what changed

**Player Partial Updates** (7–10 bytes each):
- Position: 4 bytes (`writePosition` quantized to map bounds)
- Rotation: 2 bytes (`writeRotation2`)
- Animation/action: 1 byte
- Optional item data: 1–2 bytes
- **Range: 7–10 bytes per visible player**

**Player Full Serialization** (25–50 bytes):
- Sent only when object becomes visible or resets
- Layer, team ID, active weapon, cosmetics, armor, health state
- **Cost: 25–50 bytes per player create**

### Bandwidth Calculation

At **40 TPS (25ms per tick)**, with **230 bytes per UpdatePacket**:

$$\text{Downstream bandwidth per player} = 230 \text{ bytes/tick} \times 40 \text{ ticks/sec} = 9,200 \text{ bytes/sec} = \boxed{9.2 \text{ KB/sec}}$$

**For a 4-player squad:**  
4 × 9.2 KB/sec = **36.8 KB/sec** upstream per player on the server.

### Space Optimization Techniques

Three key techniques are used:

1. **Ranged Float Quantization** — health [0, 200] packed into 2 bytes instead of 4-byte float32
2. **Delta Encoding** — send only changed objects via `partialDirtyObjects` vs. full state via `fullDirtyObjects`
3. **Visibility Culling** — each client receives only ~30 visible objects (vs. 200+ on server) via spatial grid

### Network Behavior

- **UpdatePacket sent every 25ms** (40 TPS = `GameConstants.tps`)
- **Flags field** (`uint16`) gates 16 sections — only set flags are transmitted
- **WebSocket transport** — multiple packets multiplexed per frame via `PacketStream`
- **Position precision** — quantized to ~0.029 units (imperceptible at 1924×1924 map scale)

---

## References

- **Tier 1:** `docs/api-reference.md` — packet protocol overview
- **Tier 2:** `docs/subsystems/networking/README.md` — architecture & data flow
- **Tier 3:** `docs/subsystems/networking/modules/updatepacket.md` — complete struct + breakdown
- **Tier 2:** `docs/subsystems/serialization-system/README.md` — encoding techniques
