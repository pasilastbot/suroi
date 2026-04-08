# Rendering — Spritesheets Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/rendering/README.md -->
<!-- @source: client/src/scripts/utils/pixi.ts, vite/plugins/image-spritesheet-plugin -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents spritesheet loading and `SuroiSprite`: mode-specific atlases, `loadSpritesheets`, texture lookup, and the Vite plugin that generates spritesheet configs from SVG sources.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/utils/pixi.ts | `loadSpritesheets`, `SuroiSprite`, `spritesheetLoadPromise` | Medium |
| `vite/plugins/image-spritesheet-plugin` | Generates spritesheet JSON from `client/public/img/` | High |

## loadSpritesheets

- **Args:** `modeName`, `renderer`, `highResolution`
- **Source:** `virtual:image-spritesheets-importer-high-res` or `-low-res`
- **Flow:** Import spritesheet config → load images → parse with PixiJS Spritesheet → cache in Assets
- **Fallback:** If `MAX_TEXTURE_SIZE` < 4096, force low-res (2048 atlases)

## Spritesheet Sources

- **Path:** `client/public/img/game/<variant>/` — shared, normal, winter, etc.
- **Mode:** From `Modes[modeName].spriteSheets` (e.g. ["shared", "normal", "winter"])
- **Virtual import:** Vite plugin scans SVG dirs, generates JSON meta

## SuroiSprite

- Extends PixiJS `Sprite`
- **anchor.set(0.5)** — Center pivot
- **getTexture(frame)** — Lookup by frame name; fallback `_missing_texture`
- **Constructor(frame?)** — Create sprite from frame string

## spritesheetLoadPromise

- Resolves when all spritesheets loaded
- Used to block game start until assets ready

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Rendering overview
- **Tier 2:** [../definitions/](../../definitions/) — Modes.spriteSheets
- **Tier 1:** [docs/architecture.md](../../../architecture.md) — Client entry, asset loading
