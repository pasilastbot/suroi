# Q: How do I profile and diagnose server performance problems?

<!-- @tags: server, performance, game-loop, Bun -->
<!-- @related: docs/subsystems/game-loop/README.md, docs/development.md -->

## Symptoms of Performance Problems

- Tick duration > 25 ms (game runs slower than 40 TPS)
- CPU usage spikes
- Players experience lag / delayed inputs
- `bun stressTest` shows degraded performance under load

## 1. Stress Test (Baseline)

```bash
cd tests && bun stressTest
```

Runs `tests/src/stressTest.ts` which connects bots to the server and measures
performance under simulated load. Use this as a baseline before and after changes.

## 2. Bun's Built-In Profiler

Bun supports the V8 CPU profiler via `--cpu-prof`:

```bash
# Run the server with CPU profiling
bun --cpu-prof dev:server
```

Or instrument a specific run:

```bash
bun --smol --cpu-prof --cpu-prof-dir /tmp/profiles server/src/server.ts
```

This generates a `.cpuprofile` file. Open it in Chrome DevTools:
1. Open `chrome://inspect`
2. Click **Open dedicated DevTools for Node**
3. Go to **Profiler** tab → Load `.cpuprofile`

Look for functions with high **self time** (time spent in the function itself,
not callees). These are the hot spots.

## 3. Manual Tick Timing

Add timing around the tick phases in `server/src/game.ts` to identify which
phase is slow:

```typescript
// Temporary timing instrumentation in tick():
const t0 = performance.now();

// ...phase you want to measure...

const elapsed = performance.now() - t0;
if (elapsed > 5) console.log(`Phase X took ${elapsed.toFixed(2)} ms`);
```

The full tick budget is 25 ms. Remove instrumentation before committing.

## 4. Common Performance Bottlenecks

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Bullet update slow | Too many bullets alive | Reduce bullet `range`, add `noBulletCollision` to foliage |
| Player update slow | N² player collision | Use grid spatial queries (already done) |
| Serialization slow | Objects always full-dirty | Fix `setDirty()` vs `setFullDirty()` usage |
| Memory growing | Objects not removed from grid | Ensure `dead` objects are removed in `postTick()` |
| Grid query slow | Cell size too small | `GameConstants.gridSize` (default 32 units) |

## 5. Grid Query Analysis

If spatial queries are slow, check:

- Are you using `grid.intersectsHitbox()` with a minimal bounding box?
- Are dead objects being removed from the grid?
- Is `gridSize` appropriate for the density of objects?

The grid is a spatial hash in `server/src/utils/grid.ts`.

## 6. Identifying Memory Leaks

```bash
# Check Bun memory usage
bun --inspect dev:server
```

Then connect Chrome DevTools for heap snapshots. Common causes:
- Event listeners not removed (`this.off()` in plugin cleanup)
- Objects staying in `game.objects` after `dead = true`
- Uncleaned timeouts (`game.addTimeout` accumulation)

## 7. Tick Rate Monitoring

Add a tick time logger to `game.ts`:

```typescript
private tickTimes: number[] = [];

tick(): void {
    const start = performance.now();
    // ...normal tick...
    const elapsed = performance.now() - start;
    this.tickTimes.push(elapsed);
    if (this.tickTimes.length >= 100) {
        const avg = this.tickTimes.reduce((a, b) => a + b) / 100;
        if (avg > 20) console.warn(`Slow tick average: ${avg.toFixed(1)} ms`);
        this.tickTimes = [];
    }
}
```

## Related

- [Game Loop](../subsystems/game-loop/README.md) — full tick order with 18 phases
- [Tick & Serialization](../subsystems/game-loop/modules/tick-serialization.md) — dirty tracking performance
- [Development Guide](../development.md) § Testing — `bun stressTest`
