# Melee Combat — Weapon Variants & Properties

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/melee-combat/README.md -->
<!-- @source: common/src/definitions/items/melees.ts, server/src/inventory/meleeItem.ts, common/src/constants.ts -->

## Purpose
Documents melee weapon variants and their properties: range/damage stats, attack animation times, material-specific multipliers (ice, stone, obstacles), special abilities (reflect surfaces, spin attacks), and visual asset mappings.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/definitions/items/melees.ts` | Melee definitions with all variants | High |
| `server/src/inventory/meleeItem.ts` | MeleeItem logic, damage calculation, hit detection | High |
| `common/src/utils/vector.ts` | Vec type, offset calculations for attack range | Medium |
| `client/src/scripts/objects/player.ts` | Client-side melee animation rendering | Medium |

## Business Rules

- **Weapon Tiers:** Melees have tier (A, B, C, etc.) reflecting rarity and power
- **Range/Radius:** Each melee defines `radius` (hit sphere) and `offset` (relative to player position)
- **Damage Scaling:** Base damage × multipliers (obstacle, piercing, ice, player armor)
- **Cooldown:** Time in milliseconds between attacks; enforced by inventory action system
- **Attack Delay:** Hit delay (milliseconds) from attack start to damage application
- **Fire Mode:** Can be single-hit or pierce (hits multiple targets in sequence)
- **Animation Timing:** Swing frame sequence with position/rotation at each frame
- **Surface Variants:** Reflective surfaces (crosshairs, mirrors) for special attack paths
- **Back Position:** On-back positioning/angle for visual carry state
- **Sound Audio:** Swing, hit, stop sounds played at appropriate frames

## Data Lineage

### Weapon Selection & Equip
```
Loot pickup: pickaxe
  ↓
Inventory.equip(pickaxe)
  ↓
Loots.reify(pickaxe) → MeleeDefinition loaded
  ↓
MeleeItem created with:
    definition: pickaxe definition
    owner: player
    damage, radius, cooldown, etc. copied
  ↓
Player.inventory.melee = meleItem
  ↓
Client received in UpdatePacket
  ↓
Display sprite with melee texture
```

### Attack Execution
```
Player presses attack button
  ↓
InputManager.onAttack()
  ↓
Player.executeMeleeAttack()
  ↓
Extract: definition, target position, arc radius
  ↓
For each entity in hit sphere:
    Calculate damage:
        baseDamage = definition.damage
        if target.isObstacle:
            dmg = baseDamage × definition.obstacleMultiplier
            if obstacle.material == ice:
                dmg = dmg × (definition.iceMultiplier ?? 0.01)
        if target.isPlayer:
            dmg = baseDamage
            if player.armor:
                dmg = dmg × (1 - armor.mitigation)
        if target.isPiercing:
            dmg = dmg × (definition.piercingMultiplier ?? 1.0)
  ↓
Apply damage to all targets in range
  ↓
Sound effects played
  ↓
Knockback applied (varies by target type)
```

### Animation Playback
```
Attack pressed at time T
  ↓
Melee.animation[] defines frame sequence
  ↓
Each frame:
    duration: milliseconds to show this frame
    fists: left/right hand position {x, y}
    image: if melee has weapon sprite:
        position: {x, y}
        angle: rotation in degrees
  ↓
Client Animation loop:
    currentFrame = 0
    elapsedInFrame = 0
    ↓ each client tick:
        elapsedInFrame += deltaTime
        if elapsedInFrame >= frame.duration:
            currentFrame += 1
            elapsedInFrame -= frame.duration
        ↓ update sprite:
            fistLeft = frame.fists.left
            fistRight = frame.fists.right
            meleeSprite.position = frame.image.position
            meleeSprite.rotation = frame.image.angle
```

## Melee Weapon Variants

### Fist Weapons
| Weapon | Tier | Damage | Range | Cooldown | Special |
|--------|------|--------|-------|----------|---------|
| **Fists** | N/A | 15 | 1.5 | 100ms | Natural weapon, unlimited |
| **Falchion** | A | 30 | 2.5 | 320ms | Slashing sword, armor penetration |

### Axe/Heavy Weapons
| Weapon | Tier | Damage | Range | Cooldown | Special |
|--------|------|--------|-------|----------|---------|
| **Hatchet** | B | 40 | 2.0 | 420ms | High obstacle damage (×2), ice damage (×1.5) |
| **Maul** | A | 54 | 2.3 | 450ms | Highest full damage, ice specialist (×5) |
| **Fireaxe** | B | 48 | 2.1 | 400ms | Balanced, reflective surface |

### Polearms/Spears
| Weapon | Tier | Damage | Range | Cooldown | Special |
|--------|------|--------|-------|----------|---------|
| **Polearms** | C | 25 | 3.0 | 400ms | Longest range, lower damage |
| **Pike** | A | 35 | 3.2 | 450ms | Extended range, piercing optimized |

### Special Abilities

#### Ice Specialist Weapons
- **Maul:** iceMultiplier = 5.0 (deals 5× damage to ice obstacles)
- **Ice Axe:** iceMultiplier = 2.0
- **Pickaxe:** iceMultiplier = 5.0 (ice mining specialty)
- Use case: Breaking ice obstacles efficiently in winter mode

#### Reflective Surfaces
Some melees have `reflectiveSurface` with pointA/pointB (line segment):
- Used for visual crosshair placement in attack arc
- Represents the reflection point for special physics (if implemented)

#### On-Back Positioning
`onBack` property defines how melee looks when carried:
- `angle`: rotation when on back
- `position`: offset from player center {x, y}
- `reflectiveSurface`: alternate reflection for back position

## Complex Functions

### `MeleeItem.getMeleeDamage()` — @file server/src/inventory/meleeItem.ts
**Purpose:** Calculate actual damage for a target based on all multipliers.

**Implicit behavior:**
- Start with `baseDamage = definition.damage`
- Check target type and apply multipliers:
  - **vs Player:** `dmg = baseDamage` (no multiplier, but armor reduces it in health application)
  - **vs Obstacle (wood/fabric/etc):** `dmg = baseDamage × definition.obstacleMultiplier`
    - If ice material: `dmg = dmg × (definition.iceMultiplier ?? 0.01)`
    - If piercing material: `dmg = dmg × (definition.piercingMultiplier ?? 1.0)`
  - **vs Generic object:** `dmg = baseDamage`
- Return calculated damage (before armor reduction for players)

**Called by:** `MeleeItem.executeAttack()` for each target in attack range

**Example:**
```typescript
// Maul (54 damage) hitting ice obstacle
// dmg = 54 × 2.0 (obstacleMultiplier) × 5.0 (iceMultiplier)
// dmg = 540 (destroys most ice obstacles in 1 hit)
```

### Animation Frame Definition — @file common/src/definitions/items/melees.ts
**Purpose:** Define each frame of the attack animation.

**Structure:**
```typescript
animation: [
    {
        duration: 100,        // Show this frame for 100ms
        fists: {
            left: Vec(0, -8),    // Left hand position relative to player
            right: Vec(5, 10)    // Right hand position relative to player
        },
        image: {
            position: Vec(3, -5),  // Weapon sprite position
            angle: -90             // Rotation in degrees
        }
    },
    // ... more frames
]
```

**Frame Lookup:**
- Client loops through frames based on elapsed time
- Interpolation: if needed, Lerp between frame positions for smooth motion
- Duration: total animation time = sum of all frame durations

## Configuration

| Setting | Source | Effect | Default |
|---------|--------|--------|---------|
| `damage` | Definition | Base attack damage | Varies (15–54) |
| `radius` | Definition | Hit sphere radius | Varies (1.5–3.2) |
| `cooldown` | Definition | Milliseconds between attacks | Varies (100–450ms) |
| `hitDelay` | Definition | Milliseconds before hit applies | ~180ms typically |
| `obstacleMultiplier` | Definition | Damage × this vs obstacles | 0.5–2.0 |
| `piercingMultiplier` | Definition | Damage × this vs piercing objects | 0.5–2.0 |
| `iceMultiplier` | Definition | Damage × this vs ice obstacles | 0.01–5.0 |
| `fireMode` | Definition | Attack pattern (single/pierce) | Single |
| `maxTargets` | Definition | Max entities hit in one swing | 1–10 |

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Attack mechanics, hit detection, knockback
- **Tier 2:** [../../health-damage/README.md](../../health-damage/README.md) — Damage application to players
- **Tier 2:** [../../inventory/README.md](../../inventory/README.md) — Item equipping, inventory slots
- **Tier 1:** [../../../../datamodel.md](../../../../datamodel.md) — MeleeDefinition structure
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — Object-driven combat system
- **Patterns:** [../patterns.md](../patterns.md) — Melee attack lifecycle
