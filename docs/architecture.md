# System Architecture

<!-- @tier: 1 -->
<!-- @see-also: docs/subsystems/ -->

## Overview

This document describes the runtime architecture, monorepo layout, tech stack,
process model, build system, and key configuration of the suroi game server and
client. All values are extracted directly from the source files listed in the
_Key Sources_ column.

---

## Tech Stack

| Layer | Technology | Version | Key Source |
|-------|-----------|---------|-----------|
| Runtime | Bun (native) | `@types/bun ^1.3.4` | `server/package.json` |
| Language | TypeScript | `^5.9.3` | `package.json` (devDeps) |
| Client rendering | PixiJS | `^8.14.3` | `client/package.json` |
| Client audio | @pixi/sound | `^6.0.1` | `client/package.json` |
| Client UI | Svelte | `^5.46.0` | `client/package.json` |
| Client build | Vite | `^6.4.1` | `client/package.json` |
| Linter / formatter | Biome | `2.2.4` | `biome.json` (`$schema`) |
| Server scheduling | croner | `^9.1.0` | `server/package.json` |
| Test runner (unit) | Jest + ts-jest | `^29.7.0` / `^29.4.4` | `tests/package.json` |
| Test runner (scripts) | Bun test | native | `package.json` (`test` script) |
| CSS preprocessor | Sass | `^1.97.0` | `client/package.json` |
| Spritesheet packing | maxrects-packer | `^2.7.3` | `client/package.json` |
| Canvas (spritesheet gen) | skia-canvas | `^2.0.2` | `client/package.json` / `trustedDependencies` |

---

## Monorepo Structure

```
suroi/                          ← Bun workspace root (name: "suroi", v0.30.2)
├── client/                     ← @suroi/client — browser game client
│   ├── src/
│   │   ├── scripts/            ← TypeScript game logic (PixiJS rendering, input, UI)
│   │   │   ├── game.ts         ← PixiJS Application entry, ObjectPool, client game state
│   │   │   ├── ui.ts           ← HUD setup and DOM event wiring
│   │   │   ├── managers/       ← Camera, gas, input, map, particle, sound, UI managers
│   │   │   ├── objects/        ← Renderable game objects (Player, Obstacle, Loot, …)
│   │   │   ├── console/        ← In-game developer console
│   │   │   └── utils/          ← Client-side utilities (pixi, tween, graph, translations)
│   │   ├── scss/               ← Stylesheets (Sass)
│   │   └── translations/       ← HJSON locale files
│   ├── vite/                   ← Vite config split (vite.common.ts, vite.dev.ts, vite.prod.ts)
│   │   └── plugins/            ← Custom Vite plugins (spritesheet, audio, news, translations)
│   ├── public/                 ← Static assets (audio, images, manifest)
│   ├── index.html              ← Main game entry HTML
│   └── vite.config.ts          ← Vite root config; dev vs prod dynamic import
│
├── common/                     ← @suroi/common — shared types, packets, definitions, math
│   └── src/
│       ├── constants.ts        ← GameConstants (tps, gridSize, protocolVersion, …)
│       ├── definitions/        ← ObjectDefinitions registries (bullets, guns, obstacles, …)
│       ├── packets/            ← Binary packet definitions and PacketStream
│       └── utils/              ← SuroiByteStream, hitbox, math, vector, objectPool, …
│
├── server/                     ← @suroi/server — authoritative game server
│   └── src/
│       ├── server.ts           ← Primary process entry: Bun.serve (HTTP + /team WS)
│       ├── gameManager.ts      ← Cluster orchestration + worker Bun.serve (/play WS)
│       ├── game.ts             ← Game class: tick loop, object sets, dirty tracking
│       ├── gas.ts              ← Gas (safe-zone) simulation
│       ├── map.ts              ← Map layout and object placement
│       ├── pluginManager.ts    ← Server-side plugin event bus
│       ├── inventory/          ← GunItem, MeleeItem, ThrowableItem
│       ├── objects/            ← Server-side game objects (Player, Obstacle, Bullet, …)
│       ├── data/               ← Static data (maps, gas stages)
│       └── utils/              ← Grid, IDAllocator, config, helpers
│
└── tests/                      ← @suroi/tests — unit tests + validation scripts
    └── src/
        ├── math.test.ts        ← Jest unit tests for common/src/utils/math.ts
        ├── validateDefinitions.ts ← Checks object definition consistency
        ├── validateSvgs.ts     ← Validates SVG assets
        └── stressTest.ts       ← Headless WebSocket load test
```

### Package Name Fields

| Directory | `name` field | Role |
|-----------|-------------|------|
| `client/` | `@suroi/client` | Browser game client |
| `common/` | `@suroi/common` | Shared types, packets, math |
| `server/` | `@suroi/server` | Authoritative game server |
| `tests/` | `@suroi/tests` | Test suite |
| _(root)_ | `suroi` | Bun workspace root |

---

## Component Map

```
Browser
  │
  │  HTTP GET /api/serverInfo   → game info (protocolVersion, playerCount, mode…)
  │  HTTP GET /api/getGame      → assigns gameID
  │  WS  /team                  → custom-team lobby (JSON messages)
  │                                port = Config.port
  ▼
┌──────────────────────────────────────────────────────┐
│  Primary Process  (server.ts)                        │
│  Bun.serve on Config.hostname:Config.port            │
│  Routes: /api/serverInfo, /api/getGame, /team (WS)   │
│                                                      │
│  GameManager                                         │
│  ├── games[]: GameContainer[] (one per active game)  │
│  ├── Switcher<TeamMode>  — cron-scheduled rotation   │
│  └── Switcher<string>    — map rotation              │
└──────────┬───────────────────────────────────────────┘
           │  Cluster.fork({ id, teamMode, map, mapScaleRange })
           │  IPC: WorkerMessages (UpdateTeamMode | UpdateMap |
           │                       UpdateMapOptions | NewGame)
           ▼
┌──────────────────────────────────────────────────────┐
│  Worker Process  (gameManager.ts — !Cluster.isPrimary)│
│  One worker per active game                          │
│  Bun.serve on Config.hostname : Config.port + id + 1 │
│  Route: /play  → WebSocket upgrade                   │
│                                                      │
│  Game instance (game.ts)                             │
│  ├── Grid (spatial hash)                             │
│  ├── GameMap                                         │
│  ├── Gas                                             │
│  ├── PluginManager                                   │
│  └── tick() loop (setTimeout self-rescheduling)      │
└──────────────────────────────────────────────────────┘
           ▲
           │  WS /play   (binary frames, SuroiByteStream)
           │  port = Config.port + gameID + 1
           │
         Browser (game client — client/src/scripts/game.ts)
```

### Process Details

| Process | Entry file | `Bun.serve` port | Responsible for |
|---------|-----------|-----------------|-----------------|
| Primary | `server/src/server.ts` | `Config.port` | HTTP API, `/team` WebSocket, spawning workers |
| Worker N | `server/src/gameManager.ts` (worker block) | `Config.port + N + 1` | Running one `Game` instance, `/play` WebSocket |

Workers are forked via `Cluster.fork()` with environment variables (`id`,
`teamMode`, `map`, `mapScaleRange`). Ongoing IPC uses `process.send()` /
`worker.on("message")` with typed `WorkerMessage` objects.

### Custom-Team Lobby

A separate WebSocket endpoint `/team` on the primary port carries JSON
`CustomTeamMessage` frames. Players negotiate a team and then receive a
`gameID` from `/api/getGame` before connecting to the game worker.

---

## Path Aliases

| Alias | Resolves to | Defined in |
|-------|-------------|-----------|
| `@common/*` | `./common/src/*` | `tsconfig.json` (`compilerOptions.paths`) |
| `@common` | `../../common/src` | `client/vite/vite.common.ts` (`resolve.alias`) |

The root `tsconfig.json` extends all three sub-package tsconfigs and adds the
`@common/*` path mapping. Vite needs its own alias entry because it processes
the client bundle independently of `tsc`.

---

## Build System

### Vite Configuration

Source: `client/vite.config.ts`, `client/vite/vite.common.ts`

The root `vite.config.ts` dynamically imports either `vite.dev.ts` or
`vite.prod.ts` depending on the `command` and `mode`. The shared base is
`vite/vite.common.ts`.

**Dev server:** port `3000`, host `0.0.0.0`.

**Multi-page output** (via `rollupOptions.input`):

| Entry key | HTML file |
|-----------|----------|
| `main` | `index.html` |
| `changelog` | `changelog/index.html` |
| `news` | `news/index.html` |
| `rules` | `rules/index.html` |
| `privacy` | `privacy/index.html` |
| `editor` | `editor/index.html` |

**Output asset routing:**

| Asset type | Output path pattern |
|-----------|-------------------|
| CSS | `styles/[name]-[hash].css` |
| Fonts (ttf, woff, woff2) | `fonts/[name]-[hash].[ext]` |
| Other assets | `assets/[name]-[hash][ext]` |
| Entry scripts | `scripts/[name]-[hash].js` |
| Chunk scripts | `scripts/[name]-[hash].js` |
| `node_modules` chunk | `scripts/vendor-[hash].js` |

**Chunk size warning limit:** 2000 KB. Source maps are enabled.

**Custom Vite plugins** (all in `client/vite/plugins/`):

| Plugin | Purpose |
|--------|---------|
| `imageSpritesheet()` | Packs SVG/PNG assets into WebGL spritesheets |
| `audioSpritesheet()` | Packs audio clips into a single audio spritesheet |
| `newsPosts()` | Transforms `client/src/newsPosts/` Markdown+front-matter into a virtual module |
| `translations()` | Bundles HJSON locale files into a typed virtual translations module |

**Additional plugins:** `@sveltejs/vite-plugin-svelte` (Svelte 5 compiler),
`vite-plugin-minify` (HTML minification).

**SCSS API:** `modern-compiler` (Sass `^1.97.0`).

**Compile-time defines:**

| Define | Value |
|--------|-------|
| `IS_CLIENT` | `true` |
| `APP_VERSION` | `pkg.version` (from root `package.json`) |
| `VITE_APP_VERSION` | same, via `process.env` |
| `DEBUG_CLIENT` | `"true"` in dev, `"false"` in prod |

### Bun Workspace Config

Source: `bunfig.toml`

```toml
[install]
linker = "hoisted"
```

Module resolution uses the hoisted strategy (all packages share one top-level
`node_modules`). The server workspace has its own `node_modules` for Bun
native types.

---

## Known Issues & Gotchas

- **Worker process overhead:** Each active game runs in a separate Node.js worker process spawned via `Cluster.fork()` (`server/src/gameManager.ts:66`). This incurs OS context-switch overhead (~1-2 ms latency) and IPC serialization cost for each message. Games with very high tick budgets may be bottlenecked by inter-process communication.
- **Map changes require full restart:** Map rotation via `Switcher<string>` (`server/src/gameManager.ts`) requires the worker to terminate and re-fork with new parameters. **No live map hot-reload exists.** Players must reconnect to switch maps.
- **TeamMode switching via IPC:** Team mode rotation also flows through `Switcher<TeamMode>` and requires IPC to worker. Solo→Duo mode switches force all active games to close and restart.
- **PortAssignment collision risk:** Worker N listens on `Config.port + N + 1`. If conflicts exist (e.g., another service on that port), workers fail silently with no clear error. Port ranges must be carefully allocated.
- **Vite bundle size warnings:** Chunk size warnings threshold is 2000 KB. No hard limit exists; exceeding it logs a warning but does not block build. Monitor bundle size manually for production.

## Dependencies on This Document

This document explains the **system-wide architecture** that all subsystems operate within:

- [Utilities & Support Systems](docs/subsystems/utilities-support/) — Foundational utilities: terrain, layer system, logging, tweening, sprites
- [Game Loop](docs/subsystems/game-loop/) — Depends on understanding the worker process model and tick loop architecture
- [Networking](docs/subsystems/networking/) — Depends on the WebSocket architecture (primary port /team, worker ports /play)
- [Object Definitions](docs/subsystems/object-definitions/) — Loaded at startup across all processes; must be in common/ for sharing
- [Spatial Grid](docs/subsystems/spatial-grid/) — Depends on understanding per-worker Game instance isolation
- [Map Generation](docs/subsystems/map/) — Depends on Switcher architecture for map rotation
- [Gas System](docs/subsystems/gas/) — Runs inside each Game instance; depends on tick timing
- [Client Rendering](docs/subsystems/client-rendering/) — Depends on knowing client connects via worker WebSocket ports
- [Inventory](docs/subsystems/inventory/) — Runs on server inside Game instance
- [Plugins](docs/subsystems/plugins/) — Plugin system runs inside worker Game instances, not primary process

## Related Documents

### Tier 1 — Architecture & High-Level
- [Project Description](description.md) — Purpose, features, and business domain
- [Data Model](datamodel.md) — Core enumerations and GameConstants definitions
- [Development Guide](development.md) — Setup, build commands, code standards
- [API Reference](api-reference.md) — WebSocket protocol and packet encoding

### Tier 2 — Subsystems
- [Utilities & Support Systems](docs/subsystems/utilities-support/) — Terrain, layers, logging, tweening, PixiJS helpers
- [Game Loop](docs/subsystems/game-loop/) — Tick orchestration and game state
- [Networking](docs/subsystems/networking/) — WebSocket transport and packet multiplexing
- [Object Definitions](docs/subsystems/object-definitions/) — Registry pattern and serialization
- [Spatial Grid](docs/subsystems/spatial-grid/) — Collision detection grid
- [Map Generation](docs/subsystems/map/) — Procedural map layout with Switcher system
- [Gas System](docs/subsystems/gas/) — Safe-zone shrinking mechanics
- [Client Rendering](docs/subsystems/client-rendering/) — PixiJS WebGL rendering and display
- [Inventory](docs/subsystems/inventory/) — Item management and equipment system
- [Plugins](docs/subsystems/plugins/) — Server-side event system and extensibility

---

## Code Standards (Biome)

Source: `biome.json` (schema version `2.2.4`)

`recommended: false` — all rules are explicitly opted in.

### Enforced Rules (errors)

**Complexity**
- `noAdjacentSpacesInRegex`, `noExtraBooleanCast`, `noStaticOnlyClass`,
  `noUselessCatch`, `noUselessConstructor`, `noUselessEscapeInRegex`,
  `noUselessThisAlias`, `noUselessTypeConstraint`
- `useLiteralKeys`, `useOptionalChain`

**Correctness**
- `noConstAssign`, `noConstantCondition`, `noEmptyCharacterClassInRegex`,
  `noEmptyPattern`, `noGlobalObjectCalls`, `noInvalidBuiltinInstantiation`,
  `noInvalidConstructorSuper`, `noNonoctalDecimalEscape`, `noPrecisionLoss`,
  `noSelfAssign`, `noSetterReturn`, `noSwitchDeclarations`,
  `noUndeclaredVariables`, `noUnreachable`, `noUnreachableSuper`,
  `noUnsafeFinally`, `noUnsafeOptionalChaining`, `noUnusedLabels`,
  `noUnusedPrivateClassMembers`, `noUnusedVariables`
- `useIsNan`, `useValidForDirection`, `useValidTypeof`, `useYield`

**Nursery (errors)**
- `noFloatingPromises`, `noMisusedPromises`, `noNonNullAssertedOptionalChain`
- `useConsistentTypeDefinitions` — enforces `interface` over `type` for object shapes

**Style (errors)**
- `noCommonJs` — ESM only
- `noInferrableTypes`, `noNamespace`, `noNonNullAssertion`, `noYodaExpression`
- `useArrayLiterals`, `useAsConstAssertion`, `useForOf`, `useLiteralEnumMembers`,
  `useReadonlyClassProperties`, `useShorthandFunctionType`, `useThrowOnlyError`,
  `useUnifiedTypeSignatures`

**Style (warnings)**
- `useArrowFunction`, `useConsistentArrayType`, `useTemplate`

**Suspicious (errors)**
- `noAsyncPromiseExecutor`, `noCatchAssign`, `noClassAssign`, `noCompareNegZero`,
  `noConfusingVoidType`, `noConstantBinaryExpressions`, `noControlCharactersInRegex`,
  `noDebugger`, `noDoubleEquals` (strict equality required), `noDuplicateCase`,
  `noDuplicateClassMembers`, `noDuplicateElseIf`, `noDuplicateObjectKeys`,
  `noDuplicateParameters`, `noEmptyBlockStatements`, `noExplicitAny`,
  `noExtraNonNullAssertion`, `noFallthroughSwitchClause`, `noFunctionAssign`,
  `noGlobalAssign`, `noImportAssign`, `noIrregularWhitespace`,
  `noMisleadingCharacterClass`, `noMisleadingInstantiator`, `noPrototypeBuiltins`,
  `noRedeclare`, `noShadowRestrictedNames`, `noSparseArray`, `noTsIgnore`,
  `noUnsafeDeclarationMerging`, `noUnsafeNegation`, `noUselessRegexBackrefs`, `noWith`
- `useAdjacentOverloadSignatures`, `useAwait`, `useGetterReturn`, `useNamespaceKeyword`

### Per-file Overrides

| Glob | Extra rules |
|------|------------|
| `**/*.js, *.mjs, *.cjs` (non-TS) | `noVar: error`, `useConst: error`; relaxes TS-specific correctness/suspicious rules |
| `**/*.ts, *.tsx, *.mts, *.cts` | Same override (duplicated entry in config) |
| `**/*.svelte, *.astro, *.vue` | `useConst: off`, `useImportType: off`, `noUnusedVariables: off`, `noUnusedImports: off`, `noUndeclaredVariables: off`, `noSelfAssign: off` |

### Excluded from Linting
`common/dist/**`, `client/dist/**`, `server/dist/**`, `tests/dist/**`

---

## Game Constants

Source: `common/src/constants.ts` — `export const GameConstants`

| Constant | Value | Notes |
|----------|-------|-------|
| `protocolVersion` | `73` | Increment on any byte-stream change or new definition item |
| `tps` | `40` | Server ticks per second → 25 ms ideal tick budget |
| `gridSize` | `32` | Spatial grid cell size (world units) |
| `maxPosition` | `1924` | Maximum world coordinate |
| `objectMinScale` | `0.15` | |
| `objectMaxScale` | `3` | |
| `defaultMode` | `"normal"` | |
| `player.radius` | `2.25` | |
| `player.baseSpeed` | `0.06` | |
| `player.defaultHealth` | `200` | |
| `player.maxAdrenaline` | `100` | |
| `player.maxShield` | `100` | |
| `player.maxInfection` | `100` | |
| `player.maxWeapons` | `4` | Slot types: Gun, Gun, Melee, Throwable |
| `player.nameMaxLength` | `16` | |
| `player.defaultName` | `"Player"` | |
| `player.defaultSkin` | `"hazel_jumpsuit"` | |
| `player.killLeaderMinKills` | `3` | |
| `player.maxMouseDist` | `256` | |
| `player.reviveTime` | `8` | Seconds |
| `player.maxReviveDist` | `5` | |
| `player.bleedOutDPMs` | `0.002` | = 2 damage/sec |
| `player.maxPerkCount` | `1` | |
| `player.maxPerks` | `4` | |
| `gas.damageScaleFactor` | `0.005` | Extra damage per distance unit into gas |
| `gas.unscaledDamageDist` | `12` | Damage not scaled within this distance |
| `airdrop.fallTime` | `8000` ms | |
| `airdrop.flyTime` | `30000` ms | |
| `airdrop.damage` | `300` | |
| `lootSpawnMaxJitter` | `0.7` | |

### Tick Scheduling

The tick loop uses **self-rescheduling `setTimeout`** with drift compensation —
not `setInterval`:

```typescript
// End of each tick in Game.tick():
setTimeout(this.tick.bind(this), this.idealDt - (Date.now() - now));
// where idealDt = 1000 / (Config.tps ?? GameConstants.tps) = 25 ms
```

---

## Deployment Architecture

Source: `server/src/server.ts`, `server/src/gameManager.ts`

### Primary Process

`server/src/server.ts` runs once (`Cluster.isPrimary && require.main === module`).
It:
- Creates one `GameManager` instance.
- Starts a `Bun.serve()` on `Config.hostname:Config.port`.
- Logs RAM and CPU usage every 60 seconds, and calls
  `gameManager.updateMapScaleRange()` for dynamic map scaling.
- Starts an initial game via `gameManager.newGame(0)`.

Signal handling: `SIGINT`, `SIGTERM`, `SIGUSR2` → graceful shutdown (kill all
workers). Bun hot-reload (`bun --hot`) is supported: stale workers from the
previous run are killed on startup via `Cluster.workers` cleanup.

### Worker Processes

Each `GameContainer` corresponds to one `Cluster.fork()` worker. Workers run
the bottom half of `gameManager.ts` (`if (!Cluster.isPrimary)` block), which:
- Parses game configuration from `process.env` (`id`, `teamMode`, `map`,
  `mapScaleRange`).
- Creates a `Game` instance.
- Starts its own `Bun.serve()` on `Config.port + id + 1`.
- Listens for `process.on("message")` IPC to live-update team mode, map, or
  start a new game without worker restart.
- Logs RAM usage every 60 seconds.

### Configuration

Server configuration is loaded from `server/config.json` (schema:
`server/config.schema.json`). Key fields used in the code (exact names from
code): `Config.hostname`, `Config.port`, `Config.maxGames`,
`Config.maxPlayersPerGame`, `Config.teamMode`, `Config.map`,
`Config.maxCustomTeams`, `Config.mapScaleRanges`, `Config.tps` (overrides
`GameConstants.tps`), `Config.maxSimultaneousConnections`, `Config.maxJoinAttempts`.

---

## Subsystem References

Navigate to Tier 2 for subsystem details:

- [Game Loop](subsystems/game-loop/) — server-side 40 TPS tick loop, object dirty tracking, packet dispatch
- [Networking](subsystems/networking/) — binary WebSocket protocol, `SuroiByteStream`, `PacketStream`, `UpdatePacket` delta encoding
- [Object Definitions](subsystems/object-definitions/) — `ObjectDefinitions<T>` registry, `idString` lookup, binary index serialization
- [Spatial Grid](subsystems/spatial-grid/) — `Grid` spatial hash, broad-phase collision culling
- [Map Generation](subsystems/map/) — `GameMap` construction, obstacle/building/loot placement, `data/maps.ts`
- [Gas System](subsystems/gas/) — shrinking safe-zone simulation, `Gas` class, `data/gasStages.ts`
- [Client Rendering](subsystems/client-rendering/) — PixiJS `Application`, `ObjectPool`, renderable object lifecycle
- [Inventory](subsystems/inventory/) — `GunItem`, `MeleeItem`, `ThrowableItem`, inventory slot management
- [Plugin System](subsystems/plugins/) — `PluginManager` event bus, `GamePlugin` base class, ~30 typed events

---

## Related Documents

- **Tier 1:** [description.md](description.md) — Project purpose, features, and target users
- **Tier 1:** [datamodel.md](datamodel.md) — Core entities and their relationships
- **Tier 1:** [development.md](development.md) — Setup commands, scripts, and local dev workflow
- **Tier 1:** [api-reference.md](api-reference.md) — WebSocket packet protocol and message types
