# Development Guide

<!-- @tier: 1 -->
<!-- @see-also: docs/architecture.md -->

## Overview

This guide covers how to set up a local development environment, run the game, build for production, execute tests, and follow the project's code standards. Suroi is a Bun monorepo with four packages: `client`, `common`, `server`, and `tests`.

## Prerequisites

- [Bun](https://bun.sh) — the project's runtime, package manager, and test runner (required for all commands)
- [Git](https://git-scm.com/) — to clone the repository
- A modern browser for local play (navigate to `http://127.0.0.1:3000`)

No specific minimum Bun version is declared in `package.json`. Install the latest stable release.

## Setup

### Clone and Install

```bash
git clone https://github.com/HasangerGames/suroi.git
cd suroi
bun install
```

`bun install` resolves all workspace dependencies in one pass. The `bunfig.toml` configures the linker as `hoisted` (see [Bun Configuration](#bun-configuration)).

### Server Configuration

Copy `server/config.example.json` to `server/config.json` and edit as needed:

```bash
cp server/config.example.json server/config.json
```

The server reads `server/config.json` at startup. Required fields (from `config.schema.json`):

| Option | Type | Default (example) | Description |
|--------|------|-------------------|-------------|
| `$schema` | string | `"config.schema.json"` | JSON Schema reference |
| `hostname` | string | `"127.0.0.1"` | Hostname to bind the server to |
| `port` | number (0–65535) | `8000` | Main server port; game servers occupy consecutive ports (8001, 8002, …) |
| `map` | string \| rotation object | `"normal"` | Map name from `server/src/data/maps.ts`, or a `{ rotation, cron }` schedule. Parameters can be appended with colons (e.g. `"singleObstacle:rock"`) |
| `teamMode` | `"solo"` \| `"duo"` \| `"squad"` \| rotation object | `"solo"` | Team mode, or a `{ rotation, cron }` schedule |
| `maxPlayersPerGame` | number (≥1) | `80` | Maximum players per game instance |
| `maxGames` | number (≥1) | `5` | Maximum concurrent game instances |

Optional fields from `config.schema.json`:

| Option | Description |
|--------|-------------|
| `spawn.mode` | `"default"` \| `"random"` \| `"fixed"` — controls where players spawn |
| `spawn.position` | `[x, y]` or `[x, y, z]` — used when `spawn.mode` is `"fixed"` |
| `spawn.radius` | Spawn circle radius for `"fixed"` mode |
| `gas.disabled` | Set `true` to disable the gas zone entirely |
| `gas.forcePosition` | Force gas to shrink to a specific `[x, y]` or to the map center (`true`) |
| `gas.forceDuration` | Override gas stage duration in seconds |
| `minTeamsToStart` | Minimum teams (or players in solo) required to start a game |
| `tps` | Overrides `GameConstants.tps` (default 40) |
| `plugins` | List of plugin filenames (without extension) from `server/src/plugins/` to load |
| `apiServer` | `{ url, apiKey, reportWebhookUrl? }` — external API server integration |
| `ipHeader` | HTTP header for real IP behind a proxy (e.g. `"X-Real-IP"`, `"CF-Connecting-IP"`) |
| `maxSimultaneousConnections` | Per-IP connection limit |
| `maxJoinAttempts` | Rate-limit join attempts: `{ count, duration }` (duration in ms) |
| `maxCustomTeams` | Max custom teams creatable per IP simultaneously |
| `mapScaleRanges` | Dynamic map/game scaling by player count: array of `{ minPlayers, maxPlayers, scale, maxMajorBuildings?, gameSpawnWindow? }` |
| `usernameFilters` | Array of regex strings — matching usernames are replaced with the default |
| `roles` | Map of role name → `{ password, isDev? }` for dev/contributor access |
| `allowLobbyClearing` | Allow dev cheats via lobby clearing |
| `disableBuildingCheck` | Allow scopes/flares inside buildings |

**Roles and dev cheats:** Append `?password=PASSWORD&role=ROLE` to the URL to activate a role. Add `&lobbyClearing=true` (and set `allowLobbyClearing: true` in config) to enable dev cheats.

## Development Commands

### Root (run from workspace root `/`)

All day-to-day commands are run from the workspace root.

| Command | Script value | What it does |
|---------|-------------|--------------|
| `bun dev` | `bun -F '*' dev` | Starts all workspace packages' `dev` scripts in parallel (client Vite dev server + server hot-reload) |
| `bun dev:client` | `cd client && bun dev` | Starts only the Vite client dev server |
| `bun dev:server` | `cd server && bun dev` | Starts only the game server with hot-reload (`NODE_ENV=development`) |
| `bun dev:test` | `cd tests && bun stressTest` | Runs the WebSocket stress test with nodemon watch |
| `bun watch:server` | `cd server && bunx --bun tsc --noEmit --watch` | TypeScript type-checking for server in watch mode (no emit) |
| `bun build:client` | `cd client && bun build:client` | Runs the Vite production build for the client |
| `bun start` | `cd server && bun .` | Starts the game server in production mode |
| `bun lint` | `bunx --bun biome lint --fix` | Runs Biome linter across all packages and auto-fixes issues |
| `bun lint:check` | `bunx --bun biome lint` | Runs Biome linter in check-only mode (no auto-fix, exits non-zero on findings) |
| `bun test` | `bun test` | Runs the Bun test suite (unit tests at project root) |
| `bun validateDefinitions` | `cd tests && bun validateDefinitions` | Validates all `ObjectDefinitions` registries |
| `bun validateSvgs` | `cd tests && bun validateSvgs` | Validates SVG assets |
| `bun updateConfigSchema` | `cd server && bun updateConfigSchema` | Regenerates `server/src/utils/config.d.ts` from `config.schema.json` |
| `bun fullReinstall` | `rm -r bun.lock node_modules server/node_modules && bun install` | Full clean reinstall on Unix/macOS |
| `bun fullReinstallWin` | `del /f /s /q bun.lock node_modules\* server\node_modules\* & bun install` | Full clean reinstall on Windows |

After starting `bun dev`, open **http://127.0.0.1:3000** in a browser.

### Client (run from `client/`)

| Command | Script value | What it does |
|---------|-------------|--------------|
| `bun dev` | `vite` | Starts the Vite dev server with HMR |
| `bun build:client` | `bunx --bun vite build` | Builds the client for production output to `client/dist/` |

### Server (run from `server/`)

| Command | Script value | What it does |
|---------|-------------|--------------|
| `bun dev` | `NODE_ENV=development bun --hot src/server.ts` | Starts the server in development mode with Bun hot-reload |
| `bun updateConfigSchema` | `json2ts -i config.schema.json -o src/utils/config.d.ts` | Generates TypeScript types from `config.schema.json` using `json-schema-to-typescript` |

### Tests (run from `tests/`)

| Command | Script value | What it does |
|---------|-------------|--------------|
| `bun mathUnitTests` | `cd src && jest` | Runs Jest unit tests (math, collision) |
| `bun validateDefinitions` | `bun src/validateDefinitions.ts -print-top -print-bottom` | Validates all game object definition registries |
| `bun validateSvgs` | `bun src/validateSvgs.ts` | Validates SVG assets |
| `bun stressTest:start` | `bun src/stressTest.ts` | Runs the WebSocket stress test once |
| `bun stressTest` | `nodemon -r ts-node/register -r tsconfig-paths/register --watch ./src --watch ../common src/stressTest.ts` | Runs the stress test with nodemon auto-restart on file changes |

## Testing

### Test Framework

Two test systems coexist:

- **Jest 29 + ts-jest** — unit tests in `tests/src/math.test.ts`. Run via `bun mathUnitTests` (invokes `jest` from `tests/`).
- **Bun test** — runs from the workspace root via `bun test`.

### Running Tests

```bash
# Unit tests (math / collision) via Jest
bun run mathUnitTests   # from tests/ directory
# or from root:
cd tests && bun mathUnitTests

# Validate all ObjectDefinitions registries
bun validateDefinitions

# Validate SVG assets
bun validateSvgs

# Bun native test suite (root)
bun test
```

### Test Files

| File | Purpose |
|------|---------|
| `tests/src/math.test.ts` | Jest unit tests for collision math and geometry utilities |
| `tests/src/validateDefinitions.ts` | Validates all `ObjectDefinitions` registries for consistency |
| `tests/src/validateSvgs.ts` | Validates SVG game assets |
| `tests/src/stressTest.ts` | WebSocket stress test — simulates many simultaneous clients |
| `tests/src/validationUtils.ts` | Shared helpers used by the validation scripts |

## Code Standards

### Linter / Formatter

[Biome](https://biomejs.dev) **2.2.4** replaces both ESLint and Prettier. It lints and formats all TypeScript, JavaScript, and Svelte source files (excluding `*/dist/**`).

```bash
bun lint        # auto-fix all issues
bun lint:check  # report issues without modifying files
```

### Enforced Rules

Rules are grouped by Biome category. `recommended: false` is set — only the rules listed below are active.

#### complexity

| Rule | Level | What it enforces |
|------|-------|-----------------|
| `noAdjacentSpacesInRegex` | error | No double spaces inside regex literals |
| `noExtraBooleanCast` | error | No redundant `Boolean()` or `!!` casts |
| `noStaticOnlyClass` | error | No classes with only static members (use plain objects/modules) |
| `noUselessCatch` | error | No catch blocks that only re-throw |
| `noUselessConstructor` | error | No constructor that merely calls `super()` with the same args |
| `noUselessEscapeInRegex` | error | No needless escape sequences in regex |
| `noUselessThisAlias` | error | No `const self = this` aliasing |
| `noUselessTypeConstraint` | error | No `T extends any` or `T extends unknown` constraints |
| `useArrowFunction` | **warn** | Prefer arrow functions over `function` expressions |
| `useLiteralKeys` | error | Use dot notation instead of `obj["key"]` where possible |
| `useOptionalChain` | error | Replace `a && a.b` with `a?.b` |

#### correctness

| Rule | Level | What it enforces |
|------|-------|-----------------|
| `noConstAssign` | error | No reassignment of `const` |
| `noConstantCondition` | error | No always-true/false conditions in `if`/`while` |
| `noEmptyCharacterClassInRegex` | error | No `[]` in regex (always matches nothing) |
| `noEmptyPattern` | error | No empty destructuring patterns |
| `noGlobalObjectCalls` | error | No calling `Math()`, `JSON()` etc. as constructors |
| `noInvalidBuiltinInstantiation` | error | No `new Symbol()` or `new BigInt()` |
| `noInvalidConstructorSuper` | error | `super()` must be called correctly |
| `noNonoctalDecimalEscape` | error | No `\8` or `\9` in strings |
| `noPrecisionLoss` | error | No numeric literals that lose precision |
| `noSelfAssign` | error | No `x = x` assignments |
| `noSetterReturn` | error | Setters must not return a value |
| `noSwitchDeclarations` | error | No `let`/`const`/`function` declarations directly inside `switch` cases |
| `noUndeclaredVariables` | error | No use of undeclared variables |
| `noUnreachable` | error | No unreachable code after `return`/`throw` |
| `noUnreachableSuper` | error | `super()` must be reachable in all constructor paths |
| `noUnsafeFinally` | error | No control-flow statements (`return`/`throw`) inside `finally` |
| `noUnsafeOptionalChaining` | error | No `(a?.b).c` — unsafe dereference of optional chain result |
| `noUnusedLabels` | error | No unused `label:` statements |
| `noUnusedPrivateClassMembers` | error | No private class members that are never read |
| `noUnusedVariables` | error | No declared variables that are never used |
| `useIsNan` | error | Use `Number.isNaN()` instead of `=== NaN` |
| `useValidForDirection` | error | `for` loop condition must make progress |
| `useValidTypeof` | error | `typeof x === "..."` comparisons must use valid type strings |
| `useYield` | error | Generator functions must contain at least one `yield` |

#### nursery

| Rule | Level | What it enforces |
|------|-------|-----------------|
| `noFloatingPromises` | error | Promises must be `await`-ed or `.catch()`-ed |
| `noMisusedPromises` | error | No returning `Promise` where a non-Promise is expected |
| `noNonNullAssertedOptionalChain` | error | No `a?.b!` — optional chain result non-null assertion |
| `useConsistentTypeDefinitions` | error | Use `interface` for object shapes, not `type` aliases |

#### style

| Rule | Level | What it enforces |
|------|-------|-----------------|
| `noCommonJs` | error | ESM only — no `require()` or `module.exports` |
| `noInferrableTypes` | error | No type annotations TypeScript can infer (e.g. `const x: number = 1`) |
| `noNamespace` | error | No TypeScript `namespace` declarations |
| `noNonNullAssertion` | error | No `!` non-null assertions |
| `noYodaExpression` | error | No yoda conditions (`1 === x`) |
| `useArrayLiterals` | error | Use `[]` instead of `new Array()` |
| `useAsConstAssertion` | error | Use `as const` instead of casting to explicit tuple/literal types |
| `useConsistentArrayType` | **warn** | Consistent array type syntax (`T[]` or `Array<T>`) |
| `useForOf` | error | Prefer `for...of` over indexed `for` loops when index is not used |
| `useLiteralEnumMembers` | error | Enum members must have literal (not computed) values |
| `useReadonlyClassProperties` | error | Class properties that are never reassigned must be `readonly` |
| `useShorthandFunctionType` | error | Use `() => T` shorthand instead of `{ (): T }` interface call signatures |
| `useTemplate` | **warn** | Prefer template literals over string concatenation |
| `useThrowOnlyError` | error | Only `Error` instances (or subclasses) may be thrown |
| `useUnifiedTypeSignatures` | error | Merge overloaded function signatures into one where possible |

#### suspicious

| Rule | Level | What it enforces |
|------|-------|-----------------|
| `noAsyncPromiseExecutor` | error | No `async` functions as `new Promise()` executors |
| `noCatchAssign` | error | No reassigning `catch` binding variables |
| `noClassAssign` | error | No reassigning class names |
| `noCompareNegZero` | error | No `x === -0` comparisons |
| `noConfusingVoidType` | error | No `void` in union types where `undefined` is meant |
| `noConstantBinaryExpressions` | error | No always-true/false binary expressions |
| `noControlCharactersInRegex` | error | No raw control characters in regex literals |
| `noDebugger` | error | No `debugger` statements |
| `noDoubleEquals` | error | Strict equality only (`===`, not `==`) |
| `noDuplicateCase` | error | No duplicate `case` labels in `switch` |
| `noDuplicateClassMembers` | error | No duplicate class member names |
| `noDuplicateElseIf` | error | No identical `else if` conditions |
| `noDuplicateObjectKeys` | error | No duplicate keys in object literals |
| `noDuplicateParameters` | error | No duplicate parameter names |
| `noEmptyBlockStatements` | error | No empty `{}` blocks |
| `noExplicitAny` | error | No `any` type annotations |
| `noExtraNonNullAssertion` | error | No `!!` double non-null assertions |
| `noFallthroughSwitchClause` | error | All `switch` cases must `break`/`return`/`throw` |
| `noFunctionAssign` | error | No reassigning function declarations |
| `noGlobalAssign` | error | No reassigning built-in globals |
| `noImportAssign` | error | No reassigning imported bindings |
| `noIrregularWhitespace` | error | No unusual whitespace characters outside strings |
| `noMisleadingCharacterClass` | error | No multi-codepoint characters inside `[...]` character classes |
| `noMisleadingInstantiator` | error | `new` and `constructor` must only appear where they make sense |
| `noPrototypeBuiltins` | error | No calling `obj.hasOwnProperty()` etc. directly |
| `noRedeclare` | error | No re-declaring variables in same scope |
| `noShadowRestrictedNames` | error | No shadowing of `undefined`, `NaN`, `Infinity`, etc. |
| `noSparseArray` | error | No sparse arrays (`[1,,3]`) |
| `noTsIgnore` | error | No `@ts-ignore` comments (use `@ts-expect-error` with a reason) |
| `noUnsafeDeclarationMerging` | error | No unsafe declaration merging (interface + class) |
| `noUnsafeNegation` | error | No `!x in y` or `!x instanceof Y` |
| `noUselessRegexBackrefs` | error | No backreferences to groups that don't exist |
| `noWith` | error | No `with` statements |
| `useAdjacentOverloadSignatures` | error | Overloaded function signatures must be adjacent |
| `useAwait` | error | `async` functions must contain at least one `await` |
| `useGetterReturn` | error | Getters must always return a value |
| `useNamespaceKeyword` | error | Use `namespace` keyword instead of `module` for TS namespaces |

---

## Known Issues & Gotchas

- **Biome enforces non-null assertion ban:** `noNonNullAssertion: "error"` means the `!` operator is **completely forbidden** in all TypeScript files. Use optional chaining (`var?.prop`) + nullish coalescing (`var ?? default`) instead. Refactoring away `!` is tedious but mandatory for commit.
- **No `any` type allowed:** `noExplicitAny: "error"` bans `any` entirely. Must use proper typing or `unknown` with type guards. `unknown` requires cast-checking before use.
- **TPS and grid size can be overridden:** `tps` and `gridSize` are configurable at runtime via `config.json` (see `[tps]` and `maxSimultaneousConnections` above). Changed values affect all tick-loop timing and spatial hash performance. Server restart required for changes.
- **Config schema auto-generation:** `bun updateConfigSchema` regenerates `config.d.ts` from `config.schema.json`. If you modify the schema, you **must** run this command; the TypeScript types will be stale otherwise.
- **Worker vs primary process split:** Game logic runs in worker processes. The primary process only handles `/team` WebSocket and HTTP API. If you add server-side code, be careful about which process it runs in.
- **Hot-reload limitations:** Server hot-reload (`bun dev:server`) rebuilds the primary process entry file, but if worker logic changes, you must manually restart workers. There is no automatic worker hot-reload.

## Dependencies on This Document

This document provides the **setup and workflow foundation** for all development on suroi:

- [Game Loop](docs/subsystems/game-loop/) — Developers must understand tick loop run `bun dev:server` to test
- [Networking](docs/subsystems/networking/) — Developers test WebSocket via `bun dev` and validate packets with `bun validateDefinitions`
- [Object Definitions](docs/subsystems/object-definitions/) — Changes require validation via `bun validateDefinitions` and schema updates
- [Spatial Grid](docs/subsystems/spatial-grid/) — Developers test collision via stress test: `bun dev:test`
- [Map Generation](docs/subsystems/map/) — Map changes tested via dev server with config overrides
- [Client Rendering](docs/subsystems/client-rendering/) — Client code built/served via `bun dev:client`; Vite HMR live-updates
- [Inventory](docs/subsystems/inventory/) — Inventory logic tested via `bun dev` with game client
- [Plugins](docs/subsystems/plugins/) — Plugins loaded from `server/src/plugins/` as declared in config

## Related Documents

### Tier 1 — Architecture & High-Level
- [Project Description](description.md) — What suroi is and its features
- [System Architecture](architecture.md) — Tech stack, process model, build plugins
- [Data Model](datamodel.md) — Core types and GameConstants
- [API Reference](api-reference.md) — Packet protocol for testing

### Tier 2 — Subsystems
- [Game Loop](docs/subsystems/game-loop/) — Tick loop development and testing
- [Networking](docs/subsystems/networking/) — Packet validation and WebSocket testing
- [Object Definitions](docs/subsystems/object-definitions/) — Definition validation and schema
- [Spatial Grid](docs/subsystems/spatial-grid/) — Collision testing
- [Map Generation](docs/subsystems/map/) — Map development and testing
- [Gas System](docs/subsystems/gas/) — Gas zone development
- [Client Rendering](docs/subsystems/client-rendering/) — Client build and Vite configuration
- [Inventory](docs/subsystems/inventory/) — Inventory system
- [Plugins](docs/subsystems/plugins/) — Plugin development and testing

#### TypeScript-file overrides (applied to `*.ts`, `*.tsx`, `*.mts`, `*.cts`)

| Rule | Level | What it enforces |
|------|-------|-----------------|
| `style.useConst` | error | Prefer `const` over `let` wherever the binding is never reassigned |
| `suspicious.noVar` | error | No `var` declarations |

#### Svelte/Astro/Vue overrides (applied to `*.svelte`, `*.astro`, `*.vue`)

`useConst`, `useImportType`, `noUnusedVariables`, `noUnusedImports`, `noUndeclaredVariables`, and `noSelfAssign` are all turned **off** in these files to accommodate framework-specific patterns.

### TypeScript Configuration

The root `tsconfig.json` extends the three package configs and adds the workspace-wide `@common/*` path alias:

```json
{
  "extends": ["./client/tsconfig.node.json", "./common/tsconfig.json", "./server/tsconfig.json"],
  "include": ["package.json", "client/src", "common/src", "server/src"],
  "compilerOptions": {
    "paths": { "@common/*": ["./common/src/*"] }
  }
}
```

Per-package compiler options:

| Setting | client | common | server | node (vite config) |
|---------|--------|--------|--------|--------------------|
| `target` | `ESNext` | `ES2022` | `esnext` | — |
| `module` | `ESNext` | `CommonJS` | `esnext` | `ESNext` |
| `moduleResolution` | `Bundler` | `node` | `node` | `Bundler` |
| `strict` | (via `@tsconfig/svelte`) | `true` | `true` | `true` |
| `strictNullChecks` | (via `@tsconfig/svelte`) | `true` | `true` | `true` |
| `composite` | — | — | — | `true` |
| `skipLibCheck` | — | — | `true` | `true` |
| `sourceMap` | — | — | `true` | — |
| `resolveJsonModule` | `true` | `true` | `true` | — |
| `allowJs` / `checkJs` | `true` / `true` | `true` | — | — |
| `isolatedModules` | `true` | — | — | — |
| `useDefineForClassFields` | `true` | — | — | — |
| `outDir` | `./dist` | `./dist` | `./dist` | — |

The `client` config extends [`@tsconfig/svelte`](https://github.com/tsconfig/bases#svelte) for Svelte-compatible settings, and includes `**/*.svelte` files.

### Path Aliases

| Alias | Resolves to | Used in |
|-------|-------------|---------|
| `@common/*` | `common/src/*` | `client`, `server`, root `tsconfig.json` |

### Code Style Conventions

Derived from Biome rules above — key conventions enforced automatically:

- **ESM only** — no `require()` or `module.exports` (`noCommonJs`)
- **No `!` non-null assertions** — use optional chaining or explicit null checks (`noNonNullAssertion`)
- **`interface` not `type` for object shapes** — (`useConsistentTypeDefinitions`)
- **`===` always** — never `==` (`noDoubleEquals`)
- **Optional chaining** — `a?.b?.c` not `a && a.b && a.b.c` (`useOptionalChain`)
- **Arrow functions** — prefer over `function` expressions where possible (`useArrowFunction`)
- **Template literals** — prefer over string concatenation (`useTemplate`)
- **`const` by default** — `let` only when rebinding is required; never `var` (`useConst`, `noVar`)
- **Readonly class properties** — add `readonly` to any property never reassigned (`useReadonlyClassProperties`)
- **No `any`** — use explicit types or generics (`noExplicitAny`)
- **No `@ts-ignore`** — use typed suppression with `@ts-expect-error` instead (`noTsIgnore`)
- **Await all promises** — no floating promises (`noFloatingPromises`)

## Bun Configuration

`bunfig.toml` (workspace root):

```toml
[install]
linker = "hoisted"
```

`linker = "hoisted"` places all `node_modules` at the workspace root rather than nesting them per-package. This is the npm-compatible layout required by Vite, PixiJS, and other tools that rely on top-level `node_modules` resolution.

## Workspace Structure

The project is a Bun workspace (`workspaces: ["client", "common", "server", "tests"]`):

| Package | Name | Role |
|---------|------|------|
| `client/` | `@suroi/client` | Browser client — PixiJS rendering, Svelte UI, Vite build |
| `common/` | _(unnamed)_ | Shared code — packet definitions, object definitions, math/hitbox utilities, constants |
| `server/` | `@suroi/server` | Game server — Bun WebSockets, game loop, map generation, gas, inventory |
| `tests/` | `@suroi/tests` | Test suite — Jest unit tests, definition validators, SVG validators, stress test |

See [docs/architecture.md](architecture.md) for the full system architecture.

## Related Documents

- **Tier 1:** [docs/architecture.md](architecture.md) — Full system architecture, component map, tech stack
- **Tier 1:** [docs/description.md](description.md) — Project overview and feature summary
- **Tier 1:** [docs/api-reference.md](api-reference.md) — Binary WebSocket packet protocol
- **Tier 2:** [docs/subsystems/networking/README.md](subsystems/networking/README.md) — Packet encoding details
- **Tier 2:** [docs/subsystems/object-definitions/README.md](subsystems/object-definitions/README.md) — `ObjectDefinitions` registry and validation
- **Tier 2:** [docs/subsystems/plugins/README.md](subsystems/plugins/README.md) — Plugin system and `config.plugins`
