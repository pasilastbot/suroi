# Explosion Physics Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/explosions-hazards/README.md -->
<!-- @source: server/src/objects/explosion.ts -->

## Purpose

Implements explosion damage calculation, raycasting for line-of-sight checks, falloff curves, knockback application, and environmental interaction (decals, shrapnel).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/explosion.ts` | Explosion creation, raycasting, damage falloff | High |
| `common/src/definitions/explosions.ts` | Explosion definition schema and registry | Medium |
| `server/src/objects/bullet.ts` | C4 explosion trigger logic | Medium |
| `server/src/objects/decal.ts` | Decal (scorch mark) rendering post-explosion | Low |

## Business Rules

- **Initialization:** Explosion requires source (`GameObject`), position, layer, weapon reference, and optional damage modifier
- **Raycast-Based Damage:** Uses raycasting to check line-of-sight (obstacles block damage behind them)
- **Damage Falloff:** Linear interpolation from `min` to `max` radius
  - Distance ≤ `min`: Full damage (100%)
  - Distance between `min` and `max`: `(max - distance) / (max - min)` (linear)
  - Distance > `max`: No damage
- **Obstacle Multiplier:** Obstacles take `damage * obstacleMultiplier` (usually < 1.0 for durability balance)
- **Perk Modifiers:** Players with `LowProfile` perk reduce explosion damage via `explosionMod`
- **Per-Object Damage Cap:** Each object damaged once per explosion (tracked by `damagedObjects` Set)
- **Raycasting Algorithm:** Angular steps computed from `explosionRayDistance` constant; rays extend to `maxRadius * 2` in grid query
- **Knockback Application:** Loot and projectiles pushed away (formula: `(max - distance) * multiplier`)

## Data Lineage

```
Explosion Creation
├─ Parameters: position, source, definition, layer, weapon, damageMod
├─ Spatial Query
│  └─ Grid.intersectsHitbox(Circle(radius.max * 2))
├─ Raycasting loop (angle = -π to π, step size from explosionRayDistance)
│  ├─ Line endpoint: Vec.fromPolar(angle, radius.max)
│  ├─ Hitbox.intersectsLine() for each grid object
│  ├─ Sort collisions by distance (closest first)
│  └─ Damage application (stops at first non-passable obstacle)
├─ Damage calculation per object
│  ├─ Base multipliers: (isObstacle ? obstacleMultiplier : 1)
│  ├─ Perk check: LowProfile → explosionMod
│  ├─ Distance falloff: (max - dist) / (max - min)
│  └─ Final: amount * obstacleMultiplier * perkMod * falloff
├─ Knockback queue for loot/projectiles
├─ Shrapnel bullet generation (bulletCount × bullets)
├─ Decal placement (scorch mark)
└─ Decal fade timeout (if defined)
```

## Complex Functions

### `explode()` — @file server/src/objects/explosion.ts:32

**Purpose:** Execute the complete explosion pipeline: grid query, raycasting, damage calculation, knockback, decal/shrapnel creation.

**Key Variables:**
- `definition`: Explosion data (radius min/max, damage, multipliers)
- `damagedObjects`: Set tracking which objects were already damaged (prevents double-hit)
- `step`: Angular increment for raycasting rays
- `lineCollisions`: Intersection points for current ray (sorted by distance)

**Implicit Behavior:**
- Rays fan out from `-π` to `π` radians (full circle)
- Ray density scales inversely with explosion radius (larger radius = sparser rays)
- Obstacles/buildings stop raycasting (break statement); loot/projectiles don't block further rays
- Stairs are passable (collision check but no break)

### Raycast Angular Step Calculation — @file server/src/objects/explosion.ts:34

**Formula:**
```typescript
const step = Math.acos(1 - ((GameConstants.explosionRayDistance / definition.radius.max) ** 2) / 2);
```

**Derivation (chord spacing):**
- Goal: Rays spaced `explosionRayDistance` mm apart along a circle of radius `radius.max`
- Chord length: `c = 2r sin(θ/2)` → rearranging: `θ = 2 arcsin(c / 2r)`
- Approximation used: `θ ≈ arccos(1 - (c/r)² / 2)` (avoids arcsin)
- **Example:** Explosion with radius 300 mm, ray spacing 20 mm
  - `step = arccos(1 - (20/300)²/2) ≈ 0.134 rad ≈ 7.7°`
  - Total rays ≈ 360° / 7.7° ≈ 47 rays (manageable)

### Damage Falloff Calculation — @file server/src/objects/explosion.ts:70–80

**Formula:**
```typescript
const { min, max } = definition.radius;
const dist = Math.sqrt(collision.squareDistance);
let damage = definition.damage
    * (isObstacle ? definition.obstacleMultiplier : 1)
    * (isPlayer ? object.mapPerkOrDefault(PerkIds.LowProfile, ({ explosionMod }) => explosionMod, 1) : 1)
    * ((dist > min) ? (max - dist) / (max - min) : 1);
```

**Logic:**
1. Distance ≤ `min`: Multiply by `1.0` (full damage)
2. `min` < Distance < `max`: Linear falloff from 1.0 to 0.0
3. Distance ≥ `max`: No damage (raycasting terminates before this)

**Example (Grenade):**
- Base damage: 100
- Min radius: 75 mm, Max radius: 300 mm
- Distance 150 mm: `100 * (300 - 150) / (300 - 75) = 100 * 150 / 225 = 66.7 damage`
- Distance 75 mm: `100 * 1.0 = 100 damage`

### Knockback Application — @file server/src/objects/explosion.ts:90–100

**Applied to:** Loot, Projectiles (not walls/buildings)

**Formula:**
```typescript
const multiplier = isProjectile ? 0.002 : 0.01;
object.push(
    Angle.betweenPoints(object.position, this.position),  // Direction away from explosion
    (max - dist) * multiplier
);
```

**Logic:**
- Direction: From explosion center toward object (`betweenPoints` angle normalized)
- Impulse magnitude: Scales with remaining falloff distance
- Loot multiplier (0.01) > Projectile multiplier (0.002) — loot thrown harder

## Dependencies

**Internal:**
- [Hitbox System](../../collision-hitbox/) — Intersection checks (`hitbox.intersectsLine()`)
- [Spatial Grid](../../spatial-grid/) — Nearby object queries via grid
- [Health & Damage](../../health-damage/) — Object.damage() method
- [Perks & Passives](../../perks-passive/) — LowProfile explosion modifier lookup
- [Knockback & Velocity](../../knockback-velocity/) — object.push() impulse application

**External:**
- Explosion Definition Schema (`@common/definitions/explosions`)
- Game Constants: `GameConstants.explosionRayDistance` (mm between rays)

## Configuration

| Setting | Effect | Default | Type |
|---------|--------|---------|------|
| `ExplosionDefinition.radius.min` | Minimum damage radius (mm) | Varies | number |
| `ExplosionDefinition.radius.max` | Maximum damage radius (mm) | Varies | number |
| `ExplosionDefinition.damage` | Base damage at center | Varies | number |
| `ExplosionDefinition.obstacleMultiplier` | Damage modifier for obstacles (0–1) | Varies | number |
| `ExplosionDefinition.shrapnelCount` | Number of shrapnel bullets spawned | Varies | number |
| `ExplosionDefinition.decal` | Decal definition string (scorch mark) | Optional | string |
| `ExplosionDefinition.decalFadeTime` | Milliseconds before decal disappears | Optional | number |
| `GameConstants.explosionRayDistance` | Raycast angular spacing (mm chord) | 20 | number |

## Known Issues & Gotchas

- **Ray Density:** Very large explosions (radius > 1000 mm) generate 100+ rays; consider performance under many simultaneous explosions
- **Stair Passability:** Stairs are passable (`if (isObstacle && isStair)` does not break), allowing damage through stair openings
- **Decal Layer:** Decals spawn on same layer as explosion; in multi-layer maps, this can block view of explosions on other layers
- **Overkill Damage:** Damage values not capped; large damage modifiers can exceed intended balance (see damage mitigation in Health & Damage subsystem)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Explosions & Hazards subsystem
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Hitbox:** [../../collision-hitbox/README.md](../../collision-hitbox/README.md) — Line-of-sight collision
- **Grid:** [../../spatial-grid/README.md](../../spatial-grid/README.md) — Spatial queries
- **Damage:** [../../health-damage/README.md](../../health-damage/README.md) — Damage processing
- **Knockback:** [../../knockback-velocity/README.md](../../knockback-velocity/README.md) — Impulse application
