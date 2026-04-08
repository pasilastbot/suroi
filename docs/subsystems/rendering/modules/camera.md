# Rendering — Camera Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/rendering/README.md -->
<!-- @source: client/src/scripts/managers/cameraManager.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the camera/viewport system: layer containers (basement, ground, upstairs), zoom, pan, layer transitions, shake, and shockwave effects.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/managers/cameraManager.ts | `CameraManager` — viewport, zoom, layers | High |

## Architecture

```
CameraManager.container
    ├── layerContainers[0] — basement
    ├── layerContainers[1] — ground
    ├── layerContainers[2] — upstairs
    └── tempLayerContainer — prevents flicker during layer transitions
```

## Layer Transitions

- Objects on stairs move between basement/ground/upstairs
- `tempLayerContainer` — holds object during fade-in to avoid flicker
- `LAYER_TRANSITION_DELAY` — fade duration

## Zoom

- `zoom` — from scope definition (default `DEFAULT_SCOPE.zoomLevel`)
- `resize(animation)` — recalculates scale from screen size
- `zoomTween` — smooth zoom transitions

## Effects

- **Shake** — `shakeStart`, `shakeDuration`, `shakeIntensity`
- **Shockwave** — `Shockwaves` set, `ShockwaveFilter` from pixi-filters

## World → Screen

- `toPixiCoords(position)` — world position to screen (used by objects)
- Scale: `PIXI_SCALE` * zoom

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Rendering overview
- **Tier 2:** [../input/](../../input/) — Aim uses camera-relative direction
- **Tier 1:** [docs/datamodel.md](../../../datamodel.md) — Layer enum
