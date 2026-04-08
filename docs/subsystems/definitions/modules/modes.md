# Definitions — Modes Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/modes.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents game mode definitions: `ModeName`, `ModeDefinition`, colors, spritesheets, and mode-specific features (particle effects, weapon swap, bunkers, etc.).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/definitions/modes.ts | `Modes`, `ModeDefinition`, `ModeName` | Medium |

## ModeName

`normal`, `fall`, `halloween`, `infection`, `hunted`, `birthday`, `winter`, `nye`

## ModeDefinition Fields

| Field | Purpose |
|-------|---------|
| `similarTo?` | Mode inheritance |
| `colors` | Terrain colors (grass, water, border, beach, gas, void) — HSL/HSLA |
| `spriteSheets` | Spritesheet load order (shared, normal, winter, etc.) |
| `ambience?` | Ambient sound |
| `particleEffects?` | Falling leaves, snowflakes etc. |
| `obstacleVariants?` | Mode-specific obstacle variants (e.g. `barrel_winter`) |
| `unlockStage?` | H.U.N.T.E.D. bunker doors unlock at stage |
| `weaponSwap?` | Enable weapon swap (infection) |
| `replaceWaterBy?` | Replace water with ice (winter) |
| `canvasFilters?` | Brightness, saturation (infection, halloween) |
| `playButtonImage?` | Custom play button image |

## Spritesheet Names

`shared` + mode name (e.g. `normal`, `winter`, `halloween`). Loaded from `client/public/img/game/<variant>/`.

## Business Rules

- `similarTo` allows mode inheritance (e.g. nye inherits from winter)
- Each mode has its own loot tables, gas stages per `LootTables[mode]`, `GasStages[mode]`

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 2:** [../map/](../../map/) — Map generation uses mode
- **Tier 2:** [../rendering/](../../rendering/) — Spritesheet loading
