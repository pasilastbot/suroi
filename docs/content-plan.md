# Documentation Content Plan

<!-- @tier: index -->
<!-- @updated: 2026-03-04 -->

## Dev Questions

All 30 developer how-to questions in `docs/dev-questions/` have been answered.
See [docs/dev-questions/GAPS.md](dev-questions/GAPS.md) for the gap analysis and
improvement roadmap that drove this work.

## Overview

Suroi is an open-source 2D battle royale game inspired by surviv.io. Players drop onto a map, collect weapons and items, and fight to be the last one standing. The game runs at 40 TPS on a Bun-powered server and renders in the browser using PixiJS v8.

## Documentation Index

| # | Module / Area | Tier | Status | Path | Priority | Last Updated |
|---|---------------|------|--------|------|----------|--------------|
| 1 | Project Description | T1 | Done | docs/description.md | Critical | 2026-03-04 |
| 2 | System Architecture | T1 | Done | docs/architecture.md | Critical | 2026-03-04 |
| 3 | Game Data Model | T1 | Done | docs/datamodel.md | Critical | 2026-03-04 |
| 4 | Development Guide | T1 | Done | docs/development.md | Critical | 2026-03-04 |
| 5 | Protocol Reference | T1 | Done | docs/protocol.md | Critical | 2026-03-04 |
| 6 | Definitions Subsystem | T2 | Done | docs/subsystems/definitions/ | Critical | 2026-03-04 |
| 7 | Packets Subsystem | T2 | Done | docs/subsystems/packets/ | Critical | 2026-03-04 |
| 8 | Object Model Subsystem | T2 | Done | docs/subsystems/objects/ | Critical | 2026-03-04 |
| 9 | Game Loop Subsystem | T2 | Done | docs/subsystems/game-loop/ | High | 2026-03-04 |
| 10 | Inventory Subsystem | T2 | Done | docs/subsystems/inventory/ | High | 2026-03-04 |
| 11 | Map Generation Subsystem | T2 | Done | docs/subsystems/map/ | High | 2026-03-04 |
| 12 | Rendering Subsystem | T2 | Done | docs/subsystems/rendering/ | High | 2026-03-04 |
| 13 | Input & Controls Subsystem | T2 | Done | docs/subsystems/input/ | Medium | 2026-03-04 |
| 14 | Gas System Subsystem | T2 | Done | docs/subsystems/gas/ | Medium | 2026-03-04 |
| 15 | Plugin System Subsystem | T2 | Done | docs/subsystems/plugins/ | Medium | 2026-03-04 |
| 16 | UI & Translations Subsystem | T2 | Done | docs/subsystems/ui/ | Medium | 2026-03-04 |
| 17 | Definitions — Items | T3 | Done | docs/subsystems/definitions/modules/items.md | High | 2026-03-04 |
| 18 | Definitions — Buildings | T3 | Done | docs/subsystems/definitions/modules/buildings.md | High | 2026-03-04 |
| 19 | Definitions — Obstacles | T3 | Done | docs/subsystems/definitions/modules/obstacles.md | Medium | 2026-03-04 |
| 20 | Packets — UpdatePacket | T3 | Done | docs/subsystems/packets/modules/update-packet.md | High | 2026-03-04 |
| 21 | Packets — InputPacket | T3 | Done | docs/subsystems/packets/modules/input-packet.md | Medium | 2026-03-04 |
| 22 | Object Model — Server Objects | T3 | Done | docs/subsystems/objects/modules/server-objects.md | High | 2026-03-04 |
| 23 | Object Model — Client Objects | T3 | Done | docs/subsystems/objects/modules/client-objects.md | High | 2026-03-04 |
| 24 | Inventory — Weapons | T3 | Done | docs/subsystems/inventory/modules/weapons.md | High | 2026-03-04 |
| 25 | Map — Loot Tables | T3 | Done | docs/subsystems/map/modules/loot-tables.md | Medium | 2026-03-04 |
| 26 | Game Console Subsystem | T2 | Done | docs/subsystems/console/ | Medium | 2026-03-04 |
| 27 | Validation Subsystem | T2 | Done | docs/subsystems/validation/ | Medium | 2026-03-04 |
| 28 | Game Loop — Tick & Serialization | T3 | Done | docs/subsystems/game-loop/modules/tick-serialization.md | High | 2026-03-04 |
| 29 | Map — Generation | T3 | Done | docs/subsystems/map/modules/generation.md | High | 2026-03-04 |
| 30 | Gas — Stages | T3 | Done | docs/subsystems/gas/modules/stages.md | Medium | 2026-03-04 |
| 31 | Rendering — Object Lifecycle | T3 | Done | docs/subsystems/rendering/modules/object-lifecycle.md | High | 2026-03-04 |
| 32 | Plugins — Events | T3 | Done | docs/subsystems/plugins/modules/events.md | Medium | 2026-03-04 |
| 33 | Input — Key Bindings | T3 | Done | docs/subsystems/input/modules/key-bindings.md | Medium | 2026-03-04 |
| 34 | UI — Translations | T3 | Done | docs/subsystems/ui/modules/translations.md | Medium | 2026-03-04 |
| 35 | Packets — JoinPacket | T3 | Done | docs/subsystems/packets/modules/join-packet.md | Medium | 2026-03-04 |
| 36 | Console — Commands | T3 | Done | docs/subsystems/console/modules/commands.md | Medium | 2026-03-04 |
| 37 | Validation — validateDefinitions | T3 | Done | docs/subsystems/validation/modules/validate-definitions.md | Medium | 2026-03-04 |
| 38 | Definitions — Bullets | T3 | Done | docs/subsystems/definitions/modules/bullets.md | Medium | 2026-03-04 |
| 39 | Definitions — Modes | T3 | Done | docs/subsystems/definitions/modules/modes.md | Medium | 2026-03-04 |
| 40 | Packets — MapPacket | T3 | Done | docs/subsystems/packets/modules/map-packet.md | Medium | 2026-03-04 |
| 41 | Rendering — Camera | T3 | Done | docs/subsystems/rendering/modules/camera.md | Medium | 2026-03-04 |
| 42 | Inventory — Items | T3 | Done | docs/subsystems/inventory/modules/items.md | Medium | 2026-03-04 |
| 43 | Validation — validateSvgs | T3 | Done | docs/subsystems/validation/modules/validate-svgs.md | Low | 2026-03-04 |
| 44 | Packets — JoinedPacket | T3 | Done | docs/subsystems/packets/modules/joined-packet.md | Medium | 2026-03-04 |
| 45 | Definitions — Explosions | T3 | Done | docs/subsystems/definitions/modules/explosions.md | Medium | 2026-03-04 |
| 46 | Game Loop — GameManager | T3 | Done | docs/subsystems/game-loop/modules/game-manager.md | High | 2026-03-04 |
| 47 | Game Loop — Grid | T3 | Done | docs/subsystems/game-loop/modules/grid.md | Medium | 2026-03-04 |
| 48 | Packets — GameOverPacket | T3 | Done | docs/subsystems/packets/modules/game-over-packet.md | Low | 2026-03-04 |
| 49 | Packets — KillPacket | T3 | Done | docs/subsystems/packets/modules/kill-packet.md | Medium | 2026-03-04 |
| 50 | Object Model — Hitbox | T3 | Done | docs/subsystems/objects/modules/hitbox.md | High | 2026-03-04 |
| 51 | UI — HUD | T3 | Done | docs/subsystems/ui/modules/hud.md | Medium | 2026-03-04 |
| 52 | Packets — PickupPacket | T3 | Done | docs/subsystems/packets/modules/pickup-packet.md | Low | 2026-03-04 |
| 53 | Inventory — Actions | T3 | Done | docs/subsystems/inventory/modules/actions.md | Medium | 2026-03-04 |
| 54 | Map — Terrain | T3 | Done | docs/subsystems/map/modules/terrain.md | Medium | 2026-03-04 |
| 55 | Packets — SpectatePacket | T3 | Done | docs/subsystems/packets/modules/spectate-packet.md | Low | 2026-03-04 |
| 56 | Console — CVars | T3 | Done | docs/subsystems/console/modules/cvars.md | Medium | 2026-03-04 |
| 57 | Rendering — Spritesheets | T3 | Done | docs/subsystems/rendering/modules/spritesheets.md | Medium | 2026-03-04 |
| 58 | Packets — ReportPacket | T3 | Done | docs/subsystems/packets/modules/report-packet.md | Low | 2026-03-04 |
| 59 | Packets — DebugPacket | T3 | Done | docs/subsystems/packets/modules/debug-packet.md | Low | 2026-03-04 |
| 60 | Definitions — SyncedParticles | T3 | Done | docs/subsystems/definitions/modules/synced-particles.md | Medium | 2026-03-04 |
| 61 | Definitions — Decals | T3 | Done | docs/subsystems/definitions/modules/decals.md | Low | 2026-03-04 |
| 62 | Plugins — Creating Plugins | T3 | Done | docs/subsystems/plugins/modules/creating-plugins.md | Medium | 2026-03-04 |
| 63 | Input — Mobile | T3 | Done | docs/subsystems/input/modules/mobile.md | Medium | 2026-03-04 |
| 64 | Definitions — Emotes | T3 | Done | docs/subsystems/definitions/modules/emotes.md | Low | 2026-03-04 |
| 65 | Dev Q01 — Add New Gun | HowTo | Done | docs/dev-questions/01-add-new-gun.md | High | 2026-03-04 |
| 66 | Dev Q02 — Add New Obstacle | HowTo | Done | docs/dev-questions/02-add-new-obstacle.md | High | 2026-03-04 |
| 67 | Dev Q03 — Add New Building | HowTo | Done | docs/dev-questions/03-add-new-building.md | High | 2026-03-04 |
| 68 | Dev Q04 — Protocol Version Bump | HowTo | Done | docs/dev-questions/04-protocol-version-bump.md | High | 2026-03-04 |
| 69 | Dev Q05 — Create Server Plugin | HowTo | Done | docs/dev-questions/05-create-server-plugin.md | High | 2026-03-04 |
| 70 | Dev Q06 — Add New Game Mode | HowTo | Done | docs/dev-questions/06-add-new-game-mode.md | High | 2026-03-04 |
| 71 | Dev Q07 — Add New Perk | HowTo | Done | docs/dev-questions/07-add-new-perk.md | High | 2026-03-04 |
| 72 | Dev Q08 — Add New Throwable | HowTo | Done | docs/dev-questions/08-add-new-throwable.md | High | 2026-03-04 |
| 73 | Dev Q09 — Dirty Tracking | HowTo | Done | docs/dev-questions/09-dirty-tracking-serialization.md | High | 2026-03-04 |
| 74 | Dev Q10 — Add New Map | HowTo | Done | docs/dev-questions/10-add-new-map.md | High | 2026-03-04 |
| 75 | Dev Q11 — Add Loot Table | HowTo | Done | docs/dev-questions/11-add-loot-table.md | Medium | 2026-03-04 |
| 76 | Dev Q12 — Run Locally | HowTo | Done | docs/dev-questions/12-run-locally.md | Critical | 2026-03-04 |
| 77 | Dev Q13 — Add Packet Type | HowTo | Done | docs/dev-questions/13-add-packet-type.md | High | 2026-03-04 |
| 78 | Dev Q14 — Add Player Action | HowTo | Done | docs/dev-questions/14-add-player-action.md | High | 2026-03-04 |
| 79 | Dev Q15 — Add UI Element | HowTo | Done | docs/dev-questions/15-add-ui-element.md | High | 2026-03-04 |
| 80 | Dev Q16 — Add Translation String | HowTo | Done | docs/dev-questions/16-add-translation-string.md | Medium | 2026-03-04 |
| 81 | Dev Q17 — Add Console Command | HowTo | Done | docs/dev-questions/17-add-console-command.md | Medium | 2026-03-04 |
| 82 | Dev Q18 — Add CVar | HowTo | Done | docs/dev-questions/18-add-cvar.md | Medium | 2026-03-04 |
| 83 | Dev Q19 — Add Player Stat | HowTo | Done | docs/dev-questions/19-add-player-stat.md | High | 2026-03-04 |
| 84 | Dev Q20 — Add Emote | HowTo | Done | docs/dev-questions/20-add-emote.md | Medium | 2026-03-04 |
| 85 | Dev Q21 — Layer System | HowTo | Done | docs/dev-questions/21-layer-system.md | High | 2026-03-04 |
| 86 | Dev Q22 — Add Explosion | HowTo | Done | docs/dev-questions/22-add-explosion.md | Medium | 2026-03-04 |
| 87 | Dev Q23 — Add Melee | HowTo | Done | docs/dev-questions/23-add-melee.md | Medium | 2026-03-04 |
| 88 | Dev Q24 — Add Skin | HowTo | Done | docs/dev-questions/24-add-skin.md | Medium | 2026-03-04 |
| 89 | Dev Q25 — Airdrop System | HowTo | Done | docs/dev-questions/25-airdrop-system.md | Medium | 2026-03-04 |
| 90 | Dev Q26 — Bullet Ballistics | HowTo | Done | docs/dev-questions/26-bullet-ballistics.md | High | 2026-03-04 |
| 91 | Dev Q27 — Add Decal | HowTo | Done | docs/dev-questions/27-add-decal.md | Low | 2026-03-04 |
| 92 | Dev Q28 — Debug Rendering | HowTo | Done | docs/dev-questions/28-debug-rendering.md | High | 2026-03-04 |
| 93 | Dev Q29 — Profile Server | HowTo | Done | docs/dev-questions/29-profile-server.md | High | 2026-03-04 |
| 94 | Dev Q30 — WearerAttributes | HowTo | Done | docs/dev-questions/30-wearer-attributes.md | Medium | 2026-03-04 |
| 95 | Dev Questions Gap Analysis | Meta | Done | docs/dev-questions/GAPS.md | High | 2026-03-04 |

## Status Legend

- **Done** — Written, reviewed, up-to-date with current code
- **In Progress** — Draft exists, may be incomplete or outdated
- **Not Started** — Needs to be written
- **Outdated** — Exists but doesn't match current code

## Generation Order

1. **Tier 1 first** — Architecture, data model, development guide, protocol reference ✓ _complete_
2. **Core Tier 2 next** — Definitions, Packets, Object Model (everything depends on these) ✓ _complete_
3. **Server Tier 2** — Game Loop, Inventory, Map Generation, Gas, Plugins ✓ _complete_
4. **Client Tier 2** — Rendering, Input, UI & Translations ✓ _complete_
5. **Tier 3 last** — Module docs within each subsystem (ordered by complexity)

## Source → Documentation Coverage

| Source Area | Documented At | Status |
|-------------|---------------|--------|
| `common/src/constants.ts` | `docs/datamodel.md`, `docs/protocol.md` | Done |
| `common/src/definitions/` | `docs/subsystems/definitions/` | Done |
| `common/src/packets/` | `docs/protocol.md`, `docs/subsystems/packets/` | Done |
| `common/src/utils/` | `docs/subsystems/objects/` | Done |
| `server/src/game.ts` | `docs/subsystems/game-loop/` | Done |
| `server/src/gameManager.ts` | `docs/subsystems/game-loop/modules/game-manager.md` | Done |
| `server/src/inventory/` | `docs/subsystems/inventory/` | Done |
| `server/src/map.ts` | `docs/subsystems/map/` | Done |
| `common/src/utils/terrain.ts` | `docs/subsystems/map/modules/terrain.md` | Done |
| `server/src/gas.ts` | `docs/subsystems/gas/` | Done |
| `server/src/plugins/` | `docs/subsystems/plugins/` | Done |
| `client/src/scripts/objects/` | `docs/subsystems/rendering/` | Done |
| `client/src/scripts/managers/` | `docs/subsystems/rendering/`, `docs/subsystems/input/` | Done |
| `client/src/translations/` | `docs/subsystems/ui/` | Done |
| `client/src/scripts/console/` | `docs/subsystems/console/` | Done |
| `tests/src/` | `docs/subsystems/validation/` | Done |
| `server/src/team.ts` | `docs/subsystems/game-loop/` (Team section) | Done |
