# Development Guide

<!-- @tier: 1 -->
<!-- @see-also: docs/architecture.md, docs/description.md -->
<!-- @source: package.json, biome.json, server/config.example.json -->
<!-- @updated: 2026-03-04 -->

## Overview

This guide covers local setup, available commands, code standards, and the workflows for contributing to Suroi.

**Documentation index:** See [content-plan.md](content-plan.md) for the full documentation index and status.

## Prerequisites

| Tool | Required Version | Purpose |
|------|-----------------|---------|
| [Bun](https://bun.sh) | Latest stable | Runtime, package manager, test runner |
| Node.js | Not required (Bun handles this) | — |
| Git | Any | Version control |

No database, Docker, or external services are required for local development.

## Initial Setup

```bash
# 1. Clone the repository
git clone https://github.com/HasangerGames/suroi.git
cd suroi

# 2. Install all dependencies (monorepo)
bun install

# 3. Configure the server
cp server/config.example.json server/config.json
# Edit server/config.json if needed (defaults work for local dev)
```

The server will not start without `server/config.json`.

## Development Commands

All commands are run from the **workspace root**.

### Starting the Game

```bash
# Start everything (client on :3000, server on :8000)
bun dev

# Client only (Vite dev server)
bun dev:client

# Server only
bun dev:server
```

The client hot-reloads on file changes. The server uses `bun --hot` for hot module reloading.

### Building for Production

```bash
# Build client bundle
bun build:client

# Start production server
bun start
```

### Linting

```bash
# Lint and auto-fix all files (Biome)
bun lint

# Lint check only (no changes — used in CI)
bun lint:check
```

### Validation

```bash
# Validate all game definitions
bun validateDefinitions

# Validate all SVG assets
bun validateSvgs
```

Run `bun validateDefinitions` any time you add or modify definitions in `common/src/definitions/`.

### Type Checking

```bash
# TypeScript type check (server package, watch mode)
bun watch:server
```

This runs `tsc --noEmit --watch` on the server package. For client type checking, Vite handles it during `bun dev:client`.

### Other

```bash
# Full reinstall (clears lock file and node_modules)
bun fullReinstall

# Run stress test
cd tests && bun stressTest
```

## Adding New Content

Most new game content is added by:

1. **Adding a definition** in `common/src/definitions/`
2. **Registering it** in the relevant `ObjectDefinitions` collection
3. **Bumping the protocol version** in `common/src/constants.ts`
4. **Adding server logic** in `server/src/objects/` or `server/src/inventory/`
5. **Adding client rendering** in `client/src/scripts/objects/`
6. **Adding assets** (sprites in `client/public/img/game/`)

See [docs/architecture.md](architecture.md) for a full map of where code lives.

### Protocol Version

Increment `GameConstants.protocolVersion` in `common/src/constants.ts` whenever:
- A packet format changes (fields added, removed, or reordered)
- A definition list changes (items added or removed — because indices shift)
- Any `SuroiByteStream` encoding changes

```typescript
// common/src/constants.ts
export const GameConstants = {
    protocolVersion: 73,  // ← bump this
    ...
};
```

Mismatched protocol versions prevent clients from connecting.

## Code Standards

### Linter: Biome

The project uses [Biome](https://biomejs.dev) (`biome.json` in root) with strict rules. Key enforced rules:

| Rule | Level | Description |
|------|-------|-------------|
| `noExplicitAny` | error | No `any` types |
| `noNonNullAssertion` | error | No `!` non-null assertions |
| `noVar` | error | Use `const`/`let` only |
| `useConst` | error | Prefer `const` |
| `useReadonlyClassProperties` | error | Class properties must be `readonly` by default |
| `noFloatingPromises` | error | All promises must be awaited or handled |
| `noMisusedPromises` | error | No promises in non-async contexts |
| `noTsIgnore` | error | No `@ts-ignore` comments |
| `noCommonJs` | error | ESM only (`import`/`export`) |
| `useConsistentTypeDefinitions` | error | Consistent `interface` vs `type` |

### TypeScript

- **Strict mode** is enabled across all packages (`tsconfig.json`)
- Use `interface` for object shapes, `type` for unions/intersections
- No `any` — use `unknown` and narrow, or proper generics
- No `!` — use optional chaining (`?.`) or explicit checks

### Naming Conventions

| Entity | Convention | Example |
|--------|------------|---------|
| Files | kebab-case | `game-object.ts`, `input-manager.ts` |
| Classes | PascalCase | `BaseGameObject`, `PacketStream` |
| Enums | PascalCase | `ObjectCategory`, `PacketType` |
| Enum members | PascalCase | `ObjectCategory.Player` |
| Constants | camelCase | `GameConstants`, `protocolVersion` |
| Functions | camelCase | `writePosition()`, `fromString()` |

### File Organization

- **`common/`** — Only code that both client and server need. No platform-specific imports.
- **`server/`** — Server-side only. Can import from `@common/*`, not from `client/`.
- **`client/`** — Client-side only. Can import from `@common/*`, not from `server/`.
- **`tests/`** — Validators and stress tests. Can import from all packages.

### Path Aliases

```typescript
// Use @common/* instead of relative paths crossing packages
import { GameConstants } from "@common/constants";          // ✓
import { GameConstants } from "../../common/src/constants"; // ✗
```

## Project Structure Cheatsheet

```
Where to put new code:
  New game object type?
    → Definition:  common/src/definitions/
    → Server obj:  server/src/objects/
    → Client obj:  client/src/scripts/objects/

  New packet field?
    → common/src/packets/<packet>.ts  +  bump protocolVersion

  New server mechanic (no new object)?
    → server/src/game.ts  or  server/src/objects/player.ts

  New UI element?
    → client/src/scripts/managers/uiManager.ts  +  Svelte component

  New sound?
    → client/public/ (audio asset)  +  client/src/scripts/managers/soundManager.ts

  New map?
    → server/src/data/maps.ts  +  client/public/img/game/<variant>/

  New plugin?
    → server/src/plugins/<name>.ts  +  register in server/src/server.ts
```

## Testing

### Definition Validation

```bash
bun validateDefinitions
```

Runs `tests/src/validateDefinitions.ts`, which checks:
- All `idString` values are unique within their collection
- Required fields are present
- Cross-references (e.g., a gun's bullet type) point to valid definitions
- No circular references or other structural issues

### SVG Validation

```bash
bun validateSvgs
```

Runs `tests/src/validateSvgs.ts`, which checks that all SVG assets referenced by definitions exist and are well-formed.

### Stress Test

```bash
cd tests && bun stressTest
```

Runs `tests/src/stressTest.ts` to verify server performance under load.

### Unit Tests

Math utilities have unit tests in `tests/src/math.test.ts`, run with:

```bash
cd tests && bun test
```

## Common Issues

See [AGENTS.md — Learnings](../AGENTS.md#learnings) for solutions to common errors including:
- Protocol version mismatch (client can't connect)
- Definition validation failure
- Object not rendering on client
- `@common/*` import alias not resolving

## CI

GitHub Actions runs `bun lint:check`, `bun validateDefinitions`, and `bun validateSvgs` on every pull request. All must pass before merging.

## Known Issues / Tech Debt

- **No E2E tests:** Manual playtesting is the primary verification for gameplay changes.
- **Bun-specific:** Server requires Bun. No Node.js fallback.

## Related Documents

- **Tier 1:** [architecture.md](architecture.md) — System design
- **Tier 1:** [datamodel.md](datamodel.md) — Game entities and constants
- **Tier 1:** [protocol.md](protocol.md) — Network protocol
- **Tier 2:** [subsystems/definitions/](subsystems/definitions/) — How to write definitions
