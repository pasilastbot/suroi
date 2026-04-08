# Q: How does the airdrop system work and how do I configure when airdrops spawn?

<!-- @tags: game-loop, airdrops, map, plugins -->
<!-- @related: docs/subsystems/game-loop/README.md, docs/subsystems/gas/modules/stages.md -->

## Overview

Airdrops are crates dropped from a plane. They are triggered either:
- Automatically by gas stage progression (`summonAirdrop: true` on a `GasStage`)
- Via a plugin event
- Via the `give airdrop` dev command

The airdrop system has three phases:
1. **Plane phase** — A `Parachute` object flies across the map and drops a crate
2. **Fall phase** — The crate parachutes down (8000 ms)
3. **Land phase** — The crate lands and becomes an obstacle, dealing crush damage

## Key Constants (`common/src/constants.ts`)

| Constant | Value | Effect |
|----------|-------|--------|
| `airdrop.fallTime` | 8000 ms | Time from drop to landing |
| `airdrop.flyTime` | 30000 ms | Time the plane is in flight |
| `airdrop.crushDamage` | 300 HP | Damage dealt if a player is under it |

## Configuring Airdrops via Gas Stages

In `server/src/data/gasStages.ts`, set `summonAirdrop: true` on any
`Advancing` stage to trigger an airdrop when that stage begins:

```typescript
// Stage 3 — Waiting
{
    state: GasState.Waiting,
    duration: 45,
    oldRadius: 0.55,
    newRadius: 0.55,
    dps: 3,
},
// Stage 4 — Advancing + airdrop
{
    state: GasState.Advancing,
    duration: 20,
    oldRadius: 0.55,
    newRadius: 0.35,
    dps: 3,
    summonAirdrop: true,    // ← triggers airdrop when this stage starts
},
```

Multiple stages can have `summonAirdrop: true` for multiple drops.

## Triggering via Plugin

```typescript
// server/src/plugins/myPlugin.ts
this.on("game_started", () => {
    this.game.airdrop.summon();  // trigger an airdrop immediately
});
```

## Loot Quality

Airdrop crates use loot quality tiers. The quality is passed to
`getLootFromTable(modeName, tableID, quality)`. Airdrop tables (`airdrop_limited`,
`airdrop_common`, etc.) use quality to determine what drops.

To configure airdrop loot, edit `LootTables["normal"]["airdrop_limited"]` (and
other airdrop tables) in `server/src/data/lootTables.ts`.

## Tick Integration

During each tick (step 2), `game.ts` checks:
```
if (mode.airdropInterval elapsed) → game.airdrop.summon()
```

The interval comes from the mode definition or is triggered by gas stage.

## Server Objects Involved

| Object | File | Role |
|--------|------|------|
| `Parachute` | `server/src/objects/parachute.ts` | Flying plane + falling crate animation |
| `Obstacle` | `server/src/objects/obstacle.ts` | The crate after landing |

The `Parachute` is the airborne crate. When it lands, it converts to a standard
`Obstacle` (airdrop crate definition) that players can open.

## Client Rendering

The `Parachute` `ObjectCategory` is rendered on the client as a falling container
with a parachute sprite. It uses `ObjectCategory.Parachute` in the `UpdatePacket`
`GameObjects` split.

## Related

- [Game Loop](../subsystems/game-loop/README.md) — tick step 2 (airdrops)
- [Gas Stages](../subsystems/gas/modules/stages.md) — `summonAirdrop` field
- [Game Data Model](../datamodel.md) — airdrop fall/fly time constants
- [Loot Tables](11-add-loot-table.md) — airdrop loot tables
- [Create Server Plugin](05-create-server-plugin.md) — triggering airdrops from a plugin
