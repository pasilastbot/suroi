# Q16: What profiling tools or metrics exist for measuring server tick performance in production?

**Answer:**

Suroi has **built-in tick performance metrics** at the server level and **system-level monitoring** for RAM/CPU. There is no external profiling infrastructure (APM, Datadog, etc.), but tick timing is closely tracked for production diagnostics.

---

## Server Tick Performance Metrics

### 1. **Per-Tick Latency Tracking** (Game Loop)

`server/src/game.ts` maintains a rolling buffer of tick execution times:

**Implementation:**
```typescript
private readonly _tickTimes: number[] = [];  // @line 227

// Each tick (from line 552):
const tickTime = Date.now() - now;
this._tickTimes.push(tickTime);

// Every 200 ticks, report stats:
if (this._tickTimes.length >= 200) {
    const mspt = Statistics.average(this._tickTimes);
    const stddev = Statistics.stddev(this._tickTimes);
    this.log(`ms/tick: ${mspt.toFixed(2)} ± ${stddev.toFixed(2)} | Load: ${((mspt / this.idealDt) * 100).toFixed(1)}%`);
    this._tickTimes.length = 0;
}
```

**Metrics reported:**
- **Mean tick time** (average milliseconds per tick across 200-tick window)
- **Standard deviation** (timing stability/jitter)
- **Load percentage** — tick time relative to ideal time budget
  - **Formula:** `(mspt / idealDt) * 100` where `idealDt = 1000 / 40 = 25ms` (40 TPS target)
  - **Example:** If avg tick is 18.5ms, load = `18.5 / 25 * 100 = 74%`
- **No blocking** — measurements don't slow the game loop (post-hoc timing)

---

### 2. **System-Level Metrics** (Process Monitor)

`server/src/server.ts` logs process metrics every **60 seconds**:

```typescript
setInterval(() => {
    const memoryUsage = process.memoryUsage().rss;
    let perfString = `RAM usage: ${Math.round(memoryUsage / 1024 / 1024 * 100) / 100} MB`;
    
    if (os.platform() !== "win32") {
        const load = os.loadavg().join("%, ");
        perfString += ` | CPU usage (1m, 5m, 15m): ${load}%`;
    }
    serverLog(perfString);
}, 60000);
```

**Metrics reported:**
- **RSS (Resident Set Size)** — actual memory used by primary process (in MB)
- **CPU load averages** — 1-minute, 5-minute, 15-minute averages (Unix/Linux/macOS only, skipped on Windows)

**Note:** This tracks the **primary process only** (GameManager), not individual game worker processes.

---

### 3. **Game State Metrics** (API + Logging)

**Available via HTTP endpoints:**

| Metric | Source | Scope |
|--------|--------|-------|
| `playerCount` | `GameManager.playerCount` | Aggregate across all active games |
| `aliveCount` | Per-game `GameData` | Players still alive in each game |
| `teamMode` | `GameManager.teamMode` | Current team mode (solo/duo/squad) |
| `map` | `GameManager.map` | Current map |

**Available at `/api/serverInfo`** — queryable from client/monitoring tools.

**Also logged to console** when players join/leave and on game startup.

---

## Statistics Utilities

`common/src/utils/math.ts` provides the math for profiling:

```typescript
export const Statistics = Object.freeze({
    average(values: readonly number[]): number { ... },
    stddev(values: readonly number[]): number {
        const avg = Statistics.average(values);
        return Math.sqrt(Statistics.average(values.map(v => (v - avg) ** 2)));
    }
});
```

This enables statistical analysis of tick times (mean, variance, outlier detection).

---

## Client-Side Monitoring

The [Console & Debug System](docs/subsystems/console-debug/README.md) provides **real-time graphs** on the client:

- **FPS graph** — frame rate visualization
- **Network latency graph** — ping/RTT over time
- **Update packet size graph** — bandwidth usage per tick

Accessible via backtick (`) to open the in-game console.

---

## What's NOT Available

- **Per-object/system profiling** — no breakdown of which game systems consume CPU (collision, visibility, damage processing, etc.)
- **Per-player tracking** — no per-client latency, packet loss, CPU impact metrics
- **External monitoring** — no Datadog, Prometheus, or APM integration
- **Historical time series** — metrics reset every 200 ticks; no persistent database
- **Worker process metrics** — system monitor only tracks primary process (game workers are separate OS processes)

---

## Summary Table

| Metric | Collection Point | Frequency | Scope | Output |
|--------|------------------|-----------|-------|--------|
| **Tick time (ms)** | `game.ts:552` | Every tick, logged every 200 ticks | Per-game | Console log |
| **Load %** | `game.ts:558` (calculated) | Every 200 ticks | Per-game | Console log |
| **Tick jitter (stddev)** | `game.ts:557` | Every 200 ticks | Per-game | Console log |
| **RAM usage** | `server.ts:70` | Every 60 sec | Primary process | Console log |
| **CPU load avg** | `server.ts:75` | Every 60 sec | System-wide | Console log |
| **Player count** | `gameManager.ts:91` | On join/leave + `/api/serverInfo` | Global | API endpoint |
| **Alive count** | Per-game `GameData` | Every tick (in game state snapshots) | Per-game | API endpoint, player packets |

---

## References

- **Tier 2:** `docs/subsystems/game-loop/README.md` — tick structure with profiling section
- **Tier 2:** `docs/subsystems/server-utilities/README.md` — Config and GameManager responsibilities
- **Tier 2:** `docs/subsystems/console-debug/README.md` — client-side monitoring graphs
- **Source:** `server/src/game.ts` — `_tickTimes` implementation
- **Source:** `server/src/server.ts` — system metrics loop
- **Source:** `common/src/utils/math.ts` — Statistics utilities
