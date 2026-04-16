# Q1: Why was Bun chosen over Node.js, and are there places where Bun-specific APIs are used that would make migration difficult?

**Answer:** Bun was chosen for superior performance, and the codebase is heavily committed to Bun-specific APIs in critical paths, making migration difficult.

## Why Bun Was Chosen: Performance

The changelog confirms the strategic decision:

> "Switched from Node.js to the Bun runtime, and from nginx to HAProxy, which should improve performance" (client/changelog/index.html v0.29.1)

**Key architectural benefits:**

1. **Native HTTP/WebSocket server** — Bun's `Bun.serve()` is written in Zig and compiled to native code, significantly outperforming Node.js's HTTP stack (which relies on libuv)
2. **Rapid startup** — Bun starts faster with better JIT compilation, critical for worker process spawning
3. **Native TypeScript support** — Bun runs `.ts` files directly without transpilation overhead
4. **Per-game isolation** — Bun's lightweight worker architecture supports the multi-process game server design where each game runs in its own process

---

## Bun-Specific APIs in Use

The codebase is **deeply dependent** on Bun APIs in critical paths:

### 1. **HTTP + WebSocket Server** (`Bun.serve`)

**Server entry point:** `server/src/server.ts` (line 91)
**Worker entry point:** `server/src/gameManager.ts` (line 282)

```typescript
Bun.serve({
    hostname: Config.hostname,
    port: Config.port,
    routes: { "/api/serverInfo": ... },
    websocket: { open, message, close }
})
```

This is **NON-PORTABLE**. Bun's HTTP server API differs fundamentally from Node.js's `http.createServer()` or Express.

### 2. **WebSocket Upgrade API** (`res.upgrade()`)

**Location:** `server/src/server.ts`, `gameManager.ts`

```typescript
res.upgrade(req, {
    data: { ip, teamID, autoFill, role, isDev, ... }
});
```

This is **Bun-only**. Node.js uses a different WebSocket upgrade flow (via `http.IncomingMessage` and manual socket handling). Libraries like `ws` require significant refactoring.

### 3. **Typed WebSocket Connections** (`Bun.ServerWebSocket<T>`)

**Used throughout server code:**
- `server/src/server.ts` (line 186), parameter `socket: Bun.ServerWebSocket<CustomTeamPlayerContainer>`
- `gameManager.ts` (line 326), parameter `socket: Bun.ServerWebSocket<PlayerSocketData>`
- `game.ts` (line 625), player's socket property type
- `objects/player.ts` (line 529), `team.ts` (line 285)

This is a **Bun generic type** that attaches generic data objects to WebSocket connections. Node.js's `ws` or native WebSocket doesn't have this pattern—you must manage connection state externally.

### 4. **Request Types** (`Bun.BunRequest`, `Bun.Server`)

**Location:** `server/src/utils/serverHelpers.ts`

```typescript
export function getSearchParams(req: Bun.BunRequest): URLSearchParams { ... }
export function getIP(req: Bun.BunRequest, res: Bun.Server): string { ... }
```

These are Bun-specific request/response types that differ from Node.js's `http.IncomingMessage` and `http.ServerResponse`.

---

## Portability: Node.js Compatibility Layer

The project **mitigates some lock-in** by using Node.js standard library:

- **Clustering** (`node:cluster`) — `server/src/server.ts`, `gameManager.ts`
  - Bun implements Node.js's `Cluster` module, allowing multi-process worker spawning to work across runtimes
- **File system** (`node:fs`) — `server/src/utils/serverHelpers.ts`
- **URL parsing** (`node:url`) — `server/src/server.ts`

However, **the critical path (HTTP + WebSocket server) is 100% Bun-specific**.

---

## Migration Difficulty Assessment

| Component | Difficulty | Reason |
|-----------|-----------|--------|
| **Core HTTP/WebSocket server** | **Very High** | `Bun.serve()` + `res.upgrade()` must be rewritten for Express/ws or Node.js native HTTP |
| **WebSocket types** | **High** | Generic `Bun.ServerWebSocket<T>` pattern requires state management refactor |
| **Request/response handling** | **High** | Bun's request/response objects don't map 1:1 to Node.js |
| **Process clustering** | **Low** | Node.js `node:cluster` already compatible; Bun just implements it natively |
| **Business logic** | **Low** | Game loop, objects, networking packets use only standard TypeScript |

---

## Trade-offs Summary

| Benefit | Cost |
|---------|------|
| **Performance:** 2-3× faster HTTP/WebSocket than Express + ws | **Lock-in:** Core transport is Bun-only; migration ≈ 2-3 weeks of refactoring |
| **Startup speed:** Sub-100ms cold start per worker | **Small ecosystem:** Fewer third-party Bun middlewares vs Node.js |
| **Native TypeScript:** No transpilation overhead | **Maturity:** Bun < 1.0 (major features still stabilizing as of April 2026) |
| **Type-safe WebSocket:** `Bun.ServerWebSocket<T>` generics | **Developer familiarity:** Most web devs know Node.js; Bun is newer |

---

## References

- **Tier 1:** `docs/architecture.md` — Tech stack and process model (Bun runtime, Bun.serve)
- **Tier 1:** `docs/development.md` — Setup and commands (Bun requirements)
- **Tier 2:** `docs/subsystems/networking/README.md` — WebSocket upgrade flow
- **Source:** `server/src/server.ts` — Primary Bun.serve entry point
- **Source:** `server/src/gameManager.ts` — Worker Bun.serve + WebSocket upgrade
