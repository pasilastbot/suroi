# Gas System Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/gas/modules/ -->
<!-- @source: server/src/gas.ts, server/src/data/gasStages.ts -->

## Purpose

Manages the shrinking safe zone (the "gas") that forces players toward the center of the map. Players outside the current safe-zone circle take distance-scaled piercing damage every second, with escalating damage the longer they remain outside.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/gas.ts` | `Gas` class — state machine, position/radius tracking, damage helper |
| `server/src/data/gasStages.ts` | `GasStages` array + `GasStage` interface — full stage schedule |
| `common/src/constants.ts` | `GasState` enum, `GameConstants.gas` damage constants |
| `server/src/objects/player.ts` | Per-player damage application (lines ~1415–1430) |
| `common/src/packets/updatePacket.ts` | `UpdateFlags.Gas` / `UpdateFlags.GasPercentage` serialization |
| `client/src/scripts/managers/gasManager.ts` | `GasManager` state tracker + `GasRender` PixiJS overlay |

---

## Architecture

The gas system is a **linear state machine** with three states (see `GasState` below). All state is held in a single `Gas` instance owned by `Game`. Transitions are driven by `addTimeout` callbacks scheduled at the end of each `advanceGasStage()` call. The `tick()` method runs on every game tick (40 TPS) to update `completionRatio` and, once per wall-clock second, lerp the current position/radius and set the damage flag.

---

## GasState Enum

Defined in `common/src/constants.ts` as a `const enum`:

```typescript
export const enum GasState {
    Inactive,   // 0 — initial state, damage off, zone not yet shown
    Waiting,    // 1 — countdown running, safe zone circle visible but not moving
    Advancing   // 2 — zone actively shrinking toward new position
}
```

---

## Gas Stages

### Stage Data Structure

Defined in `server/src/data/gasStages.ts`:

```typescript
export interface GasStage {
    readonly state: GasState          // Which GasState this stage runs in
    readonly duration: number          // Stage length in seconds (0 = infinite)
    readonly oldRadius: number         // Start radius as a fraction of mapSize
    readonly newRadius: number         // End radius as a fraction of mapSize
    readonly dps: number               // Damage per second for players in gas
    readonly summonAirdrop?: boolean   // Whether to drop a supply crate on stage entry
    readonly scaleDamageFactor?: number // Additional HP/s added after 10s in gas (final stages)
    readonly finalStage?: boolean      // Client-side text: "Final Zone" messaging
}
```

`mapSize` is computed as `(map.width + map.height) / 2`. Radii are multiplied by `mapSize` to get world-space units.

The predefined radius breakpoints (defined locally as `gasStageRadii`):

```typescript
const gasStageRadii: number[] = [
    0.76, // 0 — initial zone
    0.55, // 1
    0.43, // 2
    0.32, // 3
    0.20, // 4
    0.09, // 5
    0     // 6 — final zone fully closed
];
```

### Stage Schedule

`GAME_SPAWN_WINDOW = 84` seconds (player joining is blocked 81 s into the game). Total game timer (sum of all stage durations): **390 s = 6 m 30 s**.

| Index | State | Duration (s) | Old Radius | New Radius | DPS | summonAirdrop | scaleDamageFactor | finalStage |
|-------|-------|-------------|-----------|-----------|-----|---------------|-------------------|------------|
| 0 | Inactive | 0 | 0.76 | 0.76 | 0 | — | — | — |
| 1 | Waiting | 75 | 0.76 | 0.55 | 0 | — | — | — |
| 2 | Advancing | 20 | 0.76 | 0.55 | 1 | — | — | — |
| 3 | Waiting | 45 | 0.55 | 0.43 | 1 | ✓ | — | — |
| 4 | Advancing | 20 | 0.55 | 0.43 | 1 | — | — | — |
| 5 | Waiting | 40 | 0.43 | 0.32 | 2 | — | — | — |
| 6 | Advancing | 15 | 0.43 | 0.32 | 2 | — | — | — |
| 7 | Waiting | 35 | 0.32 | 0.20 | 3 | ✓ | — | — |
| 8 | Advancing | 15 | 0.32 | 0.20 | 3 | — | — | — |
| 9 | Waiting | 30 | 0.20 | 0.09 | 5 | — | — | — |
| 10 | Advancing | 15 | 0.20 | 0.09 | 5 | — | — | — |
| 11 | Waiting | 20 | 0.09 | 0 | 10 | ✓ | 1 | ✓ |
| 12 | Advancing | 60 | 0.09 | 0 | 10 | — | 1 | ✓ |
| 13 | Waiting | 0 (∞) | 0 | 0 | 10 | — | 1 | ✓ |

Stage 0 is the initial constructor state. `advanceGasStage()` always reads `GasStages[this.stage + 1]`, so the game starts by moving to index 1 (first Waiting stage).

---

## State Machine

```
[Constructor]
    └─ stage=0, state=Inactive, radii and positions set from GasStages[0]

[game start — enough teams joined, +3 s delay]
    └─ advanceGasStage() → stage=1, state=Waiting, duration=75 s
           └─ addTimeout(75 s) → advanceGasStage() → stage=2, state=Advancing, duration=20 s
                  └─ addTimeout(20 s) → advanceGasStage() → ... (continues through all stages)

[stage=13, duration=0]
    └─ no timeout scheduled — gas stays in final Waiting state forever
```

### What happens in each state

**`Inactive`** (stage 0 only):
- No damage. `completionRatio` is not updated.
- Zone circle is sent to clients via `Gas` flag on first packet.

**`Waiting`** (odd stages 1, 3, 5, 7, 9, 11, 13):
- On entry via `advanceGasStage()`: a **new target position** is chosen. It is a random point inside a circle of radius `(oldRadius − newRadius) × mapSize` centred on the previous position, then clamped so that 75% of the new safe zone stays inside map bounds.
- `completionRatio` ticks upward from ~0 to 1. Client shows a countdown timer.
- Damage is active at the stage's `dps` rate for players already outside.

**`Advancing`** (even stages 2, 4, 6, 8, 10, 12):
- On entry: `oldPosition`/`oldRadius` are not changed; `newPosition`/`newRadius` were already set in the preceding Waiting stage.
- Every second: `currentPosition` and `currentRadius` are lerped by `completionRatio`.
- Client shows "Gas is advancing" message.

---

## Gas Tick

`gas.tick()` is called once per game tick in `Game.tick()` (`server/src/game.ts` line 461):

```typescript
tick(): void {
    // 1. Update completion ratio every tick for all non-Inactive states
    if (this.state !== GasState.Inactive) {
        this.completionRatio = (this.game.now - this.countdownStart) / (1000 * this.currentDuration);
        this.completionRatioDirty = true;
    }

    // 2. Reset damage flag
    this._doDamage = false;

    // 3. Every 1000 ms (wall-clock): fire damage tick + lerp position if advancing
    if (this.game.now - this._lastDamageTimestamp >= 1000) {
        this._lastDamageTimestamp = this.game.now;
        this._doDamage = true;

        if (this.state === GasState.Advancing) {
            this.currentPosition = Vec.lerp(this.oldPosition, this.newPosition, this.completionRatio);
            this.currentRadius = Numeric.lerp(this.oldRadius, this.newRadius, this.completionRatio);
        }
    }
}
```

Key behaviours:
- `completionRatioDirty = true` is set every tick (40×/s) so clients receive smooth progress updates.
- Position/radius lerp only fires once per second (tied to the damage interval), not every tick.
- `doDamage` is read by `player.update()` to trigger per-player damage.

---

## Damage Calculation

### Base scaled damage (`gas.ts`)

```typescript
scaledDamage(position: Vector): number {
    const distIntoGas = Geometry.distance(position, this.currentPosition) - this.currentRadius;
    return this.dps
        + Numeric.clamp(distIntoGas - GameConstants.gas.unscaledDamageDist, 0, Infinity)
          * GameConstants.gas.damageScaleFactor;
}
```

Constants from `common/src/constants.ts`:
- `GameConstants.gas.unscaledDamageDist = 12` — first 12 units inside the gas get no extra scaling
- `GameConstants.gas.damageScaleFactor = 0.005` — +0.005 HP/s per unit past the 12-unit threshold

$$\text{damage} = \text{dps} + \max(0,\ \text{distIntoGas} - 12) \times 0.005$$

### Escalating damage after prolonged exposure (`player.ts`)

Applied in `player.update()` when `gas.doDamage && gas.isInGas(position)`:

```
if (player has been in gas continuously for ≥ 10 s):
    total damage += scaleDamageFactor + additionalGasDamage
    additionalGasDamage += scaleDamageFactor   // accumulates each damage tick
```

`scaleDamageFactor` is only defined on the final three stages (stages 11–13, value `1`). This mechanic makes the endgame circle increasingly lethal the longer a player lingers inside the gas. The accumulated extra damage resets to 0 when a player re-enters the safe zone:

```typescript
} else if (!gas.isInGas(this.position)) {
    this.timeWhenLastOutsideOfGas = now;
    this.additionalGasDamage = 0;
}
```

All gas damage is applied as **piercing damage** (bypasses armour), sourced as `DamageSources.Gas`.

---

## Gas State in UpdatePacket

Gas data is sent inside `UpdatePacket` using two separate bit flags (defined in `common/src/packets/updatePacket.ts`):

### `UpdateFlags.Gas` (bit 7) — full state change

Sent when `gas.dirty === true` (set in `advanceGasStage()`) or on a player's first packet.

| Field | Encoding |
|-------|----------|
| `state` | `uint8` — `GasState` enum value |
| `currentDuration` | `uint8` — stage duration in seconds |
| `oldPosition` | `writePosition` (map-space x/y) |
| `newPosition` | `writePosition` (map-space x/y) |
| `oldRadius` | `float(0, 2048, 2)` — 2 bytes, range 0–2048 |
| `newRadius` | `float(0, 2048, 2)` — 2 bytes, range 0–2048 |
| `finalStage` | `writeBooleanGroup` (1 bool) |

### `UpdateFlags.GasPercentage` (bit 8) — per-tick progress

Sent when `gas.completionRatioDirty === true` (every tick when not Inactive) or on first packet.

| Field | Encoding |
|-------|----------|
| `gasProgress` | `float(0, 1, 2)` — 2 bytes, normalised 0–1 completion ratio |

The server builds these fields in `player.secondUpdate()` (`server/src/objects/player.ts` lines ~1998–2004):

```typescript
if (gas.dirty || this._firstPacket) {
    packet.gas = gas;
}
if (gas.completionRatioDirty || this._firstPacket) {
    packet.gasProgress = gas.completionRatio;
}
```

---

## Client Rendering

Handled by `client/src/scripts/managers/gasManager.ts`.

### `GasManager` — state tracker

A singleton (`GasManagerClass`) that receives `gas` and `gasProgress` fields from `UpdatePacket`:

- **On `gas` update** (full state): stores `state`, `currentDuration`, `oldPosition`, `newPosition`, `oldRadius`, `newRadius`. Displays HUD message (`gas_inactive`, `gas_waiting`, `gas_advancing`, or `final_gas_*` variants). Shows/hides advancing CSS class on the timer element.
- **On `gasProgress` update**: updates countdown timer text (`M:SS` format). If `state === Advancing`: stores `lastPosition`/`lastRadius` and recalculates `position`/`radius` by lerping with the new progress value.

### `GasRender` — PixiJS overlay

Draws the gas fog-of-war using a PixiJS `Graphics` object:

- **Geometry**: a massive rectangle (`±100 000` units overdraw) filled with `Game.colors.gas`, with a 512-segment circular **hole** cut using `.cut()` (the safe zone).
- **`zIndex = 996`** — renders above most scene objects (see `ZIndexes.Gas` in constants.ts).
- **`update()`** called each render frame:
  - During `Advancing`: **client-side interpolation** between the last server-received position/radius and the current one, using `(Date.now() - lastUpdateTime) / serverDt` as the interpolation factor. This hides the 1-second server lerp granularity.
  - Otherwise: directly uses `GasManager.position` and `GasManager.radius`.
  - When `radius < 0.1` world units the hole is clamped to `1.0` and the centre is offset — this avoids a degenerate zero-scale transform.
- The `scale` of the `Graphics` object is set to `radius × pixelsPerUnit`; `position` is set to `center × pixelsPerUnit`.

---

## Data Flow

```
Game init
  └─ Gas constructor: stage=0 (Inactive), radii/position from GasStages[0]

Enough players join (≥ minTeamsToStart), 3 s later:
  └─ gas.advanceGasStage()
       ├─ Reads GasStages[1] (Waiting, 75 s)
       ├─ Calculates random newPosition within allowed drift radius
       ├─ Sets gas.dirty = true  →  UpdateFlags.Gas sent to all clients
       └─ Schedules addTimeout(75 s) → next advanceGasStage()

Every game tick (40 TPS) in Game.tick():
  └─ gas.tick()
       ├─ Updates completionRatio  →  completionRatioDirty = true
       └─ Once per second:
            ├─ doDamage = true
            └─ (if Advancing) lerp currentPosition, currentRadius

After gas.tick(), player.update() iterates living players:
  └─ if doDamage && player outside safe zone:
       └─ player.piercingDamage(scaledDamage + escalation, source=Gas)

player.secondUpdate() builds UpdatePacket:
  ├─ gas.dirty  →  Gas field (full state)
  └─ completionRatioDirty  →  GasPercentage field (progress float)

Client receives UpdatePacket:
  └─ GasManager.updateFrom(data)
       ├─ Full gas update: stores positions, radii, state; updates HUD message
       └─ Progress update: updates timer; (if Advancing) lerps position/radius

Each render frame:
  └─ GasRender.update()
       └─ Sets Graphics position + scale from (interpolated) GasManager data
```

---

## Interfaces & Contracts

### `Gas` class (`server/src/gas.ts`)

Main API exposed to other subsystems:

| Method / Property | Type | Purpose |
|---|---|---|
| `state: GasState` | readonly property | Current state: `Inactive`, `Waiting`, or `Advancing` |
| `stage: number` | readonly property | Current stage index (0–13) |
| `currentPosition: Vector` | property | Current safe-zone center (lerped each damage tick in Advancing state) |
| `currentRadius: number` | property | Current safe-zone radius in game units |
| `completionRatio: number` | readonly property | Stage progress: 0 (start) to 1 (end) |
| `doDamage: boolean` | readonly property | `true` only on wall-clock seconds when damage should be applied |
| `tick()` | method | Called once per game tick; updates `completionRatio`, applies position/radius lerp |
| `isInGas(position)` | method `→ boolean` | Returns `true` if position is outside current safe zone |
| `scaledDamage(position)` | method `→ number` | Calculates HP/s damage at the given position |
| `advanceGasStage()` | method | Transitions to next stage; schedules timeout for subsequent stage |

### Events (from plugin system)

The gas system does not emit plugin events directly. Player damage is emitted via `player_damage` and `player_will_piercing_damaged` from the Player subsystem.

### Data contracts

- `GasStage` interface — single-stage configuration (state, duration, old/new radius, DPS, flags)
- `GasState` enum — `Inactive | Waiting | Advancing`
- `GasData` (serialization) — position, radius, state, progress — sent in `UpdatePacket` `Gas` and `GasPercentage` flags

---

## Module Index (Tier 3)

No separate Tier 3 modules documented yet. All logic is in the single [server/src/gas.ts](../../../../../../server/src/gas.ts) class.

---

## Dependencies

- **Depends on:**
  - [Game Loop](../game-loop/) — `gas.tick()` is called inside `Game.tick()` each frame
  - Config (`server/src/utils/config.ts`) — optional `gas.disabled`, `gas.forceDuration`, `gas.forcePosition` overrides
- **Depended on by:**
  - [Networking](../networking/) — `UpdateFlags.Gas` / `UpdateFlags.GasPercentage` included in every `UpdatePacket`
  - Players (`server/src/objects/player.ts`) — reads `gas.doDamage`, `gas.isInGas()`, `gas.scaledDamage()` each tick
  - [Client Rendering](../client-rendering/) — `GasRender` draws the overlay; `GasManager` drives the HUD

---

## Gotchas & Tech Debt

- **Damage fires on wall-clock, not game ticks.** `_lastDamageTimestamp` uses `Date.now()` / `this.game.now`, so if the server is under load and ticks spike, the 1-second window could compress or stretch slightly.
- **Position lerp is also once-per-second.** Server-side `currentPosition`/`currentRadius` only update each damage tick, not each game tick. The client compensates with client-side interpolation in `GasRender.update()`.
- **`completionRatio` is set to `1` in `advanceGasStage()`** before being immediately overwritten by the next `tick()` call. It is a stale initialisation value, not a bug.
- **`scaleDamageFactor` escalation is unbounded.** `additionalGasDamage` accumulates without a cap in the final stages. A player who remains in the closed final zone continuously will take increasingly high damage per second.
- **Config overrides bypass normal gameplay.** `gas.disabled`, `gas.forceDuration`, and `gas.forcePosition` exist for development/custom modes and are not validated; `forceDuration = 0` on a stage with `duration !== 0` is handled correctly.

---

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview
- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — Core entity model
- **Tier 2:** [Game Loop](../game-loop/) — orchestrates `gas.tick()` each frame
- **Tier 2:** [Networking](../networking/) — `UpdatePacket` binary protocol
- **Tier 2:** [Client Rendering](../client-rendering/) — `GasRender` and HUD integration
- **Patterns:** [patterns.md](patterns.md) — Reusable gas patterns
