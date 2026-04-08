# Inventory — Actions Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/action.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents timed inventory actions: `Action` base class, `HealingAction`, `ReloadAction`, `ReviveAction`. Actions run for a duration and can be cancelled; they affect player state on completion.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/inventory/action.ts | `Action`, `HealingAction`, `ReloadAction`, `ReviveAction` | Medium |

## Action Base

- **Constructor:** `player`, `time` (seconds) — schedules `execute()` via `game.addTimeout`
- **execute()** — Called when timer completes; clears `player.action`
- **cancel()** — Kills timeout, clears action
- **speedMultiplier** — Reduces player movement during action (default 1)

## Action Types

| Action | Purpose |
|--------|---------|
| HealingAction | Use bandage, medikit, etc. — heals over time |
| ReloadAction | Gun reload — consumes ammo, refills magazine |
| ReviveAction | Revive downed teammate — target.revive() |

## HealingAction

- Uses `HealingItemDefinition` (heal amount, heal type)
- Duration from definition
- On execute: apply heal, consume item

## ReloadAction

- Uses `GunItem` — `fullReload` if magazine empty and `reloadFullOnEmpty`
- Duration from gun definition
- On execute: refill ammo from inventory, consume ammo items

## ReviveAction

- Target: downed `Player`
- Duration: `GameConstants.player.reviveTime` (modified by Field Medic perk)
- On execute: `target.revive()`
- On cancel: `target.beingRevivedBy = undefined`

## Business Rules

- Only one action per player at a time
- `player.setPartialDirty()` on start/cancel/execute
- Actions cancelled if player downed during execution

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Inventory overview
- **Tier 3:** [weapons.md](weapons.md) — GunItem, reload
- **Tier 3:** [items.md](items.md) — Healing items
- **Tier 2:** [../game-loop/](../../game-loop/) — addTimeout
