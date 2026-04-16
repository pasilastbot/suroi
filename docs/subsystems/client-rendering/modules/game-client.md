# Game Client Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/client-rendering/README.md -->
<!-- @source: client/src/scripts/game.ts -->

## Purpose
The central client-side singleton (`Game`) that owns the PixiJS application instance, the WebSocket connection to the game server, all renderable game object lifecycle management, the render loop, and cross-manager coordination.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/game.ts` | `Game` singleton — PixiJS app, WebSocket, object pool, render loop | High |
| `client/src/scripts/utils/pixi.ts` | `SuroiSprite` helper class, spritesheet loading utilities | Medium |
| `client/src/scripts/utils/constants.ts` | `PIXI_SCALE`, `HITBOX_COLORS`, `TEAMMATE_COLORS`, etc. | Low |
| `client/src/scripts/managers/cameraManager.ts` | Camera pan/zoom, layer containers, shake/shockwave effects | Medium |

## Business Rules
- **Singleton pattern:** `Game` is exported as a `new` expression of an anonymous class at module level — `export const Game = new (class Game { ... })();`. It is not instantiated by callers; `Game.init()` bootstraps it.
- **`PIXI_SCALE = 20`:** Game world coordinates are multiplied by this factor to obtain canvas pixels. A position of `(1, 1)` in game space maps to pixel `(20, 20)`.
- **Single initialization guard:** Both `Game.init()` and `CameraManager.init()` throw if called a second time, preventing double-bootstrap.
- **WebSocket binary type:** `this._socket.binaryType = "arraybuffer"`. All messages arrive as `ArrayBuffer`.
- **Packet multiplexing:** A single WebSocket message may contain multiple packets. `PacketStream.deserialize()` is called in a `while(true)` loop until it returns `undefined`.
- **Object pool keyed by numeric ID:** `Game.objects` is an `ObjectPool<ObjectMapping>`. On a `fullDirtyObject` entry, if no object with that ID exists (or it has been destroyed), a new instance is constructed via `ObjectClassMapping[type]`. If the object exists and is live, `updateFromData(data, false)` is called instead.
- **Partial updates require prior existence:** `partialDirtyObjects` entries assume the object already exists in the pool. A `console.warn` fires if the id is unknown, but no object is created.
- **Object deletion:** Objects in `deletedObjects` have `.destroy()` called, then are removed from the pool. A `console.warn` fires if the id is unknown.
- **`Bullet` and `Plane` use separate collections:** They are tracked in `Game.bullets` (a `Set<Bullet>`) and `Game.planes` (a `Set<Plane>`) rather than the typed object pool.
- **`pixi.autoStart = false`:** The PixiJS ticker does not start automatically. It starts inside `_socket.onopen` via `this.pixi.start()` and stops inside `_socket.onclose`/`onerror` via `this.pixi.stop()`.
- **Game UI pointer relay:** Because the game UI overlay covers the canvas, `pointerdown` events on the UI are manually re-dispatched to the canvas so that spectate-on-click still works.
- **Layer transition delay:** `LAYER_TRANSITION_DELAY = 200 ms` — object layer updates during transitions are deferred via `CameraManager.objectUpdateTimeout`.

## Game Initialization

The entry point is `client/src/index.ts`, which calls `void Game.init()` after importing FontAwesome CSS and the game SCSS.

`Game.init()` runs the following sequence:

1. Throws if already initialized (`_initialized` guard).
2. `GameConsole.init()` — sets up the in-game console.
3. `setUpCommands()` — registers console commands.
4. `await initTranslation()` — loads i18n strings.
5. `InputManager.init()` — registers keyboard/mouse/touch events.
6. Optional: loads `DebugMenu` if `DEBUG_CLIENT` is set.
7. `await setUpUI()` — wires jQuery-based HUD event handlers.
8. `await fetchServerData()` — fetches server config (modes, etc.) and sets `Game.modeName`.
9. `new GasRender(PIXI_SCALE)` — creates the gas rendering object.
10. `MapManager.init()` — initializes the minimap.
11. `CameraManager.init()` — creates the three layer containers and the temp container.
12. `GasManager.init()` — initializes gas data structures.
13. `initPixi()` (async, runs in parallel with `SoundManager.init()` and `finalizeUI()`):
    - Calls `pixi.init({ resizeTo: window, autoStart: false, canvas: #game-canvas, ... })` with renderer preferences, antialias, and resolution from CVars.
    - Optionally disables `preferCreateImageBitmap` to reduce RAM usage.
    - Calls `loadSpritesheets(modeName, renderer, highRes)` which populates the `Assets.cache` from packed spritesheets; triggers `EmoteWheelManager.init()` and `MapPingWheelManager.init()` on completion.
    - Relays `pointerdown` events from the game UI overlay to the canvas.
    - Registers `this.render` on the PixiJS ticker.
    - Creates two `Container` objects with `isRenderGroup: true`:
        - `gameContainer`: holds `CameraManager.container` (and optionally `DebugRenderer.graphics`).
        - `uiContainer`: holds `MapManager.container`, `MapManager.mask`, `netGraph` containers, `EmoteWheelManager.container`, and `MapPingWheelManager.container`.
    - Adds both containers to `pixi.stage`.
14. Menu music is loaded and auto-played via `@pixi/sound`.
15. Once all three parallel `Promise.all` tasks resolve: `unlockPlayButtons()` and `resetPlayButtons()`.

## WebSocket Connection

`Game.connect(address: string)` opens the connection:

- **URL:** the `address` parameter passed by the lobby/matchmaker UI.
- **Binary type:** `"arraybuffer"`.
- **`onopen`:** starts the PixiJS ticker (`pixi.start()`), stops menu music, resets UI, sends a `JoinPacket` with skin/badge/emote loadout from CVars, adds `gasRender.graphics` to the camera, and optionally starts a particle emitter.
- **`onmessage`:** see [Packet Dispatch](#packet-dispatch) below. Also records byte length via `netGraph.receiving.addEntry()`.
- **`onerror`:** stops the ticker, sets `this.error = true`, shows a splash error message, resets play buttons.
- **`onclose`:** stops the ticker, fires `socketCloseCallback`, resets play buttons; if the reason starts with `"Invalid game version"` it reloads the page; if the game is not yet over it shows a disconnect message and calls `endGame()`.

## Packet Dispatch

`_socket.onmessage` handler:

```typescript
this._socket.onmessage = (message: MessageEvent<ArrayBuffer>): void => {
    const stream = new PacketStream(message.data);
    let iterationCount = 0;
    while (true) {
        if (++iterationCount === 1e3) {
            console.warn("1000 iterations of packet reading; possible infinite loop");
        }
        const packet = stream.deserialize(splits);
        if (packet === undefined) break;
        this.onPacket(packet);
    }
    this.netGraph.receiving.addEntry(message.data.byteLength, splits);
};
```

`onPacket(packet: PacketDataOut)` routes via a **`switch` on `packet.type`** (`PacketType` enum):

| `PacketType` | Handler |
|---|---|
| `Joined` | `this.startGame(packet)` — sets up ambience, emotes, team mode, shows canvas |
| `Map` | `MapManager.updateFromPacket(packet)` |
| `Update` | `this.processUpdate(packet)` — the main per-tick game state update |
| `GameOver` | `UIManager.showGameOverScreen(packet)` |
| `Kill` | `UIManager.processKillPacket(packet)` |
| `Report` | `UIManager.processReportPacket(packet)` |
| `Pickup` | Plays item pickup sound; shows inventory message (not enough space, etc.) |

## Object Pool API

The pool is `Game.objects: ObjectPool<ObjectMapping>` where `ObjectMapping` maps each `ObjectCategory` key to its instance type.

**`ObjectClassMapping`** — a frozen object that maps `ObjectCategory` enum values to constructor classes:

```
ObjectCategory.Player       → Player
ObjectCategory.Obstacle     → Obstacle
ObjectCategory.DeathMarker  → DeathMarker
ObjectCategory.Loot         → Loot
ObjectCategory.Building     → Building
ObjectCategory.Decal        → Decal
ObjectCategory.Parachute    → Parachute
ObjectCategory.Projectile   → Projectile
ObjectCategory.SyncedParticle → SyncedParticle
```

**Full dirty objects** (new or re-spawned):

```typescript
for (const { id, type, data } of updateData.fullDirtyObjects ?? []) {
    const object = this.objects.get(id);
    if (object === undefined || object.destroyed) {
        const _object = new (ObjectClassMapping[type])(id, data);
        this.objects.add(_object);
    } else {
        object.updateFromData(data, false);
    }
}
```

If the id is unknown or the existing object is destroyed, a new instance is created and added. Otherwise `updateFromData(data, false)` is called.

**Partial dirty objects:**

```typescript
for (const { id, data } of updateData.partialDirtyObjects ?? []) {
    const object = this.objects.get(id);
    if (object === undefined) { console.warn(...); continue; }
    (object as GameObject).updateFromData(data, false);
}
```

No object creation — only updates. Warns if id is missing.

**Deleted objects:**

```typescript
for (const id of updateData.deletedObjects ?? []) {
    const object = this.objects.get(id);
    if (object === undefined) { console.warn(...); continue; }
    object.destroy();
    this.objects.delete(object);
}
```

## PixiJS Container Architecture

The PixiJS stage (`pixi.stage`) has two top-level render groups:

| Container | Contents |
|---|---|
| `gameContainer` (`isRenderGroup: true`) | `CameraManager.container`, optionally `DebugRenderer.graphics` |
| `uiContainer` (`isRenderGroup: true`) | `MapManager.container`, `MapManager.mask`, net graph containers, `EmoteWheelManager.container`, `MapPingWheelManager.container` |

**`CameraManager.container`** holds the entire world scene. Its `scale` is adjusted on every resize to implement zoom. Within it:

| Sub-container | Variable | Description |
|---|---|---|
| Basement layer | `layerContainers[0]` | All objects on `Layer.Basement`; `zIndex` toggles between `0` and `999` when transitioning to/from basement |
| Ground layer | `layerContainers[1]` | All objects on `Layer.Ground` |
| Upstairs layer | `layerContainers[2]` | All objects on `Layer.ToUpstairs` / upstairs |
| Temp transition container | `tempLayerContainer` | Objects moving between layers during the fade transition to prevent flickering |

The `gasRender.graphics` sprite is added directly to `CameraManager.container` on `onopen`, with `zIndex = 1000` (above all layer containers).

Z-ordering within each layer container is controlled by PixiJS's `zIndex` property set on each object's `container`, using values from the `ZIndexes` enum (see [ZIndex Layers](#zindex-layers)).

## PIXI_SCALE

**Value: `20`**

Converts game-world units to canvas pixels:

```
canvas_x = game_x × PIXI_SCALE    // e.g. game position 50 → pixel 1000
canvas_y = game_y × PIXI_SCALE
```

The camera zoom formula in `CameraManager.resize()` uses `PIXI_SCALE` as the denominator to compute the container scale from the scope's `zoomLevel`:

```
scale = (maxScreenDim × 0.5) / (zoomLevel × PIXI_SCALE)
```

## ZIndex Layers

Defined in `common/src/constants.ts` as `enum ZIndexes` (auto-incremented from 0):

| ZIndex | Value | Renders |
|--------|-------|---------|
| `Ground` | 0 | Ground plane / terrain base |
| `BuildingsFloor` | 1 | Building floor tiles |
| `Decals` | 2 | Decal objects (`Decal` class) |
| `DeadObstacles` | 3 | Destroyed/dead obstacles |
| `DeathMarkers` | 4 | Death marker objects (`DeathMarker` class) |
| `Explosions` | 5 | Explosion visuals |
| `ObstaclesLayer1` | 6 | Default obstacle layer (crates, barrels, etc.) |
| `Loot` | 7 | Loot items on the ground |
| `GroundedThrowables` | 8 | Throwables resting on the ground |
| `ObstaclesLayer2` | 9 | Obstacles drawn above loot (fences, etc.) |
| `TeammateName` | 10 | Teammate name labels |
| `Bullets` | 11 | Bullet trails |
| `DownedPlayers` | 12 | Downed/knocked player sprites |
| `Players` | 13 | Active player sprites |
| `ObstaclesLayer3` | 14 | Bushes, tables, and similar obstacles above players |
| `AirborneThrowables` | 15 | Throwables in flight |
| `ObstaclesLayer4` | 16 | Trees (leaves above airborne throwables) |
| `BuildingsCeiling` | 17 | Building ceiling tiles |
| `ObstaclesLayer5` | 18 | Obstacles that show on top of ceilings |
| `Emotes` | 19 | Emote bubbles |
| `Gas` | 20 | Gas zone overlay |

`gasRender.graphics` overrides this with `zIndex = 1000` to ensure the gas always renders above all scene objects.

`PLAYER_PARTICLE_ZINDEX` (defined in `client/src/scripts/utils/constants.ts`) is set to `ZIndexes.Players + 0.5`, placing player particles between `Players` and `ObstaclesLayer3`.

## Camera System Integration

`CameraManager` is a singleton instantiated as `new CameraManagerClass()`.

**Camera position** is set during the render loop in `Game.render()`:

```typescript
if (hasMovementSmoothing && this.activePlayer) {
    CameraManager.position = this.activePlayer.container.position;
}
```

**Zoom** is driven by the active scope: `CameraManager.zoom = scope.zoomLevel`. The setter calls `resize(true)` which animates the container scale via a 1250 ms `cubicOut` tween.

**Scale formula:** `scale = (maxScreenDim × 0.5) / (zoom × PIXI_SCALE)` where `maxScreenDim = max(min(w, h) × (16/9), max(w, h))`.

**Camera shake:** `CameraManager.shake(duration, intensity)` randomizes the position by `randomPointInsideCircle(Vec(0,0), intensity)` each frame while `Date.now() - shakeStart < shakeDuration`. Requires `cv_camera_shake_fx` CVar to be enabled.

**Shockwave effects:** `CameraManager.shockwave(duration, position, amplitude, wavelength, speed, layer)` adds a `Shockwave` instance that applies a `ShockwaveFilter` (from `pixi-filters`) to the relevant layer container.

**Layer transitions:** `CameraManager.updateLayer(newLayer, initial, oldLayer)` fades the three layer containers in/out over `LAYER_TRANSITION_DELAY` ms using `sineOut` tweens. A temporary container (`tempLayerContainer`) holds objects mid-transition to prevent flickering.

`Game.updateLayer()` delegates to `CameraManager.updateLayer()` and additionally updates sound layers, river/ocean ambience, terrain graphics visibility, and animates the background color from grass to void (basement).

## SuroiSprite

`SuroiSprite` extends PixiJS `Sprite`. Defined in `client/src/scripts/utils/pixi.ts`.

**Constructor:** requires no arguments. Centers the anchor at `(0.5, 0.5)` and sets position to `(0, 0)`. Accepts an optional `frame` string to load a texture immediately.

**Texture resolution:** `SuroiSprite.getTexture(frame: string): Texture` — looks up the frame key in `Assets.cache`. If not found, logs a warning and substitutes `"_missing_texture"`.

**Key methods (all return `this` for chaining):**

| Method | Description |
|--------|-------------|
| `setFrame(frame: string)` | Swaps texture to the given spritesheet frame key |
| `setAnchor(anchor: Vector)` | Copies a `Vector` into `this.anchor` |
| `setPivot(pivot: Vector)` | Copies a `Vector` into `this.pivot` |
| `setPos(x: number, y: number)` | Sets position |

**Spritesheet loading** (`loadSpritesheets`): detects whether the WebGL renderer supports 4096×4096 textures; if not, forces low-resolution (2048×2048) spritesheets. Dynamically imports the correct virtual module (`virtual:image-spritesheets-importer-high-res` or `…-low-res`) generated by the Vite spritesheet plugin, parses each spritesheet, uploads to the GPU via `renderer.prepare.upload()`, and populates `Assets.cache`. Progress is reflected in the loader text.

## Data Lineage

```
WebSocket message (ArrayBuffer)
  → PacketStream(message.data)
  → while loop: stream.deserialize(splits) → PacketDataOut
  → onPacket(packet) switch(packet.type):
      PacketType.Joined   → startGame()        → UI setup, ambience, emotes, CameraManager.updateLayer(initial)
      PacketType.Map      → MapManager.updateFromPacket() → terrain, rivers, buildings rendered
      PacketType.Update   → processUpdate():
          newPlayers      → Game.playerNames.set()
          playerData      → UIManager.updateUI() + updateWeaponSlots()
          fullDirtyObjects→ ObjectClassMapping[type].new(id, data) or updateFromData()
          partialDirtyObjects → object.updateFromData(data, false)
          deletedObjects  → object.destroy() + pool.delete()
          bullets         → new Bullet(data) → Game.bullets.add()
          explosions      → explosion(def, pos, layer)
          emotes          → player.showEmote(def)
          GasManager.updateFrom()
          planes          → new Plane(pos, dir) → Game.planes.add()
          mapPings        → MapManager.addMapPing()
          mapIndicators   → MapManager.updateMapIndicator()
          killLeader      → UIManager.updateKillLeader()
          → tick()
      PacketType.GameOver → UIManager.showGameOverScreen()
      PacketType.Kill     → UIManager.processKillPacket()
      PacketType.Report   → UIManager.processReportPacket()
      PacketType.Pickup   → SoundManager.play() + UIManager inventory message
```

## Complex Functions

### `Game.processUpdate(updateData)` — @file client/src/scripts/game.ts
**Purpose:** Processes the `UpdatePacket` — the primary per-tick game state message. Updates player names, creates/updates/destroys renderable objects, spawns bullets/explosions/emotes, updates gas and map indicators, then calls `tick()`.
**Implicit behavior:** Records `serverDt` (milliseconds since last update call) for interpolation. `fullDirtyObjects` can both create new objects and re-sync existing live ones. `partialDirtyObjects` only updates — silently skips unknown ids with a warning. Object destruction is deferred to this method; objects are not garbage-collected mid-frame.
**Called by:** `onPacket()` when `PacketType.Update`.

### `Game.init()` — @file client/src/scripts/game.ts
**Purpose:** Bootstraps the entire client — console, translations, input, UI, server data, gas/map/camera managers, PixiJS renderer, spritesheets, and sound.
**Implicit behavior:** `fetchServerData()` must resolve before PixiJS init because `modeName` (needed for spritesheet selection) is set during that call. PixiJS init, sound init, and UI finalization run in parallel via `Promise.all`. The play buttons remain locked until all three resolve.
**Called by:** `client/src/index.ts` (`void Game.init()`).

### `Game.render()` — @file client/src/scripts/game.ts
**Purpose:** The per-frame render callback registered on the PixiJS ticker. Runs pending `Timeout` callbacks, calls `object.update()` and `object.updateInterpolation()` on all pooled objects, advances tweens, updates bullets, particles, map, gas, planes, and the camera.
**Implicit behavior:** Returns early if `!this.gameStarted`. Camera position is only set from `activePlayer.container.position` when movement smoothing is enabled.
**Called by:** PixiJS ticker via `pixi.ticker.add(this.render.bind(this))`.

### `Game.connect(address)` — @file client/src/scripts/game.ts
**Purpose:** Opens the WebSocket, wires all socket event handlers, and sends the `JoinPacket` on open.
**Implicit behavior:** Guard `if (this.gameStarted) return` prevents double-connect. The `JoinPacket` is sent synchronously in `onopen` — no server handshake wait.
**Called by:** Play button handler in `client/src/scripts/ui.ts`.

## Configuration
| Setting | Effect | Source |
|---------|--------|--------|
| `PIXI_SCALE` | Game coordinate → pixel scale factor (value: `20`) | `client/src/scripts/utils/constants.ts` |
| `LAYER_TRANSITION_DELAY` | Duration (ms) of layer fade transitions and object update deferral (value: `200`) | `client/src/scripts/utils/constants.ts` |
| `UI_DEBUG_MODE` | When `true`, skips most HUD resets on connect (value: `false`) | `client/src/scripts/utils/constants.ts` |
| `FORCE_MOBILE` | Forces mobile layout regardless of device detection (value: `false`) | `client/src/scripts/utils/constants.ts` |
| `cv_renderer` / `cv_renderer_res` | PixiJS renderer backend and resolution from in-game console CVars | `client/src/scripts/console/variables.ts` |
| `cv_high_res_textures` / `mb_high_res_textures` | Selects 4096 vs 2048 px spritesheets | `client/src/scripts/console/variables.ts` |
| `cv_movement_smoothing` | Toggles interpolation call in render loop | `client/src/scripts/console/variables.ts` |
| `cv_camera_shake_fx` | Enables `CameraManager.shake()` | `client/src/scripts/console/variables.ts` |

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Client Rendering subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Rendering patterns
