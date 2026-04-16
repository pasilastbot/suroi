# Translation System Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: client/src/scripts/utils/translations/translations.ts, client/src/translations/*.hjson, vite/plugins/translations-plugin.ts -->

## Purpose

Manages client-side multi-language support through lazy-loaded HJSON translation files, language detection, and dynamic switching without page reload.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/utils/translations/translations.ts` | Translation API, loader, string lookup, language switching | High |
| `client/src/translations/*.hjson` | Language-specific translation files (en.hjson, fr.hjson, etc.) | Low |
| `client/src/scripts/utils/translations/typings.ts` | TypeScript types for translation keys | Medium |
| `vite/plugins/translations-plugin.ts` | Vite plugin: HJSON parsing, virtual module generation | High |

## Business Rules

- **HJSON Format:** All translations stored as Human JSON (comments allowed, trailing commas permitted)
- **Lazy Loading:** Only mandatory translations and selected language loaded at startup; others loaded on-demand
- **Fallback Chain:** Missing key → fallback to default language → fallback to key literal → item definition name
- **Language Switch:** Client can change language without reload; triggers DOM re-translation
- **Easter Egg:** Language `hp18` returns literal "HP-18" for all strings
- **Key Namespacing:** Keys separated by dots (e.g., `ui.menu.play` for nested objects)
- **Definition Rehydration:** Some strings auto-generated from item definitions (gun names, ammo types, etc.)

## Data Lineage

```
HJSON Files (28 languages: en.hjson, fr.hjson, zh.hjson, etc.)
    ↓
Vite translations-plugin:
  - Parse HJSON → JavaScript objects
  - Generate virtual module "virtual:translations-manifest"
  - Export { manifest, importTranslation }
    ↓
At Client Startup (initTranslation()):
  - Detect default language (from cvar cv_language or browser)
  - Load mandatory translations (marker: "mandatory": true)
  - Load selected language (if different)
  - Pre-load commonly used languages (optional optimization)
  - Merge into TRANSLATIONS.translations{[lang]: {key: string}}
    ↓
Runtime Lookup (getTranslatedString(key)):
  - Special case: badge_* → resolve from Badges registry
  - Special case: hp18 → return "HP-18"
  - Look up in selected language
  - Fallback to default language
  - Fallback to Loots definitions (if item)
  - Fallback to key literal
```

## Language Support

| Code | Name | Status | File |
|------|------|--------|------|
| en | English | Default | en.hjson |
| fr | Français | Full | fr.hjson |
| zh | 中文 (Simplified) | Full | zh.hjson |
| de | Deutsch | Full | de.hjson |
| es | Español | Full | es.hjson |
| ja | 日本語 | Full | jp.hjson |
| ru | Русский | Full | ru.hjson |
| +23 others | Various | Full | *.hjson |
| hp18 | HP-18 (Easter Egg) | Hardcoded | — |

## HJSON Structure Example

```hjson
# en.hjson
{
  ui: {
    menu: {
      play: "Play",
      settings: "Settings",
      quit: "Quit"
    },
    hud: {
      health: "Health",
      ammo: "Ammo"
    }
  },
  weapons: {
    ak47: "AK-47",
    m16: "M16"
  }
}
```

## Complex Functions

### `initTranslation(): Promise<void>` — @file client/src/scripts/utils/translations/translations.ts  
**Purpose:** Load translation data at client startup.

**Implicit Behavior:**
1. Check if already initialized (throw if re-init attempted)
2. Detect default language (from cvar or browser locale)
3. Get selected language (from game console cvar `cv_language`)
4. Import manifest + importTranslation function from virtual module
5. Determine which languages to load:
   - Always load: mandatory languages + default + selected
   - Optionally load: other languages if browser idle
6. Merge each language into `TRANSLATIONS.translations[lang]`
7. Call `translateCurrentDOM()` to apply translations to UI

**Async Behavior:** `Promise.all()` loads multiple languages concurrently

**Error Handling:** If language data missing, fallback chain (see below) handles empty keys

**File:** `client/src/scripts/utils/translations/translations.ts:29–60`

### `getTranslatedString(key: TranslationKeys, replacements?: Record<string, string>): string`  
**Purpose:** Look up a translation string with fallbacks.

**Implicit Behavior:**
1. Validate setup (throw if called before `initTranslation()`)
2. Special case: if language is "hp18", return "HP-18"
3. Special case: if key starts with "badge_", reify Badges registry and convert to idString
4. Look up in selected language data
5. If not found, look up in default language data
6. If still not found, try item definitions (e.g., Loots.fromStringSafe(key)?.name)
7. If still not found, return key literal
8. Apply replacements (if provided)

**Replacements Pattern:**
```typescript
getTranslatedString("damage_taken_by", { amount: "50", source: "rifle" })
// Returns interpolated string with {amount} and {source} replaced
```

**Gotcha:** Replace pattern typically uses `${...}` template syntax; depends on HJSON template format.

**File:** `client/src/scripts/utils/translations/translations.ts:72–100`

### `translateCurrentDOM(): void`  
**Purpose:** Update all DOM elements with translated strings.

**Implicit Behavior:**
1. Scan DOM for elements with `data-i18n` attribute
2. For each element:
   - Read `data-i18n` (key name)
   - Call `getTranslatedString(key)`
   - Set element text or HTML (depending on element type)
3. Trigger re-render for Svelte components (if applicable)

**Gotcha:** Does not re-run for dynamically created DOM; those must call `getTranslatedString()` on creation.

**File:** `client/src/scripts/utils/translations/translations.ts` (function location may vary)

## Language Switching

**User Action:** Click language button in UI settings
  ↓ 
**Handler:** Updates `cv_language` console variable
  ↓ 
**Event:** Language change listener triggers
  ↓ 
**Actions:**
1. Set `selectedLanguage = newLanguage`
2. Call `initTranslation()` (will load new language if not already loaded)
3. Call `translateCurrentDOM()` (re-render all UI strings)

**Result:** UI updated without page reload.

## Related Documents

- **Tier 2:** [Translations Subsystem](../README.md) — Translation management overview
- **Tier 2:** [UI Management](../../ui-management/README.md) — DOM elements and event handling
- **Tier 1:** [Development Guide](../../../development.md) — Language file contributing
- **Patterns:** [Translation Patterns](../patterns.md) — Common usage patterns
