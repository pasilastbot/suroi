# Utilities & Support Systems

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: common/src/utils/ and client/src/scripts/utils/ -->

## Purpose

General-purpose utility modules providing terrain queries, vertical layer management, logging infrastructure, animation tweening, PixiJS sprite helpers, and common game-domain functions. These utilities are used across nearly all subsystems.

## Key Files & Entry Points

| Utility | File | Responsibility |
|---------|------|-----------------|
| **Terrain** | `common/src/utils/terrain.ts` | Floor type at position, river queries, collision regions |
| **Layer System** | `common/src/utils/layer.ts` | 5-layer vertical system (basement, ground, upstairs) with predicates |
| **Logging** | `common/src/utils/logging.ts` | Colored console output with timestamps and severity levels |
| **Tween** | `client/src/scripts/utils/tween.ts` | Animation easing and numeric interpolation |
| **PixiJS Utils** | `client/src/scripts/utils/pixi.ts` | Spritesheet loading, sprite building, hitbox rendering |
| **GameHelpers** | `common/src/utils/gameHelpers.ts` | Raycasting, melee hitbox calculations, targeting |
| **Random** | `common/src/utils/random.ts` | Seeded RNG; simple random values (see recap below) |
| **Misc** | `common/src/utils/misc.ts` | Array operations, promise timeouts, deep cloning, queue/stack |

---

## Terrain System

### Purpose

Query what floor type exists at any world position. Shared between server and client. Supports:
- **Floor regions:** grass, sand, water, stone, wood, carpet, metal, ice, void
- **River queries:** Find rivers overlapping a position or hitbox
- **Collision regions:** Beach and grass boundaries for spawn/loot logic

### Main Class: `Terrain`

```typescript
terrain = new Terrain(
  width: number,
  height: number,
  oceanSize: number,
  beachSize: number,
  seed: number,
  rivers: readonly River[],
  waterType?: FloorNames
)
```

#### Core Query Methods

```typescript
// Get floor type at position on a specific layer
getFloor(position: Vector, layer: number): FloorNames

// Find all rivers intersecting a world position
getRiversInPosition(position: Vector): River[]

// Find all rivers intersecting a hitbox
getRiversInHitbox(hitbox: Hitbox, ignoreTrails?: boolean): River[]
```

#### Floor Types

```typescript
export const enum FloorNames {
  Grass = "grass",    // Normal terrain
  Stone = "stone",    // Hard surface, no speed penalty
  Wood = "wood",      // Wood floor, medium traction
  Log = "log",        // Fallen log, rough surface
  Sand = "sand",      // Beach/river banks (special audio)
  Metal = "metal",    // Metallic surfaces
  Carpet = "carpet",  // Interior flooring
  Water = "water",    // Rivers/lakes (72% speed, overlay effect)
  Ice = "ice",        // Slippery (no speed penalty, slippery: true)
  Void = "void"       // Out-of-bounds / higher floors
}
```

### Floor Definition Properties

Each floor type defines:
- `debugColor` — hex color for debug visualization
- `speedMultiplier?` — movement speed scalar (water: 0.72)
- `overlay?` — render water overlay effect
- `particles?` — show particle effects (water spray)
- `slippery?` — reduce friction/traction (ice)

### Example: Get Floor at Position

```typescript
// Server: on each game tick, check what floor a player is standing on
const floorType = terrain.getFloor(player.position, player.layer);
const floorDef = FloorTypes[floorType];

// Apply speed modifier
player.speedMod *= floorDef.speedMultiplier ?? 1.0;

// Show water overlay on client if water
if (floorType === FloorNames.Water && floorDef.overlay) {
  // Render water effect
}
```

### Terrain Grid Optimization

Terrain internally uses a spatial grid (`cellSize = 64px`) to accelerate queries:
- Divides world into 64×64 cells
- Each cell caches which rivers and floors intersect it
- Query: round position to cell, check that cell's data
- **Gotcha:** Large obstacles added to terrain after construction may not be in grid cells they should occupy

---

## Layer System

### Purpose

Vertical layer management for multi-floor maps (basements, ground level, upstairs). Handles:
- Collision between layers (same layer vs. staircase transitions)
- Visibility culling (don't render upstairs when underground)
- Movement restrictions (can't walk through basement floor to ground)

### Layer Enum

```typescript
export enum Layer {
  Basement = -2,     // Underground level
  ToBasement = -1,   // Stairs/ramp going down
  Ground = 0,        // Main outdoor level
  ToUpstairs = 1,    // Stairs/ramp going up
  Upstairs = 2       // Second floor / building interior
}
```

### Layer Classification Functions

```typescript
// Ground floors: Basement (-2), Ground (0), Upstairs (2)
isGroundLayer(layer: number): boolean
  // Returns: layer % 2 === 0

// Stair layers: ToBasement (-1), ToUpstairs (1)
isStairLayer(layer: number): boolean
  // Returns: layer % 2 !== 0

// Exact layer equality
equalLayer(ref: Layer, eval: Layer): boolean

// Same layer OR one level above
equalOrOneAboveLayer(ref: Layer, eval: Layer): boolean
  // Returns: ref === eval || ref + 1 === eval

// Same layer OR one level below
equalOrOneBelowLayer(ref: Layer, eval: Layer): boolean
  // Returns: ref === eval || ref - 1 === eval

// Adjacent layers (one above/below, not including same)
isAdjacent(num1: Layer, num2: Layer): boolean
  // Returns: (num1 - 1 === num2) || (num1 + 1 === num2)

// Same OR adjacent (includes walls and stairs)
adjacentOrEqualLayer(ref: Layer, eval: Layer): boolean
  // Returns: (ref - 1 === eval) || (ref + 1 === eval) || (ref === eval)
```

### Layer Collision Rules

Objects use a `collideWithLayers` definition property to control what they interact with:

```typescript
export const enum Layers {
  All,      // Collide objects on ALL layers (e.g., poison gas)
  Adjacent, // Collide on same/adjacent layers (most stairs, walls)
  Equal     // Only collide on same layer (default)
}
```

Stairs (`isStair: true`) always use `Adjacent` logic, allowing transition between layers.

### Example: Can Two Objects Collide?

```typescript
function canCollide(obj1: GameObject, obj2: GameObject): boolean {
  // Check if their layers are compatible
  const collideRule = obj1.definition.collideWithLayers ?? Layers.Equal;
  
  switch (collideRule) {
    case Layers.All:
      return true;
    case Layers.Adjacent:
      return adjacentOrEqualLayer(obj1.layer, obj2.layer);
    case Layers.Equal:
    default:
      return equalLayer(obj1.layer, obj2.layer);
  }
}
```

### Gotchas

1. **Layer is discrete** — No smooth Z-offset transitions; you're either on Basement or Ground, never in-between
2. **Stair semantics** — Objects with `isStair: true` ignore `collideWithLayers` setting; always allow adjacent collision
3. **Basement access** — Only stairs with `ToBasement` layer allow basement access; fall damage may prevent safe descent

---

## Logging System

### Purpose

Structured colored console output for debugging and diagnostics. Provides:
- **Color codes** — ANSI 256-color escape sequences
- **Timestamps** — ISO date + locale time
- **Log levels** — Info, Warning, Error
- **Font styles** — Bold, italic, underline, strikethrough

### Logger Interface

```typescript
Logger.log(...message: unknown[]): void    // Blue timestamp + message
Logger.warn(...message: unknown[]): void   // Yellow [WARNING] + message
Logger.error(...message: unknown[]): void  // Red [ERROR] + message
```

### Color & Font Styles

```typescript
const ColorStyles = {
  foreground: {
    black: { normal: 30, bright: 90 },
    red: { normal: 31, bright: 91 },
    green: { normal: 32, bright: 92 },
    yellow: { normal: 33, bright: 93 },
    blue: { normal: 34, bright: 94 },
    magenta: { normal: 35, bright: 95 },
    cyan: { normal: 36, bright: 96 },
    white: { normal: 37, bright: 97 },
    default: { normal: 39, bright: 39 }
  },
  background: { /* similar structure */ }
};

const FontStyles = {
  bold: 1,
  faint: 2,
  italic: 3,
  underline: 4,
  blinkSlow: 5,
  blinkFast: 6,
  invert: 7,
  conceal: 8,
  strikethrough: 9,
  overlined: 53
};

// Apply formatting
styleText(string, ColorStyles.foreground.red.bright, FontStyles.bold)
```

### Example Usage

```typescript
Logger.log("Server started on port 3000");
Logger.warn("Player connection slow:", latency, "ms");
Logger.error("Database connection failed:", err);
```

---

## Tween System

### Purpose

Smooth numeric animation over time. Used for:
- Camera zoom transitions
- Health bar fills
- UI element fades and slides
- Any property needing eased interpolation

### Tween Class

```typescript
tween = new Tween<T>({
  target: object,                     // Object to animate
  to: { propertyName: endValue },     // Target property values
  duration: 500,                      // Milliseconds
  ease?: (x: number) => number,       // Easing function (0 → 1 → 0-1)
  yoyo?: boolean,                     // Reverse at end?
  infinite?: boolean,                 // Loop forever?
  onUpdate?: () => void,              // Update callback each frame
  onComplete?: () => void             // Completion callback
})
```

### Core Methods

```typescript
tween.update(): void       // Call each frame; updates target properties
tween.complete(): void     // Force completion; trigger onComplete
tween.kill(): void         // Stop tween and remove from Game manager
```

### Easing Functions

Tween accepts a custom easing function: `(progress: 0.0–1.0) => 0.0–1.0`

Common easing functions (not built-in; provide your own):
- `linear: (t) => t` — constant speed
- `easeInQuad: (t) => t * t` — accelerate
- `easeOutQuad: (t) => 1 - (1-t)*(1-t)` — decelerate
- `easeInOutQuad: (t) => t < 0.5 ? 2*t*t : -1 + (4-2*t)*t` — smooth both ends

Default (if `ease` not provided): `(t) => t` (linear)

### Example: Camera Zoom

```typescript
new Tween({
  target: camera,
  to: { zoom: 2.0 },
  duration: 300,
  ease: (t) => t < 0.5 ? 2*t*t : -1 + (4-2*t)*t, // easeInOutQuad
  onComplete: () => console.log("Zoom complete")
});
```

### Gotchas

1. **Container.destroyed check** — Tweens check if target is destroyed PixiJS Container; if so, auto-kill to avoid crashes
2. **Custom easing only** — No built-in easing library; define your own functions
3. **yoyo + infinite** — If `yoyo: true` and `infinite: true`, reverses indefinitely

---

## PixiJS Utilities

### Purpose

Helper functions for:
- Spritesheet loading (hi-res 4096×4096, lo-res 2048×2048)
- Sprite creation with fluent API
- Hitbox visualization (draw for debug)
- Coordinate conversion (world ↔ canvas pixel)

### Spritesheet Loading

```typescript
// Load spritesheets for a game mode
loadSpritesheets(
  modeName: ModeName,
  renderer: Renderer,
  highResolution: boolean
): Promise<void>
```

Logic:
- Checks `MAX_TEXTURE_SIZE` capability; forces lo-res (2048×2048) if < 4096
- Loads virtual modules: `virtual:image-spritesheets-importer-high-res` or `virtual:image-spritesheets-importer-low-res`
- Parses each spritesheet and caches textures in PixiJS Assets
- Fires `spritesheetsLoaded = true` when complete

### SuroiSprite Class

Builder-pattern sprite wrapper around PixiJS `Sprite`:

```typescript
sprite = new SuroiSprite("frame_name")

sprite.setFrame(frame: string): this           // Change texture
sprite.setPos(x: number, y: number): this      // Move (pixels)
sprite.setVPos(pos: Vector): this              // Move (Vector)
sprite.setAnchor(anchor: Vector): this         // Rotation/scale pivot
sprite.setPivot(pivot: Vector): this           // Offset pivot
sprite.setVisible(visible: boolean): this      // Show/hide
sprite.setAngle(angle?: number): this          // Rotation in degrees
sprite.setRotation(rotation?: number): this    // Rotation in radians
sprite.setScale(scaleX?: number, scaleY?: number): this    // Scale
sprite.setTint(tint: ColorSource): this        // Color overlay
sprite.setZIndex(zIndex: number): this         // Depth ordering
sprite.setAlpha(alpha: number): this           // Opacity (0–1)
```

All methods return `this` for chaining.

### Coordinate Conversion

```typescript
// Convert world coordinates to PixiJS canvas coordinates
toPixiCoords(pos: Vector): Vector
  // Returns: Vec.scale(pos, PIXI_SCALE)

// PIXI_SCALE is defined per build (usually 1.0 for client)
```

### Hitbox Visualization (Debug)

```typescript
// Draw hitbox outline on Graphics object
drawGroundGraphics(
  hitbox: Hitbox,
  graphics: Graphics,
  scale?: number = PIXI_SCALE
): void

// Trace hitbox path (move/line/poly)
traceHitbox<T extends Graphics>(
  hitbox: Hitbox,
  graphics: T
): T

// Draw filled/stroked hitbox
drawHitbox<T extends Graphics>(
  hitbox: Hitbox,
  color: ColorSource,
  graphics: T,
  alpha?: number
): T
```

### Gotchas

1. **Spritesheet device limits** — If GPU doesn't support 4096×4096 textures, automatically switches to 2048×2048; affects asset resolution silently
2. **Texture cache keyed by frame name** — Changing a frame name breaks existing sprite references
3. **SuroiSprite destroys only Pixi Container** — If texture/spritesheet changes at runtime, sprite won't update (use `setFrame()`)
4. **toPixiCoords assumes PIXI_SCALE constant** — If scale changes at runtime, coordinate conversion breaks

---

## GameHelpers

### Purpose

Game-domain utility functions for:
- Raycasting (line-of-sight, bullet paths)
- Melee collision detection
- Target acquisition

### Core Functions

```typescript
// Cast a ray from start to end; find nearest solid object hit
raycastSolids<T extends CommonGameObject>(
  objects: Iterable<T>,
  start: Vector,
  end: Vector,
  filterFn: (object: T & Solid) => boolean
): RaycastResponse<T & Solid> | null

// Returns: { object, position: hit point, normal, distance }
```

Used for:
- Sniper rifle shots (find what's in line)
- Explosion radii (trace rays to find blocked areas)
- AI line-of-sight checks

```typescript
// Calculate melee attack hitbox relative to player
getMeleeHitbox(
  player: CommonObjectMapping[ObjectCategory.Player],
  definition: MeleeDefinition
): CircleHitbox

// Returns: Circle at (player.position + offset rotated by player.rotation)
```

```typescript
// Find all objects within melee attack hitbox
getMeleeTargets<T extends MeleeObject>(
  hitbox: CircleHitbox,
  definition: MeleeDefinition,
  player: CommonObjectMapping[ObjectCategory.Player],
  teamMode: boolean,
  objects: Iterable<CommonGameObject>,
  debugRenderer?: { addLine: (a: Vector, b: Vector, color: number, alpha?: number) => void }
): MeleeTarget<T>[]

// Returns: List of targets (object, position, direction, distance)
// Filters out: dead objects, self, non-damageable, stairs, different layers
```

---

## Miscellaneous Utilities

### Array Operations

```typescript
// Remove item from array (mutating)
removeFrom<T>(array: T[], value: T): void
  // Example: removeFrom(players, deadPlayer)

// Partition array by predicate
splitArray<T>(
  target: T[],
  predicate: (ele: T, index: number, array: T[]) => boolean
): { true: T[], false: T[] }

// Group array by mapper function
groupArray<In, Out>(
  target: In[],
  picker: (ele: In, index: number) => Out
): Map<Out, In[]>
  // Example: groupArray(players, p => p.team) → Map<Team, Player[]>
```

### Object Deep Operations

```typescript
// Recursive deep copy
cloneDeep<T>(object: T): T

// Deep freeze (all nested properties readonly)
freezeDeep<T>(object: T): DeepReadonly<T>

// Merge multiple objects (deep merge)
mergeDeep<T extends object>(
  target: T,
  ...sources: readonly DeepPartial<T>[]
): T
```

### Timeout/Callbacks

```typescript
// Cancellable timeout callback
class Timeout {
  callback: () => void
  end: number              // Absolute time to fire
  killed: boolean
  
  kill(): void             // Cancel this timeout
}
```

Not a Promise; designed for game tick/frame callbacks.

### Collections

```typescript
// Extended Map with get-or-set pattern
class ExtendedMap<K, V> extends Map<K, V> {
  getOrInsert(key: K, supplier: () => V): V
}

// FIFO queue
class Queue<T> {
  enqueue(item: T): void
  dequeue(): T | undefined
  peek(): T | undefined
  size: number
}

// LIFO stack
class Stack<T> {
  push(item: T): void
  pop(): T | undefined
  peek(): T | undefined
  size: number
}

// Linked-list node (immutable)
interface LinkedList<T> {
  readonly value: T
  next?: LinkedList<T>
}
```

---

## Random Utilities (Recap)

See [Core Math & Physics — Random](../core-math-physics/README.md#random-number-generation) for detailed docs.

Brief summary:

```typescript
// Pseudo-random float [min, max)
randomFloat(min: number, max: number): number

// Pseudo-random integer [min, max]
random(min: number, max: number): number

// Random boolean (50% chance)
randomBoolean(): boolean

// Random sign: -1 or 1
randomSign(): -1 | 1

// Random vector with per-axis range
randomVector(minX: number, maxX: number, minY: number, maxY: number): Vector

// Seeded RNG (deterministic)
class SeededRandom {
  get(min: number, max: number): number
}
```

---

## Dependencies

### Depends on:
- [Core Math & Physics](../core-math-physics/) — Vector, hitbox geometry, `Numeric.clamp`, `Numeric.lerp`, angle math
- [Object Definitions](../object-definitions/) — Definition types, `ObjectCategory`

### Depended on by:
Nearly all subsystems use at least one utility:
- [Game Loop](../game-loop/) — terrain, layer, logging
- [Networking](../networking/) — logging
- [Spatial Grid](../spatial-grid/) — math utilities
- [Client Rendering](../client-rendering/) — PixiJS utils, tweens, sprites
- [Game Objects (Server/Client)](../game-objects-server/, ../game-objects-client/) — gameHelpers, terrain, layer
- [Map Generation](../map/) — terrain, random, logging
- [Gas System](../gas/) — layer system
- [Inventory](../inventory/) — misc utilities

---

## Known Issues & Gotchas

### Terrain

1. **Quadrant size fixed** — `cellSize = 64px` can cause uneven distribution if map dimensions don't align
2. **Rivers added at construction only** — Adding rivers after Terrain creation won't update grid cells
3. **Floor type fallback** — If no floor hitbox at position, defaults to water (layer 0) or void (higher layers)
4. **TODO: Winter mode floor detection** — Code has TODO comment about toggling grass/sand based on season

### Layer System

1. **Discrete layers** — No smooth Z-offset; objects are fully on one layer or transitioning via stairs
2. **Stair semantics complex** — `isStair: true` + `collideWithLayers` combo is implicit; easy to forget implications
3. **Off-map behavior undefined** — No safeguard for layer queries outside valid range (-2 to 2)

### Logging

1. **ANSI codes may not work universally** — Console color codes are terminal/environment-specific; may not work in browser, cloud logs
2. **Performance note** — Heavy logging in tight loops (e.g., every game tick for every object) can impact perf

### Tweens

1. **No built-in easing library** — You must define easing functions; common tweening libraries (GSAP, Anime.js) not included
2. **Destroyed container check only** — Only detects destroyed PixiJS Container; plain JS objects can't detect lifecycle
3. **yoyo swaps start/end** — `yoyo: true` reverses direction; can cause unexpected stuttering if duration is very short
4. **No pause/resume** — Once started, tween runs to completion; no pause API

### PixiJS Utilities

1. **Spritesheet resolution switch is silent** — Device without 4096×4096 support auto-downgrades; hard to debug if assets look blurry
2. **toPixiCoords assumes constant PIXI_SCALE** — If rendering at different scales (e.g., mini-map), conversions break
3. **SuroiSprite texture caching** — Changing frame names after sprite creation breaks lookups (use `setFrame()` to update)
4. **Debug hitbox functions use PIXI_SCALE** — If you change scale in Game, debug drawing won't match world coordinates

### GameHelpers

1. **Raycast doesn't account for layer** — Must manually filter by layer before raycasting
2. **Melee targets include dead objects if not filtered** — `getMeleeTargets` respects `object.dead` flag, but only if passed to function

### Miscellaneous

1. **Timeout not a Promise** — Don't mix with `async/await`; designed for frame-based callbacks
2. **ExtendedMap.getOrInsert creates even if not used** — Supplier called immediately; avoid expensive operations
3. **cloneDeep doesn't clone Functions** — Functions are shared references
4. **freezeDeep shallow if property is Object** — Works recursively but only if properties are plain objects

---

## Related Documents

### Tier 1
- [Architecture](../../architecture.md) — System overview
- [Data Model](../../datamodel.md) — Layer enum, Vector type, FloorNames enum

### Tier 2
- [Core Math & Physics](../core-math-physics/) — Vector, angle, hitbox math (foundational for terrain, gameHelpers)
- [Object Definitions](../object-definitions/) — Definition types, ObjectCategory
- [Client Rendering](../client-rendering/) — Uses PixiJS utils, tweens for animations
- [Game Loop](../game-loop/) — Uses terrain, layer for collision checks

### Tier 3
See module-level docs for TerrainQueries, LayerPredicates, etc. (if documented)

### Patterns
See [patterns.md](patterns.md) (if created) for reusable utility patterns
