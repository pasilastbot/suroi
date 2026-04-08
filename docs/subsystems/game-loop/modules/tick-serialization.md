# Game Loop — Tick & Serialization Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/game-loop/README.md -->
<!-- @source: server/src/game.ts, server/src/objects/player.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the per-tick execution order and the dirty-tracking serialization pipeline that produces `UpdatePacket` for each player.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/game.ts | `tick()` — full tick order, dirty serialization loop | High |
| @file server/src/objects/player.ts | `secondUpdate()` — per-player packet assembly, visibility | High |
| @file server/src/objects/gameObject.ts | `setDirty()`, `setFullDirty()` — dirty registration | Medium |

## Tick Order (per tick)

1. **Timeouts** — Run queued `addTimeout` callbacks
2. **Airdrops** — Summon airdrops if mode interval elapsed
3. **Loot** — `loot.update()` (physics)
4. **Parachutes** — Update parachutes
5. **Projectiles** — Update thrown projectiles
6. **Synced Particles** — Update particles
7. **Bullets** — Update bullets, collect damage records
8. **Damage** — Apply bullet damage to hit objects
9. **Explosions** — Process explosion queue
10. **Gas** — `gas.tick()`
11. **Players (1st pass)** — `player.update()` — movement, inputs, actions
12. **Serialization** — Serialize dirty objects (partial, then full)
13. **Players (2nd pass)** — `player.secondUpdate()` — build UpdatePacket per player
14. **Players (3rd pass)** — `player.postPacket()` — cleanup
15. **Map Indicators** — Clean up dead indicators
16. **Reset** — Clear dirty sets, packets, explosions
17. **Winning** — Check win condition
18. **Schedule** — `setTimeout(tick, 25)` for next tick

## Dirty Tracking

- **partialDirtyObjects** — Objects with changed fields; send delta only
- **fullDirtyObjects** — Objects needing full state (new, respawn, major change)
- Objects call `setDirty()` or `setFullDirty()` when state changes
- If an object is in both sets, only full serialization is sent

## Serialization Flow

```
For each object in partialDirtyObjects (skip if in fullDirtyObjects):
    → object.serializePartial(stream)
    → Append to partial buffer

For each object in fullDirtyObjects:
    → object.serializeFull(stream)
    → Append to full buffer

Per player (secondUpdate):
    → Filter objects by visibility (in view, alive)
    → Build UpdatePacket from partial + full buffers
    → Send
```

## Business Rules

- Serialization runs once per tick; buffers are shared across all players
- Each player receives only objects they can see (visibility culling)
- `fullDirtyObjects` takes precedence over `partialDirtyObjects` for the same object

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Game loop overview
- **Tier 2:** [../packets/](../packets/) — UpdatePacket format
- **Tier 3:** [../objects/modules/server-objects.md](../../objects/modules/server-objects.md) — fullStream/partialStream
