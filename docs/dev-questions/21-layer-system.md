# Q: How does the layer system work (basement, ground, upstairs) and how do I use it for a new building?

<!-- @tags: layers, buildings, rendering, camera -->
<!-- @related: docs/datamodel.md, docs/subsystems/rendering/modules/camera.md -->

## Layers

The world has 5 layer values (from `common/src/constants.ts`):

| Layer | Value | Purpose |
|-------|-------|---------|
| `Basement` | −2 | Underground level |
| `ToBasement` | −1 | Staircase / transition zone to basement |
| `Ground` | 0 | Default street level (most objects) |
| `ToUpstairs` | +1 | Staircase / transition zone to upper floor |
| `Upstairs` | +2 | Upper floor of a building |

`ToBasement` and `ToUpstairs` are **transition layers** — players on them can
interact with objects on both the adjacent layers.

## Collision Modes

Each object specifies which layers it physically interacts with:

| Mode | Behavior |
|------|----------|
| `Layers.All` | Collides with objects on every layer |
| `Layers.Adjacent` | Collides with same layer ± 1 |
| `Layers.Equal` | Collides only with same layer |

Bullets, players, and obstacles each declare their collision layer. Two objects
only collide if their collision modes overlap.

## Rendering on Client

The `CameraManager` has three `layerContainers`:

```
layerContainers[0] — basement
layerContainers[1] — ground (default)
layerContainers[2] — upstairs
```

Objects are placed into the container matching their layer. When a player walks
up/down stairs (enters a `ToUpstairs`/`ToBasement` zone), they are smoothly
faded between containers via `tempLayerContainer` to prevent flicker.

## Z-Index Within a Layer

Within a layer container, render order is controlled by `ZIndexes` (from
`common/src/constants.ts`). Ground-level objects are ordered:
`BuildingsFloor → Decals → DeadObstacles → Loot → Players → ObstaclesLayer3 → BuildingsCeiling`

## Using Layers in a Building Definition

When defining a multi-floor building in `common/src/definitions/buildings.ts`,
assign each component its layer:

```typescript
{
    idString: "my_building",
    // Ground floor floor/walls
    floorImages: [{ key: "my_building_floor", position: Vec.create(0, 0) }],
    ceilingImages: [{ key: "my_building_ceiling", position: Vec.create(0, 0) }],
    obstacles: [
        {
            idString: "door",
            position: Vec.create(0, 10),
            layer: Layer.Ground,  // ground floor door
        }
    ],
    // Basement
    subBuildings: [
        {
            idString: "my_basement",
            position: Vec.create(0, -30),
            layer: Layer.Basement,
        }
    ],
    // Staircase
    staircase: {
        position: Vec.create(0, 5),
        orientation: 0,
    }
}
```

The staircase object transitions players between `Ground` and `Upstairs` (or
`Basement`). Players in the staircase zone get `ToUpstairs` or `ToBasement`
layer, making them visible and collidable with both floors.

## Related

- [Game Data Model](../datamodel.md) § Layer System — Layer enum, collision modes
- [Camera Module](../subsystems/rendering/modules/camera.md) — layerContainers, layer transitions
- [Buildings](../subsystems/definitions/modules/buildings.md) — building definition structure
