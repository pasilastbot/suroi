# Camera Management

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/camera-management/modules/ -->
<!-- @source: client/src/scripts/managers/cameraManager.ts -->

## Purpose

Controls the PixiJS viewport transformation: camera positioning relative to the local player, zoom level based on the equipped scope, screen shake effects, visual distortion (shockwaves), and layer (basement/ground/upstairs) visibility transitions. The camera is the spatial anchor for all client-side rendering—all game objects are positioned relative to the camera's view.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [client/src/scripts/managers/cameraManager.ts](client/src/scripts/managers/cameraManager.ts) | `CameraManager` singleton — viewport transformation, zoom, effects |
| [client/src/scripts/game.ts](client/src/scripts/game.ts) | Game owns CameraManager instance; updates camera position each frame |
| [client/src/scripts/ui.ts](client/src/scripts/ui.ts) | Applies global filters (color matrix) to camera; anchors HUD elements on mobile |

## Architecture

### Camera Position & Rendering Pipeline

```
Player position (from server state)
  ↓
CameraManager.position = activePlayer.container.position (direct follow, no look-ahead)
  ↓
CameraManager.update() — applies screen shake + shockwave offsets
  ↓
container.position = (-cameraPos.x, -cameraPos.y) with scale applied
  ↓
PixiJS viewport rendered; objects outside camera bounds culled by Pixi
```

The camera **directly follows the player position** with no mouse-based look-ahead offset. Screen shake and shockwave effects are computed as temporary position offsets during `update()` and are not persistent.

### Camera Container Hierarchy

```
CameraManager.container (scaled & positioned by camera)
├── layerContainers[0]  (basement layer, alpha-blended)
├── layerContainers[1]  (ground layer, alpha-blended)
├── layerContainers[2]  (upstairs layer, alpha-blended)
├── tempLayerContainer  (fade-in buffer for layer transitions)
└── [filters applied]   (shockwaves, global effects)
```

All game objects are children of one of the three layer containers. During layer transitions (e.g., entering a bunker), objects are moved to `tempLayerContainer` while the target layer fades in, preventing visual janking.

## Camera Behavior

### Following Player

Camera position is **synchronously set** to the local player's position each frame:

```typescript
// From game.ts render loop
if (hasMovementSmoothing && this.activePlayer) {
    CameraManager.position = this.activePlayer.container.position;
}
```

Then during `CameraManager.update()`, the position is transformed to screen space and applied to the container as a negative offset (the viewport moves in the opposite direction of the player).

**No easing or smoothing** is applied to camera position itself; the smoothing comes from player object interpolation (`object.updateInterpolation()`), which is controlled by the server-side movement sync rate (TPS=40).

### Zoom Levels

Zoom is controlled by the equipped **scope item definition**:

```typescript
private _zoom = DEFAULT_SCOPE.zoomLevel;
get zoom(): number { return this._zoom; }
set zoom(zoom: number) {
    this._zoom = zoom;
    this.resize(true);  // true = animate transition
}
```

When a scope is equipped, the server sends the scope data to the client. The zoom value is stored in the scope definition (e.g., `DEFAULT_SCOPE.zoomLevel` for no scope).

Scale calculation:

```typescript
const minDimension = Numeric.min(this.width, this.height);
const maxDimension = Numeric.max(this.width, this.height);
const maxScreenDim = Numeric.max(minDimension * (16 / 9), maxDimension);
const scale = (maxScreenDim * 0.5) / (this._zoom * PIXI_SCALE);
```

This maintains a 16:9 aspect ratio even on non-standard screens. Zoom **must be > 0** (no division by zero protection).

**Zoom Transition Animation:**
- **Duration:** 1250 ms
- **Easing:** `EaseFunctions.cubicOut`

```typescript
this.zoomTween = Game.addTween({
    target: this.container.scale,
    to: { x: scale, y: scale },
    duration: 1250,
    ease: EaseFunctions.cubicOut,
    onComplete: () => { this.zoomTween = undefined; }
});
```

When `resize(animation=true)` is called, the tween animates the container scale to the new zoom level. If `animation=false`, the scale is set instantly.

### Screen Shake Effect

Screen shake is a temporary **position offset** within a circular radius, applied each frame until the shake duration expires:

```typescript
shake(duration: number, intensity: number): void {
    if (!GameConsole.getBuiltInCVar("cv_camera_shake_fx")) return;  // toggleable
    this.shaking = true;
    this.shakeStart = Date.now();
    this.shakeDuration = duration;
    this.shakeIntensity = intensity;
}

// In update():
if (this.shaking) {
    position = Vec.add(position, randomPointInsideCircle(Vec(0, 0), this.shakeIntensity));
    if (Date.now() - this.shakeStart > this.shakeDuration) this.shaking = false;
}
```

- **Intensity**: Radius of the random offset circle (pixels)
- **Duration**: Milliseconds the shake persists
- **Console Variable:** `cv_camera_shake_fx` (boolean) — toggles entire effect

**Example:** Explosion at range applies `shake(300, 8)` → camera offset random point within 8-pixel circle for 300ms.

Shake is **NOT** eased out; it stops abruptly when duration expires. The random offset is recomputed every frame, creating the shake effect.

### Shockwave Visual Effect

Shockwave is a **post-processing filter** that creates a ripple distortion emanating from a point:

```typescript
shockwave(duration: number, position: Vector, amplitude: number, wavelength: number, speed: number, layer: Layer): void {
    this.shockwaves.add(new Shockwave(duration, position, amplitude, wavelength, speed, layer));
}

class Shockwave {
    filter: ShockwaveFilter;  // from pixi-filters library
    // Each frame: update filter center, wavelength, amplitude (eased out), time
}
```

- **Duration**: Milliseconds the shockwave persists
- **Position**: World position of the shock origin
- **Amplitude**: Pixel displacement magnitude at peak
- **Wavelength**: Distance between wave ridges (pixels)
- **Speed**: How fast the wave travels (pixels/ms)
- **Layer**: Which layer container gets the filter applied

**Amplitude easing:** Linearly fades from full amplitude → 0 over the duration:

```typescript
this.filter.amplitude = this.amplitude * EaseFunctions.linear(1 - ((now - lifeStart) / (lifeEnd - lifeStart)));
```

## Layer Transitions

### Visibility Management

When the player moves between layers (e.g., descending into a bunker):

```typescript
updateLayer(newLayer: Layer, initial = false, oldLayer?: Layer): void {
    // For basement (i=0), ground (i=1), upstairs (i=2):
    // Determine which layers should be visible based on newLayer
    // Apply smooth alpha transition to each layer's visibility
}
```

**Layer visibility rules:**
- **Basement (0):** Visible only when `newLayer <= Layer.ToBasement`
- **Ground (1):** Visible only when `newLayer >= Layer.ToBasement`
- **Upstairs (2):** Visible based on `hideSecondFloor` flag and whether player is in transition

**Z-index special case:** Basement layer's Z-index is set to `999` when on stairs or below (to render on top of buildings), then reverted to `0` when above ground.

### Fade Transition Animation

When layer visibility changes:

```typescript
const targetAlpha = visible ? 1 : 0;
this.layerTweens[i] = Game.addTween({
    target: container,
    to: { alpha: targetAlpha },
    duration: LAYER_TRANSITION_DELAY,  // 200 ms
    ease: EaseFunctions.sineOut,
    onComplete: () => { container.visible = visible; }
});
```

- **Duration:** 200 ms (defined in [client/src/scripts/utils/constants.ts](client/src/scripts/utils/constants.ts))
- **Easing:** `EaseFunctions.sineOut` (ease out sine curve)
- **Flickering prevention:** During transition, objects are moved to `tempLayerContainer` (intermediate Z-index) so they fade smoothly without jumping between containers

### Object Layer Assignment During Transition

Objects are assigned to layer containers based on their current layer:

```typescript
// Normal state
container = this.layerContainers[containerIndex];

// During transition, if the object's target layer differs from current layer:
if (this.layerTransition && oldContainerIndex !== undefined && containerIndex !== oldContainerIndex) {
    return this.tempLayerContainer;  // fade in buffer
}
```

This prevents an object from vanishing and reappearing as containers alpha-transition.

## CameraManager API

### Properties

| Property | Type | Purpose |
|----------|------|---------|
| `container` | `PixiJS.Container` | Root container that all game objects are children of; scaled/positioned by camera |
| `position` | `Vector` | Current camera position in world space (unscaled) |
| `zoom` | `number` | Current zoom level (reciprocal of scope zoom factor) |
| `width` | `number` | Viewport width in pixels |
| `height` | `number` | Viewport height in pixels |
| `shaking` | `boolean` | Whether screen shake is currently active |
| `shockwaves` | `Set<Shockwave>` | Active shockwave effects |
| `layerContainers[3]` | `Container[]` | Three layer containers (basement, ground, upstairs) |

### Methods

| Method | Purpose | Notes |
|--------|---------|-------|
| `init()` | Initialize camera; set up layer containers | Called once at game start |
| `resize(animation?)` | Recalculate zoom scale on window resize or scope change | `animation=true` → smooth tween; `animation=false` → instant |
| `update()` | Apply camera transform each frame; update shake/shockwaves | Called after all objects update |
| `shake(duration, intensity)` | Trigger screen shake effect | Toggleable via `cv_camera_shake_fx` console var |
| `shockwave(duration, pos, amp, wavelength, speed, layer)` | Trigger ripple distortion effect | Applied as post-processing filter to layer |
| `updateLayer(newLayer, initial?, oldLayer?)` | Handle layer visibility transition | Called when player changes layer |
| `getContainer(layer, oldContainerIndex?)` | Get the render container for a layer | Returns `tempLayerContainer` during transitions |
| `addFilter(layer, filter)` | Add a post-processing filter to a layer | Combines with existing filters into array |
| `removeFilter(layer, filter)` | Remove a post-processing filter from a layer | |
| `addObject(...objects)` | Add object to camera root (for non-layer objects like gas boundary) | |
| `reset()` | Clear all objects; reset zoom to default | Called on game end |

## Input Integration

Camera does **not** read input directly. Player position comes from the **Game object**, which receives server state updates. Mouse position is read by the [Input Management](../input-management/) subsystem but is **not** used to offset camera aim.

See [Game Loop](../game-loop/) for how player position is synchronized from the server.

## Dependencies

### Depends On

- **[Client Rendering](../client-rendering/)** — Pixi.js Container and Filter API
- **[Game Loop](../game-loop/)** — Player position updates each frame
- **Common Math Utils** (`@common/utils/math.ts`) — `EaseFunctions`, `Numeric`, vector math

### Depended On By

- **[Client Rendering](../client-rendering/)** — All game object rendering uses this camera's viewport transformation
- **[Game Loop](../game-loop/)** — Player follows camera position
- **[UI Management](../ui-management/)** — HUD repositioning on mobile based on camera dimensions
- **[Map Manager](../map-management/)** — Mini-map culling uses camera position
- **[Particle Manager](../particle-management/)** — Particles spawn within camera bounds
- **[Input Management](../input-management/)** — Emote/Map Ping wheels reposition on mobile

## Known Issues & Gotchas

### 1. **No Mouse Look-Ahead**

The camera follows the player **directly**, with no offset toward the mouse aim direction. This is a design choice (not a bug). If aiming/look-ahead is needed, it would require:
- Reading `InputManager.mousePosition` (already available)
- Computing a biased target position between player and mouse
- Lerping camera to target instead of snapping

Currently **not implemented**.

### 2. **Zoom Lock on Scope Change**

When the player switches scopes, `zoom` is set and `resize(animation=true)` is called, triggering a 1250ms tween. **During this tween, if the player switches scopes again, the old tween is killed and a new one starts.** This can cause visual stuttering if scope switching happens rapidly. No overflow protection exists.

### 3. **Screen Shake Not Eased**

Screen shake applies a hard cut-off when duration expires. There's no easing out, so visual impact stops abruptly. A fade-out would be more polished:

```typescript
// Not implemented:
shake amplitude *= easeOut(1 - timeRemaining / totalDuration);
```

### 4. **Shockwave Position Tracking**

Shockwave position is **static** (set at creation). If the shock origin object moves, the ripple effect does not follow. This works correctly for explosions (which are instantaneous, not persistent), but would not work for a moving shockwave source.

### 5. **Layer Transition Flickering Edge Case**

If multiple objects change layers simultaneously during a transition, and their `updateLayer()` methods are called in different order, they may briefly appear in wrong containers. The `tempLayerContainer` mitigation helps but doesn't fully solve out-of-order updates.

### 6. **Mobile HUD Repositioning**

On mobile, `EmoteWheelManager` and `MapPingWheelManager` are repositioned to the center of the screen each time `resize()` is called. This happens on window resize or zoom change but **not on camera position change**. If the player expects the wheels to stay centered, they will drift if the viewport resizes mid-game.

### 7. **Shockwave Scale Relative to Zoom**

Shockwave parameters (`amplitude`, `wavelength`, `speed`) are **not** adjusted for the current zoom level. At high zoom (small scope), the shockwave visual effect may appear too large; at low zoom (wide view), it may be too small. The filter `scale()` method attempts to compensate:

```typescript
scale(): number {
    return PIXI_SCALE / CameraManager.zoom;
}
```

But this may not fully account for perceptual differences.

### 8. **Console Variable Typo Risk**

The camera shake toggle is `cv_camera_shake_fx` (hardcoded string). If the console variable name is ever changed, the check will fail silently and shake will never trigger.

## Configuration & Console Variables

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `cv_camera_shake_fx` | `boolean` | `true` | Enable/disable screen shake effects |
| `cv_movement_smoothing` | `boolean` | `true` | Enable camera position smoothing (client-side player interpolation) |
| `cv_renderer` | `enum` | `"webgl2"` | Renderer type (affects filter support) |

## Performance Considerations

### Per-Frame Cost

**Camera.update()** is O(1) + O(n_shockwaves):
- Screen shake: Single random vector generation if active
- Shockwave updates: Iterate all active shockwaves, update filter params

**Typical cost:** < 0.1ms (negligible in 40 TPS budget of 25ms/frame)

### Viewport Culling

PixiJS **automatically culls** objects outside the camera bounds before rendering. Objects are still updated (physics, animation), but not rendered. For thousands of objects, this culling is essential.

**No manual frustum culling:** The camera system doesn't explicitly expose a "visible bounds" query; frustum queries are done by the renderer.

## Related Documents

### Tier 1

- [System Architecture](../../architecture.md) — Rendering pipeline, camera role in viewport
- [Data Model](../../datamodel.md) — Scope definitions, zoom levels

### Tier 2

- [Client Rendering](../client-rendering/) — PixiJS integration, sprite rendering
- [Game Loop](../game-loop/) — Player position synchronization, frame update cycle
- [Input Management](../input-management/) — Mouse position, keyboard input
- [UI Management](../ui-management/) — HUD, menu rendering

### Tier 3

- (None yet; see Modules section for module-level docs)

### Patterns

- (No subsystem-specific patterns yet; see [Patterns](patterns.md) if created)
