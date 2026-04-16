# Server Utilities

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/utils/ -->

## Purpose

Server-side utility modules providing foundational services: unique ID generation and recycling for game objects, loot table queries and weighted spawning, server configuration loading and validation, and miscellaneous HTTP/networking helpers. These utilities support core server systems but do not contain business logic themselves.

## Key Files & Entry Points

| Utility | File | Responsibility |
|---------|------|-----------------|
| **IDAllocator** | `server/src/utils/idAllocator.ts` | NetworkID (0–65535, uint16) generation and recycling |
| **LootHelpers** | `server/src/utils/lootHelpers.ts` | Loot table queries, weighted random selection, item spawning |
| **Config** | `server/src/utils/config.ts` | Load and expose game configuration from `config.json` |
| **ServerHelpers** | `server/src/utils/serverHelpers.ts` | HTTP utilities, rate limiting, role parsing, scheduled rotation |
| **Switcher** | `server/src/utils/serverHelpers.ts` | Cron-based value rotation for maps and team modes |

---

## IDAllocator

### Purpose

Manage a pool of unique numerical identifiers (NetworkIDs). Game objects require unique uint16 IDs (0–65535) for network synchronization. When objects are destroyed, their IDs return to the pool for reuse. IDAllocator provides O(1) allocation and deallocation.

### API

| Method | Returns | Purpose |
|--------|---------|---------|
| `constructor(bits: number)` | — | Create allocator managing 2^`bits` IDs (e.g., `bits=16` → 65536 IDs) |
| `takeNext()` | uint16 | Allocate next available ID; throws if pool exhausted |
| `give(id: uint16)` | void | Return ID to pool; throws if ID out of range |
| `hasIdAvailable()` | boolean | Check if pool has available IDs |

### Implementation Details

**Internal Structure:**
- Backing: `Queue<number>` FIFO data structure (imported from `@common/utils/misc`)
- Initialized with all IDs 0 through 2^`bits`−1 enqueued
- `takeNext()` dequeues from front, `give()` enqueues to back

**Example:**
```typescript
// From server/src/game.ts
const idAllocator = new IDAllocator(16); // 65536 IDs
const networkID = idAllocator.takeNext(); // Returns 0, then 1, 2, ...
// ... object lifetime ...
idAllocator.give(networkID); // Return to pool
```

### Validation & Safety

- Constructor throws `RangeError` if `bits` is negative or non-integer
- `give()` throws `RangeError` if ID is negative, non-integer, or exceeds 2^`bits`−1
- **WARNING:** No deduplication check—caller must ensure same ID is never given twice (would corrupt pool)

### Known Gotchas

1. **Pool Exhaustion:** If all 65536 IDs allocated simultaneously with none returned, `takeNext()` throws. This indicates a memory leak (destroyed objects not returning IDs).
2. **ID Collision Risk:** Buggy code that gives the same ID twice causes silent corruption. IDs should only be given to the allocator once per object lifetime.
3. **No Hot-Reload:** Allocator state lost on server restart (next game starts from ID 0).

---

## LootHelpers

### Purpose

Query loot tables and generate randomized loot spawns. Supports nested tables, weighted selection, rarity-based filtering, and special handling for guns (auto-spawn ammo and scopes based on gun definition).

### Key Concepts

**LootTable Types:**
- **Simple:** Array of `WeightedItem` (single roll) or array of arrays (multiple rolls)
- **Full:** Object with `min`, `max`, `loot` array, optional `noDuplicates`

**WeightedItem Structure:**
```typescript
{
  item?: string,              // Reference to LootDefinition (null for nothing)
  table?: string,             // Reference to another loot table
  weight: number,             // Higher = more likely (relative weight)
  spawnSeparately?: boolean,  // Each item as separate drop (default false)
  count?: number              // How many to spawn (default 1)
}
```

### API

| Function | Purpose | Parameters |
|----------|---------|------------|
| `getLootFromTable(modeName, tableID, quality?)` | Get loot items from a table | mode: `ModeName` (e.g., "normal"), tableID: string, quality: optional number (−1=ignore) |
| `resolveTable(modeName, tableID)` | Resolve table by ID or fall back to normal mode | modeName, tableID |
| `getLoot(modeName, items, noDuplicates, quality?)` | Internal: weighted random selection from array | — |

### Data Flow

```
Server needs loot
  ↓
getLootFromTable("normal", "chest_rare")
  ↓
resolveTable: lookup LootTables["normal"]["chest_rare"]
  ↓
if full table { validate min/max, mutate if noDuplicates }
  ↓
getLoot: weightedRandom() select item by weight
  ↓
if item is gun {
  auto-add ammo (split count), auto-add scope if specified
}
  ↓
return LootItem[] (id + count pairs)
```

### Special Behaviors

**Gun Spawning:**
When a gun is selected:
- `ammoType` → determined from gun definition
- `ammoSpawnAmount` → from gun definition
- Ammo split into two LootItem: `Math.floor(amount/2)` + `Math.ceil(amount/2)` (two drops)
- If gun has `spawnScope` defined → add 1 scope to loot

**Quality Filtering:**
- Optional `quality` parameter filters items: only weights < `quality` selected
- If no items pass filter → fall back to all items

**No Duplicates:**
- When `noDuplicates: true` → selected item removed from original array before next roll
- Only affects items in current table, not nested tables

**Mode Fallback:**
- `resolveTable(modeName, id)` → checks `LootTables[modeName][id]` first
- If not found → falls back to `LootTables.normal[id]`
- Allows mode-specific overrides with normal mode as base

### Implementation Details

- Uses `weightedRandom()` from `@common/utils/random` (probability ∝ weight)
- Uses `random(min, max)` for quantity ranges
- Recursive table resolution via `getLootFromTable()` (supports nested tables)

### Gotchas

1. **Undefined Table:** `resolveTable()` returns `undefined` if table doesn't exist in either mode or normal. `getLootFromTable()` throws `ReferenceError`.
2. **Undefined Item:** If a WeightedItem references non-existent item, throws `ReferenceError`.
3. **Mutation Risk:** Full tables with `noDuplicates` mutate the loot array — cloned internally but shared references could cause issues.
4. **Cache Invalidation:** Definition changes (item name, gun ammo) require server restart.

---

## Config System

### Purpose

Load server configuration from `config.json` at startup, validate against schema, and expose settings globally. Supports:
- Static values (port, hostname, max players)
- Scheduled rotation for maps and team modes (via cron expressions)
- Dynamic scaling by player count
- Per-IP rate limiting, role permissions, username filters

### Loading Flow

```
Server startup (server/src/server.ts)
  ↓
import { Config } from "./utils/config"
  ↓
config.ts checks: does config.json exist?
  ├─ NO: copy config.example.json → config.json
  └─ YES: proceed
  ↓
dynamic import "../../config.json" as ConfigSchema
  ↓
Config object exposed globally
  ↓
server/src/gameManager.ts uses: Config.port, Config.maxPlayersPerGame, etc.
```

### ConfigSchema Structure

| Setting | Type | Required | Purpose |
|---------|------|----------|---------|
| `$schema` | string | Yes | JSON schema URI (for validation tools) |
| `hostname` | string | Yes | Bind address (e.g., "0.0.0.0", "localhost") |
| `port` | number | Yes | Main server port; workers on port+1, port+2, … |
| `map` | string \| object | Yes | Map name (e.g., "main", "debug") or rotation schedule |
| `mapScaleRanges` | array | No | Dynamic map scaling by player count |
| `teamMode` | string \| object | Yes | "solo" \| "duo" \| "squad" or rotation schedule |
| `spawn` | object | No | Player spawn mode ("default", "random", "fixed") and position |
| `gas` | object | No | Disable gas, force shrink position, or override duration |
| `minTeamsToStart` | number | No | Min teams before game starts (default 1) |
| `maxPlayersPerGame` | number | Yes | Hard cap on players per game |
| `maxGames` | number | Yes | Max concurrent games |
| `tps` | number | No | Override GameConstants.tps (ticks/sec, default 40) |
| `plugins` | array | No | List of plugin names to load |
| `apiServer` | object | No | External API for VPN checks and bans |
| `ipHeader` | string | No | HTTP header for real IP (e.g., "X-Real-IP" for nginx) |
| `maxSimultaneousConnections` | number | No | Per-IP connection limit |
| `maxJoinAttempts` | object | No | Rate limit: max joins per duration |
| `maxCustomTeams` | number | No | Per-IP custom team creation limit |
| `usernameFilters` | array | No | Regex patterns to replace offensive usernames |
| `roles` | object | No | Dev roles with passwords and cheats enabled |
| `allowLobbyClearing` | boolean | No | Enable dev cheats (requires isDev role) |
| `disableBuildingCheck` | boolean | No | Allow scopes/flares to work indoors |

### Map & Team Mode Rotation

**Static configuration:**
```json
{
  "map": "main",
  "teamMode": "solo"
}
```

**Scheduled rotation:**
```json
{
  "map": {
    "rotation": ["main", "debug"],
    "cron": "0 */4 * * *"
  },
  "teamMode": {
    "rotation": ["solo", "duo", "squad", "squad"],
    "cron": "0 */4 * * *"
  }
}
```

Cron expression `0 */4 * * *` = every 4 hours, rotate to next in list.

### Dynamic Map Scaling

```json
{
  "mapScaleRanges": [
    { "minPlayers": 0, "maxPlayers": 10, "scale": 0.5, "maxMajorBuildings": 5 },
    { "minPlayers": 11, "maxPlayers": 50, "scale": 1.0, "maxMajorBuildings": 15 },
    { "minPlayers": 51, "maxPlayers": 100, "scale": 1.5, "maxMajorBuildings": 25 }
  ]
}
```

When a game has 15 players: matching range is 11–50 → apply scale 1.0 and allow 15 major buildings.

### Validation

- JSON schema enforced by TypeScript types (generated from `config.schema.json`)
- Ranges checked: port 0–65535, TPS > 0, player counts ≥ 1, etc.
- Enum validation: teamMode ∈ ["solo", "duo", "squad"], spawn.mode ∈ ["default", "random", "fixed"]
- Invalid config → TypeScript compilation error or runtime JSON parse error

### Known Issues

1. **No Hot-Reload:** Config changes require server restart. Cron rotation applies across games, but static changes don't take effect mid-game.
2. **Cron Typos Silent:** Invalid cron pattern in schema doesn't validate. The `Cron` parser may fail silently or throw at runtime.
3. **Relative Paths:** config.json must be in server root. Paths in config are hardcoded relative to `process.cwd()`.

---

## ServerHelpers

### Purpose

Miscellaneous server-wide utilities: logging with color, CORS headers, IP detection, role/permission parsing, rate limiting, and scheduled rotation via cron expressions.

### Functions

| Function | Purpose | Usage |
|----------|---------|-------|
| `serverLog(...msg)` | Log with "[Server]" prefix in magenta | `serverLog("Game started")` |
| `serverWarn(...msg)` | Warn with "[Server] [WARNING]" prefix in yellow | `serverWarn("Low memory")` |
| `serverError(...msg)` | Error with "[Server] [ERROR]" prefix in red | `serverError("Socket closed")` |
| `getSearchParams(req)` | Parse query string from HTTP request URL | `getSearchParams(req).get("password")` |
| `getIP(req, res)` | Extract client IP from request (respects proxy headers) | `getIP(req, res)` → "203.0.113.42" |
| `parseRole(searchParams)` | Parse role password from URL; validate against config | Returns `{ role?, isDev, nameColor? }` |
| `getPunishment(ip)` | Async: fetch VPN/ban status from external API | Returns `PunishmentMessage \| undefined` |

### CORS Headers

```typescript
export const corsHeaders = {
  headers: {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
    "Access-Control-Allow-Headers": "origin, content-type, accept, x-requested-with",
    "Access-Control-Max-Age": "3600"
  }
};
```

Used in HTTP responses to allow browser clients.

### RateLimiter Class

Track request counts per IP, limit concurrency:

```typescript
const limiter = new RateLimiter(
  max: 100,              // Allow up to 100 requests
  resetInterval: 60_000  // Reset every 60 seconds
);

// In request handler:
limiter.increment(ip);
if (limiter.isLimited(ip)) {
  return error(429, "Too Many Requests");
}
// ... process request ...
limiter.decrement(ip);
```

**Methods:**
- `increment(ip)` — increment counter for IP
- `decrement(ip)` — decrement counter for IP
- `isLimited(ip)` — true if counter > max
- `reset()` — clear all counters

**Implementation:** In-memory object `_ipMap: Record<string, number>`. If `resetInterval` provided, entire map cleared periodically via `setInterval()`.

### Switcher<T> Class

Manage scheduled rotation of values (e.g., map rotations) via cron:

```typescript
// From server/src/gameManager.ts
const teamModeSwitcher = new Switcher(
  name: "teamMode",
  schedule: {
    rotation: ["solo", "duo", "squad"],
    cron: "0 */4 * * *"  // Every 4 hours
  },
  callback: (current, next) => {
    serverLog(`Team mode changed: ${current} → ${next}`);
  }
);

// Access current value:
const mode = teamModeSwitcher.current; // "solo"
const nextMode = teamModeSwitcher.next; // "duo"
const switchTime = teamModeSwitcher.nextSwitch; // Timestamp of next cron trigger
```

**API:**

| Property/Method | Returns | Purpose |
|-----------------|---------|---------|
| `current` getter | T | Current active value |
| `next` getter | T \| undefined | Next scheduled value (undefined if static) |
| `nextSwitch` getter | number \| undefined | Millisecond timestamp of next cron trigger |
| `index` getter | number | Current position in rotation |

**Implementation:**

1. If `schedule` is static (string/number) → no cron, `_current` set once, `_next` undefined
2. If `schedule` is object with rotation & cron:
   - Load persistent index from `${name}.txt` file (if exists) or start at 0
   - Initialize current/next from rotation array
   - Parse cron expression with `Cron` library
   - On each cron trigger:
     - Increment index
     - Update current/next from rotation (wraparound with modulo)
     - Write new index to file (persist across restarts)
     - Call user callback

**Persistence:**
- Index saved to `${name}.txt` in server root
- Allows rotation to resume at correct position after restart
- Example: `teamMode.txt` contains "2" → next start uses rotation[2]

**Gotchas:**

1. **Cron Timing:** Cron is evaluated against server system clock. Time zone issues if server clock wrong.
2. **File Permissions:** Requires write access to server root for persistence file. Silently fails if read/write fails.
3. **Callback Side Effects:** Callback fires on every cron trigger. Long-running callbacks block scheduler.

---

## Dependencies

### Depends on:

- **@common/utils/misc** — `Queue<T>` for IDAllocator
- **@common/utils/random** — `random()`, `weightedRandom()` for loot selection
- **@common/utils/objectDefinitions** — `ObjectDefinitions<T>` registry for loot lookups (Loots, Guns, etc.)
- **@common/definitions/loots** — LootDefinition interfaces and Loots registry
- **@common/definitions/items/** — Item definitions (ammos, guns, etc.)
- **Internal modules** — `server/src/data/lootTables.ts`, `server/src/data/maps.ts`

### Depended on by:

- **[Game Loop](../game-loop/)** — IDAllocator for object creation, Config for TPS, Switcher for rotations
- **[Game Objects (Server)](../game-objects-server/)** — IDAllocator for NetworkID assignment
- **[Map](../map/)** — LootHelpers for spawning loot crates, Config for scaling
- **[Inventory](../inventory/)** — LootHelpers for random item drops
- **Game Manager** (`server/src/gameManager.ts`) — Switcher for map/mode rotation, Config for all settings
- **Server** (`server/src/server.ts`) — Config, ServerHelpers for HTTP, RateLimiter for connection gating

---

## Data Flow Example: Loot Spawning

```
server/src/objects/lootCrate.ts creates loot crate
  ↓
calls getLootFromTable("normal", "loot_crate_common")
  ↓
lootHelpers.ts resolves table from LootTables
  ↓
weightedRandom selects items by weight
  ↓
for each gun selected:
  auto-append ammo (split), auto-append scope
  ↓
return LootItem[] array
  ↓
server/src/objects/lootCrate.ts populates item list
  ↓
client receives UpdatePacket with item IDs
  ↓
client renders as object instances
```

---

## Known Issues & Gotchas

1. **IDAllocator Pool Exhaustion** — No check for simultaneous allocation of >65K objects. If all IDs allocated, server crashes with "No ID available."
2. **Config Change Requires Restart** — Hot-reload not supported. Changing config.json requires server restart for static values to take effect.
3. **Cron Typos Silent** — Invalid cron pattern in Config.map or Config.teamMode may not trigger properly. No validation at config load time.
4. **LootHelpers Cache Stale** — Changing item definitions or loot tables requires server restart. No reload mechanism.
5. **RateLimiter Memory Leak** — If IPs never repeat, `_ipMap` grows unbounded. The `resetInterval` clears entire map; more sophisticated LRU eviction would help.
6. **Switcher Persistence File Safe** — Index persists across restart, resuming rotation. However, deleting `.txt` file resets position to 0.
7. **Race Condition on Map Rotation** — If Cron fires mid-game, `Config.map` is updated but in-flight games unaffected. New games start with new map.

---

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — System overview, deployment model
- [Data Model](../../datamodel.md) — Core entities, NetworkID usage

### Tier 2 — Subsystems
- **[Game Loop](../game-loop/)** — Uses IDAllocator, Config.tps; calls Switcher
- **[Game Objects (Server)](../game-objects-server/)** — Objects manage NetworkIDs from IDAllocator
- **[Map](../map/)** — Uses LootHelpers for spawning, Config for scaling
- **[Object Definitions](../object-definitions/)** — Provides loot definitions
- **[Networking](../networking/)** — Uses Config for port/hostname

### Tier 3 — Modules (if created)
- `docs/subsystems/server-utilities/modules/idAllocator.md` — Queue implementation details
- `docs/subsystems/server-utilities/modules/lootHelpers.md` — Table resolution algorithm, weighted selection
- `docs/subsystems/server-utilities/modules/config.md` — JSON schema validation, rotation scheduling
