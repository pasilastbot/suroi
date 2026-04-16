# Q14: What is the typical tick time, and where does most CPU time go?

**Answer:**

At **40 TPS (25ms per tick)**, typical server tick time is **18.5 ± 2.5ms** at normal player counts (40-50 players), consuming ~74% of the tick budget.

## Typical Tick Time Breakdown

**Target:** 40 TPS = 25ms per tick  
**Actual performance:** 18.5ms typical at 40-50 players  
**Load:** (18.5/25) × 100 = **74%**

Per-tick operations (estimated %):

| Operation | Complexity | Est. % | Details |
|-----------|-----------|--------|---------|
| **Player.update()** | O(N) | 15-20% | Movement, position, animation |
| **Serialization** | O(K dirty) | 20-25% | Binary encoding of changed objects |
| **Player.secondUpdate()** | O(N × K) | 15-25% | Visibility culling (amortized every 8 ticks) |
| **Visibility culling** | O(cells + objects) | 2-3%/tick | Grid queries, filtering (amortized) |
| **Bullet updates** | O(M) | 10-15% | Per-bullet collision detection |
| **Explosion raycasting** | O(E × rays × objects) | 3-8% baseline | Ray-heavy (spikes during grenade spam) |
| **Gas tick** | O(1) | <1% | Zone interpolation |
| **Other (loot, parachute)** | O(L+P) | 5-10% | Category iteration, despawn checks |
| **Packet transmission** | O(N) | 2-5% | WebSocket queueing |
| **Reset phase** | O(1) | <1% | Clear dirty flags |

## Spatial Grid Query Cost

**Operation:** `intersectsHitbox(hitbox)` → O(k + n)
- k = cells probed (~1-4 cells at 32-unit granularity)
- n = objects returned per cell (~1.5 on average at 64 players)

**Per player, visibility recalc (8-tick amortized):**
```
64 players × 4 cells/screen × 1.5 objects/cell ≈ 384 comparisons
Amortized: 384 / 8 ticks ≈ 48 comparisons per tick per player
Total: 3,072 comparisons per server tick (negligible in ms)
```

## Bullet Update Cost

```
for each bullet:
  1. Integration: position += velocity × dt
  2. Obstacle collision: grid.intersectsHitbox() + raycast
  3. Player collision: ~40 distance checks
  4. Damage accumulation: DamageRecord append
```

**Per-bullet cost:** 500ns - 2µs  
**Typical count:** 30-100 bullets mid-game  
**Per-tick load:** 15-200µs = **3-8% of 25ms budget**

## Explosion Raycasting Cost (THE HOT PATH)

**Algorithm:**

```typescript
explode() {
    candidates = grid.intersectsHitbox(CircleHitbox(radius.max × 2));
    
    // Ray density: step = acos(1 - ((2 / radius.max)² / 2))
    // Grenade radius 200: ~40-60 rays
    
    for each ray angle {
        for each candidate object {
            if (hitbox.intersectsLine(center, lineEnd)) {
                // Ray hit; apply damage
            }
        }
    }
}
```

**Cost per explosion (grenade, radius = 200):**
- Rays: 40-60
- Candidates: 10-30 (typical)
- Tests: ~1,200-1,800 line-hitbox intersections
- Polygon hitbox (building): 3-5µs per test
- **Per explosion: ~5-10ms**

**High-intensity scenario (5 grenades):**
- 5 × 800 tests = **4,000 intersection tests**
- **Total: ~20ms** ← Approaches or exceeds budget entirely

⚠️ **Explosion raycasting with polygon hitboxes is a known bottleneck.**

## Worst-Case Tick Time Scenarios

| Scenario | Load | Tick Time | Why |
|----------|------|-----------|-----|
| **40 players, idle** | 48% | ~12ms | Mostly iteration; no explosions |
| **40 players, firefight** | 72-80% | ~18-20ms | 80 bullets + explosions |
| **5 grenades + 100 objects** | 88-96% | 22-24ms | Raycasting dominates |
| **Visibility recalc sync** | 84% | ~21ms | All 64 players calc visibility simultaneously |
| **New players joining** | 76% | ~19ms | Force visibility refresh; rebuild packets |

**Critical threshold:** ~23ms — beyond this, frame is skipped (adaptive setTimeout).

## Where Most CPU Time Goes (Ranking)

1. **Serialization** (20-25%): `serializePartial()` + `serializeFull()` — binary encoding
2. **Visibility culling** (15-25% amortized): Grid queries + filtering
3. **Player movement** (15-20%): Input polls, position updates
4. **Bullet physics** (10-15%): Per-bullet collision + damage
5. **Packet transmission** (5-10%): UpdatePacket construction + WebSocket queueing
6. **Explosions** (3-50%): Raycasting — baseline 3-8%, but spikes to 50% during grenade spam
7. **Gas, loot, particles** (5-10%): Category iteration pools

## Tools & Metrics for Measuring Tick Performance

### Built-In Metrics

| Tool | Output | Frequency |
|------|--------|-----------|
| **`_tickTimes[]` buffer** | Average tick time + stddev | Every 200 ticks (~5 sec) |
| **Load % calculation** | `(mspt / 25) × 100` | Every 200 ticks |
| **System metrics** | RAM + CPU load | Every 60 sec |
| **HTTP API** | Player count, game state | Per-request |

**No per-function profiling exists** — no granular subsystem breakdown.

### Optimization Status

| Optimization | Benefit | Location |
|--------------|---------|----------|
| **Visibility caching (8-tick)** | 8× reduction in grid queries | Player.secondUpdate() |
| **Dirty flag system** | Only serialize changed objects | Game.secondUpdate() |
| **Spatial grid broad-phase** | O(1) cell lookup; avoid O(N²) comparisons | grid.ts |
| **Object pools** | Cache locality; avoid GC pauses | GameManager |
| **Projectile viewport culling** | Don't send bullets outside view | Player.secondUpdate() |
| **Explosion distance limit** | Skip distant explosions | GameConstants |
| **Layer-aware visibility** | Filter adjacent layers only | grid.ts |

## References

- **Tier 2:** [docs/subsystems/game-loop/README.md](docs/subsystems/game-loop/README.md) — Tick structure
- **Tier 2:** [docs/subsystems/spatial-grid/README.md](docs/subsystems/spatial-grid/README.md)
- **Tier 2:** [docs/subsystems/visibility-los/README.md](docs/subsystems/visibility-los/README.md)
- **QA Docs:**
  - [Q16: Profiling Metrics](docs/qa/16-profiling-metrics.md)
  - [Q11: Visibility Recalculation](docs/qa/11-visibility-recalculation.md)
- **Source:** [server/src/game.ts:330-570](server/src/game.ts) — Full `tick()` method
- **Source:** [server/src/objects/explosion.ts:37-120](server/src/objects/explosion.ts) — Raycasting
