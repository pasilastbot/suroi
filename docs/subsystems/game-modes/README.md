# Game Modes System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/game-modes/modules/ -->
<!-- @source: common/src/definitions/modes.ts, server/src/gameManager.ts, server/src/game.ts -->

## Purpose

Defines and manages game mode theming, visual styling, and mode metadata across the game. Modes control environment colors, particle effects, ambient audio, menu music, scope defaults, special mechanics (e.g., H.U.N.T.E.D. bunker unlock stages), and map assignment. Modes are distinct from **team modes** (Solo/Duo/Squad), which control player grouping; see [Team System](../team-system/) for team assignment logic.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/definitions/modes.ts` | `ModeDefinition` interface; `Modes` registry (normal, fall, halloween, infection, hunted, birthday, winter, nye) |
| `server/src/gameManager.ts` | GameManager initializes mode via Switcher; mode switching via cron schedule |
| `server/src/game.ts` | Game accepts `modeName`; configures map, spritesheet, particles via `this.mode` |
| `server/src/utils/misc.ts` | `modeFromMap()` — extracts mode from map string (`"map_name:mode_name"`) |
| `server/src/data/maps.ts` | MapDefinition.mode — assigns mode to map; mode-specific loot tables |
| `server/src/data/lootTables.ts` | Per-mode loot table definitions |
| `common/src/constants.ts` | `GameConstants.defaultMode = "normal"` |

## Game Mode Registry

### Core Modes

| Mode | Type | Visual Style | Ambient Audio | Default Scope | Special Features |
|------|------|--------------|---------------|----|---|
| **normal** | Default | Grassland (green 95°, 41%, 38%) | wind_ambience | — | Standard gameplay; no particle effects |
| **fall** | Seasonal | Autumn (yellow-brown) | wind_ambience | 2x scope | Falling leaves particles; themed menu music |
| **halloween** | Seasonal | Dark (grass 65° hue, 100% saturation) | graveyard_ambience | 2x scope | Dark canvas filter (brightness 0.6); Halloween particles; gold airdrop at stage 5 |
| **birthday** | Event | Grassland (identical to normal) | wind_ambience | — | Birthday-themed sprites overlaid; optional cake items (disabled in code) |
| **winter** | Seasonal | Snowy (light blue 210°) | snowstorm_ambience | — | Snowflake particles; **no river ambience**; replaces water with ice; obstacle variants (e.g., `barrel_winter`) |
| **infection** | Game Mode | Purple-tinted (grass 300° hue) | wind_ambience | — | **Weapon swap enabled** (unique mechanic); canvas filter (brightness/saturation 0.8); infection-mode spritesheet |
| **hunted** | Game Mode | Dark green (grass 140° hue) | wind_ambience | — | **Bunker unlock at stage 3**; gold airdrop at stage 5; **equipment level cap increased to 4** (highest in game); special spritesheet |
| **nye** | Seasonal | Snowy (identical to winter) | snowstorm_ambience | — | Winter sprite base + special NYE overlay; bullet glow filters enabled; confetti particles; canvas filter (brightness/saturation 0.85); gold airdrop + enhanced particles at stage 30s |

---

## Mode Configuration (ModeDefinition)

Each mode is defined as a `ModeDefinition` object in `common/src/definitions/modes.ts`:

```typescript
export interface ModeDefinition {
    // Terrain colors (HSL/HSLA format)
    readonly colors: {
        grass: string       // Ground terrain base color
        water: string       // Water/river color
        border: string      // Map boundary color
        beach: string       // Sand/beach zones
        riverBank: string   // River edge color
        trail: string       // Path/trail overlay
        gas: string         // Safe zone edge glow
        void: string        // Out-of-bounds void
    }
    
    // Rendering & assets
    readonly spriteSheets: readonly SpritesheetNames[]  // [..base, mode-specific]
    readonly ambience?: string                          // Background audio file (.mp3)
    readonly ambienceVolume?: number                    // Relative volume multiplier
    readonly particleEffects?: {                        // Falling/floating particles
        frames: string | string[]                       // Sprite frame names
        delay: number                                   // ms between spawns
        gravity?: boolean                               // Downward-only movement
    }
    
    // UI & menu
    readonly specialLogo?: boolean                      // Custom splash screen logo
    readonly playButtonImage?: string                   // Custom play button image
    readonly replaceMenuMusic?: boolean                 // Use `menu_music_[MODE].mp3`
    readonly defaultScope?: ReferenceTo<ScopeDefinition>  // Scope override (e.g., 2x)
    
    // Terrain & object variants
    readonly replaceWaterBy?: FloorNames                // Replace water terrain (e.g., FloorNames.Ice)
    readonly defaultGroundFloor?: FloorNames            // Default ground type
    readonly obstacleVariants?: boolean                 // Load `barrel_winter`, `tree_halloween`, etc.
    
    // Gameplay mechanics
    readonly unlockStage?: number                       // H.U.N.T.E.D.: unlock bunkers at gas stage N
    readonly forcedGoldAirdropStage?: number            // Force premium airdrop at stage N
    readonly summonAirdropsInterval?: number            // Airdrop schedule (ms), overrides gas stages
    readonly maxEquipmentLevel?: number                 // Equipment level cap (default 3; hunted=4)
    readonly weaponSwap?: boolean                       // Enable weapon swap mechanic (infection only)
    readonly overrideUpstairsFunctionality?: boolean    // H.U.N.T.E.D. bunker mechanics
    
    // Canvas effects
    readonly canvasFilters?: {                          // Global brightness/saturation
        brightness: number  // 0.0 = black, 1.0 = normal
        saturation: number  // 0.0 = grayscale, 1.0 = normal
    }
    readonly bulletTrailAdjust?: string                 // Bullet trail color overlay (HSL)
    readonly bulletFilters?: boolean                    // Enable bullet glow (NYE, infection)
    
    // Audio & rendering tweaks
    readonly noRiverAmbience?: boolean                  // Disable water ambience (winter)
    readonly enhancedAirdropParticles?: boolean         // Extra particle effects on airdrops
    
    // Mode inheritance
    readonly similarTo?: ModeName                       // Inherit styling from base mode (e.g., nye→winter)
}
```

### Mode Inheritance

Some modes **inherit** styling from another mode via `similarTo`. This allows sharing sprite sheets without duplication:

```
nye → similarTo: "winter"
  // Uses winter's spritesheet, water replacement, particle effects
  // But overrides colors, audio, and particle models with NYE-specific assets
```

---

## Mode-to-Map Assignment

### Map-Mode Binding

Maps are assigned to modes via one of three methods (checked in order by `modeFromMap()`):

**1. Map string syntax:**
```typescript
// Input map string: "office:halloween"
// Extracted mode: "halloween"
// Syntax: "mapName:modeName"
```

**2. MapDefinition.mode field:**
```typescript
// File: server/src/data/maps.ts
const Maps: Record<MapName, MapDefinition> = {
    office: { mode: "normal", /* ... */ },
    hangar: { mode: "infection", /* ... */ },  // Hangar only spawns in infection
    // no mode specified → falls through
}
```

**3. Default fallback:**
```typescript
GameConstants.defaultMode = "normal"
// Used if map name not found in MapDefinition and not in Modes registry
```

### Loot Tables

Mode-specific loot spawning is controlled in `server/src/data/lootTables.ts`:

```typescript
// Each mode can override loot frequency
const LootTables: Record<ModeName, LootTable> = {
    normal: { /* standard loot */ },
    halloween: { /* more healing items, pumpkins */ },
    infection: { /* infected-player weapons */ },
    // ...
}
```

Maps reference mode-specific tables via `MapDefinition.loots`:

```typescript
const Maps: Record<MapName, MapDefinition> = {
    office: {
        loots: { // Mode name → spawn weight
            normal: 50,
            halloween: 60,  // +20% more loot in halloween
            infection: 45
        }
    }
}
```

---

## Mode Switching

### GameManager Mode Scheduling

Modes are switched via a cron schedule at the global level:

```typescript
// server/src/gameManager.ts
export class GameManager {
    mode: ModeName              // Current mode
    nextMode?: ModeName         // Upcoming mode (for client preload)
    
    // Mode switching is tied to map switching
    this.map = new Switcher("map", Config.map, (nextMap) => {
        this.mode = modeFromMap(nextMap)
        this.nextMode = modeFromMap(this.map.next)
    })
}
```

### Configuration

Mode (and map) switching is configured in `server/config.json`:

```jsonc
{
    "map": "office",
    
    // OR (with rotation):
    "map": {
        "rotation": ["office", "office:halloween", "compound"],
        "cron": "0 */30 * * * *"  // Every 30 minutes
    }
}
```

### Effect on Running Games

- Mode switches apply **globally** to new games only
- Existing games continue with their assigned mode
- Client preloads `nextMode` sprites to reduce load-time jank

---

## Key Subsystem Interactions

### Depends on:
- **[Server Data Systems](../server-data/)** — MapDefinition, LootTables, mode definitions
- **[Object Definitions](../object-definitions/)** — Scope definitions, obstacle variants

### Depended on by:
- **[Game Loop](../game-loop/)** — Mode config affects tick behavior (e.g., infection spread, bunker unlocks)
- **[Client Rendering](../client-rendering/)** — Spritesheet and particle loading per mode
- **[Map](../map/)** — Map instantiation uses `game.modeName` to select terrain colors and obstacle variants
- **[Team System](../team-system/)** — Orthogonal; modes and team modes are independent
- **[Networking](../networking/)** — Mode broadcasted to clients in `JoinedPacket`

---

## Architectural Patterns

### 1. Mode Definition as Configuration

Modes are **data-driven**: all mode properties are static `readonly` fields in the `Modes` registry. No mode-specific branching in game logic.

```typescript
// ✅ Good: Mode styling via definition
this.ambience = Sounds[this.game.mode.ambience]

// ❌ Avoid: Mode branching in code
if (this.modeName === "infection") { /* special logic */ }
```

### 2. Spritesheet Inheritance

Spritesheets are **stacked** (loaded in array order), with later sheets overriding earlier ones:

```typescript
halloween: {
    spriteSheets: ["shared", "fall", "halloween"]
    // Loads: base → fall theme → halloween-specific overlays
}
```

### 3. Terrain Color Customization

Each mode defines **8 terrain colors** (grass, water, beach, etc.) independently. No color inheritance; each mode specifies all 8 values.

---

## Special Mode Mechanics

### Infection Mode
- Enables **weapon swap** (`Mode.weaponSwap = true`)
- Triggers **canvas filter** (brightness 0.8, saturation 0.8)
- Uses `infection` spritesheet overlay
- No unlock stages or special airdrop

### H.U.N.T.E.D. Mode
- **Bunker unlock** at gas stage 3: NPCs and bunker doors become accessible
- **Gold airdrop** at stage 5 (premium loot)
- **Equipment cap increased to 4** (highest in game vs. normal cap of 3)
- **Upstairs functionality overridden** (`overrideUpstairsFunctionality = true`)
- Used by `server/src/game.ts` when handling building entry/exit

### Seasonal Modes (Fall, Winter, NYE)
- **Obstacle variants** enabled: code appends mode name to obstacle IDs
  - Example: `barrel` → `barrel_winter`, `tree` → `tree_halloween`
- **Water replacement**: Winter/NYE replace water terrain with `FloorNames.Ice`
- **River ambience disabled**: No water sounds in winter modes

---

## Known Issues & Gotchas

### 1. Mode-Specific Loot Tables Not Fully Utilized
Some maps do not define `MapDefinition.loots`, causing all modes to use the same loot spawning. Halloween and infection loot tables exist but may be underutilized.

**File:** `server/src/data/lootTables.ts`

### 2. Infection Mode Weapon Swap Unique
Weapon swap (`Mode.weaponSwap = true`) is only enabled for **infection mode**. No other mode uses this feature, suggesting incomplete implementation or future expansion.

**File:** `common/src/definitions/modes.ts:infection` (line ~165)

### 3. H.U.N.T.E.D. Bunker Mechanics Fragile
The `overrideUpstairsFunctionality` flag is a hardcoded boolean with minimal documentation. Bunker spawning and unlock logic is tightly coupled to gas stage progression.

**Files:** `server/src/game.ts` (bunker unlock check), `server/src/map.ts` (obstacle placement)

### 4. Spritesheet Array Order Dependency
The order of spritesheets in `ModeDefinition.spriteSheets` array is significant — later entries override earlier ones. This is implicit and error-prone if sheets are reordered.

**File:** `common/src/definitions/modes.ts` (all mode definitions)

### 5. Canvas Filters Applied Globally
Canvas brightness/saturation filters are rendered to the entire game canvas, affecting visibility and visual clarity. Halloween and NYE both use darkening filters which can impair gameplay readability.

**File:** `client/src/scripts/game.ts` (canvas rendering)

### 6. Airdrop Interval Override Not Synchronized with Gas Stages
`summonAirdropsInterval` (NYE: 30 seconds) bypasses gas stage-based airdrop scheduling, potentially creating unexpected airdrop density during transitions.

**File:** `server/src/game.ts` (airdrop spawning logic)

---

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — System design and tech stack
- [Data Model](../../datamodel.md) — Mode enum, team structures

### Tier 2
- [Server Data Systems](../server-data/) — Map definitions, loot tables
- [Object Definitions](../object-definitions/) — Scope definitions, obstacle variants
- [Map Generation](../map/) — Terrain colors, obstacle placement by mode
- [Team System](../team-system/) — Player grouping (orthogonal to modes)
- [Game Loop](../game-loop/) — Mode-specific tick behavior
- [Client Rendering](../client-rendering/) — Spritesheet and particle loading

### Tier 3
- [Mode-Specific Mechanics](modules/mode-mechanics.md) — H.U.N.T.E.D., Infection, Seasonal rules
- [Spritesheet System](modules/spritesheets.md) — Spritesheet loading and inheritance

---

## Patterns (Reusable Patterns)

See [patterns.md](patterns.md) for reusable mode-related patterns (creating new modes, mode inheritance, spritesheet stacking).
