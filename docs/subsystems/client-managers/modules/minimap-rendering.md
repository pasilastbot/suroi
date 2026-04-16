# Minimap Rendering Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/client-managers/README.md -->
<!-- @source: client/src/scripts/managers/uiManager.ts, client/src/scripts/game.ts -->

## Purpose
Manages minimap canvas rendering, dynamic scaling, grid overlay, object marker placement, viewport bounds visualization, and pan/zoom controls for navigational HUD display showing map layout, teammates, and enemies.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/uiManager.ts` | Minimap canvas setup, rendering pipeline | High |
| `client/src/scripts/game.ts` | Minimap updates in game tick, object marker updates | High |
| `client/src/scripts/managers/mapManager.ts` (if separate) | Minimap tilemap/terrain rendering | Medium |
| `common/src/constants.ts` | GameConstants for minimap sizing and colors | Low |
| `client/src/scripts/utils/pixi.ts` | Minimap canvas abstraction (Pixi.js rendering) | Medium |

## Business Rules

### Canvas Rendering

**Initial setup:**
- **Canvas element:** HTML5 canvas sized to minimap viewport
  - Size: 160 × 160 pixels (default, configurable)
  - Positioning: HUD corner (typically top-left or right)
  - Background: semi-transparent dark overlay (e.g., rgba(0, 0, 0, 0.7))
- **Context:** 2D canvas context (not WebGL)
  - Drawing API: standard `ctx.drawImage()`, `ctx.fillRect()`, etc.
  - No Pixi.js overhead (simple 2D rendering)

**Rendering layers (bottom to top):**
1. **Base map:** Terrain tilemap (grass, water, beach)
   - Scaled down: map width/height down to canvas size
   - Scale factor: `mapWidth / canvasWidth` (e.g., 800 units → 160 px = 5:1)
2. **Obstacles:** Buildings, trees, props
   - Rendered as filled rectangles or small shapes
   - Color per obstacle type (wood = brown, concrete = gray)
   - Opacity: semi-transparent to not obscure terrain
3. **Gas zone:** Current safe zone boundary
   - Rendered as circle outline
   - Color: yellow/orange for visibility
   - Updates every tick as gas shrinks
4. **Grid overlay:** Optional grid lines (typically off in production)
   - Light gray grid at major intervals (e.g., every 64 units)
   - Helps with position debugging
5. **Player markers:** Local player, teammates, enemies
   - Local player: green/blue dot, center-ish (offset for camera)
   - Teammates: cyan/light blue dots
   - Enemies: red dots (only if visible/scanned)
   - Size: 2–4 pixels for clear visibility

### Dynamic Scaling

**Scale calculation:**
```
viewportWidth = 800 units (game world)
viewportHeight = 600 units (game world)
mapWidth = 1600 units
mapHeight = 1200 units

canvasWidth = 160 pixels
canvasHeight = 160 pixels (square minimap)

scaleX = mapWidth / canvasWidth = 1600 / 160 = 10 units per pixel
scaleY = mapHeight / canvasHeight = 1200 / 160 = 7.5 units per pixel
scale = max(scaleX, scaleY) = 10 to maintain aspect ratio
```

**Adjustment logic:**
- Aspect ratio maintained (non-square maps become rectangular on minimap)
- Scale factor increases as map enlarges or canvas shrinks
- Inverse relationship: larger map = smaller scale (more zoom-out)
- Recalculated on:
  - Game start (once per game)
  - Window resize (if canvas resized)
  - Map change (if map rotated, though uncommon)

### Grid Overlay

**Grid rendering (debug feature):**
- **Toggle:** Can be enabled via command (e.g., `/debug minimap-grid`)
- **Spacing:** Major gridlines every N units (e.g., 64 units)
  - Corresponds to spatial grid cell size for quick reference
- **Color:** Light gray (e.g., rgba(128, 128, 128, 0.3))
- **Thickness:** 1 pixel
- **When enabled:** Grid drawn LAST (on top of other layers)
  - Helps players visualize chunking/culling regions
- **Performance:** Grid rendering disabled by default (reduces draw calls)

**Coordinate system:**
- Grid origin (0, 0) at top-left of map
- X-axis increases rightward
- Y-axis increases downward (screen coordinates)
- Grid scaled with minimap (`gridSpacing * scale`)

### Object Markers

**Marker types:**

| Marker | Color | Size | Visibility |
|--------|-------|------|------------|
| Local player | Green | 4 px | Always (center indicator) |
| Teammate | Cyan | 3 px | Always (same team) |
| Enemy | Red | 3 px | When visible (in viewport or revealed) |
| Spectator | Gray | 2 px | Current spec target only |
| Downed player | Orange | 3 px | If teammate, always; if enemy, only if visible |
| Airdrop | Yellow | 3 px | When in game world |
| Gas center | Purple | 2 px | Always (reference point) |

**Marker updates (per tick):**
```
for each visible object in game:
  screenX = (object.worldX - viewportMinX) / scale
  screenY = (object.worldY - viewportMinY) / scale
  
  if screenX >= 0 && screenX < canvasWidth && screenY >= 0 && screenY < canvasHeight:
    canvasCtx.fillRect(screenX - markerSize/2, screenY - markerSize/2, markerSize, markerSize)
```

**Occlusion:** If markers overlap, draw order precedence:
1. Local player (always on top)
2. Teammates
3. Enemies
4. Airdrops
5. Downed players

### Viewport Bounds Visualization

**Bounds rectangle:**
- Shows current camera viewport on minimap
- Rendered as outlined rectangle (not filled)
- Color: light white/gray (e.g., rgba(255, 255, 255, 0.5))
- Thickness: 2 pixels

**Viewport rectangle calculation:**
```
viewportMinX = camera.x - (screenWidth / 2)
viewportMaxX = camera.x + (screenWidth / 2)
viewportMinY = camera.y - (screenHeight / 2)
viewportMaxY = camera.y + (screenHeight / 2)

minimapX = (viewportMinX * scale) % canvasWidth
minimapY = (viewportMinY * scale) % canvasHeight
minimapWidth = screenWidth / scale
minimapHeight = screenHeight / scale

// Draw outline rect
ctx.strokeRect(minimapX, minimapY, minimapWidth, minimapHeight)
```

**Purpose:** Players see what portion of map is on-screen; helps with spatial awareness

### Pan & Zoom Controls

**Default mode:** Minimap fixed (no pan/zoom)
- Shows entire map
- Not interactive (click-to-pan disabled)
- Players use main camera for navigation

**Optional pan/zoom (might not be implemented):**
- **Click-to-pan:** Right-click minimap → camera pans to that location
  - Convert canvas coords to world coords
  - Animate camera transition (not instant)
- **Scroll to zoom:** Mouse wheel on minimap → zooms minimap in/out
  - Zoom factor: 0.5× (show full map) to 2× (zoomed detail)
  - Maintains viewport rect visibility
- **Constraints:** Pan/zoom constrained to map bounds
  - User can't pan off edge (minimap edges clamped)
  - Zoom-out limited to fit entire map

## Data Lineage

```
Game tick update received
  ↓
[Update camera position and viewport]
  ↓
[Clear minimap canvas]
  ↓
Render base map (terrain)
  ↓
Render obstacles (buildings, trees)
  ↓
Render gas zone overlay
  ↓
[If debug enabled: render grid]
  ↓
Render object markers:
  - Local player (green)
  - Teammates (cyan)
  - Enemies (red)
  - Airdrops (yellow)
  ↓
Render viewport rectangle
  ↓
[Minimap visible on HUD]
```

### Rendering Frequency
- **Per-game-tick:** Minimap re-rendered every game tick (40 TPS = 25 ms)
- **Can be optimized:** Skip redraw if camera/objects unchanged (rare)
- **Canvas update:** `ctx.putImageData()` or `drawImage()` submits to GPU

## Dependencies

- **Internal:**
  - Camera Management — Viewport (camera position, bounds)
  - Game Objects Client — Object positions, visibility
  - UI Management — Canvas element management
- **External:**
  - [Game Loop](../../game-loop/README.md) — Tick-based minimap update
  - [Visibility & LOS](../../visibility-los/README.md) — Enemy visibility culling (only render visible enemies)
  - [Map](../../map/README.md) — Map terrain data for minimap rendering
  - [Networking](../../networking/README.md) — Player and object positions from UpdatePacket

## Complex Functions

### Canvas Initialization
**Function:** `initializeMinimap(game: Game, container: HTMLElement): Canvas`  
**Source:** `client/src/scripts/managers/uiManager.ts`  
**Purpose:** Set up minimap canvas and render context

**Logic:**
```typescript
function initializeMinimap(game, container) {
  // Create canvas element
  const canvas = document.createElement('canvas');
  canvas.width = 160;
  canvas.height = 160;
  canvas.style.position = 'absolute';
  canvas.style.top = '10px';
  canvas.style.right = '10px';
  
  // Set up context
  const ctx = canvas.getContext('2d');
  ctx.imageSmoothingEnabled = false; // pixel-perfect
  
  // Calculate scale
  const mapBounds = game.map.getBounds();
  const scaleX = mapBounds.width / canvas.width;
  const scaleY = mapBounds.height / canvas.height;
  const scale = Math.max(scaleX, scaleY);
  
  // Store for later use
  game.minimap = {
    canvas,
    ctx,
    scale,
    mapBounds
  };
  
  container.appendChild(canvas);
  
  return canvas;
}
```

**Implicit behavior:**
- Canvas appended to container (usually HUD div)
- Scale pre-calculated (used for all subsequent renders)
- Image smoothing disabled (blocky/pixelated look for clarity)
- Context stored in game object (global access)

**Called by:**
- Game initialization — after map loaded, before first tick

### Minimap Render
**Function:** `renderMinimap(game: Game): void`  
**Source:** `client/src/scripts/managers/uiManager.ts` or `game.ts`  
**Purpose:** Update minimap each tick with current state

**Logic:**
```typescript
function renderMinimap(game) {
  const { canvas, ctx, scale, mapBounds } = game.minimap;
  
  // Clear canvas
  ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  
  // Render terrain (from tilemap)
  const terrainImage = game.map.getTerrainImage(); // pre-rendered tilemap
  const terrainScaled = scale * game.map.tileSize;
  ctx.drawImage(terrainImage, 0, 0, canvas.width, canvas.height);
  
  // Render obstacles
  ctx.fillStyle = 'rgba(100, 100, 100, 0.6)'; // gray for buildings
  for (const obstacle of game.obstacles) {
    const screenX = (obstacle.pos.x - mapBounds.minX) / scale;
    const screenY = (obstacle.pos.y - mapBounds.minY) / scale;
    const screenSize = obstacle.definition.hitbox.max / scale;
    ctx.fillRect(screenX - screenSize/2, screenY - screenSize/2, screenSize, screenSize);
  }
  
  // Render gas zone
  ctx.strokeStyle = 'rgba(255, 200, 0, 0.8)';
  ctx.lineWidth = 2;
  const gasX = (game.gas.position.x - mapBounds.minX) / scale;
  const gasY = (game.gas.position.y - mapBounds.minY) / scale;
  const gasRadius = game.gas.currentRadius / scale;
  ctx.beginPath();
  ctx.arc(gasX, gasY, gasRadius, 0, Math.PI * 2);
  ctx.stroke();
  
  // Render player markers
  renderPlayerMarkers(ctx, game, scale, mapBounds);
  
  // Render viewport bounds
  renderViewportRect(ctx, game, scale, mapBounds);
}
```

**Implicit behavior:**
- Canvas cleared and redrawn every tick (no incremental updates)
- Terrain pre-rendered (cached image, not redrawn)
- Obstacles drawn as simple shapes (not detailed)
- Gas rendered as circle arc (scalable, accurate)
- Called synchronously from game tick

**Called by:**
- Game tick handler — after updating game state, before rendering main scene

### Player Marker Rendering
**Function:** `renderPlayerMarkers(ctx: CanvasRenderingContext2D, game: Game, scale: number, mapBounds: Bounds): void`  
**Source:** `client/src/scripts/managers/uiManager.ts` (implied)  
**Purpose:** Draw player dots on minimap

**Logic:**
```typescript
function renderPlayerMarkers(ctx, game, scale, mapBounds) {
  const localPlayer = game.localPlayer;
  
  // Local player (green + crosshair)
  ctx.fillStyle = 'rgba(0, 255, 0, 1)';
  const localX = (localPlayer.pos.x - mapBounds.minX) / scale;
  const localY = (localPlayer.pos.y - mapBounds.minY) / scale;
  ctx.fillRect(localX - 2, localY - 2, 4, 4);
  
  // Teammates (cyan)
  ctx.fillStyle = 'rgba(0, 255, 255, 0.8)';
  for (const teammate of game.players) {
    if (teammate.teamId === localPlayer.teamId && teammate !== localPlayer) {
      const tmX = (teammate.pos.x - mapBounds.minX) / scale;
      const tmY = (teammate.pos.y - mapBounds.minY) / scale;
      ctx.fillRect(tmX - 1.5, tmY - 1.5, 3, 3);
    }
  }
  
  // Enemies (red, only if visible)
  ctx.fillStyle = 'rgba(255, 0, 0, 0.8)';
  for (const enemy of game.players) {
    if (enemy.teamId !== localPlayer.teamId && game.isPlayerVisible(enemy)) {
      const enemyX = (enemy.pos.x - mapBounds.minX) / scale;
      const enemyY = (enemy.pos.y - mapBounds.minY) / scale;
      ctx.fillRect(enemyX - 1.5, enemyY - 1.5, 3, 3);
    }
  }
  
  // Airdrops (yellow)
  ctx.fillStyle = 'rgba(255, 255, 0, 0.8)';
  for (const airdrop of game.airdrops) {
    const adX = (airdrop.pos.x - mapBounds.minX) / scale;
    const adY = (airdrop.pos.y - mapBounds.minY) / scale;
    ctx.fillRect(adX - 1.5, adY - 1.5, 3, 3);
  }
}
```

**Implicit behavior:**
- Local player always rendered (green, clear visibility)
- Enemies only rendered if `isPlayerVisible()` returns true (LOS/vision culling)
- Downed players (if tracked) drawn as different color (orange)
- Draw order: local → teammates → enemies → airdrops

**Called by:**
- `renderMinimap()` — as part of render pipeline

### Viewport Rectangle Rendering
**Function:** `renderViewportRect(ctx: CanvasRenderingContext2D, game: Game, scale: number, mapBounds: Bounds): void`  
**Source:** `client/src/scripts/managers/uiManager.ts` (implied)  
**Purpose:** Display current viewport bounds on minimap

**Logic:**
```typescript
function renderViewportRect(ctx, game, scale, mapBounds) {
  const camera = game.camera;
  const viewport = game.getViewport();
  
  // Convert viewport to minimap coords
  const vpX = (viewport.minX - mapBounds.minX) / scale;
  const vpY = (viewport.minY - mapBounds.minY) / scale;
  const vpWidth = viewport.width / scale;
  const vpHeight = viewport.height / scale;
  
  // Draw outline
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.6)';
  ctx.lineWidth = 1;
  ctx.strokeRect(vpX, vpY, vpWidth, vpHeight);
}
```

**Implicit behavior:**
- Viewport rect shows what's visible on main screen
- Rect updates every tick as camera pans
- Helps player understand active play area
- Outline doesn't obscure underlying content (low opacity)

**Called by:**
- `renderMinimap()` — as final render step

### Scale Recalculation
**Function:** `recalculateMinimapScale(game: Game): void`  
**Source:** `client/src/scripts/managers/uiManager.ts` (implied)  
**Purpose:** Recompute scale if canvas or map size changes

**Logic:**
```typescript
function recalculateMinimapScale(game) {
  const { canvas, mapBounds } = game.minimap;
  
  const scaleX = mapBounds.width / canvas.width;
  const scaleY = mapBounds.height / canvas.height;
  game.minimap.scale = Math.max(scaleX, scaleY);
  
  // Trigger full redraw next tick
  game.minimap.dirty = true;
}
```

**Implicit behavior:**
- Only needed on resize or map change (rare)
- Marks minimap as dirty (forces redraw next tick)
- Scale pre-calculated (not recomputed per-render)

**Called by:**
- Window resize handler — when canvas resized
- Map load — after map loaded

## Configuration

| Setting | Location | Effect | Default |
|---------|----------|--------|---------|
| `MINIMAP_WIDTH` | constants.ts | Canvas width in px | 160 |
| `MINIMAP_HEIGHT` | constants.ts | Canvas height in px | 160 |
| `MINIMAP_POSITION` | CSS / constants | Top-right corner | "10px 10px" |
| `GRID_SPACING` | constants.ts | Grid line interval in units | 64 |
| `SHOW_GRID` | debug flag | Enable grid overlay | false |
| `OBSTACLE_OPACITY` | constants.ts | Obstacle transparency on minimap | 0.6 |
| `GAS_COLOR` | constants.ts | Gas zone color (RGBA) | "255,200,0,0.8" |

## Edge Cases & Gotchas

### Gotcha: Scale Mismatch on Map Load
- **Issue:** Minimap initialized before map fully loaded
  - Map bounds undefined or incorrect
  - Scale calculated with wrong dimensions
  - Objects rendered off-screen
- **Prevention:** Defer minimap init until after map load complete; subscribe to 'map_loaded' event

### Gotcha: Marker Outside Canvas Bounds
- **Issue:** Player marker calculated outside canvas area
  - Object position not clamped to map bounds
  - Marker drawn at negative or >160px coords
  - Marker appears off-screen or wraps around
- **Prevention:** Clamp marker coords to `[0, canvasWidth]` and `[0, canvasHeight]`

### Gotcha: Gas Calculation Precision
- **Issue:** Gas shrinking with fractional coordinates
  - Math.PI × radius includes decimals
  - Arc drawn with pixel imprecision
  - Gas outline appears jagged or flickering
- **Prevention:** Round gas coords to nearest pixel; or use pixelated rendering style

### Gotcha: Enemy Visibility Stale
- **Issue:** `isPlayerVisible()` check uses cached visibility
  - LOS check frame-delayed (visibility computed last tick)
  - Enemy appears/disappears on minimap unexpectedly
- **Prevention:** Use current-frame LOS check; or reduce minimap update frequency to match LOS update rate

### Gotcha: Canvas Scaling on High-DPI
- **Issue:** Canvas rendered smaller on high-DPI displays (e.g., 4K)
  - CSS size != device pixels
  - Minimap appears tiny, hard to read
- **Prevention:** Set canvas `width/height` attributes to `devicePixelRatio * cssSize`; adjust drawing coords accordingly

### Gotcha: Viewport Rect Flicker
- **Issue:** Viewport rect drawn on top of gas circle
  - If both white-ish color, visual confusion
  - Rect appears to flicker in/out as coords change
- **Prevention:** Use distinct colors (white for rect, orange for gas); or render rect with checkerboard pattern

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Client Managers subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — HUD rendering and UI patterns
- **Related modules:**
  - [Camera Management](../../camera-management/README.md) — Viewport and camera bounds
  - [Visibility & LOS](../../visibility-los/README.md) — Enemy visibility culling for minimap
  - [Map](../../map/README.md) — Map terrain and layout rendering
  - [Game Loop](../../game-loop/README.md) — Tick-based minimap update
