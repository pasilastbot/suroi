# Q3: What is the rationale behind the monorepo workspace layout (client/, common/, server/, tests/), and what are the dependency rules between packages?

## Answer: Monorepo Architecture & Rationale

### **Monorepo Structure**

Suroi uses a **Bun workspace monorepo** with four packages:

```
suroi/
├── client/          # Web frontend (Svelte + Vite)
├── common/          # Shared TypeScript source (no build)
├── server/          # Game server (Bun runtime)
├── tests/           # Test suite (Jest + ts-jest, independent)
└── bunfig.toml      # Workspace configuration
```

**Declared in** [package.json](package.json):
```json
"workspaces": ["client", "common", "server", "tests"]
```

---

## Rationale: Why This Split?

### **Why Split Server and Client?**

1. **Different runtimes** — Server runs on Bun (native binary protocol, WebSocket API); Client runs in the browser (WASM, DOM API, PixiJS rendering)
2. **Different build pipelines** — Server has zero build steps (Bun runs TypeScript directly); Client requires Vite bundling
3. **Different dependency graphs** — Server depends on Node.js stdlib; Client depends on browser APIs and UI libraries (Svelte, PixiJS)
4. **Deployments are independent** — Client builds as static HTML/JS and ships to a CDN; Server is a single binary or Docker image

### **Why a Separate `common/` Package?**

The `common/` package provides **source code shared between client and server**:

| Code Category | Client Uses | Server Uses |
|---------------|------------|------------|
| **Packet definitions** | Serialize player input for upload | Deserialize player input; serialize game state for UpdatePacket |
| **Object definitions** | Render items, weapons, players | Spawn objects, apply damage, check collisions |
| **Constants** | GameConstants.TPS (40), grid cell size | GameConstants.TPS, GameConstants.protocolVersion |
| **Math/Hitbox** | Collision detection for animations | Raycasting, collision checks, damage calculations |
| **Typings** | TypeScript types for game objects | TypeScript types for game objects |

**Why not duplicate?** A single source of truth:
- If `weaponDamage` changes, both client and server update immediately
- Packet format stays in sync between client and server (no manual duplication risk)
- Player types (`Player`, `BaseGameObject`) are consistent on both sides

**Why separate from server package?** Because:
- Client and server have different build tools and don't share dependencies (Svelte isn't needed on server)
- Client's build would pull in unnecessary `@types/node` and server-specific modules
- Tree-shaking works better when client only imports what it needs

### **Why a Separate `tests/` Package?**

- **Independent test runner** — Uses Jest + ts-jest (NOT Bun's test runner)
- **Isolated dependencies** — Jest dependencies don't bleed into client/server builds
- **Flexible scope** — Can test server, client, or common code from one place
- **Organization** — All unit tests in one folder, separate from the packages being tested

---

## Dependency Rules

### **Valid Dependency Graph**

```
client/    → common/
server/    → common/
tests/     → client/, server/, common/  (can test anything)
common/    → (NOTHING)                    (no external dependencies)
```

### **Invalid Dependencies**

| Forbidden | Why |
|-----------|-----|
| `server/ → client/` | Server can't depend on browser APIs, Svelte components, or PixiJS |
| `client/ → server/` | Client can't depend on Node.js stdlib or Bun APIs |
| `common/ → anything` | Common must remain runtime-agnostic (no Node.js-only or browser-only imports) |
| `client/ → tests/` | Tests aren't part of the shipped client; client shouldn't depend on test code |
| `server/ → tests/` | Tests aren't deployed; server shouldn't depend on test code |

### **What's in Each Package?**

#### **common/src/** — Runtime-agnostic, shared by client and server

- `constants.ts` — `GameConstants` (TPS, grid cell size, protocol version)
- `typings.ts` — TypeScript interfaces (`Player`, `BaseGameObject`, `Hitbox`, etc.)
- `definitions/` — Game object definitions (weapons, items, clothing) — one source of truth
- `packets/` — Binary message types and serialization (e.g., `UpdatePacket`, `InputPacket`)
- `utils/` — Shared utilities: `SuroiByteStream` (binary codec), `hitbox.ts`, `math.ts`, `vector.ts`

**Node.js dependencies:** NONE. Uses only TypeScript stdlib and browser-compatible libraries (e.g., `crypto` from `node:crypto`, which Bun/browsers both support)

#### **server/src/** — Server-only, uses Node.js and Bun APIs

- `server.ts` — Primary process: `Bun.serve()`, HTTP routes, player lobby
- `gameManager.ts` — Game orchestration via `Cluster.fork()` (Node.js IPC)
- `game.ts` — Game loop controller
- `objects/` — Game object implementations (`Player`, `Loot`, `Building`)
- `utils/` — Server utilities: `grid.ts` (spatial hash), serialization helpers
- `data/` — Map data, loot tables, balance configs

**Key Node.js-only imports:**
- `node:cluster` — Multi-process worker management
- `node:fs` — File I/O for config, maps, plugins
- `node:path`, `node:url` — File path utilities
- Bun-specific: `Bun.serve()`, `Bun.ServerWebSocket<T>`

#### **client/src/** — Client-only, uses browser APIs and UI frameworks

- `scripts/` — Game logic (player input, object pooling, camera)
- `scripts/game.ts` — Main game client loop
- `scripts/objects/` — PixiJS renderers for game objects
- `scripts/managers/` — UI manager, input manager, camera manager
- `scripts/ui.ts`, `scss/` — Traditional DOM + jQuery UI
- Svelte components for UI (changelog, news, rules, leaderboard)

**Key browser-only/Svelte imports:**
- `pixi.js` — Rendering library
- `svelte` — Component framework
- DOM APIs — `document.getElementById()`, `canvas.getContext()`

#### **tests/src/** — Jest-based tests, isolated from production

- `*.test.ts` — Unit and integration tests for common, server, client
- Uses Jest + ts-jest for TypeScript support
- Can import from any package (client, server, common)

---

## Build Pipeline

### **Client Build (Vite)**

```
client/src/** (TS + Svelte)
    ↓ [Vite build]
    ├─ Resolve @common/* path alias → ../common/src/*
    ├─ ESBuild transpile (TS + Svelte)
    ├─ Tree-shake unused code
    └─ Minify + output
            ↓
        client/dist/
        ├─ index.html
        ├─ scripts/main-[hash].js
        ├─ styles/main-[hash].css
        └─ img/, audio/, fonts/
```

**Key detail:** `common/src/` is NOT compiled separately. Vite imports TS directly and bundles it into the final client bundle.

### **Server Build (Bun)**

```
server/src/** (TS)
    ↓ [Bun dev / Bun runtime]
    ├─ Resolve @common/* path alias → ../common/src/*
    ├─ JIT-compile on-the-fly (Bun.serve)
    └─ No bundling; modules loaded individually
```

**Key detail:** No build step. `bun run dev` or `bun start` directly runs TypeScript. Common source is parsed at runtime.

### **Test Build (Jest)**

```
tests/src/** (TS)
    ↓ [Jest + ts-jest]
    ├─ Load test files
    ├─ Resolve imports from any package
    └─ Execute and report
```

---

## Path Alias Convention

Both client and server use the same path alias for consistency:

**In all `tsconfig.json` files:**
```json
"compilerOptions": {
    "paths": {
        "@common/*": ["../common/src/*"]
    }
}
```

This means:
- `import { GameConstants } from "@common/constants"` works the same in client, server, and tests
- IDE autocompletion knows to look in `common/src/`
- No confusion about relative paths (`../../../common/src/` vs `./`)

---

## Dependency Chain at Runtime

When a player joins a game:

```
Client Code (client/src/scripts/game.ts)
    ↓ imports
    ├─ @common/packets/* (serialize input)
    └─ @common/definitions/* (render weapon icons)

Network (WebSocket)

Server Code (server/src/game.ts)
    ↓ imports
    ├─ @common/packets/* (deserialize input)
    ├─ @common/definitions/* (get weapon stats, damages)
    ├─ @common/utils/hitbox.ts (collision checks)
    └─ @common/constants.ts (TPS, grid cell size)
```

Both sides trust `@common/*` as the source of truth.

---

## Configuration Files

### **bunfig.toml** — Workspace configuration

```toml
[install]
linker = "hoisted"  # Flatten node_modules for workspace resolution
```

### **root package.json** — Workspace declaration

```json
{
    "workspaces": ["client", "common", "server", "tests"],
    "private": true
}
```

The root is marked `private` so it's never published as an npm package (only workspaces are deployed).

---

## References

**Tier 1 — System Architecture:**
- `docs/architecture.md` — Tech stack, monorepo layout, component map
- `docs/development.md` — Setup, build commands, workspace installation

**Source:**
- `package.json` — Workspace declaration
- `bunfig.toml` — Bun linker configuration
- `client/tsconfig.json`, `server/tsconfig.json` — Path alias configuration
- `client/package.json`, `server/package.json`, `common/package.json` — Package metadata
