# UI & Translations Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: client/src/scripts/managers/uiManager.ts, client/src/scripts/ui.ts, client/src/scripts/utils/translations/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The UI & Translations subsystem manages the game HUD (health, adrenaline, inventory, killfeed, etc.), menus (lobby, settings, game over), and internationalization (i18n) via HJSON translation files.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/managers/uiManager.ts` | `UIManager` — HUD state, inventory display, killfeed, teammate list |
| `client/src/scripts/ui.ts` | Lobby, play buttons, region selection, loadout, game over |
| `client/src/scripts/uiHelpers.ts` | DOM helpers, dropdowns |
| `client/src/scripts/utils/translations/translations.ts` | `initTranslation`, `getTranslatedString`, `TRANSLATIONS` |
| `client/src/scripts/utils/translations/typings.ts` | `TranslationKeys` — typed translation keys |
| `client/src/translations/*.hjson` | Per-language translation files (en, de, fr, etc.) |
| `client/index.html` | HTML structure — splash, game container, modals |

## Architecture

```
UIManager (singleton)
    ├── inventory — weapons, items, scope, locked slots
    ├── health, adrenaline, shield — bars
    ├── killfeed — recent kills
    ├── teammates — team panel
    ├── minimap — via MapManager
    └── Updates DOM / PixiJS overlays from game state

ui.ts
    ├── Lobby — region picker, play button, team mode
    ├── Loadout — skin, emotes, badge
    ├── Game over — stats, spectate
    └── Modals — settings, report, etc.
```

## Translation System

- **Format:** HJSON files in `client/src/translations/`
- **Keys:** `TranslationKeys` — typed keys for `getTranslatedString(key, replacements)`
- **Loading:** `initTranslation()` — loads manifest, imports selected + default language
- **Virtual import:** `virtual:translations-manifest` — Vite plugin provides manifest
- **Replacements:** `getTranslatedString("key", { progress: "1/5" })` for dynamic values

## Supported Languages

Languages include: en, de, es, fr, it, nl, pl, ru, zh, jp, and others (see `client/src/translations/*.hjson`).

## UI Stack

- **jQuery** — DOM manipulation for HUD and menus
- **PixiJS** — Overlays (killfeed sprites, minimap)
- **Svelte** — Some UI components (e.g. news, changelog)

## Key UI Elements

| Element | Purpose |
|---------|---------|
| Health/Adrenaline bars | Player stats |
| Weapon slots | Active weapon, locked state |
| Inventory panel | Items, scope, armor |
| Killfeed | Recent kills with weapon icons |
| Minimap | MapManager — player, teammates, gas |
| Teammate list | Names, health, status |
| Crosshair | Aim reticle (from crosshairs.ts) |
| Game over | Placement, kills, damage, spectate |

## Module Index (Tier 3)

- [Translations](modules/translations.md) — HJSON i18n, initTranslation, getTranslatedString
- [HUD](modules/hud.md) — Health, adrenaline, inventory, killfeed, teammates

## Protocol Considerations

- **Affects protocol:** No. UI displays data from packets; no wire format impact.

## Dependencies

- **Depends on:** Game (player state), Packets (PlayerData, KillData), Translations
- **Depended on by:** Game (updates UI on state change), Input (keybind UI)

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — Client architecture
- **Tier 2:** [../rendering/](../rendering/) — MapManager, overlay rendering
