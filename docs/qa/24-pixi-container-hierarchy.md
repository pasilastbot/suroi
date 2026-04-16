# Q24: What is the PixiJS container hierarchy, and how do the 3 layer containers map to the 5 logical game layers?

**Answer:**

The PixiJS container hierarchy in Suroi uses a **3-level rendering structure** that maps the **5 logical game layers** to **3 physical rendering containers**.

---

## PixiJS Container Hierarchy

```
PIXI.stage
├── gameContainer { isRenderGroup: true }
│   ├── CameraManager.container { scaled & translated }
│   │   ├── layerContainers[0] (zIndex=0 or 999)  ← Basement
│   │   ├── layerContainers[1] (zIndex=1)         ← Ground  
│   │   ├── layerContainers[2] (zIndex=2)         ← Upstairs
│   │   ├── tempLayerContainer                    ← Transition buffer
│   │   └── gasRender.graphics (zIndex=1000)
│   └── DebugRenderer.graphics (DEBUG_CLIENT only)
└── uiContainer { isRenderGroup: true }
    ├── MapManager.container (minimap/full map)
    ├── MapManager.mask
    ├── netGraph containers (profiling graphs)
    ├── EmoteWheelManager.container
    └── MapPingWheelManager.container
```

**Key structure attributes:**
- `gameContainer` and `uiContainer` are **render groups** — separate rendering pipelines
- `CameraManager.container` handles zoom scaling + viewport translation (camera pan)
- Three `layerContainers` hold all world objects

---

## 5 Logical Game Layers → 3 Rendering Containers Mapping

**5 logical layers:**

```typescript
enum Layer {
    Basement = -2,      // Underground rooms/bunkers
    ToBasement = -1,    // Staircase going down (transition)
    Ground = 0,         // Main outdoor level
    ToUpstairs = 1,     // Staircase going up (transition)
    Upstairs = 2        // Second floor/roofs
}
```

**Mapping function:**

```typescript
export enum LayerContainer { Basement = 0, Ground = 1, Upstairs = 2 }

export function getLayerContainer(objectLayer: Layer, activeLayer: Layer): LayerContainer {
    switch (objectLayer) {
        case Layer.Basement:
            return LayerContainer.Basement;    // Always render in container[0]
        
        case Layer.ToBasement:
            // Show in same container as player approaches stairs
            return activeLayer <= Layer.ToBasement ? LayerContainer.Basement : LayerContainer.Ground;
        
        case Layer.Ground:
            return LayerContainer.Ground;      // Always render in container[1]
        
        case Layer.ToUpstairs:
            // Show in same container as player climbs stairs
            return activeLayer >= Layer.ToUpstairs ? LayerContainer.Upstairs : LayerContainer.Ground;
        
        case Layer.Upstairs:
            return LayerContainer.Upstairs;    // Always render in container[2]
    }
}
```

**Logic summary:**
- **Basement (Layer -2)** → Always `layerContainers[0]`
- **Stairs down (Layer -1)** → `layerContainers[0]` when player ≤ -1, else `layerContainers[1]`
- **Ground (Layer 0)** → Always `layerContainers[1]`
- **Stairs up (Layer 1)** → `layerContainers[2]` when player ≥ 1, else `layerContainers[1]`
- **Upstairs (Layer 2)** → Always `layerContainers[2]`

---

## Z-Index Ordering Within Each Layer Container

Each layer container contains multiple Z-indexed object types, sorted back-to-front:

| ZIndex | Value | What Renders | ZIndex | Value | What Renders |
|--------|-------|------|--------|-------|-------|
| `Ground` | 0 | Terrain floor | `Bullets` | 11 | Projectile tracers |
| `BuildingsFloor` | 1 | Building floor graphics | `DownedPlayers` | 12 | Knocked-out players |
| `Decals` | 2 | Blood/burn marks | **`Players`** | **13** | **Standing players** |
| `DeadObstacles` | 3 | Destroyed remnants | `ObstaclesLayer3` | 14 | Bushes/tables (above players) |
| `DeathMarkers` | 4 | Death location marks | `AirborneThrowables` | 15 | Flying grenades |
| `Explosions` | 5 | Explosion visuals | `ObstaclesLayer4` | 16 | Tall trees |
| `ObstaclesLayer1` | 6 | Crates/walls (default) | `ObstaclesLayer5` | 18 | Ceiling obstacles |
| `Loot` | 7 | Ground item pickups | `Emotes` | 19 | Chat bubbles |
| `GroundedThrowables` | 8 | Grenades on ground | `Gas` | 20 | Gas overlay |
| `ObstaclesLayer2` | 9 | Mid-height obstacles | | | |
| `TeammateName` | 10 | Name labels | | | |

**Special Z-indices:**
- `PLAYER_PARTICLE_ZINDEX = ZIndexes.Players + 0.5` = bleeding effects above players
- `Gas` (in world) = `zIndex=1000` (added via `CameraManager.addObject()`)
- `Plane` (airdrop) = `Number.MAX_SAFE_INTEGER - 2`

---

## References

**Documentation:**
- **Tier 2:** `docs/subsystems/client-rendering/README.md` — Container hierarchy diagram & architecture
- **Tier 3:** `docs/subsystems/client-rendering/modules/layer-ordering.md` — Complete layer system design & gotchas

**Source Code:**
- `client/src/scripts/game.ts` (lines 748–761) — Container initialization
- `client/src/scripts/managers/cameraManager.ts` (lines 19–30) — Layer container setup
- `common/src/constants.ts` (lines 146–152) — Layer enum & ZIndex enum
- `common/src/utils/layer.ts` (lines 154–168) — getLayerContainer mapping function
