# Container & Viewport Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/client-rendering/README.md -->
<!-- @source: client/src/scripts/game.ts -->

## Purpose

Manages PixiJS container hierarchy: root stage structure, game container (world objects), HUD container (UI overlays), layer containers (ground/upstairs/air), particle containers, and interactive event layers for viewport rendering.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [client/src/scripts/game.ts](../../../../../client/src/scripts/game.ts) | Stage hierarchy, container initialization, layer management, z-index sorting | High |
| [client/src/scripts/managers/cameraManager.ts](../../../../../client/src/scripts/managers/cameraManager.ts) | Camera container (world viewport), camera position updates | High |
| [client/src/scripts/managers/mapManager.ts](../../../../../client/src/scripts/managers/mapManager.ts) | Minimap container (separate viewport) | Medium |
| [client/src/scripts/managers/particleManager.ts](../../../../../client/src/scripts/managers/particleManager.ts) | Particle layer using ParticleContainer for performance | Medium |
| [client/src/scripts/managers/gasManager.ts](../../../../../client/src/scripts/managers/gasManager.ts) | Gas visualization layer (safe zone, gas ring) | Medium |

## Business Rules

- **Root Stage**: `pixi.stage` is PixiJS Application root; contains gameContainer + uiContainer
- **Game Container**: World viewport with isRenderGroup=true (render optimization)
  - Contains: CameraManager.container (world objects), DebugRenderer graphics (debug layer)
  - Clipped to game canvas bounds
  - Camera position applied to offset world view
- **UI Container**: Screen-space HUD with isRenderGroup=true
  - Contains: MapManager.container (minimap), netGraph containers, EmoteWheelManager, MapPingWheelManager
  - Fixed to screen (doesn't move with camera)
  - Always rendered on top of game world
- **Layer Containers**: Within CameraManager.container, layer hierarchy for 2.5D rendering:
  - Ground Layer: Obstacles, ground-level objects, shadows
  - Upstairs Layer: Obstacles/buildings with second floor (hidden when hideSecondFloor=true)
  - Air Layer: Particles, bullets, projectiles (rendered on top)
- **ParticleContainer**: High-performance sprite rendering for particles/bullets (ParticleContainer skips transforms for children)
- **Interactive Layers**: Mouse events routed to game world via CameraManager.container (click to spectate)
- **Z-Ordering**: Managed per layer via `zIndex` property; PixiJS sortChildren() called after updates
- **Container Cleanup**: Containers not destroyed; reused between games (reparent children, clear, reset)

## Data Lineage

```
PixiJS Application Initialization
  ↓
Game.init() → initPixi() creates pixi instance
  ↓
Create Stage Hierarchy:
  pixi.stage (root)
    ├── gameContainer (isRenderGroup: true)
    │   ├── CameraManager.container (world objects)
    │   │   ├── {Layer.Ground} container
    │   │   ├── {Layer.Upstairs} container
    │   │   └── {Layer.Air} container
    │   └── DebugRenderer.graphics (if DEBUG_CLIENT)
    │
    └── uiContainer (isRenderGroup: true, screen-space)
        ├── MapManager.container (minimap)
        ├── netGraph containers
        ├── EmoteWheelManager.container
        └── MapPingWheelManager.container

Each Game Tick:
  1. Update gameContainer position (camera offset)
  2. Update layer visibility (hideSecondFloor affects Upstairs layer)
  3. Game objects update sprite properties
  4. Call pixi.stage.sortChildren() to sort by zIndex
  5. Call pixi.render() to draw frame

Game Ends / Disconnect:
  - Clear all containers (reparent children to null or remove)
  - Reset camera position
  - Do NOT destroy containers (reuse for next game)
```

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| `hideSecondFloor` | Toggle Upstairs layer visibility | `Game.hideSecondFloor` boolean, controlled by layer transition system |
| `gameContainer.isRenderGroup` | Enable batch rendering for performance | Hardcoded in `Game.init()` |
| `uiContainer.isRenderGroup` | Batch HUD rendering | Hardcoded in `Game.init()` |
| `zIndex` thresholds | Layer rendering order (ground < upstairs < air) | `ZIndexes` enum in `@common/constants` |
| `LAYER_TRANSITION_DELAY` | Milliseconds to delay layer switch after player moves between floors | `client/src/scripts/utils/constants.ts` |

## Complex Functions

### `Game.init()` → initPixi() — @file client/src/scripts/game.ts
**Purpose:** Create PixiJS application and stage hierarchy.

**Implicit behavior:**
- Awaits font loading, spritesheet loading
- Creates new `Application()` instance with canvas resizeTo window
- Initializes `gameContainer` with isRenderGroup=true for batching
- Initializes `uiContainer` with isRenderGroup=true
- Adds both to pixi.stage
- If DEBUG_CLIENT: adds DebugRenderer.graphics to gameContainer
- Sets up resize listener for responsive canvas

### `CameraManager.container` Structure — @file client/src/scripts/managers/cameraManager.ts
**Purpose:** Manage world viewport within game container.

**Implicit behavior:**
- Position updated every frame based on player position (smooth camera follow)
- Contains all game objects (obstacles, players, loot, buildings, particles)
- Internally organized by Layer (Ground, Upstairs, Air)
- Z-index determined by object position and layer
- Scale applied for zoom effects (camera zoom in/out)
- Mouse events on world objects routed through this container

### `Game.hideSecondFloor` Transition — @file client/src/scripts/game.ts
**Purpose:** Smoothly transition Upstairs layer visibility when player moves between floors.

**Implicit behavior:**
- When player layer changes: set `hideSecondFloor = !Game.layer == Layer.Ground`
- Delay applied before actually hiding (LAYER_TRANSITION_DELAY) so player can see transition
- Upstairs container alpha tweened from 1 → 0 or vice versa
- Used to prevent visual clutter when inside buildings

### `pixi.stage.sortChildren()` — PixiJS Built-in
**Purpose:** Re-sort all children by zIndex after updates.

**Implicit behavior:**
- Called after every game tick (after all objects updated)
- Ensures correct rendering order: ground objects behind air objects
- Performance: O(n log n) sort; generally fast unless thousands of objects
- Without sortChildren(): z-ordering becomes inconsistent (objects rendered in add order)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Client rendering subsystem overview
- **Tier 2:** [./layer-ordering.md](./layer-ordering.md) — Layer system and Z-ordering details
- **Tier 2:** [../../camera-management/README.md](../../camera-management/README.md) — Camera management
- **Tier 2:** [../../particle-system/README.md](../../particle-system/README.md) — Particle rendering
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — Client rendering architecture
