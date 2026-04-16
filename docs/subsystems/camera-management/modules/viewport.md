# Viewport Positioning & Zoom Effects

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/camera-management/README.md -->
<!-- @source: client/src/scripts/managers/cameraManager.ts, client/src/scripts/game.ts, client/src/scripts/utils/constants.ts -->

## Purpose

This module manages the camera viewport — positioning, zoom levels, screen-to-world transformations, and visual effects like shake and shockwaves. It bridges the gap between the game world coordinate system and the pixel-based rendering pipeline, handling all viewport calculations that determine what players see and how it's transformed.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/cameraManager.ts` | Viewport manager, zoom control, shake/shockwave effects | High |
| `client/src/scripts/game.ts` (lines 1204, 1230) | Camera update loop, position synchronization | Medium |
| `client/src/scripts/utils/pixi.ts` | Coordinate transformations (world ↔ screen) | Medium |
| `client/src/scripts/utils/constants.ts` | PIXI_SCALE, layer transition timing | Low |
| `common/src/constants.ts` | Game constants (grid, max position) | Low |

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│ CameraManager (Viewport Control)                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  position: Vec(x, y)               ← Player follows    │
│  zoom: number                       ← Scope-based      │
│  width, height: number              ← Screen size      │
│  container: Container               ← PixiJS root      │
│  layerContainers[3]: Container[]    ← Layer rendering  │
│                                                         │
│  ┌─────────────────────────────────────────────────────┤
│  │  Visual Effects                                     │
│  ├─────────────────────────────────────────────────────┤
│  │  shaking: boolean → randomPointInsideCircle()      │
│  │  shockwaves: Set<Shockwave> → ShockwaveFilter      │
│  └─────────────────────────────────────────────────────┘
│                                                         │
└─────────────────────────────────────────────────────────┘
         ↓
    Container.scale.set(scale)    ← Transform applied
         ↓
    Game.pixi.render()            ← Final output
```

## 1. Viewport Structure

### Core Properties

```typescript
class CameraManager {
    // Position in world coordinates
    position = Vec(0, 0);  // {x, y}

    // Zoom level (from weapon scope definition)
    private _zoom = DEFAULT_SCOPE.zoomLevel;
    get zoom(): number { return this._zoom; }
    set zoom(zoom: number) {
        this._zoom = zoom;
        this.resize(true);  // Triggers animated transition
    }

    // Screen dimensions in pixels
    width = 1;   // Game.pixi.screen.width
    height = 1;  // Game.pixi.screen.height

    // PixiJS container hierarchy
    container = new Container();  // Root transform node
    layerContainers[3] = [basement, ground, upstairs];
    tempLayerContainer = new Container();  // Cross-layer transitions
}
```

### Screen-to-World Transformation

**Constants:**
- `PIXI_SCALE = 20` — World unit to pixel multiplier (in `utils/constants.ts`)
- `GameConstants.gridSize = 32` — Spatial cell size
- `GameConstants.maxPosition = 1924` — Map boundary

**Coordinate Systems:**

```
World Space    (Game logic)
     ↓
     × PIXI_SCALE
     ↓
Pixi Space     (Rendering)
     ↓
     × container.scale
     ↓
Screen Space   (Viewport)
```

**Transformation Functions** (@file `client/src/scripts/utils/pixi.ts`):

```typescript
// World → Pixi: scale by PIXI_SCALE (20)
export function toPixiCoords(pos: Vector): Vector {
    return Vec.scale(pos, PIXI_SCALE);
}

// Pixi → World: scale by 1/PIXI_SCALE
fromPixiCoords(pixiPos: Vector): Vector {
    return Vec.scale(pixiPos, 1 / PIXI_SCALE);
}
```

**Practical Example:**
```
Player world position: (100, 200)
  → toPixiCoords() → Pixi position: (2000, 4000)
  → container.position applied
  → Screen displays at transformed pixel location
```

## 2. Camera Centering & Following Player

### Position Update Loop

The camera follows the player with two modes:

**Mode 1: With Movement Smoothing** (default, `cv_movement_smoothing = true`)

```typescript
// @file client/src/scripts/game.ts:1204
if (hasMovementSmoothing && this.activePlayer) {
    CameraManager.position = this.activePlayer.container.position;
}
```

- Camera updates **per frame** to player's **rendered position** (Pixi space)
- Smooth interpolation happens via object's `updateInterpolation()` method
- Position is already in Pixi coordinates

**Mode 2: No Smoothing** (legacy, `cv_movement_smoothing = false`)

```typescript
// @file client/src/scripts/objects/player.ts:451
if (noMovementSmoothing && !(this.isActivePlayer && cv_responsive_rotation)) {
    CameraManager.position = toPixiCoords(this.position);
}
```

- Direct world → Pixi conversion
- Teleports camera instantly to player (jerky motion)
- Used when client-side interpolation is disabled

### Viewport Center Calculation

When camera updates, position determines viewport center:

```typescript
// @file client/src/scripts/managers/cameraManager.ts:107
const cameraPos = Vec.add(
    Vec.scale(position, this.container.scale.x),  // Apply zoom
    Vec(-this.width / 2, -this.height / 2)        // Center on screen
);
this.container.position.set(-cameraPos.x, -cameraPos.y);
```

**Breaking it down:**
- `position`: camera world position (what player sees)
- `container.scale.x`: zoom factor (larger zoom = smaller scale)
- `width/2, height/2`: offset to center viewport
- Negation: moves container to show desired region

## 3. Zoom System

### Scope-Based Zoom Levels

Zoom is controlled by weapon scopes (@file `common/src/definitions/items/scopes.ts`):

```typescript
interface ScopeDefinition {
    zoomLevel: number;  // e.g., 1.0 (default), 1.5x, 2.0x
}

export const DEFAULT_SCOPE = {
    zoomLevel: 1.0  // No scope
};

// Examples:
// Red dot: 1.2x
// 2x scope: 2.0x
// 8x scope: 8.0x
```

### Zoom Scale Formula

```typescript
// @file client/src/scripts/managers/cameraManager.ts:76
resize(animation = false): void {
    this.width = Game.pixi.screen.width;
    this.height = Game.pixi.screen.height;

    // Ensure viewport respects 16:9 aspect ratio minimum
    const minDim = Numeric.min(this.width, this.height);
    const maxDim = Numeric.max(this.width, this.height);
    const maxScreenDim = Numeric.max(minDim * (16 / 9), maxDim);

    // Scale calculation
    const scale = (maxScreenDim * 0.5) / (this.zoom * PIXI_SCALE);

    if (animation) {
        // Animated zoom transition (1250ms, eased)
        this.zoomTween = Game.addTween({
            target: this.container.scale,
            to: { x: scale, y: scale },
            duration: 1250,
            ease: EaseFunctions.cubicOut
        });
    } else {
        this.container.scale.set(scale);  // Instant zoom
    }
}
```

**Key Insights:**
- `maxScreenDim * 0.5`: Base viewport size (half the screen dimension)
- Divided by `(zoom * PIXI_SCALE)`: Zoom decreases scale (more world visible)
- Aspect ratio enforcement: prevents viewport stretching
- `cubicOut` easing: smooth deceleration when transitioning between scopes

### Zoom Tween Lifecycle

```typescript
zoomTween?: Tween<Vector>;

// When scope changes:
set zoom(zoom: number) {
    this._zoom = zoom;
    this.resize(true);  // Starts new tween
}

// Tween callbacks:
onComplete: () => {
    this.zoomTween = undefined;  // Cleanup
}
```

**Behavior:**
- Only one zoom tween active at a time (previous is killed)
- Scope change → smooth 1.25s transition
- While transitioning: both scale values interpolate
- On completion: reference cleared, ready for next scope change

## 4. Shake Effect

### Triggering Camera Shake

```typescript
// @file client/src/scripts/managers/cameraManager.ts:277
shake(duration: number, intensity: number): void {
    if (!GameConsole.getBuiltInCVar("cv_camera_shake_fx")) return;
    this.shaking = true;
    this.shakeStart = Date.now();
    this.shakeDuration = duration;
    this.shakeIntensity = intensity;
}
```

**Typical usage** (@file `client/src/scripts/objects/player.ts:1997`):

```typescript
if (weaponDef.cameraShake !== undefined) {
    CameraManager.shake(
        weaponDef.cameraShake.duration,    // e.g., 150ms
        weaponDef.cameraShake.intensity    // e.g., 3 units
    );
}
```

### Shake Application & Decay

```typescript
// @file client/src/scripts/managers/cameraManager.ts:107
update(): void {
    let position = this.position;

    if (this.shaking) {
        // Add random offset within circle
        position = Vec.add(
            position,
            randomPointInsideCircle(Vec(0, 0), this.shakeIntensity)
        );

        // Check if shake duration expired
        if (Date.now() - this.shakeStart > this.shakeDuration) {
            this.shaking = false;
        }
    }

    // ... rest of position calculation
}
```

**How It Works:**
1. Each frame, random point is generated inside circle of `shakeIntensity` radius
2. Point is added to camera position (offset effect)
3. When `duration` elapses, `shaking` flag is cleared
4. No fade — shake is either on or off (instantaneous decay)

**Visual Characteristic:**
- Uniform random distribution (circular pattern)
- Fixed intensity for entire duration (no decay curve)
- Off-on switch (not gradual)
- Example: 150ms shake at intensity 3 = position ±3 units each frame for 150ms

## 5. Shockwave Effects

### Shockwave Registration

```typescript
// @file client/src/scripts/managers/cameraManager.ts:281
shockwave(
    duration: number,
    position: Vector,
    amplitude: number,
    wavelength: number,
    speed: number,
    layer: Layer
): void {
    this.shockwaves.add(new Shockwave(
        duration, position, amplitude, wavelength, speed, layer
    ));
}
```

**Typical usage** (@file `client/src/scripts/objects/explosion.ts:90`):

```typescript
CameraManager.shockwave(
    definition.cameraShake.duration,  // Lifetime
    epicenterWorldPos,                 // Effect center
    1.0,                               // Wave amplitude
    100.0,                             // Wavelength in pixels
    500.0,                             // Speed
    layer                              // Which layer
);
```

### Shockwave Update & Rendering

```typescript
// @file client/src/scripts/managers/cameraManager.ts:239
class Shockwave {
    lifeStart: number;
    lifeEnd: number;
    filter: ShockwaveFilter;  // PixiJS filter
    anchorContainer: SuroiSprite;

    constructor(lifetime, position, amplitude, wavelength, speed, layer) {
        this.lifeStart = Date.now();
        this.lifeEnd = this.lifeStart + lifetime;
        this.anchorContainer = new SuroiSprite();

        // Add sprite to layer (serves as anchor point)
        CameraManager.getContainer(layer).addChild(this.anchorContainer);
        this.anchorContainer.setVPos(position);

        // Create filter + add to layer
        this.filter = new ShockwaveFilter();
        CameraManager.addFilter(layer, this.filter);
    }

    update(): void {
        const now = Date.now();

        // Auto-cleanup on expiration
        if (now > this.lifeEnd) {
            this.destroy();
            return;
        }

        // Calculate elapsed progress (0 → 1)
        const elapsed = (now - this.lifeStart) / (this.lifeEnd - this.lifeStart);
        const scope = 1 - elapsed;

        // Scale parameters by scope (wave expands & decays)
        this.filter.wavelength = this.wavelength * scale;
        this.filter.speed = this.speed * scale;
        this.filter.amplitude = this.amplitude * scope * EaseFunctions.linear(scope);

        // Update filter center (follows anchor sprite)
        const position = this.anchorContainer.getGlobalPosition();
        this.filter.centerX = position.x;
        this.filter.centerY = position.y;

        this.filter.time = now - this.lifeStart;
    }

    destroy(): void {
        CameraManager.removeFilter(this.layer, this.filter);
        CameraManager.shockwaves.delete(this);
        this.anchorContainer.destroy();
    }
}
```

**Visual Mechanics:**
- Shockwave expands from epicenter over `lifetime` milliseconds
- Amplitude decays linearly: starts full, ends at 0
- Wavelength & speed scale with remaining time
- Per-layer application: different layers can have different intensity
- Auto-cleanup: removed when lifeEnd is exceeded

## 6. Pan & Transition Effects

### Map Load Transition

When joining game (@file `client/src/scripts/managers/cameraManager.ts:160`):

```typescript
init(): void {
    for (let i = 0; i < this.layerContainers.length; i++) {
        const container = this.layerContainers[i];
        container.zIndex = i;
        this.container.addChild(container);
    }
    this.container.addChild(this.tempLayerContainer);
}
```

- All layer containers initialized and added to main container
- `tempLayerContainer` for smooth cross-layer transitions
- Initial zoom set to `DEFAULT_SCOPE.zoomLevel`

### Death & Spectating (Pan Without Player)

When player dies, camera can continue viewing area or follow other players:

```typescript
// @file client/src/scripts/utils/tween.ts (referenced)
// Spectating implements custom pan logic via tweens
```

**Behavior varies by game mode** — some modes pan to death location, others follow teammate.

## 7. Layer Management & Transitions

### Layer Container Hierarchy

```typescript
readonly layerContainers = [
    new Container(),  // [0] = Basement (Layer.ToBasement=-1)
    new Container(),  // [1] = Ground (Layer.Ground=0)
    new Container()   // [2] = Upstairs (Layer.ToUpstairs=1)
];

readonly tempLayerContainer = new Container();
```

### Layer Visibility Transitions

```typescript
// @file client/src/scripts/managers/cameraManager.ts:163
updateLayer(newLayer: Layer, initial = false, oldLayer?: Layer): void {
    for (let i = 0; i < this.layerContainers.length; i++) {
        const container = this.layerContainers[i];

        // Determine visibility based on new layer
        let visible = true;
        switch (i) {
            case 0:  // Basement
                visible = newLayer <= Layer.ToBasement;
                break;
            case 1:  // Ground
                visible = newLayer >= Layer.ToBasement;
                break;
            case 2:  // Upstairs
                visible = newLayer >= (Game.hideSecondFloor ? ... : ...);
                break;
        }

        const targetAlpha = visible ? 1 : 0;

        // Fade transition over LAYER_TRANSITION_DELAY (200ms)
        this.layerTweens[i] = Game.addTween({
            target: container,
            to: { alpha: targetAlpha },
            duration: LAYER_TRANSITION_DELAY,  // 200ms
            ease: EaseFunctions.sineOut
        });
    }
}
```

**Transitions:**
- Smooth fade in/out (not instant)
- Fading layer becomes visible=true initially (to show transition)
- On completion: set proper visible state
- Prevents flickering when objects move between layers

### Z-Index Management

```typescript
// Basement shows above all when on stairs or below
if (i === 0 && container.zIndex !== 999) {
    container.zIndex = newLayer <= Layer.ToBasement ? 999 : 0;
}
```

**Basement Z-Order:**
- Normal: zIndex = 0 (under ground layer)
- When entering bunker: zIndex = 999 (above all)
- When exiting: reverts to 0

### Temp Layer for Cross-Layer Transitions

When object moves between layers during transition:

```typescript
// Use temp container with intermediate Z-index
let tempContainerIndex = getLayerContainerIndex(Game.layer, Game.layer);
// ... complex logic to find best z-index ...
tempContainerIndex += 0.1;  // Slight offset
this.tempLayerContainer.zIndex = tempContainerIndex;
```

**Why:** Prevents object flickering when moving from fading-out layer to fading-in layer.

## 8. Bounds Checking

### Map Boundary Enforcement

Camera position is constrained by map size (@file `common/src/constants.ts`):

```typescript
GameConstants.maxPosition = 1924;  // World units
```

**Implementation:**
- No explicit clamping in CameraManager
- Player itself is constrained by collision system
- Camera follows player → implicitly bounded

**Viewport doesn't pan beyond map edges:**
- Map is typically square (0 → maxPosition)
- Player spawns at valid positions
- Gas shrinks toward map center (confines play area)

## 9. Rendering Transform (World → Screen)

### Container Transform Pipeline

```
CameraManager.container (Root)
    ├── scale: {x, y}          ← Zoom factor
    ├── position: {x, -y}      ← Camera offset
    │
    ├── layerContainers[0]     ← Basement (z=0 or 999)
    ├── layerContainers[1]     ← Ground (z=1)
    ├── layerContainers[2]     ← Upstairs (z=2)
    └── tempLayerContainer     ← Temp during transitions

        └── Game objects (Players, Obstacles, etc.)
            └── SuroiSprite.position = toPixiCoords(world.position)
```

### Transformation Example

```
World object at (100, 200):
  1. toPixiCoords(100, 200) → (2000, 4000)
  2. container.scale.set(0.5) [zoomed in]
  3. container.position.set(-100, -100)
  4. Final screen position = (2000*0.5 - 100, 4000*0.5 - 100)
                           = (900, 1900)
```

## 10. Mouse Positioning

### Screen → World Conversion

```typescript
// @file client/src/scripts/managers/inputManager.ts:376-378
gameMousePosition() {
    const pixiPos = CameraManager.container.toLocal(this.mousePosition);
    this.gameMousePosition = Vec.scale(pixiPos, 1 / PIXI_SCALE);
    this.distanceToMouse = Geometry.distance(Game.activePlayer.position, this.gameMousePosition);
}
```

**Step-by-step:**

1. **Pixel coordinates** → Mouse event `(clientX, clientY)` on screen
2. **Pixi coordinates** → `container.toLocal(mouseScreenPos)` (undoes container transform)
3. **World coordinates** → `Vec.scale(pixiPos, 1 / PIXI_SCALE)` (undoes PIXI_SCALE)
4. **Distance calculation** → Geometry.distance between player and aimed position

**Example:**

```
Mouse at screen (500, 300)
  → container.toLocal() → Pixi (2500, 1500)
  → scale by 1/20 → World (125, 75)
  → distance to player at (100, 100) = sqrt((125-100)² + (75-100)²) ≈ 39.1

Player gun fires toward (125, 75) in world space
```

### Distance to Mouse (Game Logic)

Used for weapon accuracy, aiming feedback, and rotation speed:

```typescript
this.distanceToMouse = Geometry.distance(
    Game.activePlayer.position,
    this.gameMousePosition
);
```

**Constraint:** (`GameConstants.player.maxMouseDist = 256`)
- Server caps distance to prevent far-off aiming exploits
- Limits effective aiming distance to ~256 world units

## 11. Mobile Viewport Handling

### Aspect Ratio Enforcement

```typescript
// @file client/src/scripts/managers/cameraManager.ts:79-82
const minDimension = Numeric.min(this.width, this.height);
const maxDimension = Numeric.max(this.width, this.height);
const maxScreenDim = Numeric.max(minDimension * (16 / 9), maxDimension);
```

**Logic:**
- Enforces 16:9 minimum aspect ratio
- On mobile (taller than wide), adds extra height
- Prevents distorted viewport on ultra-wide/ultra-tall screens

### Mobile UI Positioning

```typescript
// @file client/src/scripts/managers/cameraManager.ts:92-94
if (InputManager.isMobile) {
    EmoteWheelManager.container.position =
        MapPingWheelManager.position =
        Vec(Game.pixi.screen.width / 2, Game.pixi.screen.height / 2);
}
```

- Wheels centered on screen on mobile devices
- Desktop: wheels appear at mouse position (via InputManager)

### Pinch Zoom (Not Currently Implemented)

Suroi viewer does **not** currently support pinch-zoom on mobile:
- Zoom only via weapon scopes
- Touch input uses 8-directional joystick (input-management subsystem)
- Scale.setters are tied to scope only, not touch events

## 12. Performance Optimizations

### Viewport-Based Culling

Objects visibility is determined by viewport bounds:

```typescript
// Conceptually (details in visibility-los subsystem):
if (isInViewport(worldPosition, camera)) {
    renderObject();
}
```

**Benefits:**
- Off-screen rendering skipped
- Spatial grid queries (see spatial-grid subsystem) only return visible objects
- Reduces draw calls for large maps

### Dirty Object Tracking

Network serialization uses viewport-aware object tracking (@file `common/src/packets/updatePacket.ts`):

```typescript
// Pseudo-code
const partialDirtyObjects = [];   // Changed within viewport
const fullDirtyObjects = [];      // New objects in viewport

// Only transmit viewport-aware delta updates
packet.Send({
    partialDirtyObjects,  // Changed data only
    fullDirtyObjects      // Full state
});
```

### Layer Transition Scheduling

```typescript
// @file client/src/scripts/managers/cameraManager.ts:147
objectUpdateTimeout = Game.addTimeout(() => {
    for (const object of Game.objects) {
        object.updateLayer();
    }
}, LAYER_TRANSITION_DELAY);  // Delay: 200ms
```

- Deferred layer updates to avoid jank during fade
- Objects move to new layer containers after visibility fade completes

## 13. Known Gotchas & Limitations

### Aspect Ratio Mismatches

**Problem:** Very wide or very tall screens can create black borders or clipped viewport.

**Root cause:** 16:9 aspect ratio enforcement means non-standard displays get adjusted.

**Example:**
- 21:9 ultrawide: viewport forced to max(minDim * 16/9, maxDim)
- 4:3 mobile: same constraint applied

**Workaround:** Responsive UI elements reposition based on `InputManager.isMobile` flag.

### Floating-Point Precision in Zoom Transitions

**Problem:** Zoom tweens can cause pixel misalignment at sub-pixel scales.

**Root cause:** `container.scale` is interpolated continuously; objects use integer pixel positions.

**Impact:** Minor shimmering during scope transitions (imperceptible at 40 FPS).

**Mitigation:** `cubicOut` easing reduces visible artifacts.

### Zoom Limits & Scope Definition Constraints

**Hard limits:**
- Minimum zoom: `1.0` (scope.zoomLevel can't be < 1.0)
- Maximum zoom: varies by scope definition (typically 8.0 for 8x scope)

**If scope.zoomLevel = 0:** Runtime error (division by zero in scale formula).

**Prevention:** Scope definitions validated during load (@file `server/src/utils/objectDefinitions.ts`).

### Layer Transitions & Z-Order Complexity

**Gotcha:** Basement Z-index = 999 or 0 depending on player layer.

**Why complex:**
- Buildings above ground can overlap basement view
- When entering bunker, basement must render above building
- When exiting, building takes priority again

**Risk:** Z-order flickers if layer transition timing is off.

**Mitigation:** `tempLayerContainer` with fractional z-indices prevents flickering.

### Camera Shake Doesn't Fade

**Current behavior:** Shake intensity is constant for entire duration, then stops instantly.

**Missing feature:** No amplitude decay curve (unlike shockwaves which decay linearly).

**Impact:** Shake feels "sudden off" rather than settling.

**Rationale:** Weapon shake is meant to be jarring, not soothing.

## 14. Cross-Tier References

**Tier 2 (Subsystems):**
- [Camera Management](../README.md) — Full subsystem overview
- [Input Management](../../input-management/) — Mouse position tracking
- [Visibility & LoS](../../visibility-los/) — Viewport-based culling
- [Game Objects (Rendering)](../../game-objects-client/) — Per-object rendering
- [Spatial Grid](../../spatial-grid/) — Viewport-aware object queries

**Tier 1 (Architecture):**
- [System Architecture](../../../architecture.md) — Rendering pipeline, coordinate systems
- [Data Model](../../../datamodel.md) — World coordinates, Layer enum

**Related Modules in This Subsystem:**
- [Input Handling](./input.md) — Mouse/keyboard input (future Tier 3)
- [Layer Management](./layers.md) — Detailed layer logic (future Tier 3)

**Source Files Referenced:**
- @file client/src/scripts/managers/cameraManager.ts — Main implementation
- @file client/src/scripts/objects/player.ts — Camera follow logic
- @file client/src/scripts/managers/inputManager.ts — Mouse coordinate conversion
- @file client/src/scripts/utils/pixi.ts — Coordinate transformation functions
- @file client/src/scripts/utils/constants.ts — PIXI_SCALE
- @file common/src/constants.ts — GameConstants
- @file common/src/definitions/items/scopes.ts — Zoom level definitions
