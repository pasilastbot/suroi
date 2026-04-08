# Q: How do I add a new game mode?

<!-- @tags: modes, definitions, map, spritesheet -->
<!-- @related: docs/subsystems/definitions/modules/modes.md -->

## Overview

A game mode customizes: terrain colors, spritesheets, ambient sound/particles,
obstacle variants, loot tables, gas stages, and special mechanics. Modes are
defined in `common/src/definitions/modes.ts` and referenced by map configs.

## 1. Add the mode definition

Open `common/src/definitions/modes.ts` and add to the `Modes` object:

```typescript
export const Modes: Record<ModeName, ModeDefinition> = {
    // ...existing modes...
    my_mode: {
        // Optional: inherit from another mode
        // similarTo: "normal",

        colors: {
            grass: { hue: 120, saturation: 50, lightness: 35 },
            water: { hue: 200, saturation: 60, lightness: 40, alpha: 0.75 },
            border: { hue: 200, saturation: 60, lightness: 30 },
            beach: { hue: 45, saturation: 40, lightness: 65 },
            void: "#1a1a1a",
            gas: "rgba(100, 0, 150, 0.55)",
        },

        // Spritesheets to load (in order)
        spriteSheets: ["shared", "normal", "my_mode"],

        // Ambient sound
        // ambience: "forest_ambience",

        // Particle effects (falling leaves, snow, etc.)
        // particleEffects: [
        //     { frame: "leaf_particle", alpha: { start: 0.8, end: 0 }, scale: { start: 0.3, end: 0.6 }, ... }
        // ],
    },
};
```

## 2. Add the ModeName type

Add `"my_mode"` to the `ModeName` union type in `modes.ts`:

```typescript
export type ModeName =
    | "normal"
    | "fall"
    | "halloween"
    // ...
    | "my_mode";  // ← add here
```

## 3. Add a spritesheet variant (optional)

Create the mode's spritesheet directory:

```
client/public/img/game/my_mode/
```

Add SVG files here that **override** the shared sprites for this mode.
For example, `barrel.svg` in `my_mode/` replaces the shared barrel sprite
in `my_mode` games.

## 4. Add loot tables for the mode

In `server/src/data/lootTables.ts`, add a mode-specific entry:

```typescript
LootTables["my_mode"] = {
    ...LootTables["normal"],   // inherit normal tables
    // Override specific tables:
    ground_loot: [
        { item: "ak47", weight: 2 },
        { item: "m16a4", weight: 1 },
    ],
};
```

## 5. Add gas stages for the mode

In `server/src/data/gasStages.ts`, add mode-specific stages:

```typescript
GasStages["my_mode"] = [
    ...GasStages["normal"],  // inherit normal stages
    // Or define completely custom stages
];
```

## 6. Add a map config for the mode

In `server/src/data/maps.ts`, add a map that references your mode:

```typescript
const maps = {
    // ...
    my_mode_map: {
        mode: "my_mode",      // ← links mode to map
        width: 1632,
        height: 1632,
        // ...rest of map config
    },
};
```

## 7. Configure the server to use your map

```json
// server/config.json
{
    "map": "my_mode_map"
}
```

## 8. Test

```bash
bun dev
```

## Special Mechanics

Use these `ModeDefinition` fields for advanced behavior:

| Field | Purpose |
|-------|---------|
| `similarTo` | Inherit all settings from another mode |
| `obstacleVariants` | `{ "barrel": "barrel_winter" }` — swap obstacles by idString |
| `replaceWaterBy` | Replace water floor with another type (e.g. ice) |
| `canvasFilters` | CSS-style brightness/saturation filters |
| `weaponSwap` | Enable weapon swap mechanic (infection mode) |
| `unlockStage` | Gas stage at which bunker doors unlock (hunted mode) |
| `playButtonImage` | Custom play button image |

## Related

- [Modes Module](../subsystems/definitions/modules/modes.md) — full ModeDefinition field reference
- [Map Generation](../subsystems/map/modules/generation.md) — how maps reference modes
- [Loot Tables](11-add-loot-table.md) — per-mode loot configuration
- [Gas Stages](../subsystems/gas/modules/stages.md) — per-mode gas stage configuration
- [Spritesheets](../subsystems/rendering/modules/spritesheets.md) — how mode spritesheets are loaded
