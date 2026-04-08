# Packets — InputPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/inputPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

`InputPacket` carries player input from client to server: movement (WASD or mobile stick), aim direction, attack state, and discrete actions (reload, interact, use item, emote, etc.).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `inputPacket.ts` | InputPacket, InputAction types, areDifferent() | Medium |

## Business Rules

- **pingSeq:** 7-bit sequence number; high bit (128) signals "skip this frame" (no input change)
- **Movement:** `up`, `down`, `left`, `right` booleans; mobile uses `angle` instead
- **Turning:** When `turning` is true, includes `rotation` and `distanceToMouse`
- **Actions:** Variable-length array; slot-based actions pack slot in top 2 bits of type byte
- **areDifferent():** Client uses this to avoid sending duplicate packets

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 1:** [../../protocol.md](../../protocol.md) — InputActions enum
- **Tier 1:** [../../datamodel.md](../../../datamodel.md) — InputActions, GameConstants
