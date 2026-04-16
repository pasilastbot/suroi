# Gas System — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/gas/README.md -->

## Pattern: Gas Stage Configuration

**When to use:** Tuning the gas shrink schedule — adjusting zone sizes, timing, DPS, or adding airdrop triggers.

**Implementation:**

Each `GasStage` entry in the `GasStages` array describes one phase of the game. Stages always pair as **Waiting → Advancing** (except the initial Inactive stage 0 and the final infinite Waiting stage 13). To modify the schedule:

1. Adjust `gasStageRadii` fractions to change zone sizes (values are proportions of `mapSize = (width + height) / 2`).
2. Increase or decrease `duration` (seconds) to change how long each phase lasts. Set `duration: 0` only for the last stage to make it permanent.
3. Set `dps` to control base piercing damage per second in that phase.
4. Add `summonAirdrop: true` to spawn a supply crate inside the new safe zone when that stage begins (only fires when `game.mode.summonAirdropsInterval` is `undefined`).
5. Set `scaleDamageFactor` on late stages to enable escalating damage for players who remain in the gas for more than 10 continuous seconds (see _Out-of-Zone Damage Escalation_ pattern below).
6. Set `finalStage: true` to show "Final Zone" messaging on the client.

Every `advanceGasStage()` call reads `GasStages[stage + 1]`. The method schedules the next call using `game.addTimeout(() => this.advanceGasStage(), duration * 1000)`. Adding stages beyond index 13 or removing pairs will shift indices; recalculate any `mode.unlockStage` or `mode.forcedGoldAirdropStage` references that compare against `this.stage`.

**Example file:** `@file server/src/data/gasStages.ts`

---

## Pattern: New Zone Position Randomisation

**When to use:** Understanding or customising where the next safe circle appears during a Waiting stage.

**Implementation:**

When `advanceGasStage()` enters a Waiting state, the new target position is chosen as follows (`server/src/gas.ts`):

```typescript
// drift radius = how far the circle centre can move
const driftRadius = (oldRadius - newRadius) * mapSize;

// random point within driftRadius of the current centre
const { x, y } = randomPointInsideCircle(this.oldPosition, driftRadius);

// ensure at least 75% of the new zone stays inside map bounds
const safetyRadius = newRadius * 0.75;
this.newPosition = Vec(
    Numeric.clamp(x, safetyRadius, width - safetyRadius),
    Numeric.clamp(y, safetyRadius, height - safetyRadius)
);
```

Overrides are available via server config (`server/src/utils/config.ts`):
- `gas.forcePosition: true` — pin the new position to map centre (`width/2, height/2`)
- `gas.forcePosition: [x, y]` — pin to a specific coordinate

When `newRadius === 0` (final stage) the new position is cloned from the current one (no movement).

**Example file:** `@file server/src/gas.ts` — `advanceGasStage()`

---

## Pattern: Out-of-Zone Damage (Base Scaled)

**When to use:** Understanding how gas damage is calculated for a player at any given position outside the safe zone.

**Implementation:**

`Gas.scaledDamage(position)` (`server/src/gas.ts`):

```typescript
scaledDamage(position: Vector): number {
    const distIntoGas = Geometry.distance(position, this.currentPosition) - this.currentRadius;
    return this.dps
        + Numeric.clamp(distIntoGas - GameConstants.gas.unscaledDamageDist, 0, Infinity)
          * GameConstants.gas.damageScaleFactor;
}
```

- **`distIntoGas`** — how far the player is past the safe-zone edge (0 if exactly on the edge).
- **Base `dps`** — flat HP/s from the current stage (see stage table in README).
- **Distance scaling** — kicks in only beyond `unscaledDamageDist = 12` units. Each additional unit adds `damageScaleFactor = 0.005` HP/s. A player 100 units inside the gas takes an extra `(100 − 12) × 0.005 = 0.44` HP/s on top of the stage DPS.

This damage fires once per second (gated by `gas.doDamage`, which `Gas.tick()` sets to `true` once every 1000 ms). It is applied as **piercing damage** — armour provides no protection.

**Example file:** `@file server/src/gas.ts` — `scaledDamage()`  
**Constants:** `@file common/src/constants.ts` — `GameConstants.gas`

---

## Pattern: Out-of-Zone Damage Escalation (Final Stages)

**When to use:** Understanding the escalating damage mechanic in the final zone (stages 11–13).

**Implementation:**

Applied in `player.update()` (`server/src/objects/player.ts`):

```typescript
const applyScaleDamageFactor = (now - this.timeWhenLastOutsideOfGas) >= 10000;
if (gas.doDamage && gas.isInGas(this.position)) {
    this.piercingDamage({
        amount: gas.scaledDamage(this.position)
            + (applyScaleDamageFactor
               ? (gas.getDef().scaleDamageFactor ?? 0) + this.additionalGasDamage
               : 0),
        source: DamageSources.Gas
    });
    if (applyScaleDamageFactor) {
        this.additionalGasDamage += gas.getDef().scaleDamageFactor ?? 0;
    }
} else if (!gas.isInGas(this.position)) {
    this.timeWhenLastOutsideOfGas = now;
    this.additionalGasDamage = 0;
}
```

Key points:
- Escalation only activates after **10 continuous seconds** in the gas (`timeWhenLastOutsideOfGas` is reset on every safe-zone tick).
- `scaleDamageFactor` is `1` on stages 11–13. After 10 s: +1 HP/s is added. After 11 s: +2 HP/s. After 12 s: +3 HP/s. The value accumulates without a cap.
- Returning to the safe zone resets both `timeWhenLastOutsideOfGas` and `additionalGasDamage` to 0.
- Stages without `scaleDamageFactor` (undefined) are treated as 0 via the `?? 0` fallback — no escalation in early/mid game.

**Example file:** `@file server/src/objects/player.ts` — lines ~1415–1430

---

## Pattern: Gas Dirty Flags → UpdatePacket

**When to use:** Understanding when the server sends full gas state vs. lightweight progress-only updates to clients.

**Implementation:**

The `Gas` class exposes two dirty flags:

| Flag | Set by | Clears after | Triggers |
|------|--------|-------------|---------|
| `gas.dirty` | `advanceGasStage()` | After `secondUpdate()` sends the packet | `UpdateFlags.Gas` — full state (state, durations, positions, radii, finalStage) |
| `gas.completionRatioDirty` | `gas.tick()` every tick where `state !== Inactive` | After `secondUpdate()` sends the packet | `UpdateFlags.GasPercentage` — progress float (0–1) |

In `player.secondUpdate()` (`server/src/objects/player.ts`):

```typescript
if (gas.dirty || this._firstPacket) {
    packet.gas = gas;        // writes 9+ bytes: state, duration, 2 positions, 2 radii, boolean
}
if (gas.completionRatioDirty || this._firstPacket) {
    packet.gasProgress = gas.completionRatio;   // writes 2 bytes
}
```

On a player's **first packet** both fields are always included regardless of dirty state, ensuring a newly-joined client has a complete picture of the current gas phase.

**Example files:**  
`@file server/src/gas.ts` — dirty flag mutation  
`@file server/src/objects/player.ts` — dirty flag consumption  
`@file common/src/packets/updatePacket.ts` — `UpdateFlags.Gas` / `UpdateFlags.GasPercentage` encoding

---

## Pattern: Client-Side Gas Interpolation

**When to use:** Understanding how the client renders smooth gas movement despite only receiving position updates once per second from the server.

**Implementation:**

The server lerps `currentPosition`/`currentRadius` once per second (tied to the damage tick). The client must interpolate between packets to avoid jumpy rendering.

In `GasRender.update()` (`client/src/scripts/managers/gasManager.ts`):

```typescript
if (GasManager.state === GasState.Advancing) {
    const interpFactor = Numeric.clamp(
        (Date.now() - GasManager.lastUpdateTime) / Game.serverDt,
        0, 1
    );
    position = Vec.lerp(GasManager.lastPosition, GasManager.position, interpFactor);
    radius = Numeric.lerp(GasManager.lastRadius, GasManager.radius, interpFactor);
} else {
    position = GasManager.position;
    radius = GasManager.radius;
}
```

- `GasManager.lastUpdateTime` is set to `Date.now()` whenever a `gasProgress` update is processed while the state is Advancing.
- `GasManager.lastPosition`/`lastRadius` are saved before overwriting `position`/`radius` with the new server values.
- `Game.serverDt` is the expected milliseconds between server ticks (25 ms at 40 TPS). Using wall-clock elapsed time / serverDt gives a 0–1 factor that advances smoothly between frames.

This pattern is display-only. The server's authoritative `currentPosition`/`currentRadius` still govern collision (`gas.isInGas()`) and damage.

**Example file:** `@file client/src/scripts/managers/gasManager.ts` — `GasRender.update()`
