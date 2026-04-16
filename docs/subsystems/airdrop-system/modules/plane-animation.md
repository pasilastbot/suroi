# Airdrop — Plane & Parachute Animation

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/airdrop-system/README.md -->
<!-- @source: server/src/objects/parachute.ts, common/src/constants.ts, server/src/data/maps.ts -->

## Purpose
Manages visual animation and physics of airdrop delivery: plane movement trajectory, parachute deployment animation states, crate unlock animation timings, and associated visual effects (smoke, impact).

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/parachute.ts` | Parachute object logic, fall timing, landing detection | High |
| `server/src/data/gasStages.ts` | Airdrop spawn timing relative to gas phases | Medium |
| `common/src/constants.ts` | `GameConstants.airdrop.*` — fall time, damage values | Medium |
| `client/src/scripts/objects/parachute.ts` | Client-side parachute rendering (visual state) | Medium |

## Business Rules

- **Fall Duration:** Parachute falls for exactly `GameConstants.airdrop.fallTime` milliseconds (e.g., 25 seconds)
- **Plane Trajectory:** Plane enters map edge, travels diagonally across island, exits opposite edge over ~30 seconds
- **Crate Unlock Sequence:** Crate animates falling on parachute → detaches mid-air → crate accelerates to ground
- **Crush Damage:** Landing crate deals `GameConstants.airdrop.damage` to any players/obstacles in impact zone
- **Smoke Particle:** "airdrop_smoke_particle" spawned at landing position
- **Height Drop Curve:** Parachute height decreases linearly from 1.0 (sky) to 0.0 (ground) over fall duration
- **Obstacle Destruction:** Crate destroys obstacles on landing (deals Infinity damage)
- **Building Damage:** Crate damages building ceilings (`damageCeiling()`) if in impact zone

## Data Lineage

### Plane Trajectory
```
Map.width, Map.height (island bounds)
  ↓
Airdrop spawn trigger (gas stage advance or game event)
  ↓
Plane start angle (0-360°, random or mode-specific)
  ↓
Plane path = edge → island center → opposite edge (diagonal line)
  ↓
Plane position updated per tick: lerp(startPos, endPos, timeRatio)
  ↓
Client receives plane position in UpdatePacket
  ↓
PixiJS sprite positioned and rotated
```

### Parachute Fall Animation
```
Parachute spawn (when plane reaches island)
  ↓
endTime = Date.now() + GameConstants.airdrop.fallTime (e.g., +25000ms)
  ↓
height = (endTime - now) / fallTime (starts at 1.0, decreases linearly)
  ↓
Each server tick (40 TPS): height recalculated
  ↓
UpdatePacket sends height to client
  ↓
Client Parachute sprite Y-position = ground + (height × maxHeight)
  ↓
When height < 0: parachute removed, crate appears at ground
  ↓
Crate lands: crush damage + smoke particles
```

### Crate Unlock & Object Generation
```
Parachute.height < 0 (landing)
  ↓
Airdrop type determined (from spawn config)
  ↓
map.generateObstacle(airdropType, position)
  ↓
Obstacle (crate) created at landing position
  ↓
Crush damage check: for each object in crate hitbox
    if isPlayer → piercingDamage(airdrop.damage)
    if isObstacle → damage(Infinity)
    if isBuilding → damageCeiling(Infinity)
  ↓
Loot from crate: if destroyed, items scattered via grid collision resolution
```

## Complex Functions

### `Parachute.update()` — @file server/src/objects/parachute.ts
**Purpose:** Update parachute fall state each server tick; detect landing and trigger crate generation.

**Implicit behavior:**
- Recalculates `_height` based on elapsed time since spawn (`endTime - now`)
- If height becomes negative (time exceeded), triggers landing sequence
- Landing sequence:
  1. Remove parachute from game objects
  2. Remove from airdrops array
  3. Generate obstacle (crate) at parachute position
  4. Emit `"airdrop_landed"` plugin event
  5. Spawn smoke particles at landing position
  6. Apply crush damage to all colliding objects
  7. Resolve collisions with loot item velocity push

**Called by:** Game loop `update()` phase (once per server tick at 40 TPS)

**Example timing:**
```typescript
// t=0: parachute spawned, height = 1.0
// t=12.5s: height = 0.5 (halfway down)
// t=25s: height = 0.0 (ground level)
// t=25.1s: height = -0.1 (landing triggered)
```

### Plane Positioning Algorithm — @file server/src/data/maps.ts
**Purpose:** Calculate plane position along trajectory path for each UpdatePacket.

**Implicit behavior:**
- Plane path: random start edge → center of island → opposite edge
- Path is diagonal line (e.g., top-left corner to bottom-right corner)
- Plane velocity: constant across island (~duration of airdrop spawn phase, typically 30s)
- Position = lerp(startPos, endPos, (now - spawnTime) / totalTravelTime)
- Rotation angle calculated from velocity direction (angle between points)

**Called by:** `Plane.update()` (server tick)

## Visual States

| State | Duration | visuals | `_height` Range |
|-------|----------|---------|-----------------|
| **Flying (Plane)** | ~30s | Plane sprite over map, moving | — |
| **Descending (Parachute)** | 25s | Parachute sprite + crate, Y decreases | 1.0 → 0.0 |
| **Landing** | ~0.5s | Crate impact animation, smoke | 0.0 → -0.1 |
| **Crate** | Rest of game | Static obstacle at ground | — |

## Configuration
| Setting | Effect | Default |
|---------|--------|---------|
| `GameConstants.airdrop.fallTime` | Parachute descent duration (ms) | 25000 |
| `GameConstants.airdrop.damage` | Crush damage on landing | 80 |
| `GameConstants.airdrop.maxDropCount` | Max airdrops per game | varies |

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Airdrop spawning, loot contents, event integration
- **Tier 2:** [../../game-loop/README.md](../../game-loop/README.md) — Server tick at 40 TPS
- **Tier 2:** [../../map/README.md](../../map/README.md) — Obstacle generation, loot tables
- **Tier 1:** [../../../../api-reference.md](../../../../api-reference.md) — UpdatePacket serialization (plane, parachute data)
- **Patterns:** [../patterns.md](../patterns.md) — Airdrop event lifecycle
