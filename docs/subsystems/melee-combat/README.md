# Melee Combat System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/inventory/meleeItem.ts -->

## Purpose

Server-side melee weapon combat system. Handles player melee attacks, target hitbox detection, multi-hit mechanics per swing, damage application with modifier calculations, and per-weapon cooldown management. Covers all aspects of punchable/slashable/swingable weapons (hatchets, crowbars, sickles, saws, etc.).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [server/src/inventory/meleeItem.ts](../../../server/src/inventory/meleeItem.ts) | `MeleeItem` class: melee attack logic, hit detection, damage application, multiple hits per swing |
| [common/src/definitions/items/melees.ts](../../../common/src/definitions/items/melees.ts) | `MeleeDefinition` interface: weapon configuration (damage, range, cooldown, animation) |
| [common/src/utils/gameHelpers.ts](../../../common/src/utils/gameHelpers.ts) | `getMeleeHitbox()`, `getMeleeTargets()` — hitbox creation and target filtering |
| [server/src/objects/player.ts](../../../server/src/objects/player.ts) | `Player.startedAttacking` — attack trigger when input received |

## Architecture

### Attack Flow

```
Player presses/holds M (left-click)
  ↓
InputPacket received, Player.attacking = true
  ↓
Player tick: this.startedAttacking = true
  ↓
player_start_attacking event emitted (plugins can intercept)
  ↓
Player.activeItem.useItem() called
  ↓
MeleeItem.useItem() → MeleeItem._bufferAttack()
  ↓
Checks cooldown (did enough time pass?)
  ↓
MeleeItem._useItemNoDelayCheck() scheduled
  ↓
Wait hitDelay (default 50 ms)
  ↓
FOR each hit in numberOfHits (default 1):
  - Create hitbox: getMeleeHitbox() → CircleHitbox
  - Query spatial grid: game.grid.intersectsHitbox()
  - Filter targets: getMeleeTargets() [see Hit Detection below]
  - FOR each target:
    * Check multipliers (perks, material, type)
    * target.damage() called
  - Schedule next hit with delayBetweenHits (if numberOfHits > 1)
```

### MeleeDefinition Schema

Core definition fields (from [common/src/definitions/items/melees.ts](../../../common/src/definitions/items/melees.ts)):

| Field | Type | Purpose | Example |
|-------|------|---------|---------|
| `damage` | `number` | Base damage per hit (multiplied by modifiers) | `40` (Hatchet), `20` (Sickle) |
| `radius` | `number` | Hitbox radius in world units | `2.0` (Hatchet), `2.9` (Saw) |
| `offset` | `Vector` | Position offset from player center before rotation | `Vec(5.2, -0.5)` (Hatchet) |
| `cooldown` | `number` | Milliseconds between swings | `420` (Hatchet), `725` (Saw) |
| `obstacleMultiplier` | `number` | Damage multiplier vs obstacles/walls | `2.0` (standard), `2.2` (Crowbar) |
| `piercingMultiplier` | `number` (optional) | Damage multiplier vs impenetrable obstacles | `1.5` (Hatchet), `2.0` (Crowbar) |
| `iceMultiplier` | `number` (optional, default `0.01`) | Damage multiplier vs ice obstacles | `2.0` (Saw) |
| `maxHardness` | `number` (optional) | Max hardness of damageable objects; if unset, cannot damage hardness-based obstacles | — |
| `hitDelay` | `number` (optional, default `50`) | Milliseconds before first hit in swing | `180` (Hatchet/Saw) |
| `numberOfHits` | `number` (optional, default `1`) | Total hits in one swing sequence | `2` (Saw), `1` (Hatchet) |
| `delayBetweenHits` | `number` (optional, default `0`) | Milliseconds between consecutive hits | `415` (Saw) |
| `attackCooldown` | `number` (optional) | Reduced cooldown if swing hits something | `140` (Sickle) |
| `maxTargets` | `number` (optional, default `1`) | Maximum targets damaged per swing | `1` (standard) |
| `fireMode` | `FireMode` (optional) | `FireMode.Auto` allows hold-to-attack, else requires click-to-attack | `FireMode.Auto` (Sickle) |
| `speedMultiplier` | `number` | Movement speed during attack (0 = frozen, 1 = normal) | `1.0` (standard) |
| `tier` | `Tier` | Item rarity/tier (for economy) | `Tier.B` (Hatchet), `Tier.A` (Crowbar) |

Animation fields (visual/fist position):
- `fists` — idle fist positions and animation duration
- `animation` — keyframes for warmup, swing, cooldown phases
- `image` — weapon sprite position/angle/scale
- `onBack` — weapon appearance when equipped in secondary slot (on back)

## Hit Detection

### Hitbox Creation

```typescript
position = player.position + rotate(definition.offset, player.rotation) * player.sizeMod
hitbox = CircleHitbox(definition.radius * player.sizeMod, position)
```

The attack creates a **circular** hitbox centered at the rotated offset position, scaled by player size modifier.

### Target Filtering (`getMeleeTargets`)

For each object in the spatial grid cell:

1. **Type check:** Must be Player, Obstacle, Building, or C4 Projectile
2. **State check:** Not dead, is damageable
3. **Self-check:** Not the attacker
4. **Obstacle filter:** Skip stairs and obstacles with `noMeleeCollision`
5. **Layer check:** Must be on adjacent or same layer as attacker
6. **Hitbox check:** Object's hitbox must intersect attack hitbox
7. **Raycast vs walls:** 
   - For **Players/C4:** Check if target is behind a wall by raycasting from attacker to target. Target must not be farther behind wall than the wall itself (with 0.05 unit tolerance).
   - For **Obstacles:** Check if the rayecast hits a different obstacle (blocking it)
8. **Sorting:** Sort by distance (closest first), deprioritize teammates in team mode
9. **Capping:** Return only `maxTargets` closest targets

### Multi-Hit Prevention

Same entity is included in `getMeleeTargets` results each time. **No built-in per-swing deduplication**—if `maxTargets > 1` or `numberOfHits > 1`, the same target can be hit multiple times per attack sequence. Target prevention relies on:
- **Small `maxTargets`** (usually 1)
- **Boss/enemy arena timing** — hits scheduled with enough delay between them that target moves before next hit regiters

## Damage Calculation

When damage is applied:

```typescript
actualDamage = definition.damage * multiplier

multiplier = 1.0
  * perkModifier     (Berserker, Lycanthropy)
  * typeMultiplier   (obstacleMultiplier, piercingMultiplier, iceMultiplier)
```

### Perk Modifiers

- **Berserker:** Multiplicative if not Lycanthropy
- **Lycanthropy:** Multiplicative (exclusive with Berserker for stacking)

### Type-Specific Multipliers

| Target Type | Field | Default | Examples |
|-------------|-------|---------|----------|
| Obstacle (normal) | `obstacleMultiplier` | N/A (required) | `2.0`, `2.2` |
| Obstacle (impenetrable) | `piercingMultiplier` | undefined (no damage) | `1.5`, `2.0` |
| Obstacle (ice material) | `iceMultiplier` | `0.01` | `2.0` (Saw) |
| Projectile (C4) | `obstacleMultiplier` | N/A | `2.0` |
| Player | × | 1.0 | (no direct multiplier) |

### Example: Crowbar vs Hatchet

**Crowbar** (`tier: A`):
- `damage: 40`, `obstacleMultiplier: 2.2`, `piercingMultiplier: 2.0`
- 40 DMG to player → 40 × 1.0 = **40 DMG**
- 40 DMG to wood wall → 40 × 2.2 = **88 DMG**
- 40 DMG to concrete wall → 40 × 2.0 = **80 DMG**

**Hatchet** (`tier: B`):
- `damage: 40`, `obstacleMultiplier: 2.0`, `piercingMultiplier: 1.5`
- 40 DMG to player → 40 × 1.0 = **40 DMG**
- 40 DMG to wood wall → 40 × 2.0 = **80 DMG**
- 40 DMG to concrete wall → 40 × 1.5 = **60 DMG**

## Cooldown Management

### Cooldown Timing

```
[Swing A] → hitDelay (50 ms default) → [Hits registered]
                                        ↓
                        Check: did any hits connect?
                                        ↓
                     YES → use attackCooldown (if defined)
                     NO  → use base cooldown
                                        ↓
                           [wait time] → [Swing B allowed]
```

- **`cooldown`**: Time between attacks when swing misses or weapon has no `attackCooldown`
- **`attackCooldown`** (optional): Shorter cooldown if swing hits anything (encourages aggressive play)

### Example: Sickle (Auto-fire)

```javascript
{
  damage: 20,
  cooldown: 160,
  attackCooldown: 140,
  fireMode: FireMode.Auto // hold M = continuous swings
}
```

- Hit: 140 ms between hits
- Miss: 160 ms before next swing allowed
- Non-auto weapons (no `fireMode`) require new click each swing

## Dependencies

### Depends on

- **[Inventory System](../inventory/)** — `InventoryItemBase` base class, item wear mechanics (`itemData()`)
- **[Core Math & Physics](../core-math-physics/)** — spatial grid `intersectsHitbox()`, `CircleHitbox` collision detection, `raycastSolids()` line-of-sight
- **[Game Objects (Server)](../game-objects-server/)** — `Player`, `Obstacle`, `Building`, `Projectile` — damage handlers (`target.damage()`)
- **[Game Loop](../game-loop/)** — time-based scheduling (`game.addTimeout()`, `game.now`)
- **[Plugin System](../plugins/)** — `player_start_attacking`, `player_stop_attacking`, `inv_item_use` event interception

### Depended on by

- **[Game Loop](../game-loop/)** — `MeleeItem.useItem()` called during player tick when attacking
- **[Inventory System](../inventory/)** — item selection/switching affects active melee weapon
- **[Client-Side Animations](../client-rendering/)** — `AnimationType.Melee` animation playback on attacker when swing starts

## Known Issues & Gotchas

### 1. **No per-swing hit deduplication**
Same target can be hit multiple times if:
- `numberOfHits > 1` and target hasn't moved between hit delays
- `maxTargets > 1` and another instance of same target exists in grid

**Mitigation:** Most weapons use `maxTargets: 1` (default).

### 2. **Raycasting does 0.05 unit tolerance, not configurable**
Edge case: Players standing exactly on a wall may sometimes fail to be hit due to raycast distance calculation being too similar.

**Code:** [gameHelpers.ts line 145](../../../common/src/utils/gameHelpers.ts#L145)

### 3. **Hitbox is 2D circular only**
- No cone/fan shape — just a circle around the offset point
- No vertical (Z-axis) limit — hits entities at any height in the same layer
- Wide `radius` can lead to unexpectedly long reach

**Workaround:** Tune `radius` carefully. Example: Saw has `radius: 2.9`, very generous reach.

### 4. **Obstacles with `noCollisions` block raycasting but don't count as hits**
Bushes, vines, etc. won't prevent hitting entities inside them, but will block LOS to entities behind them.

### 5. **Client-side cooldown UI vs server-side enforcement**
If client UI shows cooldown ready but network lag or plugin intervention delays server, rapid clicks are sent and queued/dropped by server.

**Server behavior:** Extra clicks are ignored if `lastUse` time hasn't elapsed.

### 6. **`speedMultiplier` of 1.0 = normal speed, not 100% slow**
Confusing naming. `speedMultiplier: 0.5` means move at 50% speed. `speedMultiplier: 1.0` means move at normal speed.

### 7. **FireMode.Auto requires explicit `fireMode` field**
Weapons without `fireMode` field default to requiring single-click per attack, even if held.

## Complexity & Performance

- **Per-attack cost:** O(n) where n = objects in spatial grid cell(s)
  - O(1) hitbox check per object
  - O(1) raycast worst-case (ray hits obstacle grid)
- **Hit frequency:** Limited by `cooldown` (min ~160 ms = ~6 hits/sec per player)
- **Grid scalability:** Spatial index keeps checks fast even with many objects

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — System design, object categories
- [Data Model](../../datamodel.md) — Damage system, object lifecycle

### Tier 2
- [Inventory System](../inventory/) — Item ownership, active item selection
- [Game Loop](../game-loop/) — Tick-based attack triggering, timeout scheduling
- [Game Objects (Server)](../game-objects-server/) — Player and Obstacle damage handlers
- [Core Math & Physics](../core-math-physics/) — Collision detection, spatial grid, raycasting
- [Plugin System](../plugins/) — Attack event interception and modification

### Tier 3
- [Game Objects (Server) — Player](../game-objects-server/modules/player.md) — `startedAttacking` flow
- [Core Math & Physics — Spatial Grid](../core-math-physics/) — grid cell lookup
