# Map Rendering

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/map/README.md -->
<!-- @source: client/src/scripts/managers/mapManager.ts, client/src/scripts/game.ts -->

## Purpose
Handles client-side rendering of the game world background, including map sprites, tilesets per game mode, island boundary visualization, and viewport-relative positioning for efficient culling.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/mapManager.ts` | Main map rendering manager, sprite setup, viewport culling | High |
| `client/src/scripts/game.ts` | Game loop integration, map sprite creation | High |
| `client/src/scripts/utils/pixi.ts` | PixiJS utilities, sprite scaling, terrain graphics | High |
| `common/src/utils/terrain.ts` | Terrain type definitions, floor name mappings | Medium |

## Business Rules

- **Sprite Caching:** Map background sprite is created once at game join and cached; not recreated per frame
- **Viewport Culling:** Map sprites are hidden/shown based on viewport bounds to reduce render load (off-screen objects not rendered)
- **Mode-Specific Tilesets:** Game modes (Normal, Winter) each have unique tileset textures; switching modes reloads map background
- **Island Boundary:** Visual boundary between island and ocean rendered as border layer between zones
- **Coordinate System:** Map uses game world coordinates; PixiJS renders at scaled viewport position
- **Zoom-Relative Positioning:** Player position kept centered (or offset for ADS); map sprite position follows camera

## Data Lineage

```
Server MapData (from JoinedPacket)
  ↓
MapPacket deserialization (terrain, buildings, obstacles)
  ↓
Client MapManager.render() setup
  ↓
Map background RenderTexture created
  ↓
Terrain graphics drawn via drawGroundGraphics()
  ↓
Sprite texture applied to PixiJS Sprite
  ↓
Sprite positioned at (0, 0) in world coords
  ↓
Viewport manager updates sprite position per camera frame
  ↓
PixiJS renderer draws sprite at scaled position
```

## Complex Functions

### `MapManager.render(data: MapData)` — @file client/src/scripts/managers/mapManager.ts
**Purpose:** Initialize map rendering from server map data; create background sprite and add to display tree.

**Implicit behavior:**
- Creates a new `RenderTexture` for terrain graphics (supports A/B testing different tileset resolutions)
- Calls `drawGroundGraphics()` to render terrain types per layer
- Converts rendered graphics to sprite texture
- Positions sprite at world origin (0, 0)
- Adds sprite to the bottom of the display container (behind all game objects)
- Listens for viewport zoom changes to adjust sprite position

**Called by:** `Game.connect()` (during initial join)

**Example:**
```typescript
// From game.ts - after receiving JoinedPacket
const mapData = joinedPacket.mapData;
this.mapManager.render(mapData);  // Sprite now visible in viewport
```

### `drawGroundGraphics(graphics: Graphics, terrain: Terrain)` — @file client/src/scripts/utils/pixi.ts
**Purpose:** Rasterize terrain data into PixiJS Graphics for efficient rendering.

**Implicit behavior:**
- Iterates over terrain grid cells
- For each cell: determine tile type (grass, water, beach, etc.) by terrain enum
- Draw colored rectangle for that tile at (cellX, cellY) in world coordinates
- Grass = green, water = blue, beach = beige, etc. (colors from mode tileset)
- Clips water/beach to terrain boundaries (respects island boundary)

**Called by:** `MapManager.render()`

**Example:**
```typescript
const terrain = mapData.terrain;  // Grid of floor types
const graphics = new Graphics();
drawGroundGraphics(graphics, terrain);
// graphics now contains colored rectangles for entire island
```

## Configuration
| Setting | Effect | Default |
|---------|--------|---------|
| `RENDER_TEXTURE_SIZE` | Resolution of terrain RenderTexture | 4096×4096 |
| `PIXI_SCALE` | Game world units → screen pixels scale factor | 1.0 |
| `MAP_SPRITE_VISIBLE` | Toggle map sprite rendering on/off | true |

## Viewport-Relative Positioning

Map sprite position is updated each frame to keep the island centered in the viewport:

```typescript
// In CameraManager update
const viewportCenter = { x: player.x, y: player.y };  // or offset for ADS
mapSprite.position = toPixiCoords(
  -viewportCenter.x + viewport.width / 2,
  -viewportCenter.y + viewport.height / 2
);
```

This ensures:
- Player always at center of screen (or ADS offset)
- Map background moves to counteract camera movement
- Off-screen map areas not rendered (culled by PixiJS)

## Related Documents
- **Tier 2:** [../README.md](../README.md) — Map generation, loot spawning, obstacle placement
- **Tier 2:** [../../camera-management/README.md](../../camera-management/README.md) — Viewport positioning and zoom
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — Client rendering architecture
- **Patterns:** [../../client-rendering/README.md](../../client-rendering/README.md) — Rendering patterns (sprite pooling, dirty tracking)
