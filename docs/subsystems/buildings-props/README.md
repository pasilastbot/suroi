# Buildings & Map Props

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/buildings-props/modules/ -->
<!-- @source: server/src/objects/building.ts, server/src/objects/obstacle.ts -->

## Purpose

Environmental static and dynamic structures: buildings with doors/interior layouts/loot spawns, obstacles (trees, rocks, crates, walls), destructible props with health/loot, collision handling, layer transitions, and ceiling damage mechanics.

## Key Files & Entry Points

| File | Purpose | Complexity |
|------|---------|-----------|
| `server/src/objects/building.ts` | Building class, door management, ceiling damage tracking | High |
| `server/src/objects/obstacle.ts` | Obstacle class (trees, rocks, crates, doors, walls, stairs), health degradation, loot drops | High |
| `server/src/map.ts` | Building/obstacle placement, generation, loot spawning | High |
| `common/src/definitions/buildings.ts` | BuildingDefinition schema (obstacles, layout, loot spawners, walls to destroy) | Medium |
| `common/src/definitions/obstacles.ts` | ObstacleDefinition schema (health, material, door/stair/tree/wall properties) | Medium |

## Architecture

### Data Model

**Building**
```typescript
class Building extends BaseGameObject {
  definition: BuildingDefinition
  orientation: Orientation                 // 0=N, 1=E, 2=S, 3=W
  layer: number
  
  spawnHitbox: Hitbox                      // where building contents spawn
  hitbox?: Hitbox                          // collision bounds
  scopeHitbox?: Hitbox                     // ceiling scope for ceilings
  
  interactableObstacles: Set<Obstacle>    // doors + activatables in this building
  puzzlePieces: Obstacle[]                 // ordered puzzle triggers
  
  _wallsToDestroy: number                  // countdown to ceiling collapse
  _puzzle?: {
    inputOrder: string[]
    solved: boolean
    errorSeq: boolean
    resetTimeout?: Timeout
  }
}
```

**Obstacle**
```typescript
class Obstacle extends BaseGameObject {
  definition: ObstacleDefinition
  health: number                           // current health (0 = dead)
  maxHealth: number
  rotation: number
  layer: number
  scale: number                            // visual scale, shrinks as damaged
  variation: Variation                     // visual variant (0-N)
  
  isDoor: boolean
  door?: {
    operationStyle: "swivel" | "slide"
    isOpen: boolean
    locked: boolean
    closedHitbox: Hitbox
    openHitbox: Hitbox
    openAltHitbox?: Hitbox                 // alt hitbox for swivel from other side
    offset: number                         // animation state (0=closed, 1=open, 3=alt)
    powered: boolean
    openOnce: boolean                      // one-way door
  }
  
  parentBuilding?: Building
  puzzlePiece?: string | boolean           // puzzle identifier
  activated?: boolean                      // for activatable obstacles
  loot: LootItem[]
}
```

### Building Generation & Lifecycle

```
Map.constructor()
  → _generateBuildings(definition, count)
    → for each spawn location:
      → generateBuilding(definition, position, orientation, layer)
        → Building created
        → for each obstacle in definition.obstacles:
          → generateObstacle() creates Obstacle
          → if isDoor || isActivatable:
            → add to building.interactableObstacles
        → for each loot spawner in definition.lootSpawners:
          → getLootFromTable()
          → game.addLoot(itemId, position, ...)
```

**Key files referenced:** `server/src/map.ts` lines 645–780 (generateBuilding)

## Obstacle System

### Obstacle Types & Properties

Obstacles vary by **material**, **health**, and **special traits** (door, stair, tree, wall, window, activatable):

| Type | Health | Material | Traits | Description |
|------|--------|----------|--------|-------------|
| **House Wall** | 170 | wood | wall | Interior building wall, hideOnMap=true, 2.06 units tall |
| **Headquarters Wall** | 100–170 | wood | wall | Reinforced building wall, custom tint/color |
| **Inner Concrete Wall** | 500 | stone | wall | Solid structure wall, 2 variations |
| **Wooden Crate** | 100 | wood | loot | Contains loot drops on destruction |
| **Barrel** | 200 | metal_light | explosion | Explodes on destruction |
| **Super Barrel** | 200 | metal_light | explosion | Larger barrel variant |
| **Door** | *varies* | wood | door | Swivel/slide operation, lockable |
| **Tree** | 300–∞ | tree | tree | Renewable, visual variations |
| **Rock** | 500 | stone | — | Indestructible unless impenetrable=false |
| **Tent Wall** | 100 | stone | wall | Temporary structure, 3 particle variations |
| **Stair** | — | — | stair | Layer transition, rectangular hitbox with activeEdges |
| **Window** | *varies* | glass | window | Transparent, loses collision on destruction |

**Key files:** `common/src/definitions/obstacles.ts`

### Door Mechanics

Doors are obstacles with `isDoor: true` in the definition.

**Door States:**
- **Closed** → collision enabled, players blocked, `isOpen=false`
- **Locked** → closed + `locked=true`, cannot interact
- **Open** → collision disabled, free passage, `isOpen=true`
- **Powered** → temporary lock during animation (`powered=false` briefly)

**Door Operation Styles:**

1. **Swivel** (hinge-based)
   - Door rotates around `hingeOffset` point
   - Two open positions:
     - **offset=1**: normal open orientation
     - **offset=3**: alt open (opposite side), uses `openAltHitbox`
   - Determined by player position relative to door rotation
   ```
   if player on opposite side: offset=3 (openAltHitbox)
   else: offset=1 (openHitbox)
   ```
   
2. **Slide** (rail-based)
   - Door slides horizontally by `slideFactor`
   - Single open position (`offset=1`)
   - Uses `openHitbox` only

**Door Locking:**
- `locked: boolean` property
- Locked doors cannot be toggled unless `automatic: true`
- Can unlock via `stageable` mechanic (e.g., multi-stage events)
- `openOnce: true` → one-way door, cannot reopen

**Interaction Flow:**
```
Player.interact(door)
  → canInteract() checks:
    - if locked and not automatic → return false
    - if interactionDelay → door.powered=false, door.locked=true
  → toggleDoor(player)
    → door.isOpen = !door.isOpen
    → recalculate hitbox (closed/open/alt)
    → after interactionDelay: door.powered=true, door.locked=false
```

**Key files:** `server/src/objects/obstacle.ts` lines 340–560; `common/src/definitions/obstacles.ts` DoorMixin

### Obstacle Damage & Destruction

```
Obstacle.damage(params)
  → if indestructible or health already 0 → return
  → if impenetrable and weapon not piercing → return
  → health -= amount
  → if health > 0:
    → scale shrinks proportionally
    → update collision hitbox
  → else (health ≤ 0):
    → collidable = false (unless window)
    → scale = destroy_scale
    → if explosion defined:
      → game.addExplosion()
    → for each item in loot:
      → game.addLoot() → drops on ground
    → if wall: building.damageCeiling()
```

**Scale Degradation:**
Health visually reflected in obstacle size:
```
newScale = (health / maxHealth) * (maxScale - destroyScale) + destroyScale
```

**Loot Drops:**
- Obstacles with `hasLoot: true` spawn `lootTable` items on destruction
- Drop position: `lootSpawnOffset` if defined, else random point in hitbox
- Special: Perk "PlumpkinBlessing" improves loot quality

**Key files:** `server/src/objects/obstacle.ts` lines 160–340

### Trees

Trees are special obstacles with visual variation:

```typescript
interface TreeMixin {
  isTree: true
  trunkVariations?: number    // e.g., 4
  leavesVariations?: number   // e.g., 4
  tree?: {
    minDist?: number
    maxDist?: number
    trunkMinAlpha?: number
    leavesMinAlpha?: number
  }
}
```

Trees have separate trunk and leaves sprites rendered at different alpha values. Variations allow visual diversity.

**Key files:** `common/src/definitions/obstacles.ts` TreeMixin

### Stairs (Layer Transitions)

Stairs are rectangular obstacles with two **active edges**: one at ground layer, one at destination layer.

```typescript
interface StairMixin {
  isStair: true
  hitbox: RectangleHitbox
  activeEdges: {
    high: 0 | 1 | 2 | 3      // 0=top, 1=right, 2=bottom, 3=left
    low: 0 | 1 | 2 | 3
  }
}
```

When a player overlaps the stair hitbox:
```
resolveStairInteraction(stairDef, rotation, hitbox, baseLayer, position)
  → determine which edge player closest to
  → if near low edge: transition to low layer
  → else: transition to high layer
```

**Key files:** `common/src/utils/math.ts` resolveStairInteraction()

### Walls

Walls are obstacles with `isWall: true`:

```typescript
interface WallMixin {
  isWall: true
  wall?: {
    color: number
    borderColor: number
  }
}
```

Walls have special handling:
- **Visibility:** hideOnMap=true (not shown in map UI)
- **Collision:** blocks movement
- **Ceiling Damage:** destroying a wall damages parent building's ceiling
  ```
  if definition.isWall:
    parentBuilding.damageCeiling(damage)
  ```

**Key files:** `server/src/objects/obstacle.ts` line 311; `common/src/definitions/obstacles.ts` WallMixin

## Building System

### Building Definitions

Buildings are placed as collections of obstacles, loot spawners, and sub-buildings:

```typescript
interface BuildingDefinition {
  idString: string
  spawnHitbox: Hitbox                       // spawn zone for obstacles
  hitbox?: Hitbox                           // collision bounds
  ceilingHitbox?: Hitbox                    // visual ceiling collision
  collidable: boolean
  allowFlyover?: FlyoverPref                // which layers can fly over?
  
  obstacles?: BuildingObstacle[]            // interior obstacles/doors/walls
  lootSpawners?: LootSpawner[]              // designated loot locations
  subBuildings?: SubBuilding[]              // nested buildings
  
  wallsToDestroy?: number                   // ceiling collapse countdown (default: Infinity)
  ceilingCollapseSound?: string
  destroyOnCeilingCollapse?: string[]       // obstacle idStrings to destroy if ceiling collapses
  visibilityOverrides?: Layer[]             // cross-layer visibility exceptions
  
  noCeilingScopeEffect?: boolean
  hasSecondFloor?: boolean
  puzzle?: {
    triggerOnSolve?: string                 // obstacle to activate on puzzle solve
    delay: number
    order?: string[]                        // required sequence of puzzle pieces
    solvedSound?: boolean
  }
}
```

### Ceiling Damage & Collapse

Buildings track incoming ceiling damage:

```
wall destroyed in building
  → wall.damage() calls building.damageCeiling(damage)
    → _wallsToDestroy -= damage
    → emit "building_will_damage_ceiling"
    → if _wallsToDestroy ≤ 0:
      → building.dead = true
      → emit "building_did_destroy_ceiling"
      → for each obstacle in destroyOnCeilingCollapse:
        → obstacle.damage(Infinity) [instant destruction]
```

**Implications:**
- Buildings with `wallsToDestroy: 1` → single wall destruction collapses ceiling
- Buildings with `wallsToDestroy: Infinity` → ceiling never collapses
- When ceiling collapses, designated obstacles (e.g., support pillars) are destroyed

**Key files:** `server/src/objects/building.ts` lines 55–115

### Loot Spawning

Buildings have designated loot spawn points:

```typescript
interface LootSpawner {
  position: Vector              // relative to building
  table: string                 // e.g., "ground_loot", "weapon"
}
```

During building generation:
```
for each lootSpawner in building.definition.lootSpawners:
  lootTable = getLootFromTable(modeName, table)
  for each item in lootTable:
    addLoot(itemId, position, layer, { count: item.count })
```

Loot tables are defined per game mode and can vary by terrain/region. Items spawn at exact positions, not randomized.

**Key files:** `server/src/map.ts` lines 750–760; `server/src/utils/lootHelpers.ts`

### Puzzles

Buildings can have interactive puzzles:

```typescript
puzzle?: {
  order?: string[]             // sequence of puzzle piece idStrings
  triggerOnSolve?: string      // obstacle to activate when solved (e.g., door unlocking)
  delay: number                // delay before triggering
  setSolvedImmediately?: boolean
  unlockOnly?: boolean         // unlock trigger instead of activating
}
```

**Puzzle Flow:**
```
Player activates puzzlePiece obstacle
  → building.togglePuzzlePiece()
    → add piece idString to building.puzzle.inputOrder
    → if inputOrder matches definition.puzzle.order:
      → building.solvePuzzle()
        → await delay
        → puzzle.solved = true
        → activate triggerOnSolve obstacle
    → else if wrong sequence:
      → puzzle.errorSeq = !puzzle.errorSeq [visual feedback]
      → reset after timeout
```

**Key files:** `server/src/objects/building.ts` lines 160–260

## Client Rendering

### Building & Obstacle Rendering Pipeline

**Server → Network:**
1. Building/Obstacle serialized as `FullData` containing definition, position, layer
2. `UpdatePacket` sent to clients in viewport
3. Client deserializes, creates/updates `GameObject` pool instances

**Client → Rendering:**
```
ObjectPool[ObjectCategory.Building] → mapping to BuildingManager
ObjectPool[ObjectCategory.Obstacle] → mapping to ObstacleManager
  → each frame:
    → iterate game objects by layer
    → apply rotation, scale, position transforms
    → apply tint/color/alpha from definition
    → render sprite at correct Z-index
    → if door: render based on door.offset (closed/open/alt)
    → if dead: render destroyed/wreckage variant
    → if particle: emit particle effect
```

### Layer Rendering Order

Objects rendered bottom-up by layer:

```
Layer -2 (basement)
  → sprites at ZIndexes.BuildingsFloor or custom
Layer -1 (basement-ground transition)
  → stair visual
Layer 0 (ground)
  → building walls, obstacles, props
  → most interactive elements
Layer 1 (ground-second floor transition)
  → stair visual
Layer 2 (second floor)
  → interior second floor objects
```

Within a layer, Z-index determines render order (defined in `ZIndexes` enum).

### Visual States

**Door States:**
- `offset=0`: closed hitbox shown, door sprite normal
- `offset=1`: open hitbox shown, door sprite rotated/hidden
- `offset=3`: openAltHitbox shown, door sprite at alternate rotation

**Obstacle Damage:**
- `scale` shrinks visually: `newScale = health / maxHealth * maxScale`
- Partially destroyed obstacles still render, just smaller/faded

**Destroyed State:**
- `dead=true` → render destroyed sprite variant (if defined)
- `collidable=false` → no collision, can walk through
- Particle effect (smoke, debris) plays at destruction position

## Collision & Physics

### Hitbox Management

Each building and obstacle has collision geometry:

```
Building.spawnHitbox  → where child obstacles can spawn
Building.hitbox       → movement collision (if collidable=true)
Building.ceilingHitbox → separate collision for ceilings (can affect scoping)

Obstacle.hitbox       → movement collision
Obstacle.spawnHitbox  → where child objects spawn (usually = hitbox)
```

### Door Collision During Animation

Doors are rectangular obstacles with dynamic hitbox:

```
Door.closedHitbox    → blocks movement (closed state)
Door.openHitbox      → permits passage (open state, normal side)
Door.openAltHitbox   → permits passage (open state, alternate side)

Door.hitbox = closedHitbox | openHitbox | openAltHitbox [swaps on toggle]
game.grid.updateObject(door)  [updates spatial grid]
```

### Stair Interaction

Stairs automatically transition layers:

```
Player.collidesWith(stair.hitbox)
  → if player.layer == stair.layer:
    → resolveStairInteraction()
    → player.layer = newLayer
```

No explicit player action required; collision triggers layer change.

## Dependencies

### Depends on:
- [Object Definitions](../object-definitions/) (`@common/definitions/buildings.ts`, `obstacles.ts`) — reified definitions, dynamic lookups
- [Core Math & Physics](../core-math-physics/) (`Vector`, `Hitbox`, `Geometry`, `resolveStairInteraction`) — collision, transforms, layer math
- [Collision & Hitbox](../collision-hitbox/) (`CircleHitbox`, `RectangleHitbox`, `GroupHitbox`) — collision primitives
- [Game Objects (Server)](../game-objects-server/) (`BaseGameObject`, `GameObject`) — entity base class, serialization
- [Spatial Grid](../spatial-grid/) (`game.grid.intersectsHitbox()`, `game.grid.updateObject()`) — broad-phase collision, object tracking
- [Inventory & Loot](../inventory/) (`getLootFromTable()`, `game.addLoot()`) — loot generation and spawning
- [Networking](../networking/) (`PacketStream`, `FullData`, serialization) — client synchronization

### Depended on by:
- [Game Loop](../game-loop/) — building/obstacle updates per tick
- [Map](../map/) — building/obstacle placement during generation
- [Client Rendering](../client-rendering/) — visual display, pool management
- [Visibility & LOS](../visibility-los/) — buildings/walls block vision between layers
- [Input Management](../input-management/) — player interact with doors
- [Particles & Effects](../particle-system/) — ceiling collapse, destruction particles

## Known Issues & Gotchas

### 1. **Door Hitbox Orientation**
Swivel doors use `hingeOffset` to calculate pivot point. Rotation affects both the hitbox and the alt hitbox calculation. If rotation isn't applied correctly, doors may leave collision gaps when opening.

*Workaround:* Always apply `Numeric.addOrientations()` to combine building orientation + obstacle rotation.

### 2. **Basement Access Inconsistency**
Buildings can have `layer` offsets in obstacle definitions (e.g., `layer: -2` for basement), but layer transitions require stairs. Some buildings define basement obstacles without proper stair access, making them unreachable.

*Workaround:* Verify every basement layer (`layer < 0`) has corresponding stair with `activeEdges`.

### 3. **Indoor Loot Placement Conflicts**
Loot spawners use fixed positions. If two loot items spawn at the same position, they stack directly on top of each other, making one impossible to pickup without moving the other.

*Workaround:* Offset loot spawners by at least 2 units, or use `lootSpawnOffset` per obstacle.

### 4. **Wall Destruction Triggers Cascading Ceiling Damage**
Destroying a wall calls `building.damageCeiling()`, but if the building has `wallsToDestroy: Infinity`, ceiling won't collapse. However, destroying certain walls may trigger `destroyOnCeilingCollapse` logic even if ceiling isn't actually destroyed, leading to unintended obstacle destruction.

*Workaround:* Be explicit about `wallsToDestroy` and `destroyOnCeilingCollapse` in building definitions. Test ceiling destruction behavior.

### 5. **Puzzle Piece Reset Timing**
Puzzle error sequence has a 10-second reset timeout. If the player triggers another wrong piece before timeout expires, the timer resets but the error visual might not sync with the actual puzzle state.

*Workaround:* Use `setSolvedImmediately: true` to avoid timing issues in time-sensitive puzzles.

### 6. **Scale Shrink Causes Hitbox Misalignment**
Obstacles shrink as they take damage (scale interpolation). The hitbox is scaled proportionally, but the visual sprite may lag behind or be slightly offset, making it appear disconnected from the collision shape.

*Workaround:* Ensure `destroy_scale` value is reasonable (≥ 0.5 recommended). Test with melee weapons to verify visual alignment.

### 7. **Door Locked State & Powered Flag Mismatch**
During door animation, `door.powered = false` temporarily locks the door, but if a player interacts rapidly, the state machine might not sync properly, leaving the door in a locked state after the animation.

*Workaround:* Always check both `door.locked` and `door.powered` in interaction checks.

### 8. **Sub-Buildings & Loot Spawners**
Sub-buildings can be nested inside buildings, but loot spawners defined at the parent level may not account for nested sub-building positions, resulting in misaligned loot positions.

*Workaround:* Define loot spawners at the deepest building level, not on parent buildings with sub-buildings.

### 9. **Obstacle Variations & Rotation Mode Conflict**
Obstacles with `variations > 0` and `rotationMode: RotationMode.None` may still render visual variations, but collision hitbox won't rotate, leading to asymmetric visuals vs collision.

*Workaround:* Use `rotationMode: RotationMode.Limited` if obstacle can have asymmetric visuals.

### 10. **Cross-Layer Visibility Not Comprehensive**
`visibilityOverrides` can selectively show/hide players between layers, but the implementation only allows explicit allow-lists, not deny-lists. Omitted layers may unexpectedly reveal positions through walls.

*Workaround:* Whitelist all visible layer combinations explicitly in `visibilityOverrides`.

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — building layout in overall map
- [Data Model](../../datamodel.md) — Layer system, ObjectCategory.Building/Obstacle

### Tier 2
- [Object Definitions](../object-definitions/) — dynamic lookup/reification of building/obstacle defs
- [Map](../map/) — building/obstacle generation and placement algorithms
- [Spatial Grid](../spatial-grid/) — collision detection and broad-phase queries
- [Game Loop](../game-loop/) — per-tick updates including door/puzzle state
- [Client Rendering](../client-rendering/) — pool management, sprite rendering, Z-index ordering
- [Visibility & LOS](../visibility-los/) — walls/buildings block sightlines between layers
- [Networking](../networking/) — FullData serialization, UpdatePacket delta sync

### Tier 3
- (Future modules: door-mechanics.md, wall-destruction.md, loot-spawning.md, ceiling-collapse.md, puzzle-system.md)

