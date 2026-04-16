# Bullet Firing Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/projectiles-ballistics/README.md -->
<!-- @source: server/src/inventory/gunItem.ts | server/src/objects/bullet.ts -->

## Purpose

Handles the complete bullet spawning pipeline: gun trigger processing, multi-projectile patterns (shotguns), spread calculation, initial velocity determination, and registration into game physics.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/gunItem.ts` | Gun firing logic, spread calculation, ammo consumption | High |
| `server/src/objects/bullet.ts` | Bullet instantiation, physics initialization, damage tracking | High |
| `common/src/utils/baseBullet.ts` | Shared bullet data structure and serialization | Medium |
| `common/src/definitions/bullets.ts` | Bullet definition schema and registry | Medium |

## Business Rules

- **Fire Rate Limiting:** Enforced by `fireDelay` (milliseconds between shots)
- **Ammo Consumption:** Each shot depletes `ammo` counter; fire blocked if `ammo <= 0`
- **Spread Mechanics:** Spread radius increases with consecutive shots (bloom) and player movement
  - Reset (`spread = 0`) after `fsaReset` milliseconds of not firing
  - Movement penalty: `moveSpread > shotSpread` (in degrees, converted to radians)
  - Applied per shot: radius = `((moveSpread or shotSpread) / 2) / sqrt(projCount)` for multi-projectile guns
- **Burst Mode:** Automatic reset of consecutive shots after burst cooldown
- **Gun Cycling:** Some guns switch fire delay after N shots (e.g., pump-action shotguns)
- **Dual Guns:** Left/right offset alternates on each shot; stored in `definition.leftRightOffset`
- **Jitter:** Additional positional variance at spawn (e.g., shotgun pellet randomization)

## Data Lineage

```
Gun Input
├─ Player attacking state + ammo check
├─ Gun definition (fireMode, fireDelay, spread, cycle, bulletOffsets)
├─ Spread calculation (based on FSA reset + movement)
├─ Bullet spawn position
│  ├─ Offset by barrel length (definition.length)
│  ├─ Collision check against obstacles at spawn
│  └─ Jitter radius application (if applicable)
├─ Spread application at spawn position
├─ Multi-projectile loop (bulletCount)
└─ Bullet registration in game.bullets
     ├─ Initial velocity vector (direction + spread)
     ├─ Range calculation (definition.range + rangeOverride)
     └─ Damage modifiers (perk multipliers, etc.)
```

## Complex Functions

### `_useItemNoDelayCheck(skipAttackCheck)` — @file server/src/inventory/gunItem.ts

**Purpose:** Fire one or more bullets without delay validation (used by burst/auto-fire timers).

**Implicit Behavior:**
- Validates player state: not dead, not downed, gun still active
- Resets `_consecutiveShots` if firing stops
- Handles burst fire cooldown transition
- Applies spread-based shot clustering
- Checks ammo and blocks fire if empty
- Handles weapon cycling (pump-action reset logic)

**Called by:**
- `useItem()` (manual fire)
- `_burstTimeout` callback (burst mode)
- `_autoFireTimeout` callback (automatic fire)

### Spread Calculation Formula — @file server/src/inventory/gunItem.ts:110–130

```typescript
let spread = owner.game.now - this.lastUse >= (fsaReset ?? Infinity)
    ? 0  // Reset to zero if FSA time exceeded
    : Angle.degreesToRadians((owner.isMoving ? moveSpread : shotSpread) / 2);
```

**Logic:**
1. FSA Reset: If time since last shot ≥ `fsaReset`, spread = 0
2. Movement check: `isMoving` selects `moveSpread` or `shotSpread` (in degrees)
3. Conversion: Degrees → radians; divide by 2 for radius (not diameter)
4. Per-projectile: For shotguns with `bulletCount`, spread is divided by `sqrt(bulletCount)` to cluster pellets

**Example (12-gauge shotgun):**
- `shotSpread=8°` → 4° (radius) → 0.070 radians
- `moveSpread=14°` → 7° (radius) → 0.122 radians
- `bulletCount=8` → per-pellet deviation ≈ 0.070/√8 ≈ 0.025 radians

### Spawn Position Collision Check — @file server/src/inventory/gunItem.ts:160–190

**Purpose:** Push bullet spawn position forward until it clears obstacles at barrel tip.

**Algorithm:**
1. Start position: barrel offset by `definition.length`
2. Collision loop:
   - Query spatial grid for obstacles/buildings in line segment
   - For each collision, measure distance to intersection
   - If intersection is closer than current position, advance to intersection point
3. Result: Bullets spawn beyond obstacles, preventing "shooting through walls"

**Edge case:** Jitter applied after collision check to slightly randomize final position

### Multi-Projectile Pattern (Shotgun) — @file server/src/inventory/gunItem.ts:200–230

**Pattern function:** `getPatterningShape()` — @file server/src/utils/misc.ts

**Used for:** Shotgun pellet clustering within spread cone

**Options:**
- Radial pattern (ring distribution)
- Linear pattern (horizontal line)
- Random scatter (even distribution within circle)

**Pellet count:** `definition.bulletCount ?? 1` (defaults to single projectile)

## Dependencies

**Internal:**
- [Hitbox System](../collision-hitbox/) — Collision detection at spawn
- [Spatial Grid](../spatial-grid/) — Obstacle lookup during spawn collision check
- [Perks & Passives](../perks-passive/) — Perk multipliers on damage (applied by `Bullet` constructor, not here)

**External:**
- Gun Definition Schema (`@common/definitions/items/guns`)
- Bullet Definition Schema (`@common/definitions/bullets`)

## Configuration

| Setting | Effect | Default | Type |
|---------|--------|---------|------|
| `GunDefinition.fireDelay` | Milliseconds between shots | Varies | number |
| `GunDefinition.fireMode` | Semi / Burst / Auto | Semi | enum |
| `GunDefinition.shotSpread` | Spray radius when stationary (degrees) | Varies | number |
| `GunDefinition.moveSpread` | Spray radius while moving (degrees) | Varies | number |
| `GunDefinition.fsaReset` | Milliseconds before spread resets to zero | ∞ | number |
| `GunDefinition.bulletCount` | Projectile count per shot (shotguns) | 1 | number |
| `GunDefinition.bulletOffsets` | Per-projectile barrel offset for burst guns | Undefined | number[] |
| `GunDefinition.jitterRadius` | Positional jitter at spawn (mm) | 0 | number |
| `GunDefinition.cycle` | Pump-action fire delay cycling | Undefined | object |

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Ballistics subsystem overview
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Projectile patterns
- **Collision:** [../../collision-hitbox/README.md](../../collision-hitbox/README.md) — Hitbox collision
- **Grid:** [../../spatial-grid/README.md](../../spatial-grid/README.md) — Spatial indexing
- **Ammo:** [../../inventory/README.md](../../inventory/README.md) — Ammo tracking
