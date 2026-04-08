# UI — Translations Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/ui/README.md -->
<!-- @source: client/src/scripts/utils/translations/, client/src/translations/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the internationalization (i18n) system: HJSON translation files, loading, and typed translation keys.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/utils/translations/translations.ts | `initTranslation`, `getTranslatedString`, `TRANSLATIONS` | Medium |
| @file client/src/scripts/utils/translations/typings.ts | `TranslationKeys` — typed keys | Low |
| @file client/src/translations/*.hjson | Per-language translation files | Low |
| `vite/plugins/translations-plugin` | `virtual:translations-manifest` — lazy load | Medium |

## Translation Flow

```
initTranslation()
    → Load cv_language (default + selected)
    → Import virtual:translations-manifest
    → Mandatory + selected + default: await importTranslation(lang)
    → Others: deferred (lazy)
    → TRANSLATIONS.translations[lang] = { ...manifest, ...content }
    → translateCurrentDOM()

getTranslatedString(key, replacements?)
    → Lookup TRANSLATIONS.translations[selectedLanguage][key]
    → Apply replacements (e.g. { progress: "1/5" })
    → Return string
```

## HJSON Format

- **Location:** `client/src/translations/*.hjson`
- **Keys:** `TranslationKeys` — typed for autocomplete and validation
- **Replacements:** `getTranslatedString("key", { name: "value" })` — `{{name}}` in string

## Supported Languages

en, de, es, fr, it, nl, pl, ru, zh, jp, and others (see `client/src/translations/`).

## Business Rules

- `cv_language` cvar controls selected language
- Mandatory languages always loaded; others lazy-loaded on demand
- `translateCurrentDOM()` updates existing DOM elements with `data-i18n` attributes

## Related Documents

- **Tier 2:** [../README.md](../README.md) — UI overview
- **Tier 2:** [../console/](../../console/) — cv_language cvar
