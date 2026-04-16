# Q18: How is the game loop scheduled — setInterval, setTimeout, or a Bun-specific timer mechanism?

**Answer:** The suroi game loop uses **`setTimeout`** (not `setInterval` or Bun timers) with a **self-rescheduling mechanism** that maintains a fixed 40 TPS (25 ms/tick) rate. The timer is purely a Node.js/Bun `setTimeout` call, with no Bun-specific timer APIs in use.

---

## How It Works

**Target rate:** `GameConstants.tps` = 40 TPS  
**Ideal tick duration:** `idealDt = 1000 / 40 = 25 ms`

The tick loop is a **self-rescheduling `setTimeout`** that calculates the next delay dynamically:

```typescript
// server/src/game.ts line 565
setTimeout(this.tick.bind(this), this.idealDt - (Date.now() - now));
```

**The delay formula:**
```
next_delay = idealDt - actual_tick_duration
           = 25ms - (time spent in this tick)
```

This **adaptive mechanism** ensures the game processes at a consistent rate despite variable tick computation times. If a tick takes 20ms, the next one schedules 5ms later. If a tick takes 23ms, the next schedules 2ms later. This keeps the average tick rate at 40 TPS.

---

## Game Tick Structure

Within each `tick()` call in `server/src/game.ts`:

1. **Capture delta time** — `this._dt = now - this._now; this._now = now`
2. **Execute timeouts** — custom callbacks queued via `addTimeout()`
3. **Update game objects** — loot, parachutes, projectiles, bullets, explosions, gas, players
4. **Serialize dirty state** — objects with dirty flags get serialized to `UpdatePacket`
5. **Send network updates** — each connected player receives only visible objects
6. **Reset per-tick collections** — clear dirty flags and queued events
7. **Win condition check** — if alive count ≤ 1, game ends
8. **Emit plugin events** — `"game_tick"` event for plugin hooks
9. **Reschedule next tick** — `setTimeout(...)` with dynamically calculated delay

---

## Why `setTimeout` (Not `setInterval`)

| Feature | `setInterval` | `setTimeout` (self-rescheduling) | Explanation |
|---------|--------------|----------------------------------|------------|
| **Jitter handling** | Fixed interval, ignores tick duration | Adapts next delay to actual tick time | If tick takes 23ms, `setInterval` still fires at exactly 25ms boundary (skipping a tick); `setTimeout` fires 2ms later |
| **Frame skipping** | Can skip frames if tick > interval | No frame skipping possible | Self-rescheduling ensures every tick runs |
| **Long tick recovery** | Backs up (multiple ticks fire); requires catch-up logic | Naturally spreads load | Next tick is delayed, but catch-up occurs gradually |

`setTimeout` with self-rescheduling is the **standard game loop pattern** for maintaining frame-rate stability across variable tick times.

---

## References

**Documentation:**
- **Tier 2:** `docs/subsystems/game-loop/README.md` — "Game Tick Structure" section shows all steps
- **Tier 3:** `docs/subsystems/game-loop/modules/tick.md` — detailed per-step documentation

**Source Code:**
- `server/src/game.ts:265` — `idealDt` calculation in constructor
- `server/src/game.ts:336–565` — Full `tick()` method
- `server/src/game.ts:565` — **The actual `setTimeout` call**
- `server/src/server.ts:1–35` — Primary process entry point, worker forking via `Cluster`

**Related subsystem:** `docs/subsystems/networking/` — handles `UpdatePacket` serialization and transmission
