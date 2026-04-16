# Common Utils

<!-- @tier: 2 -->
<!-- @parent: ../../architecture.md -->
<!-- @modules: modules/ -->
<!-- @source: common/src/constants.ts, server/src/utils/config.ts -->

## Purpose

Provides shared constants, configuration management, and utility types used across both client and server, ensuring consistent behavior and centralized tuning.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/constants.ts` | `GameConstants` object with all shared game constants (TPS, player stats, physics) |
| `server/src/utils/config.ts` | Configuration loader for `config.json` |
| `server/src/utils/config.d.ts` | TypeScript schema for server config (auto-generated) |
| `common/src/utils/misc.ts` | Shared utility types and functions |

## Architecture

This subsystem centralizes two domains:

1. **Game Constants** (`GameConstants`) — immutable, shared across client/server:
   - Physics: TPS (40), grid size (32), max position (1924)
   - Player: speed, health, adrenaline, inventory slots
   - Gas: damage curve, shrink timings
   - Loot: drop radii, spawn weights
   - Projectiles: gravity, drag, arc height

2. **Server Configuration** (`config.json`) — runtime options:
   - Network: hostname, port
   - Game: map, team mode, max players, TPS override
   - Features: spawn mode, gas options, plugins list
   - Auth: roles and password configuration

## Data Flow

```
GameConstants (defined at compile time)
    ↓
Imported by both client and server
    ↓
Used for physics, UI rendering, damage calculations
    
Server config.json (defined at runtime)
    ↓
Loader reads and validates at server startup
    ↓
Overrides or supplements GameConstants for server-specific tuning
```

## Interfaces & Contracts

**`GameConstants` object:**
- Exported as singleton from `common/src/constants.ts`
- Imported by client and server separately
- Frozen to prevent mutations

**`ConfigSchema` type:**
- Generated from `config.schema.json` via json-schema-to-typescript
- Required fields: hostname, port, map, teamMode, maxPlayersPerGame, maxGames
- Optional fields: spawn, gas, minTeamsToStart, tps, plugins

**Utility Types:**
- `Result<Res, Err>` — discriminated union for error handling
- `DeepPartial<T>`, `DeepReadonly<T>` — recursive type transformations

## Dependencies

- **Depends on:** None (self-contained)
- **Depended on by:** All subsystems (game loop, networking, rendering, etc.)

## Module Index (Tier 3)

For detailed documentation, see:
- [Constants & Configuration](modules/constants-config.md) — Full constants reference and config schema

## Related Documents

- **Tier 1:** [Architecture](../../architecture.md) — System overview and tech stack
- **Tier 1:** [Data Model](../../datamodel.md) — Entity definitions
- **Tier 1:** [Development Guide](../../development.md) — Setup and configuration
- **Tier 2:** [Game Loop](../game-loop/README.md) — Uses GameConstants.tps
- **Tier 2:** [Gas System](../gas/README.md) — Uses GameConstants.gas
