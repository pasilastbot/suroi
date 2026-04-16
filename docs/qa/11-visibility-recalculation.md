# Q11: Why is visibility recalculated only every ~8 ticks instead of every tick, and what are the trade-offs?

**Direct answer:** Suroi recalculates visibility every ~8 ticks (~200ms at 40 TPS) because the **screen viewport is stable and movement is smooth and predictable**. Recalculating per-tick would waste CPU on redundant spatial grid queries.

---

## The Optimization Mechanism

### How It Works

In `Player.secondUpdate()`, the visibility culling logic runs conditionally:

```typescript
this.ticksSinceLastUpdate++;
if (this.ticksSinceLastUpdate > 8 || game.updateObjects || this.updateObjects) {
    this.ticksSinceLastUpdate = 0;
    
    // Recalculate screen hitbox and query grid
    const dim = zoom * 2 + 8;  // screen dimension in game units
    this.screenHitbox = RectangleHitbox.fromRect(dim, dim, player.position);
    
    const newVisibleObjects = game.grid.intersectsHitbox(this.screenHitbox);
    // ... detect visibility changes, send deletions, mark new objects as full
}
```

**Key points:**
- Counter `ticksSinceLastUpdate` increments every tick, resets to 0 when visibility recalculates
- Recalculates every **8+ ticks** (25ms × 8 = ~200ms latency)
- **Force refresh** on demand: if `game.updateObjects` (set when objects added/removed, buildings open) or `player.updateObjects` is true
- Time at 40 TPS: `25ms × 8 = 200ms` per full visibility refresh

### Forced Early Refreshes

The system doesn't strictly enforce the 8-tick interval. The flag `game.updateObjects` is set when:
- Objects are added to the grid (new player, loot spawned, building opened)
- Objects are removed (player dies, loot picked up, building closed)

Individual players can force an immediate refresh via `player.updateObjects = true`.

---

## Performance Trade-Offs

### Why 8 Ticks?

**Screen viewport is small and stable:**

```typescript
const dim = zoom * 2 + 8;
// Example: zoom = 1.5 (scoped) → dim = 11 units
// Unscoped: zoom = 0.5 → dim = 9 units
```

At a nominal 800×600 viewport on a 1924×1924 map, each player's screen bounds cover ~0.3–0.5% of the world. Objects entering/leaving this small viewport are rare — most ticks show the same visible object set.

### Latency Trade-Off: 200ms Visibility Latency

**Cost of the optimization:**
- Objects entering a player's viewport have **up to 8 ticks (200ms) before they're first synced**
- At 40 TPS, this is noticeable but acceptable — enemies appearing from off-screen take ~0.2s to become visible
- **Mitigated by:** movement is smooth (players move predictably), and close-range combat (< 10 units) usually happens within the viewport already

| Metric | Value |
|--------|-------|
| Ticks between recalculation | 8 |
| Time between checks (at 40 TPS) | 200ms |
| Time complexity per player | O(n + m + p) where n = neighbors, m = bullets, p = explosions |
| Spatial grid cell size | 32 units |

### CPU Savings

Per-tick cost of visibility (if it ran every tick instead of every 8):
- Grid query: O(k + n) where k = grid cells in viewport, n = candidate objects
- Typical viewport covers ~1–4 grid cells (32 units each)
- Typical candidates: 10–50 objects per cell
- **Amortization factor: 8×** — reducing cost from O(100) per player per tick to O(12.5)

At 64 players, **avoiding per-tick visibility saves ~44,800 grid cell lookups per second**.

### Stability of Player Position

The screen hitbox moves gradually with the player:
```
Tick N:   Player at (500, 500) → screen bounds query
Tick N+1: Player at (501, 502) → (no recalc, assume same objects)
Tick N+2: Player at (502, 504) → (still no recalc)
...
Tick N+8: Player at (508, 516) → (recalc, detect new objects)
```

**Why this works:** 8 ticks = 200ms. At average player speed (~150 units/s), a player moves **~30 units** in 200ms — often within the same grid cell neighborhood.

---

## Consequences & Trade-Offs

### ✅ Benefits

| Benefit | Impact |
|---------|--------|
| **Reduced CPU** | 8× amortization on grid queries per player |
| **Lower bandwidth** | Fewer visibility recalculations mean fewer `UpdatePacket` deltas |
| **Spectator efficiency** | Spectators using virtual camera from spectated player position don't add extra grid queries (same 8-tick rate) |
| **Simple implementation** | Single counter per player; no complex heuristics |

### ⚠️ Consequences

| Issue | Severity | Mitigation |
|-------|----------|-----------|
| **Visibility discovery latency** | Medium | Objects entering viewport take up to 200ms to become visible; counter-intuitive for fast-moving enemies off-screen |
| **Rapid viewport changes** | Low–Medium | If player location teleported or fast-moving (gas closing), new objects may not sync immediately until next 8-tick mark |
| **Zoom changes** | Medium | Changing scope zoom updates `_zoomOverride` but screen hitbox isn't recalculated until next 8-tick interval; temporary blind spot possible |
| **Forced refresh contention** | Low | High object spawn/destruction rates (e.g., many grenades) trigger `game.updateObjects = true` frequently, causing per-player `secondUpdate()` to recalculate early |
| **No occlusion culling** | High | Walls and buildings don't block visibility; 200m away behind a building, you're still synced if in the rectangular screen bounds (LOS is purely distance-based) |

---

## References

- **Tier 2:** `docs/subsystems/visibility-los/README.md` — full visibility pipeline and culling rules
- **Tier 2:** `docs/subsystems/spatial-grid/README.md` — grid query complexity
- **Tier 3:** `docs/subsystems/game-loop/modules/tick.md` — exact step-by-step tick execution
- **Tier 3:** `docs/subsystems/game-loop/patterns.md` — object lifecycle and grid updates
- **Source:** `server/src/objects/player.ts` (Player.secondUpdate) — actual implementation
