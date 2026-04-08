# Game Loop Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/game.ts, server/src/gameManager.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Game Loop subsystem runs the authoritative server simulation at 40 TPS (ticks per second). Each tick processes inputs, updates physics, resolves combat, advances the gas, serializes state, and broadcasts `UpdatePacket` to all connected players.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| @file server/src/game.ts | `Game` class — main tick loop, game state, object management |
| `server/src/gameManager.ts` | `GameManager`, `GameContainer` — multi-game, cluster workers |
| @file server/src/server.ts | Primary process — HTTP/WebSocket routing, spawns workers |
| `server/src/utils/grid.ts` | `Grid` — spatial hash for collision queries |
| `server/src/utils/idAllocator.ts` | `IDAllocator` — object ID allocation |

## Architecture

```
Primary Process (server.ts)
    └── GameManager
            ├── GameContainer (worker) → Game instance
            ├── GameContainer (worker) → Game instance
            └── ...

Worker Process (gameManager.ts)
    └── Game
            ├── setInterval(tick, 25)  // 40 TPS
            ├── WebSocket handlers
            └── Grid, Gas, Map, PluginManager
```

Each `Game` runs in a worker process. The primary process routes HTTP/WebSocket connections to the correct worker.

## Tick Order (per tick)

1. **Timeouts** — Execute queued timeouts (`addTimeout`)
2. **Airdrops** — Summon airdrops if mode interval elapsed
3. **Loot** — Update loot physics (`loot.update()`)
4. **Parachutes** — Update parachutes
5. **Projectiles** — Update thrown projectiles
6. **Synced Particles** — Update particles
7. **Bullets** — Update bullets, collect damage records
8. **Damage** — Apply bullet damage to hit objects
9. **Explosions** — Process explosion queue
10. **Gas** — `gas.tick()` (update position, radius, damage)
11. **Players** — First pass: `player.update()` (movement, inputs, actions)
12. **Serialization** — Serialize dirty objects (partial, then full)
13. **Players** — Second pass: `player.secondUpdate()` (visible objects, send packets)
14. **Players** — Third pass: `player.postPacket()` (cleanup)
15. **Map Indicators** — Clean up dead indicators
16. **Reset** — Clear dirty sets, packets, explosions, etc.
17. **Winning** — Check win condition, emit `game_end` if done
18. **Schedule** — `setTimeout(tick, 25)` for next tick

## Data Flow

```
InputPacket (from client)
    → Game.onMessage() → player.processInputPacket()
    → Buffered in player for next tick

Tick
    → player.update() reads buffered inputs
    → Movement, physics, actions applied
    → Objects marked dirty (partialDirtyObjects, fullDirtyObjects)
    → Serialize dirty objects to buffers
    → player.secondUpdate() builds UpdatePacket for each player
    → Send to client
```

## Key Concepts

### Dirty Tracking

- `partialDirtyObjects` — Objects with changed fields (delta send)
- `fullDirtyObjects` — Objects that need full state (new or major change)
- Objects call `setDirty()` or `setFullDirty()` when state changes

### Grid (Spatial Hash)

- Cell size: `GameConstants.gridSize` (32 units)
- `grid.pool.getCategory(ObjectCategory.X)` — Iterate objects by category
- Used for collision detection, visibility queries

### Timeouts

- `game.addTimeout(callback, ms)` — Schedule callback for future tick
- Timeouts run at start of tick; `game.now` is current timestamp

## Protocol Considerations

- **Affects protocol:** Indirectly. Game loop drives UpdatePacket content; object changes affect serialization.

## Team System

- **TeamMode:** Solo (1), Duo (2), Squad (4) — from `GameConstants`
- **Team:** `server/src/team.ts` — holds players, kill count, color index
- **Custom teams:** Players can create/join custom teams via lobby; `server/src/team.ts` and `server.ts` handle team sockets
- **Team assignment:** On join, players are assigned to teams; `teamID` is serialized in UpdatePacket

## Module Index (Tier 3)

- [Tick & Serialization](modules/tick-serialization.md) — Tick order, dirty tracking, UpdatePacket assembly
- [GameManager](modules/game-manager.md) — Multi-game, cluster workers, team/map switching
- [Grid](modules/grid.md) — Spatial hash for collision and visibility queries

## Dependencies

- **Depends on:** Objects, Map, Gas, Inventory, Packets, Plugins, Definitions
- **Depended on by:** Server (entry point), all game logic

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview
- **Tier 2:** [../objects/](../objects/) — Object model, serialization
- **Tier 2:** [../map/](../map/) — Map generation
- **Tier 2:** [../gas/](../gas/) — Gas mechanics
- **Tier 2:** [../plugins/](../plugins/) — Event hooks
