# Gas System Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/gas.ts, server/src/data/gasStages.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Gas System implements the battle royale shrinking zone. A circular gas zone moves inward over time, damaging players outside it. The zone progresses through stages (Inactive, Waiting, Advancing) with configurable radius, duration, and damage.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/gas.ts` | `Gas` class — position, radius, damage, stage advancement |
| `server/src/data/gasStages.ts` | `GasStages`, `GasStage` — stage definitions |
| `common/src/constants.ts` | `GasState`, `GameConstants.gas` — damage scaling |

## Architecture

```
GasStages (gasStages.ts)
    └── Array of GasStage
            ├── state: Inactive | Waiting | Advancing
            ├── duration: number (seconds)
            ├── oldRadius, newRadius: fraction of map size
            ├── dps: damage per second
            ├── summonAirdrop?: boolean
            └── finalStage?: boolean

Gas (gas.ts)
    ├── stage, state, currentDuration
    ├── oldPosition, newPosition, currentPosition
    ├── oldRadius, newRadius, currentRadius
    ├── tick() — update position/radius, set doDamage
    ├── scaledDamage(position) — damage per second at position
    └── advanceGasStage() — transition to next stage
```

## Data Flow

```
Game tick
    → gas.tick()
    → If Advancing: lerp currentPosition/currentRadius by completionRatio
    → If 1 second elapsed: set doDamage = true

Player update
    → If gas.doDamage: check if player outside gas
    → damage = gas.scaledDamage(player.position)
    → Apply damage

Stage advance (triggered by game logic)
    → gas.advanceGasStage()
    → Load next GasStages[stage+1]
    → Set state, duration, radii
    → If game.mode.unlockStage: unlock bunker doors (hunted mode)
```

## Gas States

| State | Behavior |
|-------|----------|
| `Inactive` | Gas not yet started |
| `Waiting` | Gas paused; countdown to next stage |
| `Advancing` | Gas moving from oldRadius to newRadius |

## Damage Formula

```
distIntoGas = distance(player, gas.center) - gas.currentRadius
damage = dps + (distIntoGas - unscaledDamageDist) * damageScaleFactor
```

- `unscaledDamageDist` (12): No extra scaling within 12 units of edge
- `damageScaleFactor` (0.005): Linear scaling per unit beyond that

## GasStages

Stages define the progression. Example:

- Stage 0: Inactive (full map)
- Stage 1–2: Waiting (75s) → Advancing (20s) — shrink to 55%
- Stage 3–4: Waiting (45s) → Advancing — shrink further, airdrop
- ...

`GAME_SPAWN_WINDOW` (84s) — players cannot join after this many seconds.

## Module Index (Tier 3)

- [Stages](modules/stages.md) — GasStage definitions, damage formula, state transitions

## Protocol Considerations

- **Affects protocol:** Gas state is in UpdatePacket. Stage/radius format changes require protocol bump.

## Dependencies

- **Depends on:** Game (now, map), Config (gas.disabled, gas.forceDuration)
- **Depended on by:** Game loop (tick), Player (damage)

## Related Documents

- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — GasState, damage constants
- **Tier 2:** [../game-loop/](../game-loop/) — Tick order
