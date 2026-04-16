# Suroi — Project Description

<!-- @tier: 1 -->
<!-- @see-also: docs/architecture.md, docs/subsystems/ -->

## Overview

Suroi is an open-source browser-based 2D battle royale game inspired by [surviv.io](https://survivio.fandom.com/wiki/Surviv.io_Wiki). Players drop into a shared map, scavenge weapons and equipment, and fight to be the last survivor while a shrinking gas zone forces them together. The game is playable at [suroi.io](https://suroi.io) and is currently a work in progress.

## Purpose & Target Users

The project aims to be an open, community-driven successor to classic browser-based battle royale IO games. It targets players who enjoy fast, low-barrier web games, as well as developers who want to contribute to or self-host an open-source game server.

## Key Features

The following features are evidenced directly in the source code and README:

- **Battle royale loop** — players compete until one survives; kill leader system activates after `killLeaderMinKills: 3` kills (`common/src/constants.ts`)
- **Shrinking gas zone** — distance-scaled damage (`damageScaleFactor: 0.005` extra damage per distance unit beyond `unscaledDamageDist: 12`), defined in `GameConstants.gas`
- **Weapon inventory** — 4 inventory slots: 2 guns, 1 melee, 1 throwable (`inventorySlotTypings` in `common/src/constants.ts`)
- **Loot system** — loot types: Gun, Ammo, Melee, Throwable, HealingItem, Armor, Backpack, Scope, Skin, Perk; each with a defined pickup radius
- **Airdrop events** — crates fall with `fallTime: 8000 ms`, `flyTime: 30000 ms`, dealing `300` damage on impact
- **Perks** — up to `maxPerkCount: 1` active perk per player (maximum `maxPerks: 4` available)
- **Custom teams** — team management via `CustomTeam` / `CustomTeamPlayer` in `server/src/team.ts`; team mode switchable at runtime
- **Game modes** — default mode is `"normal"`; additional named modes defined in `common/src/definitions/modes`
- **Player health & adrenaline** — `defaultHealth: 200`, `maxAdrenaline: 100`, `maxShield: 100`, `maxInfection: 100`
- **Ice physics** — separate friction/acceleration constants for ice surfaces
- **Rate limiting & anti-abuse** — `rateLimitPunishmentTrigger`, `emotePunishmentTime`, punishment system in server
- **Projectiles** — physics-based with gravity (`10`), air drag (`0.7`), and ground drag (`3`)
- **Translations** — multi-language UI support via HJSON translation files in `client/src/translations/`
- **In-game map editor** — standalone editor page at `client/editor/`

## Business Domain

Genre: browser-based 2D battle royale IO game. The core competitive loop is:
1. All players spawn on the same map
2. Players scavenge weapons, armor, backpacks, and healing items
3. The gas zone shrinks progressively, dealing increasing damage outside the safe area
4. Last survivor (or team) wins

The game runs at **40 ticks per second** (`tps: 40`) with a grid cell size of **32 units** (`gridSize: 32`) and a map boundary of **1924 units** (`maxPosition: 1924`). The binary protocol version at time of writing is **73** (`protocolVersion: 73`).

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime / package manager | Bun (workspaces: `client/`, `common/`, `server/`, `tests/`) |
| Language | TypeScript 5.9.3 |
| Client rendering | PixiJS (WebGL 2D renderer) |
| Client UI | Svelte 5, SCSS |
| Build tool | Vite 6 |
| Transport | Bun WebSockets (binary frames) |
| Linter / formatter | Biome 2.2.4 |

Full tech stack details, folder structure, and deployment information are in [docs/architecture.md](architecture.md).

## Repository

- **License:** GPL-3.0
- **Version:** 0.30.2 (from `package.json`)
- **GitHub:** [HasangerGames/suroi](https://github.com/HasangerGames/suroi)
- **Live game:** [suroi.io](https://suroi.io)
- **Donations:** [ko-fi.com/suroi](https://ko-fi.com/suroi)
- **Discord:** [discord.suroi.io](https://discord.suroi.io)

## Subsystem References

Navigate to Tier 2 for subsystem details:

- [Game Loop](docs/subsystems/game-loop/) — server-side 40 TPS tick loop (`server/src/game.ts`, `server/src/gameManager.ts`)
- [Networking](docs/subsystems/networking/) — binary WebSocket protocol (`common/src/packets/`, `common/src/utils/suroiByteStream.ts`)
- [Object Definitions](docs/subsystems/object-definitions/) — entity definition registry (`common/src/utils/objectDefinitions.ts`, `common/src/definitions/`)
- [Spatial Grid](docs/subsystems/spatial-grid/) — broad-phase spatial hash for collision (`server/src/utils/grid.ts`)
- [Map Generation](docs/subsystems/map/) — procedural map layout (`server/src/map.ts`, `server/src/data/maps.ts`)
- [Gas System](docs/subsystems/gas/) — shrinking zone mechanics (`server/src/gas.ts`, `server/src/data/gasStages.ts`)
- [Client Rendering](docs/subsystems/client-rendering/) — PixiJS WebGL renderer (`client/src/scripts/game.ts`, `client/src/scripts/objects/`)

## Known Issues & Gotchas

- **Tick rate timing:** Game targets 40 TPS with `setTimeout`-based self-rescheduling (`server/src/game.ts:565`), but actual tick time depends on system timer resolution and JavaScript event loop load. No strict timing guarantees. See `ms/tick` logging metric.
- **Protocol version bumping:** `GameConstants.protocolVersion` must increment on **any** change to `SuroiByteStream` serialization format — even single-bit field reorderings. Mismatch causes immediate client disconnect with no recovery.
- **Inventory slot immutability:** Inventory slot order (`[Gun, Gun, Melee, Throwable]`) is hardcoded in `GameConstants.player.inventorySlotTypings`. Changing slot types requires server/client version sync and data migration.
- **Map boundary collision:** `maxPosition: 1924` is the hard map edge; objects outside this range are removed from the game. Careful when placing obstacles near boundaries.
- **Gas damage scaling absolute distance:** Gas damage scales with **absolute distance** beyond `unscaledDamageDist: 12` units into the gas, not by damage stage. Extremely distant players outside the gas can take unpredictable damage spikes if they drift far enough.

## Dependencies on This Document

This document provides the **foundation** for understanding Suroi's purpose, features, and tech stack. Every subsystem below depends on concepts introduced here:

- [Game Loop](docs/subsystems/game-loop/) — Explains the 40 TPS server tick loop, tick delta (25 ms), and tick scheduling strategy
- [Networking](docs/subsystems/networking/) — Explains the binary WebSocket protocol, packet framing, and protocol version semantics
- [Object Definitions](docs/subsystems/object-definitions/) — Explains how loot types, weapons, armor, and cosmetics are defined and registered
- [Spatial Grid](docs/subsystems/spatial-grid/) — Explains the grid cell size (32 units) and collision query strategy
- [Map Generation](docs/subsystems/map/) — Explains map layout, boundaries, and procedural object placement
- [Gas System](docs/subsystems/gas/) — Explains the shrinking zone mechanics and damage scaling
- [Client Rendering](docs/subsystems/client-rendering/) — Explains how PixiJS renders the 2D game world
- [Inventory](docs/subsystems/inventory/) — Explains the 4-slot inventory system and item pickup
- [Plugins](docs/subsystems/plugins/) — Explains the server-side plugin event system

## Related Documents

### Tier 1 — Architecture & High-Level
- [System Architecture](architecture.md) — Tech stack, process model, build system, code standards
- [Data Model](datamodel.md) — Core enumerations, GameConstants, typings
- [Development Guide](development.md) — Setup, build commands, testing, code standards
- [API Reference](api-reference.md) — Binary WebSocket protocol, packet types, serialization

### Tier 2 — Subsystems
- [Game Loop](docs/subsystems/game-loop/) — 40 TPS tick orchestration, game state updates
- [Networking](docs/subsystems/networking/) — Binary packet protocol, WebSocket endpoints, streaming
- [Object Definitions](docs/subsystems/object-definitions/) — Definition registry, serialization formats
- [Spatial Grid](docs/subsystems/spatial-grid/) — Collision detection, broad-phase culling
- [Map Generation](docs/subsystems/map/) — Procedural layout, landmarks, object placement
- [Gas System](docs/subsystems/gas/) — Shrinking zone, damage scaling, stage progression
- [Client Rendering](docs/subsystems/client-rendering/) — PixiJS display layers, ObjectPool, animation
- [Inventory](docs/subsystems/inventory/) — Item slots, equipment, loot system
- [Plugins](docs/subsystems/plugins/) — Event bus, server-side scripting

### Tier 3 — Key Modules
- [Game Tick Module](docs/subsystems/game-loop/modules/tick.md) — Per-tick state machine and update order
- [Update Packet Serialization](docs/subsystems/networking/modules/update-packet.md) — Delta-encoded object updates
- [Spatial Grid Query](docs/subsystems/spatial-grid/modules/broad-phase.md) — Collision query performance
