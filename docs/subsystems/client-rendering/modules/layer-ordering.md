# Client Rendering — Layer System & Z-Ordering

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/client-rendering/README.md -->
<!-- @source: client/src/scripts/game.ts, common/src/constants.ts, common/src/utils/layer.ts -->

## Purpose

Manage correct visual stacking of game objects across 5 vertical layers (basement through upstairs) and ensure proper Z-index ordering within each layer for 2D depth perception. This module is essential for the isometric-style rendering where object positions must be sorted visually to create depth illusion.

## Layer Hierarchy

**5 logical game layers** with defined enum values (@file common/src/constants.ts):

```typescript
export enum Layer {
    Basement = -2,     // Underground rooms, caves
    ToBasement = -1,   // Stairs going down — visual transition layer
    Ground = 0,        // Main outdoor level — buildings, trees, players
    ToUpstairs = 1,    // Stairs going up — visual transition layer
    Upstairs = 2       // Second floor — interiors, roofs
}
```

**3 rendering containers** mapped from layers (@file common/src/utils/layer.ts):

```typescript
export enum LayerContainer { 
    Basement = 0,   // Renders basement objects
    Ground = 1,     // Renders ground + stair transition objects
    Upstairs = 2    // Renders upstairs objects
}
```

## Container Mapping Logic

Objects are assigned to **LayerContainers** based on the player's current active layer:

```typescript
// @file common/src/utils/layer.ts:154
export function getLayerContainer(objectLayer: Layer, activeLayer: Layer): LayerContainer {
    switch (objectLayer) {
        case Layer.Basement:
            return LayerContainer.Basement;  // Always render in basement container
        
        case Layer.ToBasement:
            // Transition layer — show in same container as player approaches
            return activeLayer <= Layer.ToBasement 
                ? LayerContainer.Basement 
                : LayerContainer.Ground;
        
        case Layer.Ground:
            return LayerContainer.Ground;     // Always render in ground container
        
        case Layer.ToUpstairs:
            // Transition layer — show in same container as player approaches
            return activeLayer >= Layer.ToUpstairs 
                ? LayerContainer.Upstairs 
                : LayerContainer.Ground;
        
        case Layer.Upstairs:
            return LayerContainer.Upstairs;   // Always render in upstairs container
    }
}
```

### Example: Player Moving from Ground to Upstairs

```
Step 1: Player on Ground (Layer.Ground)
  → activeLayer = 0
  → Ground objects use Container[1]
  → Upstairs objects use Container[1].alpha = 0 (invisible)
  → Floor rendering visible

Step 2: Player enters stairs (Layer.ToUpstairs)
  → activeLayer = 1
  → ToUpstairs layer: activeLayer >= 1 ? use Container[2] : Container[1]
  → Both containers fade over 200ms
  → Stair visual transitions begin

Step 3: Player climbing (Layer.ToUpstairs → Layer.Upstairs)
  → activeLayer = 2
  → ToUpstairs layer: activeLayer >= 1 ? use Container[2]
  → Upstairs objects now fully visible (alpha=1)
  → Ground container invisible (alpha=0)
```

## Display Hierarchy & Z-Indexing

**Container arrangement** (@file client/src/scripts/managers/cameraManager.ts:24):

```typescript
// Three layer containers as children of main camera container
readonly layerContainers = [
    new Container(),  // [0] Basement
    new Container(),  // [1] Ground
    new Container()   // [2] Upstairs
];

// Each container's zIndex determines stacking:
for (let i = 0; i < layerContainers.length; i++) {
    const container = layerContainers[i];
    container.zIndex = i;  // Basement: 0, Ground: 1, Upstairs: 2
    this.container.addChild(container);
}
```

When player is **on basement or approaching stairs going down**:
```typescript
// Basement gets priority (zIndex = 999) to appear above all ground buildings
if (i === 0 && container.zIndex !== 999) {
    container.zIndex = newLayer <= Layer.ToBasement ? 999 : 0;
}
```

### Z-Index Enum (Rendering Order within Container)

Objects within a container are sorted by ZIndex enum values (@file common/src/constants.ts:108):

```typescript
export enum ZIndexes {
    Ground = 0,                // Terrain / floor
    BuildingsFloor = 1,        // Building floor visuals
    Decals = 2,                // Ground decals
    DeadObstacles = 3,         // Destroyed objects
    DeathMarkers = 4,          // Death position markers
    Explosions = 5,            // Explosion visuals
    ObstaclesLayer1 = 6,       // Default obstacle layer
    Loot = 7,                  // Item pickups
    GroundedThrowables = 8,    // Grenades on ground
    ObstaclesLayer2 = 9,       // Mid-level obstacles
    TeammateName = 10,         // Teammate name labels
    Bullets = 11,              // Projectiles in flight
    DownedPlayers = 12,        // Downed player visuals
    Players = 13,              // Standing players
    ObstaclesLayer3 = 14,      // High obstacles (bushes, tables)
    AirborneThrowables = 15,   // Flying grenades
    ObstaclesLayer4 = 16,      // Very high obstacles (trees)
    BuildingsCeiling = 17,     // Building ceiling visuals
    ObstaclesLayer5 = 18,      // Above ceilings
    Emotes = 19,               // Chat bubbles, emotes
    Gas = 20                   // Gas cloud rendering
}

export const Z_INDEX_COUNT = 21;  // Total distinct Z-index levels
```

**Rendering order**: `0 (back) → 1 → 2 → ... → 20 (front)`

## Y-Position Sorting (Isometric Depth)

Within each **layer container**, PixiJS renders children **left-to-right in the display list**. To simulate isometric depth (lower Y = further back), objects must be **sorted by Y position before rendering**:

```
Display List Order (back → front):
├─ Object at y=100 (top of screen, furthest back)
├─ Object at y=250
├─ Object at y=400 (depth-sorted)
└─ Object at y=600 (bottom of screen, closest)
```

### Manual Z-Index Assignment per Object Type

Object definitions specify their **base zIndex** which is adjusted per instance. (@file client/src/scripts/objects/obstacle.ts:115):

```typescript
// Default obstacle zIndex from definition, or ZIndexes.ObstaclesLayer1
definition.graphicsZIndex ?? ZIndexes.ObstaclesLayer1;

// Particle effects above the object:
zIndex: Numeric.max((definition.zIndex ?? ZIndexes.ObstaclesLayer1) + 1, ZIndexes.Players);

// Glow effects slightly behind:
zIndex: glow.zIndex ?? this.container.zIndex - 0.5;
```

### Player Particle Z-Index

Particles emitted by players render **above the player** (@file client/src/scripts/utils/constants.ts:13):

```typescript
export const PLAYER_PARTICLE_ZINDEX = ZIndexes.Players + 0.5;
```

This ensures bleeding particles, healing effects, and status indicators appear in front of all game objects.

## Layer Transitions

**Transition timing** (@file client/src/scripts/utils/constants.ts:11):

```typescript
export const LAYER_TRANSITION_DELAY = 200;  // milliseconds
```

### Fade-In/Out Animation

When player changes layers, containers **cross-fade** (@file client/src/scripts/managers/cameraManager.ts:128):

```typescript
updateLayer(newLayer: Layer, initial = false, oldLayer?: Layer): void {
    for (let i = 0, len = this.layerContainers.length; i < len; i++) {
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
                visible = newLayer >= (Game.hideSecondFloor ? Layer.ToUpstairs : Layer.ToBasement);
                break;
        }
        
        const targetAlpha = visible ? 1 : 0;
        
        // Tween alpha over LAYER_TRANSITION_DELAY (200ms) with sineOut easing
        this.layerTweens[i] = Game.addTween({
            target: container,
            to: { alpha: targetAlpha },
            duration: LAYER_TRANSITION_DELAY,
            ease: EaseFunctions.sineOut,
            onComplete: () => {
                container.visible = visible;
            }
        });
    }
}
```

**Visual result**:
- At 0% transition: Current layer visible at 100%, new layer invisible
- At 50% transition: Both layers visible at 50% (crossfade)
- At 100% transition: New layer visible at 100%, old layer invisible

## Temporary Layer Container

**Problem**: Moving objects between containers during layer transition causes visible flickering (object vanishes from current container before appearing in new container).

**Solution** (@file client/src/scripts/managers/cameraManager.ts:29):

```typescript
// Temporary container for objects moving between layers
readonly tempLayerContainer = new Container();

// During layer transition, use temp container for objects changing layers
getContainer(layer: Layer, oldContainerIndex?: LayerContainer): Container {
    const containerIndex = getLayerContainer(layer, Game.layer);
    
    if (this.layerTransition && oldContainerIndex !== undefined && containerIndex !== oldContainerIndex) {
        // Object is moving to a different container
        // Render in temporary container during fade
        return this.tempLayerContainer;
    } else {
        return this.layerContainers[containerIndex];
    }
}
```

The temp container is **set to the zIndex of the target container** with a small offset to ensure objects appear at the right depth during transition:

```typescript
tempContainerIndex += 0.1;  // Slight elevation to render correctly during fade
this.tempLayerContainer.zIndex = tempContainerIndex;
```

## HUD (UI Layer)

The HUD is **completely separate** from game world rendering:

```
PixiJS Rendering Stack:
│
├─ Game Container (world objects)
│  ├─ LayerContainer[0] (basement, zIndex=0)
│  ├─ LayerContainer[1] (ground, zIndex=1)
│  ├─ LayerContainer[2] (upstairs, zIndex=2)
│  └─ TempLayerContainer (transitions)
│
└─ HTML DOM (HUD)
   ├─ Killfeed
   ├─ Health bar
   ├─ Inventory bar
   ├─ Minimap
   └─ Chat
```

HTML UI elements are **always rendered on top** because they exist in the CSS z-index layer above the canvas, not within PixiJS.

## Basement/Bunker Special Handling

Buildings with basements (bunkers) require special stacking rules when player is **not in basement**:

```typescript
// When viewing basement from ground/stairs, basement should be ABOVE ground
// When viewing ground from ground, basement should be BELOW
if (i === 0 && container.zIndex !== 999) {
    container.zIndex = newLayer <= Layer.ToBasement ? 999 : 0;
}
```

**Reason**: When approaching bunker stairs, ceiling blocks view of interior. By raising basement to zIndex=999, it displays above all ground buildings for visibility.

### Ceiling Rendering

Ceiling objects render with very high zIndex (@file client/src/scripts/objects/obstacle.ts:320):

```typescript
// Building ceiling gets special treatment
.setZIndex(ZIndexes.BuildingsCeiling);  // zIndex = 17
```

Objects between player and ceiling are shown; objects inside ceiling are hidden.

## Multiple Scope/Zoom Levels

**Zoom doesn't affect Z-ordering**, only the viewport scale:

```typescript
// @file client/src/scripts/managers/cameraManager.ts:79
this.zoom = scopeDefinition.zoomLevel;  // 0.5x to 4x magnification

// Affects container scale, not zIndex values
this.container.scale.set(scale);
```

All zIndex values are **absolute**, regardless of scope level.

## Hidden Second Floor (Hunting Stand Mode)

In game modes without second floors (@file client/src/scripts/managers/cameraManager.ts:152):

```typescript
visible = newLayer >= (Game.hideSecondFloor ? Layer.ToUpstairs : Layer.ToBasement);
```

When `hideSecondFloor = true`, the upstairs container never becomes visible, preventing players from seeing interior spaces.

## Performance: Z-Index Updates

Z-index is **not recalculated every frame**. Objects have static zIndex from their definition:

```typescript
// Set once during object creation
definition.zIndex ?? ZIndexes.ObstaclesLayer1  // Default or custom zIndex

// Changes only when:
// 1. Object definition is hot-reloaded
// 2. Object visibility changes (e.g., door opens, building destroyed)
// 3. Particle/effect emitted with custom zIndex
```

PixiJS batches rendering internally by zIndex, so rendering 100+ objects per container is efficient.

## Data Lineage: Position → Container → Z-Index → Render

```
Server UpdatePacket:
  Player { position: {x: 500, y: 500}, layer: 0 }
  Obstacle { position: {x: 550, y: 490}, layer: 0, definition: "tree" }
  ↓
Client updates:
  player.position = {x: 500, y: 500}
  player.layer = Layer.Ground
  obstacle.position = {x: 550, y: 490}
  obstacle.layer = Layer.Ground
  ↓
Container assignment:
  player container = layerContainers[getLayerContainer(Layer.Ground, activeLayer)]
  obstacle container = layerContainers[getLayerContainer(Layer.Ground, activeLayer)]
  ↓
Z-Index assignment:
  player.container.zIndex = ZIndexes.Players = 13
  obstacle.container.zIndex = ZIndexes.ObstaclesLayer1 = 6
  ↓
Display order within Ground container:
  1. Obstacle at zIndex=6, y=490 (renders first, furthest back)
  2. Player at zIndex=13, y=500 (renders second, in front)
  ↓
PixiJS Display List (final render):
  [Basement container, Ground container, Upstairs container]
  → Ground container children sorted by zIndex and display order
  → Result: obstacle behind player visually ✓
```

## Known Issues & Gotchas

### 1. Stair Layer Visibility Bugs

**Issue:** Transitioning between `ToBasement` and `ToUpstairs` with `activeLayer` mismatches can cause objects to disappear mid-transition.

**Cause:** `getLayerContainer()` depends on both object layer AND active layer. If active layer doesn't match object layer, mapping fails.

**Evidence:** @file common/src/utils/layer.ts:159 — transition layers use conditional logic.

### 2. Y-Sorting Doesn't Account for Object Size

**Issue:** A tall tree at y=100 may render behind a small rock at y=101 if sorted only by center position.

**Fix needed:** Sort by **bottom edge** (position.y + hitbox.radius) instead of center position.

### 3. Particle Z-Index Relative to Container

**Issue:** `PLAYER_PARTICLE_ZINDEX = ZIndexes.Players + 0.5` is only relative within a single container. If player moves layers, particle calculation may break.

**Evidence:** @file client/src/scripts/utils/constants.ts:13

### 4. buildingVisionSize Changes Not Reflected in Z-Order

**Issue:** When player enters a building, ceiling visibility changes but Z-index doesn't adapt. Building interior objects may incorrectly layer.

**Workaround:** Ceilings use `ZIndexes.BuildingsCeiling`, but this is static.

### 5. Temporary Container Z-Index Offset Fragile

**Issue:** The `tempContainerIndex += 0.1` offset is arbitrary. If multiple objects transition simultaneously, Z-index order can invert.

**Evidence:** @file client/src/scripts/managers/cameraManager.ts:183

## Related Documents

### Tier 2
- [Client Rendering](../README.md) — full rendering system
- [Camera Management](../../camera-management/) — viewport & zoom management
- [Game Objects — Client](../../game-objects-client/) — object rendering lifecycle

### Tier 3
- [camera-management/modules/viewport.md](../../camera-management/modules/viewport.md) — viewport positioning relative to layers
- [game-objects-client/modules/sprite-pooling.md](../../game-objects-client/modules/sprite-pooling.md) — object creation and container assignment

### Tier 1
- [Architecture](../../../architecture.md) — system design overview
- [Data Model](../../../datamodel.md) — Layer enum and object schema

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [common/src/constants.ts](../../../../common/src/constants.ts) | Layer enum & ZIndexes enum definitions | Low |
| [common/src/utils/layer.ts](../../../../common/src/utils/layer.ts) | Layer utility functions & LayerContainer mapping | Medium |
| [client/src/scripts/managers/cameraManager.ts](../../../../client/src/scripts/managers/cameraManager.ts) | Layer container management & transitions | High |
| [client/src/scripts/game.ts](../../../../client/src/scripts/game.ts) | Game class layer transitions & initial setup | High |
| [client/src/scripts/utils/constants.ts](../../../../client/src/scripts/utils/constants.ts) | Layer transition timing & Z-index offsets | Low |
| [client/src/scripts/objects/obstacle.ts](../../../../client/src/scripts/objects/obstacle.ts) | Obstacle Z-index assignment examples | Medium |
