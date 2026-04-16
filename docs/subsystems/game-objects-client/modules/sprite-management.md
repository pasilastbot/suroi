# Sprite Management Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/game-objects-client/README.md -->
<!-- @source: client/src/scripts/objects/*.ts -->

## Purpose

Manages texture loading from spritesheets, sprite creation and pooling per object type, sprite property updates (rotation, scale, tint), animation frame selection, and sprite cleanup.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [client/src/scripts/objects/gameObject.ts](../../../../../client/src/scripts/objects/gameObject.ts) | Base GameObject class with sprite initialization and lifecycle | High |
| [client/src/scripts/objects/player.ts](../../../../../client/src/scripts/objects/player.ts) | Player sprite (skin rendering, direction rotation, equipment overlay) | High |
| [client/src/scripts/objects/obstacle.ts](../../../../../client/src/scripts/objects/obstacle.ts) | Obstacle sprites (trees, crates, doors, rotation from server) | High |
| [client/src/scripts/objects/building.ts](../../../../../client/src/scripts/objects/building.ts) | Building sprites (walls, roofs, rotation) | Medium |
| [client/src/scripts/utils/pixi.ts](../../../../../client/src/scripts/utils/pixi.ts) | Spritesheet loading, texture caching, `SuroiSprite` wrapper class | High |
| [client/src/scripts/game.ts](../../../../../client/src/scripts/game.ts) | ObjectPool management, object pooling/reuse | Medium |

## Business Rules

- **Spritesheet Loading**: All textures pre-loaded at startup via `loadSpritesheets()` (async, awaited before game starts)
- **Sheet Resolution**: Desktop = 4096×4096 high-res, Mobile = 2048×2048 low-res (selected via Vite build plugin)
- **Texture Caching**: `TEXTURE_CACHE` Map prevents duplicate texture loads; textures are reused across all instances
- **Object Pooling**: `ObjectPool<T>` pre-allocates sprites for each category (Player, Obstacle, Building, etc.), reuses instances
- **Sprite Rotation**: Updated every tick when object orientation changes; rotation in radians (`Math.atan2`)
- **Sprite Scale**: Applied for obstacles/buildings with size variations; players have fixed scale with equipment overlays
- **Sprite Tint**: Applied for damage/status effects (e.g., player being downed = red tint, gas damage = green tint)
- **Animation**: Frame selection via sprite name suffix (e.g., `player_base_male_0`, `player_base_male_1` for frame 0 vs 1)
- **Cleanup**: Sprites are NOT destroyed; instance is returned to pool for reuse in subsequent games
- **ParticleContainer**: High-performance sprite rendering for particles and bullets (less transform overhead)

## Data Lineage

```
Game Startup
  ↓
loadSpritesheets() → load all texture atlases from build output (__SPRITESHEETS__)
  ↓
TEXTURE_CACHE populated with [textureName → Texture mapping]
  ↓
Game connects to server
  ↓
Server sends UpdatePacket with ObjectsNetData for each object category
  ↓
Game.objects.get(objectID) → retrieve or create GameObject:
  1. Check ObjectPool for unused instance
  2. If reused → call .updateFromData(packet)
  3. If new → create GameObject subclass → initialize sprite → add to layer container
  ↓
Sprite Rendering Loop (each frame):
  1. GameObject.update() called with server data
  2. Sprite rotation/scale updated based on object state
  3. Tint applied for effects (damage, downed, gas)
  4. AnimationFrame selected based on tween/state
  ↓
Game ends
  ↓
Sprites not destroyed; instances returned to ObjectPool for next game
```

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| Spritesheet URL | Built from Vite plugin; injected at runtime as `__SPRITESHEETS__` | `client/vite/plugins/` |
| Texture cache | In-memory texture reuse; cleared on game disconnect | `pixi.ts` TEXTURE_CACHE Map |
| Mobile texture res | 2048×2048 (vs 4096×4096 desktop) | Vite build-time selection |
| ObjectPool maxSize | Pre-allocated sprites per category | `GameConstants.player.maxAllowedPlayers`, etc. |

## Complex Functions

### `SuroiSprite` Class → @file client/src/scripts/utils/pixi.ts
**Purpose:** Wrapper around PixiJS `Sprite` with fluent API for common operations (tint, rotation, zIndex).

**Implicit behavior:**
- `setTint(color)` → applies `ColorMatrixFilter` for visual effects
- `setZIndex(z)` → updates internal `_currentZIndex` (handles sorting)
- `setAngle(radians)` → updates `rotation` property
- `setScale(x, y)` → scales width/height
- Chainable: `new SuroiSprite().setTexture(...).setScale(...).setTint(...)`

### `GameObject.updateFromData(newData)` — @file client/src/scripts/objects/gameObject.ts
**Purpose:** Update sprite properties from server UpdatePacket data.

**Implicit behavior:**
- Called every frame when object data changes
- Updates position, rotation, scale, animation frame
- Applies sprite tint based on status (downed, gas damage, etc.)
- May create/destroy child sprites for equipment (scope on gun, armor visual)
- Position already in world-space; camera translation applied by render pipeline

### `Player.update()` / `Obstacle.update()` / `Building.update()` — @file client/src/scripts/objects/*.ts
**Purpose:** Render object-specific sprite variants based on definition and state.

**Implicit behavior:**
- **Player**: Select skin texture, apply direction rotation, layer equipment overlays (gun, armor, helmet)
- **Obstacle**: Select texture based on obstacle type (tree=transparent, crate=solid), apply rotation
- **Building**: Select texture based on building type/layer, apply wall/roof rotation
- Handles layer transitions (ground ↔ upstairs) by reparenting to appropriate layer container

### `loadTextures()` — @file client/src/scripts/utils/pixi.ts
**Purpose:** Pre-load all spritesheet textures from `__SPRITESHEETS__` array at startup.

**Implicit behavior:**
- Async operation; game waits for promise before continuing
- Called before any objects instantiated
- Populates PixiJS texture cache; subsequent sprite creations reference cached textures
- On failure → console error; game continues (sprites will appear missing)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Client-side game objects overview
- **Tier 2:** [../../layer-ordering/README.md](../../layer-ordering/README.md) — Layer system and Z-ordering
- **Tier 2:** [../../client-rendering/README.md](../../client-rendering/README.md) — PixiJS rendering pipeline
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — Client architecture
