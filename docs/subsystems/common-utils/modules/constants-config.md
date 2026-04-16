# Constants & Configuration Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: common/src/constants.ts, server/config.schema.json, server/config.example.json, server/src/utils/config.ts -->

## Purpose

Centralizes all game-wide constants (tick rate, map size, player stats) and server configuration schema, enabling consistent behavior across client and server.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/constants.ts` | `GameConstants` object with all shared constants | High |
| `server/src/utils/config.ts` | Config loader (JSON parsing + auto-generation) | Medium |
| `server/src/utils/config.d.ts` | TypeScript schema for `config.json` (auto-generated) | Medium |
| `server/config.schema.json` | JSON Schema for config validation | Medium |
| `server/config.example.json` | Reference config with all options | Low |

## Business Rules

- **Single Source of Truth:** `GameConstants` is imported by both client and server → same physics
- **Read-Only Freezing:** `GameConstants` is frozen (deeply immutable) to prevent accidental mutations
- **Config Auto-Generation:** Missing `config.json` copied from `config.example.json` at startup
- **Nested Objects:** Constants organized by domain (player, gas, loot, projectiles) for clarity
- **Enum-Driven Design:** Item types and categories use enums to prevent string-based bugs
- **Protocol Versioning:** `protocolVersion` incremented on any byte stream change (gates client/server mismatch)

## Data Lineage

```
GameConstants (common/src/constants.ts)
    ├── player.* (health, speed, inventory, etc.)
    ├── gas.* (damage curve, shrink timings)
    ├── projectiles.* (physics, gravity, drag)
    └── loot.* (radii, spawn weights)
         ↓
Used by:
  - Client: UI displays, physics simulation
  - Server: spawn validation, damage calculation, config override
                                                                ↓
Server config.json (runtime options):
  ├── hostname, port (network)
  ├── map (name or rotation schedule)
  ├── teamMode (solo/duo/squad or rotation)
  ├── gas.disabled, gas.forcePosition (override gas logic)
  ├── maxPlayersPerGame, maxGames
  ├── plugins[] (list to load)
  └── tps (override GameConstants.tps)
       ↓
ConfigSchema (parsed + validated)
```

## GameConstants Structure

### Core Physics
```typescript
GameConstants.tps: 40                    // Game ticks per second (fixed)
GameConstants.gridSize: 32               // Spatial grid cell size (pixels)
GameConstants.maxPosition: 1924           // Max coordinate (map boundary)
GameConstants.objectMinScale: 0.15       // Minimum object scale
GameConstants.objectMaxScale: 3          // Maximum object scale
```

### Player Constants
```typescript
GameConstants.player.radius: 2.25        // Hitbox radius (pixels)
GameConstants.player.baseSpeed: 0.06     // Units per tick
GameConstants.player.defaultHealth: 200  // Starting HP
GameConstants.player.maxAdrenaline: 100  // Adrenaline cap
GameConstants.player.maxShield: 100      // Body armor cap
GameConstants.player.maxWeapons: 4       // Inventory slots
GameConstants.player.nameMaxLength: 16   // Username length limit
GameConstants.player.killLeaderMinKills: 3 // Threshold for kill leader
GameConstants.player.reviveTime: 8       // Downed → dead time (seconds)
```

### Gas Constants
```typescript
GameConstants.gas.damageScaleFactor: 0.005    // Extra DPS per unit distance
GameConstants.gas.unscaledDamageDist: 12      // Radius before scaling applies
```

### Projectile Physics
```typescript
GameConstants.projectiles.maxHeight: 5        // Bullet arc max (units)
GameConstants.projectiles.gravity: 10         // Downward acceleration
GameConstants.projectiles.drag.air: 0.7       // Air resistance
GameConstants.projectiles.drag.ground: 3      // Ground sliding friction
GameConstants.projectiles.drag.water: 5       // Water drag
```

### Loot Drop Radii
```typescript
GameConstants.lootRadius[DefinitionType.Gun]: 3.4      // Gun pickup radius
GameConstants.lootRadius[DefinitionType.Ammo]: 2       // Ammo pickup radius
// ... etc for each item type
```

## Server Config Schema

### Top-Level Options
| Option | Type | Required? | Purpose |
|--------|------|-----------|---------|
| `hostname` | string | Yes | Server IP (e.g., "127.0.0.1") |
| `port` | number | Yes | Main server port (game ports = port+1, port+2, ...) |
| `map` | string \| rotation | Yes | Map name or rotation schedule (cron-based) |
| `teamMode` | "solo" \| "duo" \| "squad" \| rotation | Yes | Team size or rotation |
| `spawn` | object | No | Spawn mode (default, random, fixed) and position |
| `gas` | object | No | Gas disable, force position, force duration |
| `maxPlayersPerGame` | number | Yes | Player limit per game |
| `maxGames` | number | Yes | Concurrent games limit |
| `minTeamsToStart` | number | No | Teams required before game starts |
| `tps` | number | No | Override GameConstants.tps (default: 40) |
| `plugins` | string[] | No | Plugin names to load (e.g., ["placeObjectPlugin"]) |
| `roles` | object | No | Authentication roles with passwords |

### Spawn Mode
```typescript
spawn: {
  mode: "default" | "random" | "fixed",
  position?: [number, number] | [number, number, number],  // [x, y] or [x, y, layer]
  radius?: number                                              // If set, random within circle
}
```

### Gas Configuration
```typescript
gas?: {
  disabled?: boolean,                  // Disable gas entirely
  forcePosition?: [number, number] | boolean,  // true = center, or specific position
  forceDuration?: number               // Fixed stage duration (seconds)
}
```

### Map Rotation (Cron-Based)
```typescript
map: {
  rotation: ["normal", "debug", "variants:snowflake"],
  cron: "0 */30 * * * *"  // Every 30 minutes
}
```

## Complex Functions

### `GameConstants` — Nested Object Structure
**Purpose:** Organize all game constants with semantic grouping.

**Implicit Behavior:**
- Imported into both client and server
- Used by UI to display player stats
- Used by physics engine for simulation
- Not included in network packets (assumed same on both sides)

**Mutation Prevention:** Use `Object.freeze()` to prevent accidental mutations

**File:** `common/src/constants.ts:1–150`

### `Config` (runtime) — @file server/src/utils/config.ts  
**Purpose:** Load server configuration from `config.json`.

**Implicit Behavior:**
1. Check if `config.json` exists
2. If missing, copy from `config.example.json`
3. Import and parse as TypeScript (via import)
4. Apply defaults for optional fields
5. Validate against schema (optional, may use separate validator)

**Gotcha:** Changes to `config.json` require server restart; no hot-reload

**File:** `server/src/utils/config.ts:1–12`

## Configuration Tuning Reference

| Setting | Use Case | Default | Typical Range |
|---------|----------|---------|----------------|
| `tps` | Game simulation rate | 40 | 20–60 |
| `maxPlayersPerGame` | Difficulty balancing | 80 | 4–200 |
| `maxGames` | Server capacity | 5 | 1–20 |
| `gas.forceDuration` | Debugging | — | 10–60 (seconds) |
| `spawn.radius` | Spawn clustering | — | 0–500 (units) |

## Related Documents

- **Tier 2:** [Common Utils Subsystem](../README.md) — Overview of shared utilities
- **Tier 1:** [Architecture](../../../architecture.md) — System-wide configuration
- **Tier 1:** [Data Model](../../../datamodel.md) — Entity definitions and structure
- **Tier 2:** [Game Loop](../../game-loop/README.md) — Tick-rate usage
- **Tier 2:** [Gas System](../../gas/README.md) — Gas constants and configuration
