# Rendering Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: client/src/scripts/objects/, client/src/scripts/managers/, client/src/scripts/utils/pixi.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Rendering subsystem draws the game world using PixiJS v8. It receives `UpdatePacket` data from the server, updates client-side game objects, interpolates position/rotation for smooth visuals, and renders via the PixiJS scene graph.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/game.ts` | `Game` — PixiJS Application, WebSocket, packet dispatch, object lifecycle |
| `client/src/scripts/objects/gameObject.ts` | `GameObject` base — Container, interpolation, layer |
| `client/src/scripts/objects/player.ts` | Player rendering, weapon, animations |
| `client/src/scripts/objects/obstacle.ts` | Obstacle sprites, destruction |
| `client/src/scripts/objects/loot.ts` | Loot sprites |
| `client/src/scripts/objects/building.ts` | Building parts, layers |
| `client/src/scripts/objects/bullet.ts` | Bullet trails |
| `client/src/scripts/objects/explosion.ts` | Explosion effects |
| `client/src/scripts/managers/cameraManager.ts` | Viewport, zoom, pan, layer transitions, shake |
| `client/src/scripts/managers/particleManager.ts` | Particle effects |
| `client/src/scripts/managers/gasManager.ts` | Gas zone rendering |
| `client/src/scripts/managers/mapManager.ts` | Minimap |
| `client/src/scripts/utils/pixi.ts` | `loadSpritesheets`, `SuroiSprite` |
| `client/src/scripts/utils/constants.ts` | `PIXI_SCALE`, layer constants |

## Architecture

```
PixiJS Application (Game.pixi)
    └── CameraManager.container
            └── layerContainers[3] — basement, ground, upstairs
                    └── Game objects (Player, Obstacle, Loot, etc.)
                            └── SuroiSprite, Container
```

Each `GameObject` extends PixiJS `Container`. Objects are added to layer containers by `Layer` (Basement, Ground, Upstairs). Z-index within a layer is from `ZIndexes`.

## Data Flow

```
UpdatePacket (from WebSocket)
    → Game.onMessage() → parse packets
    → For each object: create/update/destroy
    → ObjectClassMapping[category] — instantiate or update
    → updateFromData() / updateFull()
    → Mark position/rotation dirty

PixiJS ticker
    → updateContainerPosition() / updateContainerRotation() — lerp
    → toPixiCoords() — world → screen
    → Render
```

## Managers

| Manager | Responsibility |
|---------|----------------|
| `CameraManager` | Viewport position, zoom, layer transitions, shake, shockwave |
| `ParticleManager` | Particle effects (muzzle flash, impacts, etc.) |
| `GasManager` | Gas zone circle rendering |
| `MapManager` | Minimap canvas, player markers |
| `SoundManager` | Audio via `@pixi/sound` |
| `EmoteWheelManager` | Emote selection UI |
| `MapPingWheelManager` | Map ping selection |
| `PerkManager` | Active perk display |
| `ScreenRecordManager` | Screen recording |

## Spritesheets

- **Loading:** `loadSpritesheets(modeName, renderer, highResolution)` — loads mode-specific atlases
- **Source:** `client/public/img/game/<variant>/` — organized by map variant (normal, winter, etc.)
- **Virtual import:** Vite plugin `image-spritesheet-plugin` generates spritesheet configs
- **SuroiSprite:** Extends `Sprite`, uses `anchor.set(0.5)`, `getTexture(frame)` for lookup

## Interpolation

- `Game.serverDt` — 25ms (40 TPS)
- Position/rotation changes trigger `updateContainerPosition()` / `updateContainerRotation()`
- Lerp factor: `min((Date.now() - lastChange) / serverDt, 1)` for smooth catch-up

## Module Index (Tier 3)

- [Object Lifecycle](modules/object-lifecycle.md) — Create/update/destroy from UpdatePacket
- [Camera](modules/camera.md) — Viewport, zoom, layer containers, shake, shockwave
- [Spritesheets](modules/spritesheets.md) — loadSpritesheets, SuroiSprite, mode atlases

## Protocol Considerations

- **Affects protocol:** No. Rendering consumes UpdatePacket format; changes are server-side.

## Dependencies

- **Depends on:** Packets (UpdatePacket structure), Objects (ObjectsNetData), Definitions
- **Depended on by:** Game (main loop), Input (camera-relative aim)

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — Client architecture
- **Tier 2:** [../objects/](../objects/) — Object model, client objects
- **Tier 2:** [../packets/](../packets/) — UpdatePacket structure
