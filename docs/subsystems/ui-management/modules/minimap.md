# Minimap Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/ui-management/README.md -->
<!-- @source: client/src/scripts/managers/mapManager.ts -->

## Purpose

Renders a dynamic minimap that displays the game world in reduced scale. Tracks player position, teammates, enemy players, pings, gas zone, and safe zone on a 2D canvas.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [client/src/scripts/managers/mapManager.ts](../../../../../client/src/scripts/managers/mapManager.ts) | Minimap container, rendering logic, marker management, terrain display | High |
| [client/src/scripts/ui.ts](../../../../../client/src/scripts/ui.ts) | Menu toggle for minimap expand/collapse | Medium |
| [client/index.html](../../../../../client/index.html) | HTML structure for minimap container and close button | Low |

## Business Rules

- **Size**: Minimap dimensions scale based on map size (via `_minimapWidth`, `_minimapHeight`)
- **Visibility Toggle**: Player can toggle minimap on/off via `toggleMinimap()`
- **Expanded Mode**: Double-click (mobile) or button click expands minimap to full screen; close button collapses it
- **Terrain Rendering**: Beach/grass boundaries drawn as colored regions with rounded corners
- **Player Position**: Red/team-colored indicator at player's world position, scaled to minimap coordinates
- **Teammate Markers**: Different colored indicators per teammate (via `TEAMMATE_COLORS`)
- **Enemy Detection**: Enemy player indicators rendered via `UpdatePacket` with player object data
- **Ping Placement**: Ping markers for map pings and teammate pings rendered with distinct graphics
- **Gas Visualization**: Safe zone (white circle) and gas ring (red overlay) updated every tick
- **Container Masking**: `_objectsContainer` masked by `mask` graphics to clip content within minimap bounds

## Data Lineage

```
Server UpdatePacket (player positions, teammate data, object state)
  ↓
Game.objects ObjectPool (current player object, teammate list)
  ↓
MapManager.updateObjects() → extract position/visibility data
  ↓
Minimap rendering pipeline:
  1. drawTerrain() → terrain graphics
  2. Draw locations/places sprites
  3. GasRender.render() → gas visualization
  4. Update player indicator position (scaled)
  5. Update teammate indicators (scaled)
  6. Render ping markers
  7. Render safe zone circle
```

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| `minimapWidth` / `minimapHeight` | Dynamic minimap canvas size | `mapManager.ts`, scaled from sprite dimensions |
| `margins` | Minimap position offset on screen | Calculated during `updateZoom()` / mobile layout |
| `TEAMMATE_COLORS` | Color palette for teammate indicators | `client/src/scripts/utils/constants.ts` |
| `expanded` | Bool: true = full-screen map, false = corner minimap | `MapManager._expanded` |
| `visible` | Bool: minimap rendered on screen or hidden | `MapManager._visible` |

## Complex Functions

### `MapManager.updateObjects()` — @file client/src/scripts/managers/mapManager.ts
**Purpose:** Render all live game objects on minimap (obstacles, buildings, loot, active effects).

**Implicit behavior:**
- Called every frame from game loop
- Objects scaled by `scale` factor (minimap coords vs world coords)
- Terrain boundaries define render boundary (beach/grass hitbox)
- Obstacles/buildings drawn as small rectangles/icons
- Z-index sorting ensures correct layer order

### `MapManager.updatePosition()` — @file client/src/scripts/managers/mapManager.ts
**Purpose:** Update player indicator and camera marker on minimap as player moves.

**Implicit behavior:**
- Scales world-space position (`Vec`) to minimap coordinates
- Indicator rotation matches player's facing direction
- Clamped to minimap bounds (won't exceed `mask` graphics)

### `MapManager.switchToBigMap()` / `switchToSmallMap()` — @file client/src/scripts/managers/mapManager.ts
**Purpose:** Toggle between expanded full-screen map and corner minimap.

**Implicit behavior:**
- `switchToBigMap()`: Expand to full screen, reposition UI, increase scale
- `switchToSmallMap()`: Shrink to corner, restore original position/scale
- Calls `updateZoom()` internally to recalculate margins and dimensions
- Fade/tween animation applied via layer system (smooth transition)

### `MapManager.drawTerrain()` — @file client/src/scripts/managers/mapManager.ts
**Purpose:** Render beach and grass boundaries as rounded polygon shapes.

**Implicit behavior:**
- Uses PixiJS `Graphics.roundShape()` for anti-aliased edges
- Beach drawn first, then grass (hole-in-the-middle technique)
- Scale parameter allows desktop vs mobile resolution differences
- Called during map initialization and resize events

## Related Documents

- **Tier 2:** [../README.md](../README.md) — UI Management subsystem overview
- **Tier 2:** [../../camera-management/README.md](../../camera-management/README.md) — Camera viewport that minimap mirrors
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — Client rendering layer architecture
- **Patterns:** [../patterns.md](../patterns.md) — UI state synchronization patterns
