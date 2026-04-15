# Documentation Content Plan

## Overview

Suroi is an open-source 2D battle royale game inspired by surviv.io. It is a multiplayer browser game with a client-server architecture using WebSockets and binary protocol serialization.

## Documentation Index

| # | Module / Area | Tier | Status | Path | Priority | Last Updated |
|---|---------------|------|--------|------|----------|--------------|
| 1 | Architecture Overview | T1 | Done | [docs/architecture.md](architecture.md) | Critical | 2026-04-09 |
| 2 | Game Data Model | T1 | Done | [docs/datamodel.md](datamodel.md) | Critical | 2026-04-09 |
| 3 | Development Guide | T1 | Done | [docs/development.md](development.md) | Critical | 2026-04-09 |
| 4 | Protocol Reference | T1 | Done | [docs/protocol.md](protocol.md) | Critical | 2026-04-09 |
| 5 | Project Description | T1 | Done | [docs/description.md](description.md) | Critical | 2026-04-09 |
| 6 | Definitions Subsystem | T2 | Not Started | docs/subsystems/definitions/ | Critical | — |
| 7 | Packets Subsystem | T2 | Not Started | docs/subsystems/packets/ | Critical | — |
| 8 | Object Model Subsystem | T2 | Not Started | docs/subsystems/objects/ | Critical | — |
| 9 | Game Loop Subsystem | T2 | Not Started | docs/subsystems/game-loop/ | High | — |
| 10 | Inventory Subsystem | T2 | Not Started | docs/subsystems/inventory/ | High | — |
| 11 | Map Generation Subsystem | T2 | In Progress | docs/subsystems/map/ | High | 2026-04-09 |
| 12 | Rendering Subsystem | T2 | Not Started | docs/subsystems/rendering/ | High | — |
| 13 | Input & Controls Subsystem | T2 | Not Started | docs/subsystems/input/ | Medium | — |
| 14 | Gas System Subsystem | T2 | Not Started | docs/subsystems/gas/ | Medium | — |
| 15 | Plugin System Subsystem | T2 | Done | docs/subsystems/plugins/ | Medium | 2026-04-09 |
| 16 | UI & Translations Subsystem | T2 | Not Started | docs/subsystems/ui/ | Medium | — |

## Status Legend

- **Done** — Written, reviewed, up-to-date with current code
- **In Progress** — Draft exists, may be incomplete or outdated
- **Not Started** — Needs to be written
- **Outdated** — Exists but doesn't match current code

## Generation Order

Generate documentation in dependency order:

1. **Tier 1 first** — Architecture, data model, development guide, protocol reference, project description
2. **Core Tier 2 next** — Definitions, Packets, Object Model (everything depends on these)
3. **Server Tier 2** — Game Loop, Inventory, Map Generation, Gas, Plugins
4. **Client Tier 2** — Rendering, Input, UI & Translations
5. **Tier 3 last** — Module docs within each subsystem (ordered by complexity)

## What Each Document Must Include

- Purpose and scope
- Key files and entry points (with `@file` references to source code)
- Data flow (how data moves through the module)
- Dependencies on other modules (with links to their Tier 2/3 docs)
- Cross-tier references ("See also" links up and down the tiers)
- Protocol version implications (when changes require `GameConstants.protocolVersion` bump)
- Known issues, tech debt, or gotchas
