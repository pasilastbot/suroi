# Documentation Content Plan

## Overview
Suroi is an open-source browser-based 2D battle royale IO game (suroi.io) built with Bun, TypeScript, PixiJS, and Svelte. Players scavenge weapons, avoid the shrinking gas zone, and fight to be the last survivor.

## Documentation Index

| # | Module / Area | Tier | Status | Path | Priority | Last Updated |
|---|---------------|------|--------|------|----------|--------------|
| 1 | Project Description | T1 | Done | docs/description.md | Critical | 2026-04-16 |
| 2 | Architecture Overview | T1 | Done | docs/architecture.md | Critical | 2026-04-16 |
| 3 | Data Model | T1 | Done | docs/datamodel.md | Critical | 2026-04-16 |
| 4 | Development Guide | T1 | Done | docs/development.md | Critical | 2026-04-16 |
| 5 | API Reference | T1 | Done | docs/api-reference.md | Critical | 2026-04-16 |
| 5a | Core Math & Physics Utilities Subsystem | T2 | Done | docs/subsystems/core-math-physics/ | Critical | 2026-04-16 |
| 5a-i | Core Math & Physics — Vector Operations & Collision Shapes Module | T3 | Done | docs/subsystems/core-math-physics/modules/vectors-hitbox.md | Critical | 2026-04-16 |
| 5b | Server Utilities Subsystem | T2 | Done | docs/subsystems/server-utilities/ | Critical | 2026-04-16 |
| 5c | Serialization System Subsystem | T2 | Done | docs/subsystems/serialization-system/ | Critical | 2026-04-16 |
| 5c-i | Serialization System — Binary Encoding & SuroiByteStream Module | T3 | Done | docs/subsystems/serialization-system/modules/byte-encoding.md | Critical | 2026-04-16 |
| 5d | Collision & Hitbox System | T2 | Done | docs/subsystems/collision-hitbox/ | Critical | 2026-04-16 |
| 5d-i | Collision & Hitbox — Obstacle Collision & Response Module | T3 | Done | docs/subsystems/collision-hitbox/modules/obstacle-collision.md | Critical | 2026-04-16 |
| 5e | Utilities & Support Systems | T2 | Done | docs/subsystems/utilities-support/ | High | 2026-04-16 |
| 6 | Game Loop Subsystem | T2 | Done | docs/subsystems/game-loop/ | Critical | 2026-04-16 |
| 6a | Game Loop — Patterns | T2 | Done | docs/subsystems/game-loop/patterns.md | Critical | 2026-04-16 |
| 7 | Game Loop — Tick Module | T3 | Done | docs/subsystems/game-loop/modules/tick.md | High | 2026-04-16 |
| 7a | Game Loop — Update Phase Module | T3 | Done | docs/subsystems/game-loop/modules/update-phase.md | High | 2026-04-16 |
| 7b | Game Loop — Player Update Module | T3 | Done | docs/subsystems/game-loop/modules/player-update.md | High | 2026-04-16 |
| 7c | Game Loop — Object Update Module | T3 | Done | docs/subsystems/game-loop/modules/object-update.md | High | 2026-04-16 |
| 7d | Game Loop — Game State Module | T3 | Done | docs/subsystems/game-loop/modules/game-state.md | High | 2026-04-16 |
| 7e | Game Loop — Network Tick Module | T3 | Done | docs/subsystems/game-loop/modules/network-tick.md | High | 2026-04-16 |
| 7f | Death & Spectator System Subsystem | T2 | Done | docs/subsystems/death-spectator/ | High | 2026-04-16 |
| 8 | Networking Subsystem | T2 | Done | docs/subsystems/networking/ | Critical | 2026-04-16 |
| 8a | Networking — UpdatePacket Module | T3 | Done | docs/subsystems/networking/modules/updatepacket.md | Critical | 2026-04-16 |
| 8b | Networking — Packet Types Module | T3 | Done | docs/subsystems/networking/modules/packet-types.md | Critical | 2026-04-16 |
| 8c | Networking — Network Protocol Module | T3 | Done | docs/subsystems/networking/modules/protocol.md | Critical | 2026-04-16 |
| 9 | Object Definitions Subsystem | T2 | Done | docs/subsystems/object-definitions/ | Critical | 2026-04-16 |
| 10 | Spatial Grid Subsystem | T2 | Done | docs/subsystems/spatial-grid/ | Critical | 2026-04-16 |
| 10a | Spatial Grid — Patterns | T2 | Done | docs/subsystems/spatial-grid/patterns.md | Critical | 2026-04-16 |
| 10a-bis | Visibility & Line-of-Sight System | T2 | Done | docs/subsystems/visibility-los/ | High | 2026-04-16 |
| 10a-ter | Spatial Grid — Query & Optimization Module | T3 | Done | docs/subsystems/spatial-grid/modules/query-optimization.md | Critical | 2026-04-16 |
| 10b | Server-Side Game Objects Subsystem | T2 | Done | docs/subsystems/game-objects-server/ | Critical | 2026-04-16 |
| 10b-i | Server-Side Game Objects — Object Lifecycle Module | T3 | Done | docs/subsystems/game-objects-server/modules/object-lifecycle.md | Critical | 2026-04-16 |
| 10c-bis | Buildings & Map Props Subsystem | T2 | Done | docs/subsystems/buildings-props/ | High | 2026-04-16 |
| 10c | Projectiles & Ballistics Subsystem | T2 | Done | docs/subsystems/projectiles-ballistics/ | Critical | 2026-04-16 |
| 10c-i | Projectiles & Ballistics — Bullet Firing Module | T3 | Done | docs/subsystems/projectiles-ballistics/modules/bullet-firing.md | High | 2026-04-16 |
| 10c-ii | Explosions & Hazards Subsystem | T2 | Done | docs/subsystems/explosions-hazards/ | High | 2026-04-16 |
| 10c-ii-a | Explosions & Hazards — Explosion Physics Module | T3 | Done | docs/subsystems/explosions-hazards/modules/explosion-physics.md | High | 2026-04-16 |
| 10d | Knockback & Velocity Physics Subsystem | T2 | Done | docs/subsystems/knockback-velocity/ | High | 2026-04-16 |
| 10d-i | Knockback & Velocity — Movement Physics Module | T3 | Done | docs/subsystems/knockback-velocity/modules/movement-physics.md | High | 2026-04-16 |
| 11 | Map Generation Subsystem | T2 | Done | docs/subsystems/map/ | High | 2026-04-16 |
| 11a-i | Map — Killfeed & Events Module | T3 | Done | docs/subsystems/map/modules/killfeed-events.md | High | 2026-04-16 |
| 11a | Server Data Systems Subsystem | T2 | Done | docs/subsystems/server-data/ | Critical | 2026-04-16 |
| 12 | Gas System Subsystem | T2 | Done | docs/subsystems/gas/ | High | 2026-04-16 |
| 12a-ii | Camera Management — Spectate Mode Module | T3 | Done | docs/subsystems/camera-management/modules/spectate-mode.md | High | 2026-04-16 |
| 13b | Client-Side Game Objects Subsystem | T2 | Done | docs/subsystems/game-objects-client/ | High | 2026-04-16 |
| 13b-i | Client-Side Game Objects — Rendering Pipeline Module | T3 | Done | docs/subsystems/game-objects-client/modules/rendering-pipeline.md | High | 2026-04-16 |
| 13c | Input Management Subsystem | T2 | Done | docs/subsystems/input-management/ | High | 2026-04-16 |
| 13c-i | Input Management — Input Pipeline Module | T3 | Done | docs/subsystems/input-management/modules/input-pipeline.md | High | 2026-04-16 |
| 13c-ii | Input Management — Mobile Input Module | T3 | Done | docs/subsystems/input-management/modules/mobile-input
| 13a-i | Camera Management — Viewport Positioning & Zoom Effects Module | T3 | Done | docs/subsystems/camera-management/modules/viewport.md | High | 2026-04-16 |
| 13c | Input Management Subsystem | T2 | Done | docs/subsystems/input-management/ | High | 2026-04-16 |
| 13c-i | Input Management — Input Pipeline Module | T3 | Done | docs/subsystems/input-management/modules/input-pipeline.md | High | 2026-04-16 |
| 13d | Particle System Subsystem | T2 | Done | docs/subsystems/particle-system/ | High | 2026-04-16 |
| 13d-i | Particle System — Special Effects & Animations Module | T3 | Done | docs/subsystems/particle-system/modules/effects.md | High | 2026-04-16 |
| 13d-ii | Particle System — Emitter System Module | T3 | Done | docs/subsystems/particle-system/modules/emitter-system.md | High | 2026-04-16 |
| 13e | Sound Management Subsystem | T2 | Done | docs/subsystems/sound-management/ | Medium | 2026-04-16 |
| 13e-ii | Sound Management — Sound Events Module | T3 | Done | docs/subsystems/sound-management/modules/sound-events.md | Medium | 2026-04-16 |
| 13e-i | Sound Management — Audio Playback & Spatial Audio Module | T3 | Done | docs/subsystems/sound-management/modules/audio-playback.md | Medium | 2026-04-16 |
| 13f | UI Management Subsystem | T2 | Done | docs/subsystems/ui-management/ | High | 2026-04-16 |
| 13g | UI Management — Patterns | T2 | Done | docs/subsystems/ui-management/patterns.md | High | 2026-04-16 |
| 13g-i | UI Management — HUD Elements & State Synchronization Module | T3 | Done | docs/subsystems/ui-management/modules/hud-elements.md | High | 2026-04-16 |
| 13h | Client Auxiliary Managers Subsystem | T2 | Done | docs/subsystems/client-managers/ | Medium | 2026-04-16 |
| 13i | Console & Debug System Subsystem | T2 | Done | docs/subsystems/console-debug/ | High | 2026-04-16 |
| 14 | Inventory Subsystem | T2 | Done | docs/subsystems/inventory/ | Medium | 2026-04-16 |
| 14a | Inventory — Patterns | T2 | Done | docs/subsystems/inventory/patterns.md | Medium | 2026-04-16 |
| 14b | Melee Combat System Subsystem | T2 | Done | docs/subsystems/melee-combat/ | High | 2026-04-16 |
| 14b-i | Melee Combat — Attack Mechanics & Hit Detection Module | T3 | Done | docs/subsystems/melee-combat/modules/attack-mechanics.md | High | 2026-04-16 |
| 14c | Throwable/Grenade System Subsystem | T2 | Done | docs/subsystems/throwable-system/ | High | 2026-04-16 |
| 14c-i | Throwable System — Grenade Physics & Detonation Module | T3 | Done | docs/subsystems/throwable-system/modules/grenade-physics.md | High | 2026-04-16 |
| 14d-i | Equipment & Scopes — Scope Mechanics Module | T3 | Done | docs/subsystems/equipment-scope/modules/scope-mechanics.md | High | 2026-04-16 |
| 14d | Equipment & Scope System Subsystem | T2 | Done | docs/subsystems/equipment-scope/ | High | 2026-04-16 |
| 14e | Body Armor & Equipment Subsystem | T2 | Done | docs/subsystems/body-armor/ | High | 2026-04-16 |
| 14e-i | Body Armor — Armor Mitigation Module | T3 | Done | docs/subsystems/body-armor/modules/armor-mitigation.md | High | 2026-04-16 |
| 14f-ii | Status Effects & Conditions Subsystem | T2 | Done | docs/subsystems/status-effects/ | High | 2026-04-16 |
| 14f-ii-a | Status Effects & Conditions — Effect Application Module | T3 | Done | docs/subsystems/status-effects/modules/effect-application.md | High | 2026-04-16 |
| 14f | Health & Damage System Subsystem | T2 | Done | docs/subsystems/health-damage/ | High | 2026-04-16 |
| 14f-i | Health & Damage — Damage Application Pipeline Module | T3 | Done | docs/subsystems/health-damage/modules/damage-pipeline.md | High | 2026-04-16 |
| 15 | Plugin System Subsystem | T2 | Done | docs/subsystems/plugins/ | Medium | 2026-04-16 |
| 15a | Plugin System — Patterns | T2 | Done | docs/subsystems/plugins/patterns.md | Medium | 2026-04-16 |
| 15b | Team System Subsystem | T2 | Done | docs/subsystems/team-system/ | High | 2026-04-16 |
| 15b-i | Team System — Team Mechanics Module | T3 | Done | docs/subsystems/team-system/modules/team-mechanics.md | High | 2026-04-16 |
| 15c | Emote System Subsystem | T2 | Done | docs/subsystems/emote-system/ | Medium | 2026-04-16 |
| 15d | Airdrop System Subsystem | T2 | Done | docs/subsystems/airdrop-system/ | High | 2026-04-16 |
| 15d-i | Airdrop System — Spawning & Contents Module | T3 | Done | docs/subsystems/airdrop-system/modules/spawning.md | High | 2026-04-16 |
| 15e | Perks & Passive System Subsystem | T2 | Done | docs/subsystems/perks-passive/ | High | 2026-04-16 |
| 15e-i | Perks & Passive — Perk Effects & Application Module | T3 | Done | docs/subsystems/perks-passive/modules/perk-effects.md | High | 2026-04-16 |
| 15f | Game Modes System Subsystem | T2 | Done | docs/subsystems/game-modes/ | High | 2026-04-16 |
| 15g | Badges & Cosmetics Subsystem | T2 | Done | docs/subsystems/badges-cosmetics/ | Medium | 2026-04-16 |
| 16 | Inventory — GunItem Module | T3 | Done | docs/subsystems/inventory/modules/gun-item.md | High | 2026-04-16 |
| 16a | Inventory — Gun Mechanics & Firing System Module | T3 | Done | docs/subsystems/inventory/modules/gun-mechanics.md | High | 2026-04-16 |
| 17 | Inventory — Actions Module | T3 | Done | docs/subsystems/inventory/modules/actions.md | High | 2026-04-16 |
| 17a | Inventory — Equip & Pickup Module | T3 | Done | docs/subsystems/inventory/modules/equip-pickup.md | High | 2026-04-16 |
| 18 | Client Rendering — Game Client Module | T3 | Done | docs/subsystems/client-rendering/modules/game-client.md | High | 2026-04-16 |
| 18a | Client Rendering — Layer System & Z-Ordering Module | T3 | Done | docs/subsystems/client-rendering/modules/layer-ordering.md | High | 2026-04-16 |
| 18b | Client Rendering — Visual Effects Module | T3 | Done | docs/subsystems/client-rendering/modules/visual-effects.md | High | 2026-04-16 |
| 18c | Client Rendering — Container & Viewport Module | T3 | Done | docs/subsystems/client-rendering/modules/container-rendering.md | High | 2026-04-16 |
| 18d | Client-Side Game Objects — Sprite Management Module | T3 | Done | docs/subsystems/game-objects-client/modules/sprite-management.md | High | 2026-04-16 |
| 18e | Client-Side Game Objects — Player Module | T3 | Not Started | docs/subsystems/game-objects-client/modules/player.md | Medium | — |
| 18f | Client-Side Game Objects — Building Module | T3 | Not Started | docs/subsystems/game-objects-client/modules/building.md | Medium | — |
| 19 | UI Management — Minimap Module | T3 | Done | docs/subsystems/ui-management/modules/minimap.md | High | 2026-04-16 |
| 19a | UI Management — Settings & Menus Module | T3 | Done | docs/subsystems/ui-management/modules/settings-menus.md | High | 2026-04-16 |
| 20 | Console & Debug — Debug Tools Module | T3 | Done | docs/subsystems/console-debug/modules/debug-tools.md | High | 2026-04-16 |
| 21 | Object Definitions — Registry Module | T3 | Done | docs/subsystems/object-definitions/modules/registry.md | Medium | 2026-04-16 |
| 22 | Map Generation — Algorithm Module | T3 | Done | docs/subsystems/map/modules/generation-algorithm.md | Medium | 2026-04-16 |
| 22a | Map Generation — Terrain & Obstacle Placement Module | T3 | Done | docs/subsystems/map/modules/terrain-obstacles.md | High | 2026-04-16 |
| 23 | Common Utils Subsystem | T2 | Done | docs/subsystems/common-utils/ | High | 2026-04-16 |
| 23a | Common Utils — Constants & Configuration Module | T3 | Done | docs/subsystems/common-utils/modules/constants-config.md | High | 2026-04-16 |
| 24 | Server Utilities — Spatial Queries Module | T3 | Done | docs/subsystems/server-utilities/modules/spatial-queries.md | Critical | 2026-04-16 |
| 24a | Server Utilities — Data Structures Module | T3 | Done | docs/subsystems/server-utilities/modules/data-structures.md | High | 2026-04-16 |
| 25 | Object Definitions — Definition Loading Module | T3 | Done | docs/subsystems/object-definitions/modules/definition-loading.md | Critical | 2026-04-16 |
| 26 | Translations Subsystem | T2 | Done | docs/subsystems/translations/ | High | 2026-04-16 |
| 26a | Translations — Translation System Module | T3 | Done | docs/subsystems/translations/modules/translation-system.md | High | 2026-04-16 |
| 27 | Plugin System — Plugin Development Module | T3 | Done | docs/subsystems/plugins/modules/plugin-development.md | Medium | 2026-04-16 |
| 28 | Map — Map Rendering Module | T3 | Done | docs/subsystems/map/modules/map-rendering.md | High | 2026-04-16 |
| 29 | Airdrop System — Plane & Parachute Animation Module | T3 | Done | docs/subsystems/airdrop-system/modules/plane-animation.md | High | 2026-04-16 |
| 30 | Gas System — Gas Phases Module | T3 | Done | docs/subsystems/gas/modules/gas-phases.md | High | 2026-04-16 |
| 31 | Inventory — Healing & Utility Module | T3 | Done | docs/subsystems/inventory/modules/healing-utility.md | High | 2026-04-16 |
| 32 | Melee Combat — Weapon Variants Module | T3 | Done | docs/subsystems/melee-combat/modules/weapon-variants.md | High | 2026-04-16 |
| 33 | Map — Loot Generation Module | T3 | Done | docs/subsystems/map/modules/loot-generation.md | High | 2026-04-16 |
| 34 | Knockback & Velocity — Ice Physics Module | T3 | Done | docs/subsystems/knockback-velocity/modules/ice-physics.md | High | 2026-04-16 |
| 35 | Client Auxiliary Managers — Message System Module | T3 | Done | docs/subsystems/client-managers/modules/message-system.md | Medium | 2026-04-16 |
| 36 | Equipment & Scope — Weapon Customization Module | T3 | Done | docs/subsystems/equipment-scope/modules/weapon-customization.md | High | 2026-04-16 |
| 37 | Badges & Cosmetics — Emote System Module | T3 | Done | docs/subsystems/badges-cosmetics/modules/emote-system.md | High | 2026-04-16 |
| 38 | Buildings & Props — Door Mechanics Module | T3 | Done | docs/subsystems/buildings-props/modules/door-mechanics.md | High | 2026-04-16 |
| 39 | Perks & Passive — Perk Stacking & Conflicts Module | T3 | Done | docs/subsystems/perks-passive/modules/perk-conflicts.md | High | 2026-04-16 |
| 40 | Death & Spectator — Revival Mechanics Module | T3 | Done | docs/subsystems/death-spectator/modules/revival-mechanics.md | High | 2026-04-16 |
| 41 | Game Modes — Mode-Specific Rules Module | T3 | Done | docs/subsystems/game-modes/modules/mode-rules.md | High | 2026-04-16 |
| 42 | Client Managers — Minimap Rendering Module | T3 | Done | docs/subsystems/client-managers/modules/minimap-rendering.md | High | 2026-04-16 |

## Status Legend
- **Done** — Written, reviewed, up-to-date with current code
- **In Progress** — Draft exists, may be incomplete or outdated
- **Not Started** — Needs to be written
- **Outdated** — Exists but doesn't match current code

## Generation Order
Generate documentation in dependency order:
1. **Tier 1 first** — Architecture, data model, development guide (everything depends on these)
2. **Core Tier 2 next** — Subsystems that other subsystems depend on (object definitions in `common/`, networking packets, spatial grid)
3. **Feature Tier 2** — Business logic subsystems (ordered by developer priority: game loop, map, gas, inventory, client rendering)
4. **Tier 3 last** — Module docs within each subsystem (ordered by complexity)

## What Each Document Must Include
- Purpose and scope
- Key files and entry points (with `@file` references to source code)
- Data flow (how data moves through the module)
- Dependencies on other modules (with links to their Tier 2/3 docs)
- Cross-tier references ("See also" links up and down the tiers)
- Known issues, tech debt, or gotchas
