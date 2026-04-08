# Q: How do I debug client-side rendering issues (missing sprite, wrong position, z-index)?

<!-- @tags: rendering, debug, PixiJS, spritesheet, camera -->
<!-- @related: docs/subsystems/rendering/README.md, docs/subsystems/rendering/modules/object-lifecycle.md -->

## Missing Sprite (shows `_missing_texture`)

A pink/magenta square means `SuroiSprite.getTexture(frame)` could not find the
frame name in the loaded spritesheet.

**Diagnosis:**

1. Open browser DevTools → Console. Look for warnings like:
   `"Missing texture: <frame_name>"`

2. Check the frame name the code is requesting. In client objects, sprites are
   created like:
   ```typescript
   new SuroiSprite("my_obstacle")
   // or
   this.image.setFrame("my_obstacle_residue")
   ```

3. Verify the SVG exists at the expected path:
   ```
   client/public/img/game/shared/<frame_name>.svg
   ```
   or in the mode-specific directory if mode-specific.

4. Run `bun validateSvgs` — it reports missing or malformed SVGs.

5. Check the spritesheet load order: `Modes[modeName].spriteSheets` controls
   which atlases are loaded. If your SVG is in `normal/` but the active mode
   only loads `shared/`, the frame won't be found.

## Object Not Appearing

**Possible causes:**

1. **Not registered in `ObjectClassMapping`** — Check `client/src/scripts/game.ts`
   that your `ObjectCategory` maps to a class.

2. **Not in the player's view** — The server only sends visible objects. If the
   object is far away, it won't be in `UpdatePacket`.

3. **Wrong layer container** — Objects are placed into `layerContainers[layer]`.
   If your object has `layer = Layer.Basement` but the camera is on Ground,
   the basement container is not visible.

4. **Alpha = 0** — Check if the container or sprite has `alpha: 0`.

5. **Wrong position** — Add a temporary `console.log(this.position)` in
   `updateFromData()`. Positions are in world units; use `CameraManager.toPixiCoords()`
   to convert to screen space.

## Wrong Z-Index (Object Hidden Behind / In Front of Wrong Objects)

Z-index within a layer is controlled by `ZIndexes` (from `common/src/constants.ts`):

```
Ground → BuildingsFloor → Decals → DeadObstacles → DeathMarkers →
Explosions → ObstaclesLayer1 → Loot → ... → Players → ObstaclesLayer3 →
... → BuildingsCeiling
```

To change an object's render order:

```typescript
// In the client object constructor or updateFromData:
this.container.zIndex = ZIndexes.ObstaclesLayer2;
// Or for layer containers:
this.game.camera.layerContainers[layer].sortChildren = true;
```

Make sure `sortChildren` is enabled on the container if zIndex sorting is needed.

## Sprite at Wrong Position / Scale

1. Check `PIXI_SCALE` (`client/src/scripts/utils/constants.ts`) — all world
   positions are multiplied by this to convert to pixels.

2. Verify `SuroiSprite` anchor: `anchor.set(0.5)` means the position is the
   center of the sprite. If your sprite looks offset, the SVG may have unequal
   margins around the artwork.

3. In `updateFromData()`, log `this.container.position` after calling
   `updateContainerPosition()` to verify screen coordinates.

## Using the Debug Packet

The `DebugPacket` (dev-only) provides server-side debug state:

```bash
# In the game console (`)
debug on
```

This enables visual overlays: hitboxes, grid cells, spawn zones, etc.
(if implemented in the current build).

## PixiJS Inspector

Install the [PixiJS DevTools browser extension](https://chrome.google.com/webstore/detail/pixijs-devtools/).
It lets you inspect the scene graph, modify container properties at runtime,
and identify invisible/missing objects.

## Interpolation Issues (Objects Snapping / Jittering)

Positions interpolate between server ticks (every 25 ms). If an object snaps:

- Check `updateContainerPosition()` — it lerps based on `(Date.now() - lastPositionChange) / serverDt`
- If `lastPositionChange` is not being updated in `updateFromData()`, lerp
  won't work correctly.
- `serverDt` = 25 ms (40 TPS). If the tick rate changes, update `Game.serverDt`.

## Related

- [Rendering Subsystem](../subsystems/rendering/README.md) — architecture overview
- [Object Lifecycle](../subsystems/rendering/modules/object-lifecycle.md) — create/update/destroy flow
- [Spritesheets](../subsystems/rendering/modules/spritesheets.md) — frame lookup and fallback
- [Camera](../subsystems/rendering/modules/camera.md) — layer containers, coordinate transforms
