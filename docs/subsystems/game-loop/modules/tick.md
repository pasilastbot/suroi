# Tick Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/game-loop/README.md -->
<!-- @source: server/src/game.ts -->

## Purpose

Manages the per-tick game state update loop that runs at `GameConstants.tps` Hz — advancing every simulation object, serializing dirty state, and dispatching `UpdatePacket`s to all connected clients exactly once per tick.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/game.ts` | `Game.tick()` — the full 21-step tick loop | High |
| `server/src/objects/player.ts` | `Player.update()`, `secondUpdate()`, `postPacket()` — three per-player phases called from `tick()` | High |
| `server/src/gas.ts` | `Gas.tick()` — advances gas position, radius, and damage flag once per game tick | Medium |
| `common/src/constants.ts` | `GameConstants.tps`, `GameConstants.maxPosition` — core timing and world-bounds constants | Low |
| `common/src/utils/misc.ts` | `Timeout` class — value object used by the in-game timeout queue | Low |

## Business Rules

- **Tick rate:** `GameConstants.tps = 40` Hz → `idealDt = 1000 / (Config.tps ?? GameConstants.tps) = 25 ms` by default. `Config.tps` overrides this if present.
- **Delta time (`dt`):** At the start of each tick, `this._dt = Date.now() - this._now` captures real elapsed time. All physics and time-based calculations use `dt` (milliseconds), not `idealDt`.
- **Drift-compensating scheduler:** Next tick is scheduled with `setTimeout(this.tick.bind(this), this.idealDt - elapsed)`. A 5 ms tick sleeps 20 ms; a 30 ms tick on a 25 ms budget passes a negative delay (clamped to 0 by the runtime).
- **Dirty object tracking:** Objects that changed *position or rotation only* go in `game.partialDirtyObjects`; objects that changed *any serialized field* go in `game.fullDirtyObjects`. Full-dirty shadows partial: any object in both sets is serialized full and the partial serialize is skipped.
- **Bullet damage batching:** All bullet trajectories are swept first (`records = records.concat(bullet.update())`), then all damage records are applied at once. A shotgun volley destroying a crate on the first pellet cannot cause later pellets to phase through on the server, matching the client's visual.
- **First player loop uses `livingPlayers`; second and third use `connectedPlayers`.** A dead-but-connected spectator never runs `player.update()` but does receive `UpdatePacket`s.
- **Gas damage is gated:** `Gas.tick()` sets `_doDamage = true` at most once per 1 000 ms. Player damage is applied in `player.update()` only when `gas.doDamage` is `true`.
- **Win condition requires a 5 000 ms grace period** (`this.now - this.startedTime > 5000`) to prevent false wins during the join window.
- **Win condition disabled when `Config.minTeamsToStart <= 1`** — the check is gated by `(Config.minTeamsToStart ?? 2) > 1`.
- **Reset phase runs before win condition.** All per-tick accumulators are cleared before the win check, so no dirty state leaks into the win-exit path.
- **Collision resolution runs up to 10 iterations** per player per tick; loop exits early if no collision is detected in an iteration.
- **Visible-object set refreshed every 8 ticks or on demand.** `ticksSinceLastUpdate` increments each tick and resets on recalculation; `game.updateObjects` or `player.updateObjects` force an immediate refresh.

## Tick Execution Order

The following 21 steps execute in exact order inside `Game.tick()` (`server/src/game.ts:310`):

### Step 1: Delta time

```typescript
const now = Date.now();
this._dt = now - this._now;
this._now = now;
```

Captures the wall-clock timestamp at the start of the tick and computes `_dt` (milliseconds elapsed since the previous tick). All physics calculations use `this.game.dt` via this value.

### Step 2: Timeout queue

```typescript
for (const timeout of this._timeouts) {
    if (timeout.killed) { this._timeouts.delete(timeout); continue; }
    if (this.now > timeout.end) {
        timeout.callback();
        this._timeouts.delete(timeout);
    }
}
```

Iterates the `_timeouts: Set<Timeout>` — game-internal one-shot callbacks scheduled with `game.addTimeout(callback, delayMs)`. Callbacks whose `end` timestamp has passed fire in iteration order and are removed. Killed (cancelled) timeouts are also pruned here. These are used throughout the codebase for delayed actions: door auto-close, gas stage advancement, win-delay shutdown, perk timers, etc.

### Step 3: Periodic airdrop

```typescript
if (
    this.mode.summonAirdropsInterval !== undefined
    && (this.now - this.lastAirdropTime) >= this.mode.summonAirdropsInterval
) {
    this.summonAirdrop(...);
    this.lastAirdropTime = this.now;
}
```

If the current mode definition includes `summonAirdropsInterval`, and enough wall time has elapsed since the last airdrop, a new airdrop is placed at a random safe position outside the gas. Only active in modes that define this interval (e.g. not standard "normal" mode).

### Step 4: Loot update

```typescript
for (const loot of this.grid.pool.getCategory(ObjectCategory.Loot)) {
    loot.update();
}
```

Ticks all `Loot` objects in the spatial grid. Loot applies drag and slides to rest on the floor. Ice tiles use a lower `iceDrag` constant (`GameConstants.loot.iceDrag = 0.0008` vs normal `drag = 0.003`).

### Step 5: Parachute update

```typescript
for (const parachute of this.grid.pool.getCategory(ObjectCategory.Parachute)) {
    parachute.update();
}
```

Ticks `Parachute` objects (airdrop descenders). Advances their fall animation/physics.

### Step 6: Projectile update

```typescript
for (const projectile of this.grid.pool.getCategory(ObjectCategory.Projectile)) {
    projectile.update();
}
```

Ticks every `Projectile` (grenades, C4, throwables). See [Projectile Update](#projectile-update) below.

### Step 7: SyncedParticle update

```typescript
for (const syncedParticle of this.grid.pool.getCategory(ObjectCategory.SyncedParticle)) {
    syncedParticle.update();
}
```

Advances all `SyncedParticle` objects (e.g. smoke clouds from grenades). These are server-authoritative particles with collision and depletion effects (vision blocking, damage-over-time).

### Step 8: Bullet trajectories and dead-bullet cleanup

```typescript
let records: DamageRecord[] = [];
for (const bullet of this.bullets) {
    records = records.concat(bullet.update());
    if (bullet.dead) {
        if (!bullet.reflected) {
            const { onHitExplosion } = bullet.definition;
            if (onHitExplosion) { this.addExplosion(onHitExplosion, bullet.position, ...); }
        }
        this.bullets.delete(bullet);
    }
}
```

Each `Bullet.update()` sweeps the bullet's trajectory for this tick (a `RectangleHitbox.fromLine`), resolves collisions, and returns an array of `DamageRecord`s (one per hit object). Dead bullets are removed from `this.bullets`; if a non-reflected bullet's definition has `onHitExplosion`, the explosion is registered for processing in step 10.

### Step 9: Apply bullet damage records

```typescript
for (const { object, damage, source, weapon, position } of records) {
    object.damage({ amount: damage, source, weaponUsed: weapon, position });
    // + onHitProjectile, enemySpeedMultiplier, removePerk, HollowPoints tracking
}
```

Applies all collected `DamageRecord`s in one pass **after** all bullet trajectories are resolved. This is the intentional design: a shotgun burst that would destroy an obstacle on the first pellet cannot cause later pellets to phase through — the server behaves the same as the client rendering. Additional per-record effects: spawning `onHitProjectile` projectiles, applying `enemySpeedMultiplier` slowdowns, triggering `removePerk`, and updating `HollowPoints` map indicators.

### Step 10: Process explosions

```typescript
for (const explosion of this.explosions) {
    explosion.explode();
}
```

Calls `explode()` on every `Explosion` queued during this tick (from step 8 or from thrown grenades that detonated). Explosions apply area damage, create synced particles, and add decals.

### Step 11: Gas tick

```typescript
this.gas.tick();
```

See [Gas Tick](#gas-tick) below.

### Step 12: Player first pass — `player.update()` (livingPlayers)

```typescript
for (const player of this.livingPlayers) {
    player.update();
}
```

Iterates only **living** players. See [Player Update Phases → Phase 1](#phase-1-playerupdate-first-pass-all-living-players).

### Step 13: Dirty object serialization

```typescript
for (const partialObject of this.partialDirtyObjects) {
    if (this.fullDirtyObjects.has(partialObject)) continue; // full shadows partial
    partialObject.serializePartial();
}
for (const fullObject of this.fullDirtyObjects) {
    fullObject.serializeFull();
}
```

Serializes network state for all objects that changed during this tick. See [Dirty Object System](#dirty-object-system).

### Step 14: Player second pass — `player.secondUpdate()` (connectedPlayers)

```typescript
for (const player of this.connectedPlayers) {
    player.secondUpdate();
}
```

Iterates **all connected** players (including spectators and dead players). See [Player Update Phases → Phase 2](#phase-2-playersecondupdate-second-pass-all-connected-players).

### Step 15: Player third pass — `player.postPacket()` (connectedPlayers)

```typescript
for (const player of this.connectedPlayers) {
    player.postPacket();
}
```

Post-send cleanup. See [Player Update Phases → Phase 3](#phase-3-playerpostpacket-third-pass-all-connected-players).

### Step 16: Map indicator cleanup

```typescript
for (const indicator of this.mapIndicators) {
    if (indicator.dead) {
        this.mapIndicatorIDAllocator.give(indicator.id);
        removeFrom(this.mapIndicators, indicator);
        continue;
    }
    indicator.positionDirty = false;
    indicator.definitionDirty = false;
}
```

Releases IDs for dead `MapIndicator`s back to the `IDAllocator` and clears their dirty flags. Indicators are used for `HollowPoints`/`ThermalGoggles` player highlights and airdrop markers.

### Step 17: Reset per-tick collections

```typescript
this.fullDirtyObjects.clear();
this.partialDirtyObjects.clear();
this.newBullets.length = 0;
this.explosions.length = 0;
this.emotes.length = 0;
this.newPlayers.length = 0;
this.deletedPlayers.length = 0;
this.packets.length = 0;
this.planes.length = 0;
this.mapPings.length = 0;
this.killLeaderDirty = false;
this.aliveCountDirty = false;
this.gas.dirty = false;
this.gas.completionRatioDirty = false;
this.updateObjects = false;
```

Clears all per-tick accumulators so the next tick starts clean. The dirty object sets are cleared here (after serialization in step 13 and packet send in step 14).

### Step 18: Win condition check

```typescript
if (
    this._started
    && !this.over
    && (Config.minTeamsToStart ?? 2) > 1
    && (
        this.isTeamMode
            ? this.aliveCount <= (this.teamMode as number)
              && new Set([...this.livingPlayers].map(p => p.teamID)).size <= 1
            : this.aliveCount <= 1
    )
    && this.now - this.startedTime > 5000
) { ... }
```

**Threshold:**
- **Solo mode:** `aliveCount <= 1`
- **Team mode:** `aliveCount <= teamMode` (the numeric enum value, e.g. 2 for Duo) **and** all survivors share the same `teamID`

**On win:**
1. Each surviving player's movement inputs are cleared, attacking is stopped
2. Each survivor triggers their win emote slot (`loadout.emotes[6]`)
3. Each survivor receives a `GameOverPacket(won = true)`
4. Plugin events `player_did_win` and `game_end` are emitted
5. `allowJoin = false`, `over = true`
6. `addTimeout(() => disconnect all → _stopped = true, 1000)` — game ends 1 second later

The guard `(Config.minTeamsToStart ?? 2) > 1` means a server configured for single-player (`minTeamsToStart = 1`) will never trigger the win condition here.

### Step 19: Tick time logging

```typescript
const tickTime = Date.now() - now;
this._tickTimes.push(tickTime);
if (this._tickTimes.length >= 200) {
    const mspt = Statistics.average(this._tickTimes);
    const stddev = Statistics.stddev(this._tickTimes);
    this.log(`ms/tick: ${mspt.toFixed(2)} ± ${stddev.toFixed(2)} | Load: ${((mspt / this.idealDt) * 100).toFixed(1)}%`);
    this._tickTimes.length = 0;
}
```

Measures the wall-clock time consumed by the current tick (not the time between ticks). After accumulating 200 samples, logs the rolling mean ± standard deviation and the percentage of the 25 ms budget consumed (`Load`). The buffer is cleared after logging.

### Step 20: Plugin tick event

```typescript
this.pluginManager.emit("game_tick", this);
```

Fires the `game_tick` plugin event. Plugins can use this for custom per-tick logic (e.g. game mode scripting, analytics). Runs after all standard game state has been updated and packets sent.

### Step 21: Reschedule

```typescript
if (!this._stopped) {
    setTimeout(this.tick.bind(this), this.idealDt - (Date.now() - now));
}
```

Rescheduled via `setTimeout` with the remaining budget: `idealDt - elapsedMs`. If the tick overran the 25 ms budget, the subtraction produces a negative number, which `setTimeout` treats as `0` — the next tick fires as soon as possible after the event loop yields. There is no backfill or catch-up mechanism; overruns cause one-tick drift but do not cascade.

The first call is made directly from the `Game` constructor:

```typescript
// Start the tick loop
this.tick();
```

---

## Player Update Phases

The tick calls three separate methods across two player sets, in distinct passes.

### Phase 1: `player.update()` (first pass, all living players)

Called for every player in `livingPlayers`. Performs (in order):

1. **Building & smoke detection** — queries `nearObjects` for buildings with `scopeHitbox` collision; collects overlapping `SyncedParticle`s for scope and damage effects.
2. **Recoil multiplier** — checks recoil active flag and expiry timestamp.
3. **Speed calculation** — multiplicative stack of: `baseSpeed`, floor `speedMultiplier`, recoil, perk modifiers (`AdvancedAthletics`, `Claustrophobic`), active action speed modifier, adrenaline (logarithmic model peaking at +15% at 100 adren), downed/revive multiplier, `effectSpeedMultiplier`, and `_modifiers.baseSpeed`.
4. **Movement vector** — WASD booleans or mobile `moving`/`angle`; diagonal inputs are normalised by `Math.SQRT1_2`. `AchingKnees` perk inverts the vector.
5. **Ice physics** — if the floor is `slippery`, acceleration-based blending and friction reduce the movement vector; otherwise the vector is set directly.
6. **Position update** — adds `movementVector * dt` to position.
7. **Collision resolution** — up to 10 iteration steps resolving player hitbox against obstacles/buildings. Stair interactions update `this.layer`.
8. **Position clamp** — clamped to `[radius, mapWidth-radius]` × `[radius, mapHeight-radius]`.
9. **Grid update** — if position changed, calls `grid.updateObject(this)` and updates floor type and map indicator position.
10. **Dirty flag** — `setPartialDirty()` if moved or turning; `turning` cleared.
11. **Health regen** — `_modifiers.hpRegen` base, plus adrenaline-scaled regen (logarithmic model): drain `adrenaline -= 0.0005 * adrenDrain * dt`. `LacedStimulants` perk inverts the direction.
12. **Shield regen** — if `hasBubble`, adds `_modifiers.shieldRegen * dt / 1000`.
13. **Infection regen** — if `Infected` perk is active, increments infection per second.
14. **Item use/stop use** — processes `startedAttacking` and `stoppedAttacking` flags; calls `activeItem.useItem()` or `activeItem.stopUse()` subject to plugin gating.
15. **Gas damage** — if player is inside the gas and `gas.doDamage` is true, applies `piercingDamage` scaled by distance and (after 10 s inside gas) an additional `additionalGasDamage` accumulator.
16. **Bleed-out** — downed players who are not being revived take `bleedOutDPMs * dt` piercing damage.
17. **Revive range check** — if in `ReviveAction` and target is further than 7 units away, cancels the action.
18. **Smoke effects** — applies depletion health/adrenaline from overlapping smoke particles; updates scope target if `snapScopeTo` is set.
19. **Effective scope update** — downed or inside building without scope target → `DEFAULT_SCOPE`; otherwise scope or override.
20. **Rate-limit token drain** — decrements `emoteCount` every `rateLimitInterval` ms if above zero.
21. **Perk update tick** — for each perk in `perkUpdateMap`, fires the perk's interval-based effects if `now - lastUpdated > updateInterval`: `Bloodthirst` (bleed), `BabyPlumpkinPie` (swap weapon), `TornPockets` (drop ammo), `RottenPlumpkin` (damage + emote), `Shrouded` (spawn particle), `Necrosis` (bleed + infect), `Infected` (perk swap), `AchingKnees` (flip movement).
22. **ThermalGoggles / HollowPoints** — detects players in radius, maintains `highlightedIndicators` map, updates `dirty.highlightedPlayers`.
23. **EternalMagnetism** — drains health from nearby enemies into self, pulls loot toward self, updates `hasMagneticField` visual flag.
24. **Stuck projectiles** — repositions projectiles attached to the player (e.g. `Seedshot` seeds) according to `rotation` offset stored in `stuckProjectiles`.
25. **Automatic doors** — opens doors within 5 units that are inside the same building; schedules a close callback at 1 000 ms. Doors spaced too far from other open doors are skipped.
26. **Plugin event** — `pluginManager.emit("player_update", this)`.

### Phase 2: `player.secondUpdate()` (second pass, all connected players)

Called for every player in `connectedPlayers`. Assembles and sends the `UpdatePacket`:

1. **Visibility recalculation** (every 8 ticks, or if `game.updateObjects` or `player.updateObjects` is set):
   - Computes `screenHitbox` as a square of side `zoom * 2 + 8` centred on the observed player.
   - Objects that left the viewport are added to `packet.deletedObjects`.
   - Objects newly entering the viewport are added to `fullObjects` (force full serialization for this player).
2. **Full dirty objects** — any object in `game.fullDirtyObjects` that is within the player's `visibleObjects` set is added to `fullObjects`.
3. **Partial dirty objects** — any object in `game.partialDirtyObjects` that is visible and not already in `fullObjects` is added to `partialObjectsCache`.
4. **Player data** — all `dirty.*` flags on the observed player determine which `playerData` fields are included: health, adrenaline, shield, infection, zoom, weaponInventory, items, layer, perks, teamID, teammates, c4s, lockedSlots, highlightedPlayers. On the first packet (`_firstPacket`) all fields are force-included.
5. **Bullet culling** — `newBullets` filtered by `Collision.lineIntersectsRectTest` against `screenHitbox`.
6. **Explosion culling** — `game.explosions` filtered to those inside `screenHitbox` and within `explosionMaxDistSquared`.
7. **Emotes** — `game.emotes` filtered to those whose player is in `visibleObjects`.
8. **Gas** — included if `gas.dirty` (state changed) or `_firstPacket`; `gasProgress` included if `gas.completionRatioDirty` or first packet.
9. **New/deleted players, alive count, planes, map pings, map indicators, kill leader** — all included with appropriate dirty-flag or first-packet guards.
10. **Serialize and send** — `sendPacket(packet)` serializes the `UpdatePacket` into the player's `_packetStream` buffer and writes it to the WebSocket. Then queued `_packets` (e.g. `KillPacket`) and `game.packets` (broadcast packets) are also serialized and sent.
11. **`_firstPacket = false`** — cleared so future ticks only send incremental updates.

### Phase 3: `player.postPacket()` (third pass, all connected players)

```typescript
postPacket(): void {
    for (const key in this.dirty) {
        this.dirty[key as keyof Player["dirty"]] = false;
    }
    this._animation.dirty = false;
    this._action.dirty = false;
}
```

Resets all entries in `player.dirty` to `false`, plus resets `_animation.dirty` and `_action.dirty`. This must happen **after** all `secondUpdate()` calls complete, because multiple spectators of the same player all read `dirty` flags during their `secondUpdate()` — resetting early would cause spectators processed later to miss state changes.

---

## Dirty Object System

`Game` maintains two `Set<BaseGameObject>` instances:

| Set | Populated by | Serialized via |
|-----|-------------|----------------|
| `partialDirtyObjects` | `BaseGameObject.setPartialDirty()` | `serializePartial()` — high-frequency fields: position, rotation, animation |
| `fullDirtyObjects` | `BaseGameObject.setDirty()` (= setFullDirty) | `serializeFull()` — all fields including inventory, health, full definition |

**Priority rule:** Step 13 skips `serializePartial()` for any object also present in `fullDirtyObjects`:

```typescript
for (const partialObject of this.partialDirtyObjects) {
    if (this.fullDirtyObjects.has(partialObject)) continue; // full shadows partial
    partialObject.serializePartial();
}
```

When an object first enters a player's viewport (step 14, `secondUpdate`), it is added to that player's local `fullObjects` set regardless of the game-wide dirty sets — ensuring first-time visibility always yields a complete snapshot.

After step 14 (`secondUpdate`) reads the serialized buffers, both sets are cleared in step 17 (reset).

---

## Timeout Queue

`game._timeouts` is a `Set<Timeout>` (declared `private` in `Game`). The `Timeout` class (`common/src/utils/misc.ts:357`) stores:

| Field | Type | Meaning |
|-------|------|---------|
| `callback` | `() => void` | Logic to run when the timeout fires |
| `end` | `number` | Absolute `Date.now()` timestamp at which to fire |
| `killed` | `boolean` | If `true`, discard without firing |

**Queuing a timeout:**
```typescript
addTimeout(callback: () => void, delay = 0): Timeout {
    const timeout = new Timeout(callback, this.now + delay);
    this._timeouts.add(timeout);
    return timeout;
}
```
`delay` is in milliseconds relative to `this.now` (the tick-start timestamp). A `delay = 0` fires on the *next* tick's timeout loop.

**Cancellation:** Call `timeout.kill()` to set `killed = true`. The timeout is pruned from the set on the next tick iteration without invoking the callback. The caller must store the returned `Timeout` handle to be able to cancel.

**One-shot only:** Timeouts are removed after firing. There is no repeat/interval variant at this layer; repeating effects are implemented by queuing a new timeout inside the callback (e.g. the automatic door close loop).

---

## Gas Tick

`Gas.tick()` (step 11) does three things on every tick:

1. **Completion ratio update** — if state is not `Inactive`, updates `completionRatio = (now - countdownStart) / (1000 * currentDuration)` and sets `completionRatioDirty = true`.
2. **Damage gate** — resets `_doDamage = false`. If ≥ 1 000 ms have elapsed since `_lastDamageTimestamp`, sets `_doDamage = true` and updates `_lastDamageTimestamp`. Actual player damage is applied in `player.update()` (step 12) by checking `gas.doDamage`.
3. **Position/radius interpolation** — when `_doDamage` fires and the state is `Advancing`, lerps `currentPosition` and `currentRadius` toward their target values using `completionRatio`.

Gas stage transitions (`advanceGasStage()`) are triggered by timeouts registered in the previous stage, not directly by `tick()`.

---

## Projectile Update

`Projectile.update()` (called in step 6) operates in this order:

1. **Fuse countdown** — decrements `_fuseTime` by `game.dt` (skipped for unactivated C4). When `_fuseTime < 0`, calls `_detonate()` and returns.
2. **Collision detection** — queries `grid.intersectsHitbox` with a line rect from `_lastPosition` to `position`; checks hitbox/line intersections against buildings, obstacles, and players; sorts by distance.
3. **Collision response** — for each sorted collision: stair interactions update layer; below-height obstacles are skipped; above-height triggers `impactDamage` and bounce via `_reflect()`; resolves hitbox collision.
4. **Height / gravity** — applies `GameConstants.projectiles.gravity * dt` to `_velocityZ`, clamps height to `[0, maxHeight = 5]`. If sitting on an obstacle, height floor is the obstacle's height.
5. **Drag** — scales velocity by `1 / (1 + dt * speedDrag)` where `speedDrag` is chosen by floor type: air `0.7`, water `5`, ice `1`, ground `3`.
6. **Position update** — adds `velocity * dt`; clamps to map bounds.
7. **Angular velocity damping** — `angularVelocity *= 1 / (1 + dt * 1.2)`.
8. **Dirty flag** — `setPartialDirty()` if position, rotation, or height changed.

---

## Timing and Scheduling

```
idealDt = 1000 / (Config.tps ?? 40) = 25 ms (default)
```

After completing all 21 steps, the next tick is scheduled as:

```typescript
setTimeout(this.tick.bind(this), this.idealDt - (Date.now() - now));
```

`Date.now() - now` is the wall-clock duration of the current tick. Subtracting it from `idealDt` targets the next tick to fire at `now + idealDt`. If the tick overran (elapsed > 25 ms), the delay is negative — `setTimeout` clamps this to `0`, so the next tick fires on the next event loop iteration with no further compensation. There is no catch-up, backfill, or fixed-timestep accumulator.

---

## Tick Time Logging

Tick duration (not inter-tick interval) is pushed into `_tickTimes: number[]`. Every 200 samples:

- `mspt` — arithmetic mean via `Statistics.average()`
- `stddev` — standard deviation via `Statistics.stddev()`
- `Load` — `(mspt / idealDt) * 100` percent of 25 ms budget consumed

The buffer is cleared after each log line. Example output:

```
[Game 0] ms/tick: 3.41 ± 0.92 | Load: 13.6%
```

---

## Win Condition

Evaluated in step 18, after all state resets:

```
Solo:  aliveCount <= 1
Teams: aliveCount <= teamMode (numeric) AND all survivors share the same teamID
Guard: _started == true
       over == false
       Config.minTeamsToStart (default 2) > 1
       now - startedTime > 5000 ms
```

On win activation:
1. All surviving players have movement and attack cleared.
2. Each survivor plays win emote (`loadout.emotes[6]`) and receives `GameOverPacket(won = true)`.
3. Plugin events `player_did_win` (per winner) and `game_end` (once) are emitted.
4. `allowJoin` set to `false`, `over` set to `true`.
5. A 1 000 ms timeout disconnects all `connectedPlayers` and sets `_stopped = true`, halting the tick loop at step 21.

---

## Data Lineage

```
WebSocket InputPacket received between ticks
    → player.processInputs(packet)
    → stored in player.movement / player.startedAttacking / etc.
        ↓
Game.tick() begins:

  Step 1:  dt captured
  Step 2:  registered timeout callbacks fire
  Step 3:  periodic airdrops checked
  Steps 4–7:  loot, parachutes, projectiles, synced particles advance physics
  Step 8:  bullet trajectories swept → DamageRecord[] collected, dead bullets removed
  Step 9:  DamageRecord[] applied → objects take damage, on-hit effects spawned
  Step 10: explosion.explode() called for all queued explosions
  Step 11: gas.tick() → completionRatio updated, 1 Hz damage gate evaluated
  Step 12: player.update() per living player
           → movement/collision/position resolved
           → actions (firing, healing, reviving) processed
           → gas damage applied
           → dirty flags set (setPartialDirty / setDirty)
  Step 13: partialDirtyObjects → serializePartial()  (skip if in fullDirtyObjects)
           fullDirtyObjects    → serializeFull()
  Step 14: player.secondUpdate() per connected player
           → visibility re-calculated (every 8 ticks or on demand)
           → UpdatePacket assembled from visible / dirty sets
           → UpdatePacket + queued packets sent via WebSocket
  Step 15: player.postPacket() per connected player
           → all dirty.* flags cleared
  Steps 16–17: indicator cleanup, per-tick collections reset
  Step 18: win condition evaluated → GameOverPacket if triggered
  Step 19: tick duration logged (rolling 200-sample average)
  Step 20: pluginManager.emit("game_tick", this)
  Step 21: setTimeout(tick, idealDt - elapsed) → reschedule
```

---

## Dependencies

- **Internal:** All game object `update()` methods are called from within this module (`Bullet`, `Projectile`, `Loot`, `Parachute`, `SyncedParticle`).
- **External:**
  - [Gas System](../../gas/README.md) — `gas.tick()` called at step 11
  - [Networking](../../networking/README.md) — `UpdatePacket` serialized and sent at step 14 via `player.secondUpdate()`
  - [Spatial Grid](../../spatial-grid/README.md) — `grid.pool.getCategory()` used in steps 4–7; `grid.intersectsHitbox()` used in player collision and bullet trajectory
  - [Plugin System](../../plugins/README.md) — `player_update`, `player_start_attacking`, `player_stop_attacking`, `game_tick`, `player_did_win`, `game_end` events emitted

---

## Configuration

| Setting | Effect | Default |
|---------|--------|---------|
| `GameConstants.tps` | Target tick rate (ticks per second) | `40` |
| `Config.tps` | Optional server-config override for tick rate (`server/config.json`) | `undefined` (falls back to `GameConstants.tps`) |
| `GameConstants.maxPosition` | World boundary; player position is clamped to `[radius, maxPosition − radius]` on both axes | `1924` |
| `GameConstants.player.bleedOutDPMs` | Bleed-out damage per millisecond for downed players | `0.002` (= 2 dps) |
| `GameConstants.gas.damageScaleFactor` | Extra gas damage per world unit past the unscaled distance into gas | `0.005` |
| `GameConstants.gas.unscaledDamageDist` | Distance into gas over which no scaling is applied | `12` |
| `Config.minTeamsToStart` | Minimum teams required to start; win condition disabled if ≤ 1 | `2` |
| `mode.summonAirdropsInterval` | Ms between automatic airdrop summons (mode-defined; `undefined` = disabled) | mode-defined |

---

## Complex Functions

### `Game.tick()` — @file server/src/game.ts

**Purpose:** Advance all game state by one tick in a fixed 21-step sequence.

**Implicit behaviour:**
- The first player loop uses `livingPlayers`; the second and third use `connectedPlayers`. A dead-but-connected spectator never goes through `player.update()` but does receive `UpdatePacket`s via `secondUpdate()`.
- Full serialization always shadows partial for the same dirty object (step 13 guard).
- Damage records from bullets are batched until step 9 to keep all bullets alive for the full trajectory sweep — prevents shotgun pellets from phasing through obstacles that were destroyed by earlier pellets in the same tick.
- The win condition (step 18) checks `this.now - this.startedTime > 5000` — the game must have been running for 5 seconds before a win can register. This prevents false wins during the join window.
- `_stopped = true` halts the tick loop: the `setTimeout` at step 21 is not registered when `_stopped` is set.

**Called by:** Itself via `setTimeout` (step 21); initial call from the `Game` constructor.

### `Player.secondUpdate()` — @file server/src/objects/player.ts

**Purpose:** Perform per-player visibility culling and assemble + send the `UpdatePacket`.

**Implicit behaviour:**
- Visibility is **not recomputed every tick**. `ticksSinceLastUpdate` counts up; recalculation fires every 8 ticks, or immediately when `game.updateObjects` or `player.updateObjects` is set (e.g. after a player teleports or a large batch of objects is created).
- The viewport is a rectangle: `dim = zoom * 2 + 8` per side. Scoped-up players have larger viewports.
- Objects newly entering the viewport are always sent full-serialized that tick, regardless of whether they are in `fullDirtyObjects`.
- Spectating players observe another player's position/scope — `player.secondUpdate()` uses `this.spectating ?? this` as the observed player, but reads `this.dirty` for whose data to include.
- Bullet culling uses a line-segment vs AABB test (`Collision.lineIntersectsRectTest`) on each `newBullet`'s trajectory — bullets that will never enter the viewport are dropped.

**Called by:** `Game.tick()` step 14.

### `Player.postPacket()` — @file server/src/objects/player.ts

**Purpose:** Reset dirty flags after all packets for this tick have been sent.

**Implicit behaviour:** Must run **after** all `secondUpdate()` calls. Multiple spectators of the same player each call `secondUpdate()` and read the same `dirty.*` flags. If `postPacket()` cleared them mid-loop, later spectators would see stale data.

**Called by:** `Game.tick()` step 15.

---

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Game Loop subsystem overview (contains the full tick-step quick-reference table)
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Dirty Object Tracking and Worker-Per-Game patterns
- **Networking:** [../../networking/README.md](../../networking/README.md) — UpdatePacket serialization format
- **Spatial Grid:** [../../spatial-grid/README.md](../../spatial-grid/README.md) — `grid.intersectsHitbox()` used in collision and visibility culling
- **Gas System:** [../../gas/README.md](../../gas/README.md) — `Gas.tick()` and damage scaling details
