# Client Rendering Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/client-rendering/modules/ -->
<!-- @source: client/src/scripts/ -->

## Purpose

PixiJS 8-based WebGL/WebGPU renderer for the game world. Receives binary `UpdatePacket` messages over a WebSocket, routes each one to typed renderable objects managed by `ObjectPool`, and drives position/rotation interpolation and the particle/tween/sound systems each animation frame.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/game.ts` | `Game` singleton — PixiJS `Application`, WebSocket client, `ObjectPool`, render loop, packet dispatch |
| `client/src/scripts/objects/gameObject.ts` | `GameObject<Cat>` abstract base with container, position/rotation interpolation, layer management |
| `client/src/scripts/objects/` | Nine concrete renderable classes (see [Renderable Object Classes](#renderable-object-classes)) |
| `client/src/scripts/utils/pixi.ts` | `SuroiSprite`, spritesheet loading (`loadSpritesheets`), coord helper `toPixiCoords` |
| `client/src/scripts/utils/constants.ts` | `PIXI_SCALE`, `ZIndexes` usage, `HITBOX_COLORS`, debug flags |
| `client/src/scripts/managers/cameraManager.ts` | `CameraManager` — zoom, pan, screen shake, shockwave filters, layer containers |
| `client/src/scripts/managers/particleManager.ts` | `ParticleManager` — particle/emitter lifecycle |
| `client/src/scripts/managers/mapManager.ts` | `MapManager` — minimap / full-screen map |
| `client/src/scripts/managers/soundManager.ts` | `SoundManager` / `GameSound` — positional audio via `@pixi/sound` |

**PixiJS version:** `pixi.js ^8.14.3`, `@pixi/sound ^6.0.1`, `pixi-filters ^6.1.5`

## Architecture

A single `PIXI.Application` (stored at `Game.pixi`) owns a `<canvas id="game-canvas">`. The stage is split into two isolated render groups: `gameContainer` for the camera-transformed world, and `uiContainer` for overlay elements (minimap, emote wheel, net graphs) that are fixed to the screen. Within `gameContainer`, `CameraManager.container` handles zoom scaling and camera translation; it holds three per-layer sub-containers that are faded in/out when the local player moves between basement, ground, and upstairs floors.

Incoming `UpdatePacket` data is dispatched by `Game.processUpdate()`. Pool-managed objects are created via `ObjectClassMapping`, which maps each `ObjectCategory` enum value to a concrete class constructor. Non-pooled rendering objects — `Bullet` (set-tracked) and `Plane` (set-tracked) — are managed separately. The `explosion()` function is a standalone fire-and-forget renderer called directly from `processUpdate`.

The render loop runs at the browser's vsync rate (`PIXI.Ticker`). Each tick it advances tweens, updates each `GameObject` (interpolation, custom per-class logic), ticks `Bullet` and `Plane` instances, advances the `ParticleManager`, and calls `CameraManager.update()` to reposition the viewport.

## PIXI_SCALE

```
PIXI_SCALE = 20   // px per game unit
```

All game-logic coordinates (e.g. player position `{x, y}`) are in **game units**. Multiply by `PIXI_SCALE` to get canvas pixels. The helper `toPixiCoords(pos: Vector): Vector` in `pixi.ts` performs `Vec.scale(pos, PIXI_SCALE)`.

Example: a player at game position `(100, 50)` maps to canvas position `(2000, 1000)` before camera transform.

## Z-Index Layer System

Defined as `enum ZIndexes` in `common/src/constants.ts`. Enum values are auto-assigned starting from `0`:

| ZIndex Name | Value | What Renders Here |
|-------------|-------|--------------------|
| `Ground` | 0 | Base ground plane |
| `BuildingsFloor` | 1 | Building floor graphics |
| `Decals` | 2 | `Decal` objects (blood, burn marks) |
| `DeadObstacles` | 3 | Destroyed obstacle remnants |
| `DeathMarkers` | 4 | `DeathMarker` objects |
| `Explosions` | 5 | Explosion flash sprites |
| `ObstaclesLayer1` | 6 | Default obstacle layer (crates, walls) |
| `Loot` | 7 | `Loot` objects |
| `GroundedThrowables` | 8 | Throwables resting on the ground |
| `ObstaclesLayer2` | 9 | Mid-height obstacles |
| `TeammateName` | 10 | Teammate name labels |
| `Bullets` | 11 | `Bullet` tracers |
| `DownedPlayers` | 12 | Downed player sprites |
| `Players` | 13 | `Player` objects |
| `ObstaclesLayer3` | 14 | Bushes, tables (above players) |
| `AirborneThrowables` | 15 | Thrown grenades/throwables in flight |
| `ObstaclesLayer4` | 16 | Trees (tall obstacles above players) |
| `BuildingsCeiling` | 17 | Building ceiling sprites |
| `ObstaclesLayer5` | 18 | Obstacles that show above ceilings |
| `Emotes` | 19 | Emote speech-bubble sprites |
| `Gas` | 20 | Gas zone overlay |

Special cases outside the enum: `Parachute` uses hard-coded `994`; gas `GasRender.graphics` uses `1000`; `Plane` uses `Number.MAX_SAFE_INTEGER - 2`.

The constant `PLAYER_PARTICLE_ZINDEX = ZIndexes.Players + 0.5` (from `constants.ts`) places player particles just above player sprites.

## Container Hierarchy

```
PIXI.stage
├── gameContainer  { isRenderGroup: true }
│   ├── CameraManager.container          ← scaled (zoom) + translated (pan)
│   │   ├── layerContainers[0]  zIndex=0 (or 999)  ← Basement
│   │   ├── layerContainers[1]  zIndex=1            ← Ground
│   │   ├── layerContainers[2]  zIndex=2            ← Upstairs
│   │   ├── tempLayerContainer                      ← swap buffer for layer transitions
│   │   └── gasRender.graphics  zIndex=1000         ← gas effect, added via CameraManager.addObject()
│   └── DebugRenderer.graphics  (DEBUG_CLIENT only)
└── uiContainer  { isRenderGroup: true }
    ├── MapManager.container              ← minimap / full map
    ├── MapManager.mask
    ├── netGraph containers
    ├── EmoteWheelManager.container
    └── MapPingWheelManager.container
```

Each `GameObject` adds its own `container` to a layer container via `CameraManager.getContainer(layer)`. Individual children within a layer container are sorted by their `zIndex` properties.

Layer container fade-in/out is animated with `LAYER_TRANSITION_DELAY` (200 ms) and `EaseFunctions.sineOut` tweens whenever the local player's layer changes.

## Object Rendering (Client ObjectPool)

The `ObjectPool<ObjectMapping>` at `Game.objects` maps numeric IDs to typed `GameObject` instances. It is updated inside `Game.processUpdate()` on every incoming `UpdatePacket`:

### Object Creation

```typescript
// in Game.processUpdate(), for each entry in updateData.fullDirtyObjects:
const _object = new (ObjectClassMapping[type])(id, data);
Game.objects.add(_object);
```

The constructor of every pool-managed class calls `this.updateFromData(data, true)` internally, which fully initialises the visual state and calls `CameraManager.getContainer(layer).addChild(this.container)` to insert the object into the scene graph.

### Partial Updates

```typescript
// for each entry in updateData.partialDirtyObjects:
object.updateFromData(data, false);   // isNew = false
```

`updateFromData` accepts an `isNew` boolean. When `false`, only changed properties are updated; full re-initialisation is skipped.

### Object Deletion

```typescript
object.destroy();          // removes container from scene, cancels tweens/sounds
Game.objects.delete(object);
```

### Non-Pool Objects

| Object | Container | How created | How destroyed |
|--------|-----------|-------------|---------------|
| `Bullet` | Direct child of layer container | `new Bullet(bulletOptions)` pushed into `Game.bullets` | Removed from `Game.bullets` when `bullet.dead` after `bullet.update(delta)` |
| `Plane` | Direct child via `CameraManager.addObject()` | `new Plane(pos, dir)` pushed into `Game.planes` | Removed from `Game.planes` when flight distance exceeded |
| `explosion()` | Layer container (transient) | Called as `explosion(definition, position, layer)` | Self-removing tween-driven sprite |

## Renderable Object Classes

All live in `client/src/scripts/objects/`. Pool-managed classes extend `GameObject.derive(ObjectCategory.X)`:

| Class | File | Pooled | Renders |
|-------|------|--------|---------|
| `Player` | `player.ts` | Yes | Local and remote players — body, hands, weapon, emotes, name labels |
| `Obstacle` | `obstacle.ts` | Yes | Destructible/static world objects (crates, rocks, doors, bushes, trees…) |
| `DeathMarker` | `deathMarker.ts` | Yes | Death marker sprite + player name text at kill location |
| `Loot` | `loot.ts` | Yes | Ground loot items — background circle + item sprite |
| `Building` | `building.ts` | Yes | Multi-layer structures — floor graphics, ceiling container, room lighting |
| `Decal` | `decal.ts` | Yes | Persistent decals placed on the ground (blood splats, burn marks) |
| `Parachute` | `parachute.ts` | Yes | Airdrop parachute, descends and triggers landing particles |
| `Projectile` | `projectile.ts` | Yes | Server-tracked throwables (grenades) in flight |
| `SyncedParticle` | `syncedParticle.ts` | Yes | Server-authoritative animated particles (e.g. gas canisters) |
| `Bullet` | `bullet.ts` | No (`Game.bullets`) | Client-predicted bullet tracers |
| `Plane` | `plane.ts` | No (`Game.planes`) | Airdrop plane flyover |
| `explosion` (fn) | `explosion.ts` | No (transient) | Tween-animated flash + particles + screen shake on detonation |

## SuroiSprite

`SuroiSprite` (`client/src/scripts/utils/pixi.ts`) extends `PIXI.Sprite` with:

- **Default anchor `(0.5, 0.5)`** — sprites are centered on their position by default.
- **`static getTexture(frame: string): Texture`** — looks up `frame` in `Assets.cache`; falls back to `"_missing_texture"` with a console warning if the key is absent.
- **Fluent setter API** — every setter returns `this` to allow chaining:

| Method | Purpose |
|--------|---------|
| `setFrame(frame)` | Swap texture (re-uses `getTexture`) |
| `setPos(x, y)` | Set canvas position (already in px) |
| `setVPos(pos)` | Set position from `Vector` |
| `setAnchor(anchor)` | Override default center anchor |
| `setPivot(pivot)` | Set rotation pivot |
| `setAngle(angle)` | Set rotation in degrees |
| `setRotation(rotation)` | Set rotation in radians |
| `setScale(sx, sy?)` | Set uniform or non-uniform scale |
| `setTint(tint)` | Apply color tint |
| `setZIndex(z)` | Set render sort order |
| `setAlpha(alpha)` | Set opacity |
| `setVisible(visible)` | Toggle visibility |

**Important:** `setPos` / `setVPos` expect **pixel** coordinates (already scaled). Use `toPixiCoords(gamePos)` to convert from game units first.

## Camera System

`CameraManager` (singleton `client/src/scripts/managers/cameraManager.ts`) owns:

| Property / Method | Description |
|-------------------|-------------|
| `container` | Root `PIXI.Container` for everything in world-space |
| `layerContainers[0..2]` | Per-floor containers (Basement=0, Ground=1, Upstairs=2) |
| `position: Vector` | Camera center in **game units** |
| `zoom` (setter) | Triggers animated `cubicOut` scale tween over 1250 ms |
| `resize(animation?)` | Recalculates `container.scale` from screen size and `zoom`: `scale = (maxScreenDim * 0.5) / (zoom * PIXI_SCALE)` |
| `update()` | Each frame: add shake offset, apply `shockwave.update()`, set `container.position` to center the view |
| `getContainer(layer)` | Returns `layerContainers[containerIndex]`, or `tempLayerContainer` during layer transitions |
| `addObject(...objs)` | Adds directly to `container` (world-space, bypasses layer system) |
| `addFilter(layer, filter)` | Appends a PixiJS `Filter` to a layer container (used by shockwaves) |
| `shake(duration, intensity)` | Randomises position offset for `duration` ms by up to `intensity` game units each frame |
| `shockwave(...)` | Creates `Shockwave` which applies `ShockwaveFilter` from `pixi-filters` |
| `updateLayer(newLayer, initial?, oldLayer?)` | Fades layer containers in/out with 200 ms tween; schedules deferred `object.updateLayer()` calls |

Camera translation formula (applied in `update()`):
```
cameraPos = position * container.scale.x - (width/2, height/2)
container.position = (-cameraPos.x, -cameraPos.y)
```

In the render loop, `Game.render()` sets `CameraManager.position = activePlayer.container.position` each frame when movement smoothing is enabled (this is already in pixel space because `updateContainerPosition()` calls `toPixiCoords`).

## Spritesheet System

Build-time (`client/vite/plugins/image-spritesheet-plugin.ts`):
- All game images are packed using `maxrects-packer` into atlases at build time.
- **High-resolution:** 4096 × 4096 px per atlas.
- **Low-resolution:** 2048 × 2048 px per atlas.
- Separate atlas sets are generated per game mode (`modeName`).
- Output is a Vite virtual module (`virtual:image-spritesheets-importer-high-res` / `…-low-res`).

Run-time (`client/src/scripts/utils/pixi.ts` — `loadSpritesheets`):
1. If WebGL renderer reports `MAX_TEXTURE_SIZE < 4096`, force low-res.
2. Dynamically import the appropriate virtual module.
3. For each atlas: `Assets.load<Texture>(atlasUrl)` → `renderer.prepare.upload(texture)` → `new Spritesheet(...).parse()`.
4. Register all parsed textures into `Assets.cache` by key name.
5. Resolve all `spritesheetLoadPromise()` callbacks when the last atlas finishes.

`SuroiSprite.getTexture(frame)` looks up keys in `Assets.cache`. Missing frames log a warning and return a magenta `_missing_texture`.

## Particle System

`ParticleManager` (singleton `client/src/scripts/managers/particleManager.ts`):

| Method | Purpose |
|--------|---------|
| `spawnParticle(options: ParticleOptions)` | Creates `Particle`, adds its `SuroiSprite` to the layer container |
| `spawnParticles(count, options)` | Calls `spawnParticle` `count` times |
| `addEmitter(options: EmitterOptions)` | Creates a `ParticleEmitter` that auto-spawns particles at `options.delay` intervals |
| `update(delta)` | Called each render frame — advances all particles, removes dead ones, ticks emitters |
| `reset()` | Clears all particles and emitters (called on game end) |

`Particle` interpolates `scale`, `alpha`, and `rotation` from `start` → `end` over `lifetime` ms, optionally with a custom easing function. Position advances by `speed * (delta / 1000)` each frame.

`ParticleEmitter` has an `active` toggle and `destroy()` to stop it.

`ParticleOptions.frames` can be a single string or an array; if an array, a random frame is chosen at spawn. Frames with a matching entry in `TintedParticles` use a tinted base sprite.

## Sound System

`GameSound` wraps `@pixi/sound` and adds positional audio:

- **Distance attenuation:** volume = `falloff^(distance^2 / maxRange^2)` — volume drops exponentially with squared distance.
- **Stereo panning:** `PixiSound.filters.StereoFilter` pans left/right based on the horizontal offset between the sound position and the camera center.
- **Layer muffling:** sounds on a different `Layer` than the camera receive a volume multiplier (controlled by `SOUND_FILTER_FOR_LAYERS`).
- **Dynamic sounds:** when `SoundOptions.dynamic = true`, pan and volume are updated every frame as the camera moves. Static sounds are set at creation and not recalculated.
- **Ambient sounds:** `SoundOptions.ambient = true` routes volume through `SoundManager.ambienceVolume` instead of `SoundManager.sfxVolume`.

## Minimap

`MapManager` (singleton `client/src/scripts/managers/mapManager.ts`) renders inside `uiContainer` (screen-space, not camera-space):

- **`updateFromPacket(packet: MapData)`** — rebuilds the entire map: `Terrain` (floor types, rivers, beaches), building outlines, place name labels. Renders to a `RenderTexture` which becomes `MapManager.sprite`.
- **Minimap mode:** small corned overlay — `container.mask` clips to a circle.
- **Full-map mode:** `expanded = true` switches to a full-screen view.
- **Live overlays updated each frame:** gas safe-zone circle, player indicator (`SuroiSprite` with `"player_indicator"` frame), teammate indicators (colored dots from `TEAMMATE_COLORS`), map pings (`MapPing` entries in `pingsContainer`), and map indicators (objective markers).
- **Toggle:** `MapManager.visible` / `MapManager.toggle()` / `MapManager.expanded` — the minimap visibility and map/minimap toggle are driven by `cv_minimap_minimized` and `cv_map_expanded` CVars.

## Data Flow

```
WebSocket message (ArrayBuffer)
    │
    └─► PacketStream.deserialize()        [common/src/packets/packetStream.ts]
             │
             └─► Game.onPacket(packet)
                      │
                      ├─► PacketType.Map    → MapManager.updateFromPacket()
                      │
                      └─► PacketType.Update → Game.processUpdate(updateData)
                                  │
                                  ├── fullDirtyObjects   → new ObjectClassMapping[type](id, data)
                                  │                           └─► constructor → updateFromData(data, true)
                                  │                                                └─► CameraManager.getContainer(layer).addChild(container)
                                  │
                                  ├── partialDirtyObjects → object.updateFromData(data, false)
                                  │
                                  ├── deletedObjects      → object.destroy() → objects.delete(object)
                                  │
                                  ├── deserializedBullets → new Bullet(opts) → Game.bullets.add()
                                  │
                                  ├── explosions          → explosion(def, pos, layer)   [transient]
                                  │
                                  └── planes              → new Plane(pos, dir) → Game.planes.add()

PIXI.Ticker (vsync)
    │
    └─► Game.render()
             ├── for object of Game.objects  → object.update() + object.updateInterpolation()
             ├── for bullet of Game.bullets  → bullet.update(delta)
             ├── for plane  of Game.planes   → plane.update()
             ├── ParticleManager.update(delta)
             ├── MapManager.update()
             ├── gasRender.update()
             ├── CameraManager.update()      → repositions container
             └── PIXI renders to WebGL/WebGPU canvas
```

## Interfaces & Contracts

### `Game` class (`client/src/scripts/game.ts`)

Main client-side API:

| Class / Method | Type | Purpose |
|---|---|---|
| `Game.pixi: PIXI.Application` | static property | The PixiJS Application singleton |
| `Game.camera: CameraManager` | static property | Camera position, zoom, shake, layer management |
| `Game.socket: WebSocket` | static property | Connected WebSocket to server |
| `Game.objects: ObjectPool<ObjectMapping>` | static property | Pool-managed game objects indexed by ID |
| `Game.bullets: Set<Bullet>` | static property | Non-pooled bullet tracers (client-predicted) |
| `Game.planes: Set<Plane>` | static property | Non-pooled airdrop plane sprites |
| `Game.processUpdate(packet)` | method | Dispatches incoming `UpdatePacket` to update object pool and UI managers |
| `Game.addGameObject(type, id, data)` | method | Creates a new pooled game object |
| `Game.removeGameObject(id, skipAnimation?)` | method | Deletes a pooled game object |

### Manager interfaces

| Manager | File | Key methods |
|---|---|---|
| `CameraManager` | `managers/cameraManager.ts` | `update()`, `getContainer(layer)`, `addObject()`, `shake()`, `shockwave()` |
| `MapManager` | `managers/mapManager.ts` | `updateFromPacket()` — minimap state |
| `GasManager` | `managers/gasManager.ts` | `updateFrom()` — gas state updates |
| `UIManager` | `managers/uiManager.ts` | `processUpdatePacket()`, `showGameOverScreen()`, `processKillPacket()` |
| `ParticleManager` | `managers/particleManager.ts` | `addEmitter()`, `addParticle()` — fire-and-forget particles |
| `SoundManager` | `managers/soundManager.ts` | `play(sound, source?)` — positional audio |
| `InputManager` | `managers/inputManager.ts` | Per-frame keyboard/mouse/touch state capture |

### Renderable Object Classes

All extend `GameObject<Type>` (from `objects/gameObject.ts`):

```typescript
abstract class GameObject<Cat extends ObjectCategory> {
    id: number
    container: PIXI.Container
    position: Vector
    rotation: number
    layer: Layer
    updateFromData(data, isNew: boolean): void   // Apply definition + state updates
    destroy(): void                               // Remove from scene, cleanup
}
```

Concrete subclasses (in `objects/`):
- `Player` — local and remote player bodies, hands, weapons
- `Obstacle` — world destructibles (crates, rocks, trees)
- `Building` — multi-story structures with floor/ceiling rendering
- `Loot` — ground items with physics simulation
- `Decal` — persistent ground decals (blood, scorch marks)
- `DeathMarker` — kill location markers with names
- `Parachute` — airdrop descent animation
- `Projectile` — server-tracked throwables in flight
- `SyncedParticle` — server-authoritative particles

Non-pooled singletons:
- `Bullet` (set-tracked) — client-predicted bullet tracers
- `Plane` (set-tracked) — airdrop plane sprites
- `explosion()` function (transient) — tween-driven flash + shake

---

## Module Index (Tier 3)

For client rendering patterns and implementation details, see:

- [Game Client Module](modules/game-client.md) — Client initialization, WebSocket lifecycle, packet dispatch

## Dependencies

- **Depends on:**
  - [Networking](../networking/) — all game state arrives via binary `UpdatePacket` / `MapPacket` over WebSocket; `PacketStream` / `SuroiByteStream` deserialize the wire format
  - [Object Definitions](../object-definitions/) — renderable classes read `ObstacleDefinition`, `BuildingDefinition`, `LootDefinition`, etc. from `common/src/definitions/` to know sprite names, scale, sound IDs, and hitbox shapes
  - [Gas System](../gas/) — `GasManager` / `GasRender` update gas overlay inside camera-space and drive `MapManager`'s gas circle
- **Depended on by:** Nothing — this is the top-level output layer

## Known Gotchas & Tech Debt

- **Bullets are NOT in ObjectPool.** They are deserialized client-side from `deserializedBullets` in the update packet and tracked in `Game.bullets`. They have no server-assigned IDs in the pool.
- **`explosion()` is a function, not a class.** It creates transient PixiJS sprites that self-destroy via tweens; there's no pool entry.
- **Layer container zIndex hack:** when the player moves to basement, `layerContainers[0].zIndex` is set to `999` so the basement renders on top. A comment in the source notes this is to prevent buildings on the surface from occluding bunkers.
- **PixiJS `autoStart: false`:** the application does not auto-start. `Game.pixi.start()` is called only when the WebSocket connection opens, and `Game.pixi.stop()` is called on close/error.
- **Camera position in render loop:** `CameraManager.position` is set to `activePlayer.container.position` (pixel space) rather than `toPixiCoords(activePlayer.position)`. These are equivalent when movement smoothing interpolates the container position, but coupling them means the camera lags by one rendered frame.
- **`SOUND_FILTER_FOR_LAYERS`** is `true` by default but the source comment reads _"TODO: test this, unsure if it glitches the sound manager."_

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview and tech stack
- **Tier 1:** [docs/api-reference.md](../../api-reference.md) — Binary packet protocol
- **Tier 2:** [Networking](../networking/) — How `UpdatePacket` / `MapPacket` are serialized
- **Tier 2:** [Object Definitions](../object-definitions/) — Definition registry read by renderable classes
- **Tier 2:** [Gas System](../gas/) — Gas overlay data flow
- **Patterns:** [patterns.md](patterns.md) — How to add a new renderable object class
