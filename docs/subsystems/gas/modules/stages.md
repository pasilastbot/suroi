# Gas — Stages Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/gas/README.md -->
<!-- @source: server/src/gas.ts, server/src/data/gasStages.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents gas stage definitions, state transitions, and the damage formula. The gas zone shrinks over time through configurable stages.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/gas.ts | `Gas` — position, radius, damage, `tick()`, `advanceGasStage()` | High |
| @file server/src/data/gasStages.ts | `GasStages`, `GasStage` — stage definitions | Medium |
| @file common/src/constants.ts | `GasState`, `GameConstants.gas` — damage scaling | Low |

## GasStage Structure

| Field | Purpose |
|-------|---------|
| `state` | Inactive \| Waiting \| Advancing |
| `duration` | Seconds for this stage |
| `oldRadius`, `newRadius` | Fraction of map size (0–1) |
| `dps` | Damage per second at edge |
| `summonAirdrop?` | Spawn airdrop when advancing |
| `finalStage?` | Last stage; game ends when gas closes |

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

## Data Flow

```
Game tick → gas.tick()
    → If Advancing: lerp position/radius by completionRatio
    → If 1 second elapsed: set doDamage = true

Player update → if gas.doDamage
    → damage = gas.scaledDamage(player.position)
    → Apply damage

Stage advance → gas.advanceGasStage()
    → Load GasStages[stage+1]
    → Set state, duration, radii
```

## Business Rules

- `GAME_SPAWN_WINDOW` (84s) — players cannot join after this
- `gas.disabled` / `gas.forceDuration` in config override behavior

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Gas overview
- **Tier 2:** [../game-loop/](../../game-loop/) — Tick order
