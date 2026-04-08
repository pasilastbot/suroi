# Project Description

<!-- @tier: 1 -->
<!-- @see-also: docs/architecture.md -->
<!-- @source: package.json -->
<!-- @updated: 2026-03-04 -->

## Overview

Suroi is an open-source 2D battle royale game inspired by the now-defunct [surviv.io](https://surviv.io). Players parachute onto a shrinking map, scavenge weapons and supplies, and fight to be the last player (or team) standing. The game runs entirely in a web browser with no installation required.

Current version: **0.30.2** | License: **GPL-3.0**

## Gameplay

### Core Loop

1. Players join a lobby and drop in by parachute onto the map
2. They collect weapons, armor, healing items, and ammo from obstacles and loot crates
3. A poisonous gas zone gradually shrinks the playable area, forcing players together
4. Players fight until one player or team remains

### Game Modes

| Mode | Team Size | Description |
|------|-----------|-------------|
| Solo | 1 | Every player for themselves |
| Duo | 2 | Two-player teams |
| Squad | 4 | Four-player teams |

Downed teammates can be revived before they bleed out (8-second revive timer, 5-unit max distance).

### Map Variants

The game ships with several themed map variants that affect visuals and some gameplay:

| Variant | Theme |
|---------|-------|
| `normal` | Default grass/desert map |
| `winter` | Snow-covered terrain |
| `fall` | Autumn foliage |
| `halloween` | Halloween theme |
| `hunted` | Special hunt mode |
| `infection` | Infection game mode |

### Weapons & Items

Players carry up to 4 weapons in fixed inventory slots:

| Slot | Type |
|------|------|
| 0, 1 | Guns (primary/secondary) |
| 2 | Melee |
| 3 | Throwable |

**Guns** support Single, Burst, and Auto fire modes. Items include healing (bandages, medikit, cola, tablets), armor (vests, helmets), backpacks (expand inventory capacity), scopes (1x–15x), and perks.

## Target Audience

- **Players:** Fans of browser-based battle royale games, io-games
- **Contributors:** Open-source game developers interested in TypeScript, real-time multiplayer, and PixiJS
- **Modders:** Developers who want to add content (guns, obstacles, buildings, game modes) via the definitions system

## Key Features

- **Browser-based** — No install required; runs on desktop and mobile
- **Binary protocol** — Custom `SuroiByteStream` format for low-latency network communication
- **40 TPS server** — High-tick-rate server running on the Bun runtime
- **Definitions-driven content** — All game objects defined in TypeScript; adding content doesn't require touching engine code
- **Plugin system** — Event-based server plugins for custom game modes and server-side behavior
- **Multi-game support** — Server runs multiple concurrent game instances via Node cluster
- **i18n** — HJSON-based translations for multiple languages
- **Open source** — GPL-3.0 license; community-driven development

## Technology Stack

| Layer | Technology |
|-------|------------|
| Runtime | [Bun](https://bun.sh) |
| Language | TypeScript (strict mode) |
| Client rendering | [PixiJS v8](https://pixijs.com) |
| Client UI | [Svelte 5](https://svelte.dev) |
| Client bundler | [Vite](https://vitejs.dev) |
| Client styling | SCSS |
| Linter | [Biome](https://biomejs.dev) |
| WebSockets | Bun native WebSocket |
| Multi-process | Node `cluster` module |

## Documentation Index

See [content-plan.md](content-plan.md) for the full documentation index and status.

## Related Documents

- **Architecture:** [System Architecture](architecture.md) — System design and component map
- **Data Model:** [Game Data Model](datamodel.md) — Game objects, categories, and relationships
- **Development:** [Development Guide](development.md) — Setup and contribution guide
- **Protocol:** [Protocol Reference](protocol.md) — Network protocol reference

## Known Issues / Tech Debt

None documented. Add items here as they are discovered.
