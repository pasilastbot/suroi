# Translations

<!-- @tier: 2 -->
<!-- @parent: ../../architecture.md -->
<!-- @modules: modules/ -->
<!-- @source: client/src/scripts/utils/translations/, client/src/translations/, vite/plugins/translations-plugin.ts -->

## Purpose

Manages client-side multi-language support through lazy-loaded HJSON translation files, dynamic language switching, and fallback chains for missing translations.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/utils/translations/translations.ts` | Main translation API (`initTranslation()`, `getTranslatedString()`) |
| `client/src/scripts/utils/translations/typings.ts` | TypeScript types for translation keys |
| `client/src/translations/*.hjson` | Language files (en.hjson, fr.hjson, zh.hjson, etc.) — 28 languages supported |
| `vite/plugins/translations-plugin.ts` | Vite build plugin for HJSON parsing and virtual module generation |

## Architecture

The translation system follows a lazy-loading pattern:

```
Build Time:
  Vite parses all .hjson files
  → Virtual module "virtual:translations-manifest"
  → Metadata: which languages are mandatory, which are optional

Runtime:
  Client startup → initTranslation()
    → Load mandatory languages + default + selected language
    → Keep optional languages in metadata (load on-demand)
    → Merge into TRANSLATIONS.translations object
    ↓
  UI rendering → getTranslatedString(key)
    → Look up in selected language
    → Fallback to default language
    → Fallback to item definitions
    → Fallback to key literal
```

## Data Flow

```
HJSON Translation Files (28 languages)
    ↓
Vite translations-plugin (HJSON → JS, virtual module generation)
    ↓
Virtual module exports { manifest, importTranslation }
    ↓
Client initTranslation():
  - Detect default language (from cvar or browser)
  - Load mandatory languages + selected language
  - Merge each into TRANSLATIONS.translations[lang]
    ↓
getTranslatedString(key) → looks up key with fallback chain
    ↓
DOM translation → translateCurrentDOM() updates all UI
```

## Interfaces & Contracts

**Translation API:**
- `async initTranslation(): Promise<void>` — Initialize and load translations at startup
- `getTranslatedString(key: TranslationKeys, replacements?: Record<string, string>): string` — Look up translated string with fallbacks
- `translateCurrentDOM(): void` — Re-render DOM with current language

**HJSON Structure:**
```hjson
{
  category: {
    subcategory: {
      key: "English text"
    }
  }
}
```

**TranslationKeys type:** Union of all valid translation keys (auto-generated from HJSON files)

## Dependencies

- **Depends on:** Object definitions (for fallback item names)
- **Depended on by:** UI components, HUD, game console
- **Build:** Vite (for HJSON parsing plugin)

## Module Index (Tier 3)

For implementation details, see:
- [Translation System](modules/translation-system.md) — Loader, APIs, language switching, fallback chains

## Language Support

| Code | Name | Status |
|------|------|--------|
| en | English | Default |
| fr, de, es, ja, ru | European/Asian languages | Full |
| +23 others | Various languages | Full |
| hp18 | HP-18 (Easter Egg) | Hardcoded |

## Configuration

**Language selection:**
- Detected from browser locale on first visit
- Configurable via console cvar: `cv_language`
- Persisted to browser localStorage (optional implementation)

**Mandatory vs Optional:**
- **Mandatory:** English + default language always loaded at startup
- **Optional:** Other languages loaded on-demand when user selects them

## Related Documents

- **Tier 1:** [Architecture](../../architecture.md) — Multi-language design patterns
- **Tier 1:** [Development Guide](../../development.md) — Language file contributing
- **Tier 2:** [UI Management](../ui-management/README.md) — DOM element handling
- **Tier 2:** [Client Rendering](../client-rendering/README.md) — UI rendering pipeline
- **Patterns:** [Translation Patterns](patterns.md) — Common usage patterns
