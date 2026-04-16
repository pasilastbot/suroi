# Q15: How does the spatial grid (32-unit cells) scale as player count increases, and has the cell size been tuned?

**Answer:** The 32-unit cell size is **hardcoded without apparent tuning** and scales linearly with player count. Performance implications are **bounded but non-trivial** at high player counts.

---

## Cell Size & Design

**Cell Size Implementation:**
- The grid uses a fixed cell size of **32 pixels** in `server/src/utils/grid.ts`
- This matches the constant `GameConstants.gridSize = 32` in `common/src/constants.ts`
- The `Grid` class hardcodes this value directly in the field declaration, indicating it's treated as a fixed architectural parameter

**Map Bounds:**
```
Maximum map size: 1924 × 1924 pixels  (GameConstants.maxPosition)
Grid dimensions:  ~60 × 60 cells       (1924 / 32 ≈ 60.125)
Typical map:      32 × 32 cells        (1024 × 1024 pixels)
```

---

## Cell Size Rationale

The 32-unit cell size aligns with **typical object sizes** in the game:

- **Player collision radius:** 2.25 units
- **Most weapon hitboxes:** 4–8 units
- **Building/obstacle hitboxes:** 8–128 units (variable)
- **Loot hitboxes:** ~2–4 units

A 32-unit cell is **12–16× player radius**, large enough to avoid granular cell fragmentation while small enough to provide meaningful spatial culling (e.g., players can't see objects >100 units away; a 32-unit cell groups ~9 map quadrants).

---

## Scaling with Player Count

### Linear Candidate Growth

As player count increases, broad-phase queries return more candidates:

**Broad-Phase Query Cost: O(k + n)**
- `k` = cells in the query rectangle (fixed per viewport/hitbox size)
- `n` = candidate objects returned

For each player, visibility culling calls `grid.intersectsHitbox(screenHitbox)`:

```
Screen hitbox ≈ 11 × 11 units (varies with zoom)
Grid cells touched ≈ (11 / 32)² ≈ 1–4 cells
Objects per cell ≈ 1 + (player_count / cell_coverage)
```

**Scaling pattern:**
- 50 players on a 32×32 map (1024 cells) → ~1.5 objects per cell on average
- 128 players → ~4 objects per cell
- 256 players → ~8 objects per cell

Each visibility query must iterate candidates in those cells; with 4 cells × 8 objects, that's ~32 comparisons per player per 8-tick cycle (200ms).

### Per-Tick Cost (Server-Side Game Loop)

Each tick:

1. **Per-category iteration** (no spatial overhead):
   ```
   for each player:     scan all N players      O(N)
   for each projectile: scan all M projectiles O(M)
   for each loot:       scan all K loot         O(K)
   ```
   
2. **Collision queries** (spatial grid involved):
   ```
   Player.secondUpdate() → grid.intersectsHitbox() per player
   Projectile collision detection → grid cell traversal O(cells crossed)
   Explosion radius checks → grid.intersectsHitbox()
   ```

At **40 TPS, 128 players**:
- 128 players × ~32 visibility query objects ≈ **~4,096 object comparisons per 8-tick cycle**
- Per tick: ~512 comparisons (minimal in wall-clock time on modern hardware)

### Visibility Caching Amortization

The visibility system caches visibility per player and **only recalculates every 8+ ticks** (~200ms at 40 TPS):

```
Visibility recalculated every 8+ ticks (~200ms)
Forced refresh only on: object add/remove, building state, player-specific flags
```

This **8× amortization** moves the cost from game tick to visibility tick:
- Full re-query cost is paid once every 8 ticks, spread across all players
- **Effective per-tick cost ∝ player count / 8**

---

## Performance Implications: No Evidence of Tuning

**What the documentation does NOT mention:**
- No discussion of A/B testing different cell sizes (16, 32, 64 units)
- No performance benchmarks vs. player count
- No notes on why 32 specifically was chosen (beyond it matching object scale)
- No configuration to tune cell size per map or game mode

**Known Bottlenecks at High Player Count:**

1. **Network bandwidth** — `UpdatePacket` serialization dominates at 100+ players:
   - Each visible object = ~50–200 bytes in packet
   - 128 players × ~100 visible objects × ~150 bytes ≈ 1.9 MB/s (unsustainable)
   - Mitigation: `partialDirtyObjects` reduces per-frame packet size

2. **Memory fragmentation** — Object pool allocation stalls:
   - Garbage collector can stall render loop when reusing/allocating hundreds of objects (common in large matches)
   - Affects both server and client; **not a grid issue**, but compounds at high player count

3. **Spatial grid itself is not the bottleneck** — at 128 players on a 32×32 map:
   - 1–4 objects per cell on average
   - Grid cell access is O(1) hash map lookup
   - Iteration cost is O(n) where n ≈ objects in visible cells (~4–32)
   - This is negligible vs. network serialization cost

---

## Verdict: Cell Size is Static; Grid Scales Gracefully

**Cell size (32 units):** Appears **intentional but not empirically tuned**. Matches game object scale and provides reasonable spatial granularity. No evidence it was benchmarked against alternatives (16, 48, 64).

**Scaling:** The grid itself is **O(N) optimal** — cost grows linearly with player count only in visibility queries, and that's amortized across 8 ticks. Real limits come from:
- Network bandwidth (UpdatePacket serialization)
- Memory fragmentation (object pool allocation on client)
- Server tick budget (40 TPS = 25 ms per tick for all gameplay logic)

For a 128-player server, the grid remains a minor contributor to frame time; the network and memory pressure are the dominant constraints.

---

## References

- **Tier 2:** `docs/subsystems/spatial-grid/README.md` — Core design, data structures, query algorithms
- **Tier 3:** `docs/subsystems/spatial-grid/modules/query-optimization.md` — Grid dimensions, cell organization, performance characteristics
- **Tier 2:** `docs/subsystems/spatial-grid/patterns.md` — Object registration, movement updates, query patterns
- **Tier 2:** `docs/subsystems/visibility-los/README.md` — Visibility culling pipeline with spatial grid integration
- **Tier 3:** `docs/subsystems/game-loop/modules/object-update.md` — Grid integration in tick loop
- **Source:** `server/src/utils/grid.ts` — Grid implementation, cell size hardcoding
- **Source:** `common/src/constants.ts` — `gridSize: 32` constant
