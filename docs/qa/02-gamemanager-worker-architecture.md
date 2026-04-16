# Q2: How does the GameManager → Worker architecture work, and why is each game forked as a separate worker instead of running in-process?

## Answer: GameManager → Worker Architecture

### **How It Works**

The suroi server uses a **multi-process cluster model** where:

1. **Primary Process** runs `server.ts` — serves public HTTP routes (`/api/getGame`, `/api/serverInfo`, `/team` WebSocket) and manages game orchestration via `gameManager.ts`
2. **Broker Layer** — `GameManager` (and `Switcher` for config rotation) run in the primary, deciding when to fork new workers and coordinating live configuration changes
3. **Worker Processes** — each game gets its own isolated OS process via `Cluster.fork()`, running the full `Game` instance on a dedicated port

**Process Map:**
```
┌─────────────────────────────────────────┐
│ Primary: server.ts                       │
│ • HTTP: /api/serverInfo, /api/getGame   │
│ • WebSocket: /team (custom team lobby)  │
│ • Listens on Config.port                │
│                                         │
│ GameManager                             │
│ ├─ games[]: GameContainer[] (metadata)  │
│ ├─ Switcher<TeamMode> (cron rotation)   │
│ └─ Switcher<string> (map rotation)      │
└──────────────┬──────────────────────────┘
              │ Cluster.fork({id, teamMode, map, mapScaleRange})
              │ IPC: WorkerMessage (UpdateTeamMode|UpdateMap|NewGame)
              ▼
┌──────────────────────────────────────┐
│ Worker N (gameManager.ts worker block)│
│ • WebSocket: /play (game players)     │
│ • Listens on Config.port + id + 1     │
│ • Game instance + tick loop (40 TPS)  │
│ • Bun.serve() on dedicated port       │
└──────────────────────────────────────┘
```

### **Communication Flow**

`gameManager.ts` implements the orchestration:

```typescript
// Primary creates a GameContainer wrapping each worker
this.worker = Cluster.fork({
    id,
    teamMode: gameManager.teamMode.current,
    map: gameManager.map.current,
    mapScaleRange: gameManager.mapScaleRange
}).on("message", (data: Partial<GameData>): void => {
    this._data = { ...this._data, ...data }; // Cache GameData
});
```

**Message types** (`WorkerMessages` enum):
- `UpdateTeamMode` — Primary → Worker (team mode rotation)
- `UpdateMap` — Primary → Worker (kills current game, starts new one)
- `UpdateMapOptions` — Primary → Worker (map scale range for dynamic scaling)
- `NewGame` — Primary → Worker (restart the current game)

**State sync:**
- **Worker → Primary**: `process.send(Partial<GameData>)` via `game.updateGameData()` whenever `aliveCount`, `allowJoin`, `over`, or `startedTime` change
- **Primary → Worker**: `GameContainer.sendMessage(message)` sends control commands

The worker block (guarded by `!Cluster.isPrimary` at `gameManager.ts`) reads environment variables and constructs the `Game`:
```typescript
let game = new Game(id, teamMode, map, mapOptions);
```

### **Why Each Game Forks as a Separate Worker (Not In-Process)**

**Benefits:**

1. **Crash Isolation** — A bug in one game simulation (e.g., infinite loop, memory corruption) kills only that worker, not all games. Other games continue running.

2. **Memory Isolation** — Each game's `Game` instance, `Grid`, `GameMap`, object pools, and player set are locked in separate memory spaces. Memory leaks in one game don't bleed into others. The `IDAllocator` (uint16 pool) is per-game, not global.

3. **Horizontal Scaling** — Each worker can bind to its own port (`Config.port + id + 1`), allowing:
   - Direct WebSocket connections from clients to avoid bottlenecking through the primary
   - Future distribution across multiple machines (listen on `hostname:port`, not localhost)
   - Load balancing: primary just directs new players to eligible games via `/api/getGame` → `GameManager.findGame()`

4. **Configuration Hot-Swapping** — New team modes and maps spread to workers via IPC; old games can continue to completion while new games start with the new config.

### **Tradeoffs (The Cost)**

> **Worker process overhead:** Each active game runs in a separate process spawned via `Cluster.fork()`. This incurs OS context-switch overhead (~1-2 ms latency) and IPC serialization cost for each message.

**Specific costs:**
- **IPC serialization** — Every `GameData` update (aliveCount, state changes) and every worker message (config rotation) crosses process boundaries, requiring serialization. At 40 TPS per game, this adds latency.
- **Context-switch overhead** — ~1-2 ms per message as the OS scheduler switches threads between primary and worker
- **Port allocation complexity** — Workers listen on unique ports; port conflicts cause silent failures (no error reporting)
- **No live hot-reload** — Map changes require the worker to terminate and restart; players must reconnect

### **Key Design Pattern: Creation Lock**

`gameManager.ts` uses `GameManager.creating` to serialize game startup:

```typescript
if (this.creating) return this.creating.id; // Wait for in-flight game
```

If a player requests a game while one is starting, their request queues in `GameContainer.promiseCallbacks` and resolves when the worker sends `{ allowJoin: true }`.

---

## References

**Tier 1 — Architecture:**
- `docs/architecture.md` — Component map, process model, known overhead
- `docs/api-reference.md` — WebSocket protocol (`/play` endpoint)

**Tier 2 — Game Loop Subsystem:**
- `docs/subsystems/game-loop/README.md` — Architecture, IPC message types, data flow
- `docs/subsystems/game-loop/patterns.md` — Worker-Per-Game pattern, Dirty Object Tracking, Self-Scheduling Tick Loop

**Source Code:**
- `server/src/gameManager.ts` — `GameManager` (primary), `GameContainer`, `Cluster.fork()`, worker block
- `server/src/game.ts` — `Game` constructor, `updateGameData()`, IPC message handler
- `server/src/server.ts` — Primary process entry, HTTP routes, GameManager instantiation
