# Game Loop — GameManager Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/game-loop/README.md -->
<!-- @source: server/src/gameManager.ts, server/src/server.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents multi-game management: `GameManager`, `GameContainer`, cluster workers, team mode and map switching, and how new games are created when players join.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/gameManager.ts | `GameManager`, `GameContainer`, `WorkerMessages` | High |
| @file server/src/server.ts | Primary process — routes connections to workers | High |

## Architecture

```
Primary Process (server.ts)
    └── GameManager
            ├── games[] — GameContainer per worker
            ├── teamMode — Switcher<TeamMode> (solo/duo/squad)
            ├── map — Switcher<string> (map name)
            └── creating? — GameContainer being created

Worker Process (gameManager.ts)
    └── Game — one game per worker
    └── process.send(GameData) — aliveCount, allowJoin, over, startedTime
```

## GameContainer

- **worker** — Cluster worker process
- **aliveCount**, **allowJoin**, **over**, **startedTime** — from worker messages
- **sendMessage(WorkerMessage)** — UpdateTeamMode, UpdateMap, NewGame

## WorkerMessages

| Message | Purpose |
|---------|---------|
| `UpdateTeamMode` | Switch solo/duo/squad |
| `UpdateMap` | Switch map (e.g. normal, winter) |
| `UpdateMapOptions` | mapScaleRange |
| `NewGame` | Start new game in worker |

## Game Creation Flow

```
Player joins (no suitable game)
    → GameManager.getOrCreateGame()
    → creating = new GameContainer(worker)
    → Worker forks, creates Game
    → Worker sends { allowJoin: true }
    → creating = undefined, resolve promise
    → Route player to GameContainer
```

## Switcher

- **StaticOrSwitched** — Fixed value or rotation with cron
- **teamMode**, **map** — Can rotate on schedule (Config.teamMode, Config.map)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Game loop overview
- **Tier 3:** [tick-serialization.md](tick-serialization.md) — Tick within a Game
- **Tier 1:** [docs/architecture.md](../../../architecture.md) — Cluster setup
