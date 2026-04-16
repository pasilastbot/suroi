# Game Loop — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/game-loop/README.md -->

## Pattern: Dirty Object Tracking

**When to use:** Any time a game object's state changes and the change must be sent to clients in the next `UpdatePacket`.

**Implementation:**

`Game` maintains two `Set<BaseGameObject>` fields that are populated during the tick and cleared at the end of the reset phase:

| Set | Meaning |
|-----|---------|
| `fullDirtyObjects` | Object needs its **full** state serialized — used when an object first becomes visible to a client, or when a field covered only by full serialization changes (`fullAllocBytes`) |
| `partialDirtyObjects` | Object needs only its **partial** state serialized — position, rotation, and other high-frequency fields (`partialAllocBytes`) |

Priority rule: if an object is in **both** sets, the partial serialization is skipped and only `serializeFull()` is called (tick step 14, `if (this.fullDirtyObjects.has(partialObject)) continue`).

After serialization (steps 14–15), the second player loop (`player.secondUpdate()`) reads these serialized buffers to build `UpdatePacket`s. At the end of the tick (step 18 — reset phase) both sets are cleared:

```typescript
this.fullDirtyObjects.clear();
this.partialDirtyObjects.clear();
```

Objects add themselves to the dirty sets by calling inherited helpers on `BaseGameObject` (e.g. `setDirty()` / `setFullDirty()`). Individual object update methods (e.g. `player.update()`, `bullet.update()`) call these when state changes.

**Example files:** `@file server/src/game.ts`

---

## Pattern: Worker-Per-Game

**When to use:** Every new game instance. Each game runs in its own isolated OS process for crash safety, memory isolation, and horizontal scaling.

**Implementation:**

The primary process calls `Cluster.fork()` with game configuration in environment variables:

```typescript
this.worker = Cluster.fork({
    id,
    teamMode: gameManager.teamMode.current,
    map: gameManager.map.current,
    mapScaleRange: gameManager.mapScaleRange
});
```

The worker branch (guarded by `!Cluster.isPrimary` at the bottom of `gameManager.ts`) reads these env vars, parses them, and constructs a `Game`:

```typescript
const id = parseInt(data.id);
let game = new Game(id, teamMode, map, mapOptions);
```

The worker also starts its own `Bun.serve()` on port `Config.port + id + 1` for direct WebSocket connections from game clients.

**State synchronisation via IPC:**

- Worker → Primary: `process.send?.(partialGameData)` — the `Game` class calls this via `updateGameData()` whenever `aliveCount`, `allowJoin`, `over`, or `startedTime` changes. The primary's `GameContainer` merges the partial object into `_data`.
- Primary → Worker: `GameContainer.sendMessage(workerMessage)` — sends `UpdateTeamMode`, `UpdateMap`, `UpdateMapOptions`, or `NewGame`.

**Creation lock:** `GameManager.creating` holds the in-flight `GameContainer` while a new worker is starting. Any concurrent `findGame()` call that arrives during startup queues its resolver in `GameContainer.promiseCallbacks`. All queued resolvers are called when the worker sends `{ allowJoin: true }`.

**Example files:** `@file server/src/gameManager.ts`

---

## Pattern: Game Object Lifecycle

**When to use:** Adding or removing any entity (player, obstacle, loot, bullet, synced particle, etc.) from the simulation.

**Implementation:**

**Adding an object:**

All entities inherit from `BaseGameObject`. When created they must be registered in two places:

1. `this.grid.addObject(object)` — inserts the object into the spatial hash grid so it can be found by broad-phase queries and per-category iteration (`grid.pool.getCategory(ObjectCategory.X)`).
2. Appropriate collection on `Game` — e.g. `this.livingPlayers.add(player)`, `this.bullets.add(bullet)`, or `this.newPlayers.push(player)` (tick-scoped, flushed in the reset phase).

Objects that appear for the first time this tick are also added to `fullDirtyObjects` so clients receive a full serialisation.

**Removing an object:**

1. Call `this.grid.removeObject(object)` to remove it from the spatial hash.
2. Remove from the relevant collection (e.g. `this.livingPlayers.delete(player)`).
3. For players: push the numeric ID to `this.deletedPlayers` so `UpdatePacket` can notify all clients.
4. Return the object's `id` to `this._idAllocator` via `IDAllocator.give()` (where applicable).

**Tick-scoped arrays** (`newBullets`, `newPlayers`, `explosions`, `emotes`, `planes`, `mapPings`, `deletedPlayers`, `packets`) are cleared at the end of every tick in the reset phase. They accumulate during the tick and are consumed by `player.secondUpdate()` when building `UpdatePacket`s.

**Example files:** `@file server/src/game.ts`

---

## Pattern: Self-Scheduling Tick Loop

**When to use:** Understanding how the 40 TPS rate is maintained without a fixed `setInterval`.

**Implementation:**

`Game` uses a self-rescheduling `setTimeout` rather than `setInterval`:

```typescript
// end of tick():
if (!this._stopped) {
    setTimeout(this.tick.bind(this), this.idealDt - (Date.now() - now));
}
```

`idealDt = 1000 / (Config.tps ?? GameConstants.tps)` = `25 ms` at the default 40 TPS.

The delay passed to `setTimeout` is `idealDt` minus the time already spent in this tick, keeping the average inter-tick interval close to 25 ms even when individual ticks take variable time. If a tick overshoots `idealDt` the next tick fires immediately (delay would be negative but `setTimeout` clamps to 0).

`_tickTimes` accumulates the raw computation time per tick (not the inter-tick gap). Every 200 samples, `Statistics.average()` and `Statistics.stddev()` are used to log `ms/tick ± stddev` and a `Load %` (`mspt / idealDt * 100`).

**Example files:** `@file server/src/game.ts`

---

## Pattern: Deferred Timeout Queue

**When to use:** Any game logic that needs to fire after a delay without blocking the tick (e.g. airdrop scheduling, gas stage transitions, immunity duration, end-of-game disconnect).

**Implementation:**

`Game.addTimeout(callback, delay)` creates a `Timeout` object (from `@common/utils/misc`) stamped with `this.now + delay` and inserts it into `this._timeouts: Set<Timeout>`. At the start of every tick (step 2) the set is iterated:

```typescript
for (const timeout of this._timeouts) {
    if (timeout.killed) { this._timeouts.delete(timeout); continue; }
    if (this.now > timeout.end) {
        timeout.callback();
        this._timeouts.delete(timeout);
    }
}
```

Timeouts can be cancelled at any time by calling `timeout.kill()` — setting `timeout.killed = true` so it is dropped on the next tick sweep. Resolution granularity is one tick (≤ 25 ms).

**Example files:** `@file server/src/game.ts`
