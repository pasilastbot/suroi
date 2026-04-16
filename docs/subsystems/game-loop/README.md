# Game Loop Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/game-loop/modules/ -->
<!-- @source: server/src/ -->

## Purpose

The game loop subsystem owns every game instance's lifecycle: it spawns isolated Worker processes (one per game), drives the 40 TPS simulation tick inside each worker, processes player input events, updates all game objects, serializes dirty state into binary `UpdatePacket`s, and sends them to all connected clients. It is the top-level orchestrator — everything else in the server is called from here.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/server.ts` | Primary-process entry point. Starts `GameManager`, serves `/api/serverInfo`, `/api/getGame`, and `/team` HTTP+WebSocket endpoints via `Bun.serve()`. Calls `gameManager.newGame(0)` on startup to launch the first game. Only runs when called directly from the command line (`require.main === module`) on the primary Cluster process. |
| `server/src/gameManager.ts` | `GameManager` — manages the `games` array of `GameContainer` instances, decides which game a new player should join (`findGame()`), creates new game workers (`newGame()`), and broadcasts live-configuration changes (team mode, map, map scale range) to all workers via IPC. `GameContainer` — wraps a single `Cluster.Worker`, caches the latest `GameData` snapshot received from the worker, and forwards `WorkerMessage`s into the worker process. |
| `server/src/game.ts` | `Game` class — the full game simulation. Constructed inside each worker process. Owns the `Grid`, `GameMap`, `Gas`, `PluginManager`, all entity sets, and the tick loop driven by `setTimeout`. Implements the `GameData` interface and calls `process.send()` to report state back to the primary. |

## Architecture

The server runs as a **multi-process cluster** using Node.js `Cluster`:

- The **primary process** (`server.ts`) runs `GameManager`, serves public HTTP routes, and never touches game simulation logic.
- Each **game worker** is a separate OS process forked with `Cluster.fork()`. One `Game` instance lives in each worker.
- Workers communicate with the primary exclusively via IPC messages:
  - **Worker → Primary**: `Partial<GameData>` objects sent with `process.send?.(data)`, merged by `GameContainer` into its `_data` cache.
  - **Primary → Worker**: typed `WorkerMessage` objects sent through `GameContainer.sendMessage()`.

### Class Relationships

```
server.ts (primary)
  └─ GameManager
       ├─ games: Array<GameContainer | undefined>
       │    └─ GameContainer
       │         ├─ worker: Cluster.Worker   ←──IPC──→  Game (worker process)
       │         └─ _data: GameData (cached)
       ├─ teamMode: Switcher<TeamMode>
       └─ map: Switcher<string>
```

### IPC Message Types (`WorkerMessages` enum)

| Message | Direction | Effect in Worker |
|---------|-----------|-----------------|
| `UpdateTeamMode` | Primary → Worker | Updates `teamMode` local variable (takes effect on next new game) |
| `UpdateMap` | Primary → Worker | Calls `game.kill()` — the current game ends, new one starts with new map |
| `UpdateMapOptions` | Primary → Worker | Updates `mapOptions` (map scale range) for next game |
| `NewGame` | Primary → Worker | Calls `game.kill()` and immediately constructs a fresh `Game` |

Worker reports game state back by sending `Partial<GameData>` (`{ aliveCount, allowJoin, over, startedTime }`).

### Game Worker Startup

When `Cluster.fork()` is called, environment variables are passed: `id`, `teamMode`, `map`, `mapScaleRange`. The worker branch of `gameManager.ts` (guarded by `!Cluster.isPrimary`) parses these and calls:

```typescript
let game = new Game(id, teamMode, map, mapOptions);
```

The worker also starts its own `Bun.serve()` on `Config.port + id + 1` to accept `/play` WebSocket connections directly from clients.

### Live Switching — `Switcher` and `StaticOrSwitched`

`Config.teamMode` and `Config.map` support either a static value or a scheduled rotation (cron). The `Switcher<T>` class in `server/src/utils/serverHelpers.ts` wraps this: when a scheduled switch fires, it broadcasts the appropriate `WorkerMessage` to all running games and calls `resetTeams()`.

## Data Flow

```
Browser client
    │
    │  HTTP GET /api/getGame  (primary: server.ts)
    ▼
GameManager.findGame()
    │  eligible game found or GameContainer.newGame() called
    ▼
Client connects  ws://host:(Config.port + gameID + 1)/play  (worker: gameManager.ts)
    │
    │  WebSocket upgrade → PlayerSocketData attached
    ▼
game.addPlayer(socket)          ← creates Player, adds to Grid + connectedPlayers
    │
    │  JoinPacket received
    ▼
game.activatePlayer(player, joinData)   ← joins livingPlayers, queues JoinedPacket
    │
    ▼
game.tick()  (every ~25 ms)
    │  processes objects, sends UpdatePacket to all connectedPlayers
    ▼
player.secondUpdate()  →  socket.send(updatePacketBytes)
```

## Game Tick Structure

**Tick rate:** `GameConstants.tps = 40` TPS → `1000 / 40 = 25 ms/tick`
(`idealDt = 1000 / (Config.tps ?? GameConstants.tps)`)

The tick is driven by `setTimeout(this.tick.bind(this), this.idealDt - elapsed)` — self-rescheduling after each tick to maintain the target rate. The sequence within each `tick()` call:

| Step | Code | Description |
|------|------|-------------|
| 1 | `this._dt = now - this._now; this._now = now` | Capture delta time and current timestamp |
| 2 | `for (const timeout of this._timeouts)` | Execute expired `Timeout` callbacks; remove killed/expired entries |
| 3 | Airdrop interval check | If `mode.summonAirdropsInterval` is set and elapsed, call `this.summonAirdrop(...)` |
| 4 | `loot.update()` | Update all `Loot` objects (physics, despawn) |
| 5 | `parachute.update()` | Update all `Parachute` objects |
| 6 | `projectile.update()` | Update all `Projectile` objects |
| 7 | `syncedParticle.update()` | Update all `SyncedParticle` objects |
| 8 | `bullet.update()` → collect `DamageRecord[]` | Advance all bullets; mark dead bullets; accumulate damage records |
| 9 | Dead bullet cleanup | For dead bullets: fire `onHitExplosion` if defined; delete from `this.bullets` |
| 10 | Process damage records | For each `{object, damage, source, weapon, position}`: call `object.damage()`, apply on-hit projectiles, speed multipliers, perk removal, hollow-point highlight |
| 11 | `explosion.explode()` | Resolve all `Explosion` instances queued this tick |
| 12 | `this.gas.tick()` | Advance gas zone position/radius |
| 13 | **First player loop** `player.update()` | Movement, animation, actions for all `livingPlayers` |
| 14 | Serialize dirty objects | `partialObject.serializePartial()` for `partialDirtyObjects`; `fullObject.serializeFull()` for `fullDirtyObjects` (full takes priority — partial skipped if object is also full-dirty) |
| 15 | **Second player loop** `player.secondUpdate()` | Calculate each connected player's visible object set; build and send `UpdatePacket` |
| 16 | **Third player loop** `player.postPacket()` | Post-send cleanup per player |
| 17 | Map indicator cleanup | Remove dead `MapIndicatorSerialization` entries; release IDs; reset dirty flags |
| 18 | **Reset phase** | Clear all tick-scoped collections: `fullDirtyObjects`, `partialDirtyObjects`, `newBullets`, `explosions`, `emotes`, `newPlayers`, `deletedPlayers`, `packets`, `planes`, `mapPings`; reset all `*Dirty` flags |
| 19 | Win condition check | If game started and alive count ≤ threshold: send `GameOverPacket`, emit `player_did_win` + `game_end` plugin events, schedule disconnect in 1 s |
| 20 | Performance logging | Accumulate `tickTime` in `_tickTimes`; log `ms/tick ± stddev` and load % every 200 ticks |
| 21 | `pluginManager.emit("game_tick", this)` | Notify plugins that a tick completed |
| 22 | `setTimeout(this.tick.bind(this), ...)` | Schedule next tick (skipped if `this._stopped`) |

## GameManager — Worker Architecture

```
GameManager
├── games: Array<GameContainer | undefined>   (indexed by game ID, max Config.maxGames)
├── creating: GameContainer | undefined        (creation lock — prevents duplicate spawns)
├── teamMode: Switcher<TeamMode>
├── map: Switcher<string>
├── mode: ModeName                             (derived from current map)
└── nextMode?: ModeName                        (for UI display)

GameContainer
├── id: number
├── worker: Cluster.Worker
├── _data: GameData                            (last snapshot from worker IPC)
├── promiseCallbacks[]                         (queued resolvers waiting for allowJoin)
└── sendMessage(WorkerMessage): void
```

**Creating a game** (`newGame(id)`):
1. If `this.creating` is set, queue the resolver in `promiseCallbacks` and return — the in-progress game will notify all waiters when `allowJoin` becomes `true`.
2. Otherwise allocate a slot, set `this.creating`, call `Cluster.fork({ id, teamMode, map, mapScaleRange })`.
3. Worker starts, calls `game.setGameData({ allowJoin: true })` via `process.send()`.
4. Primary receives the message, calls all queued `promiseCallbacks`.

**Finding a game** (`findGame()`):
- Returns a random eligible `GameContainer` (one where `allowJoin && aliveCount < Config.maxPlayersPerGame`).
- Falls back to `newGame(undefined)` if no eligible game exists.

**`mapScaleRange`** is recomputed every 60 s in the primary based on total player count and `Config.mapScaleRanges` thresholds. New range is broadcast to all workers as `WorkerMessages.UpdateMapOptions`.

## Interfaces & Contracts

### Exported from `server/src/game.ts`
| Symbol | Kind | Notes |
|--------|------|-------|
| `Game` | `class` | Main game simulation; implements `GameData` |
| `Airdrop` | (implicitly typed) | Used in `game.airdrops` |

### Exported from `server/src/gameManager.ts`
| Symbol | Kind | Notes |
|--------|------|-------|
| `GameData` | `interface` | `{ aliveCount, allowJoin, over, startedTime }` — the IPC state snapshot |
| `GameContainer` | `class` | Wrapper for a worker process |
| `GameManager` | `class` | Manages all `GameContainer`s |
| `WorkerMessages` | `enum` | `UpdateTeamMode`, `UpdateMap`, `UpdateMapOptions`, `NewGame` |
| `WorkerMessage` | `type` | Discriminated union of all IPC message payloads |

## Dependencies

- **Depends on:**
  - [Object Definitions](../object-definitions/) — all entity definitions (`Bullets`, `Obstacles`, `Loots`, etc.) are looked up from `common/src/definitions/` during every tick
  - [Networking](../networking/) — `UpdatePacket`, `JoinPacket`, `KillPacket`, `PacketStream`, `SuroiByteStream` from `common/src/packets/`
  - [Spatial Grid](../spatial-grid/) — `server/src/utils/grid.ts`; `Grid.pool.getCategory()` is used to iterate object sets in every tick
  - [Map Generation](../map/) — `server/src/map.ts`; `new GameMap(this, map, mapOptions)` called in `Game` constructor
  - [Gas System](../gas/) — `server/src/gas.ts`; `this.gas.tick()` called once per tick (step 12)
  - [Inventory](../inventory/) — `GunItem`, `MeleeItem`, `ThrowableItem` affect bullet creation and damage records
  - [Plugin System](../plugins/) — `PluginManager` events emitted at `game_created`, `game_tick`, `game_end`, `player_will_connect`, `player_did_win`
- **Depended on by:** Nothing — this is the top-level orchestrator

## Module Index (Tier 3)

For implementation details, see:
- [Tick Module](modules/tick.md) — per-tick game state update logic (object updates, dirty tracking, win condition)

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview
- **Tier 2:** [Networking](../networking/) — `UpdatePacket` binary protocol and `SuroiByteStream`
- **Tier 2:** [Spatial Grid](../spatial-grid/) — `Grid` broad-phase collision and object pool
- **Tier 2:** [Gas System](../gas/) — gas tick called from step 12
- **Tier 2:** [Map Generation](../map/) — `GameMap` constructed in `Game` constructor
- **Tier 2:** [Plugin System](../plugins/) — event bus wired into the tick and lifecycle methods
- **Patterns:** [patterns.md](patterns.md) — Reusable patterns in this subsystem
