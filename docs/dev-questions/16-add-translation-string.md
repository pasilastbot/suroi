# Q: How do I add a new translatable string for internationalization (i18n)?

<!-- @tags: UI, translations, HJSON, i18n -->
<!-- @related: docs/subsystems/ui/modules/translations.md -->

## 1. Add the key to TranslationKeys

Open `client/src/scripts/utils/translations/typings.ts` and add your key:

```typescript
export interface TranslationKeys {
    // ...existing keys...
    my_new_string: string    // ← add here
}
```

This provides type-checking — `getTranslatedString("my_new_string")` will
type-check and autocomplete.

## 2. Add the string to each language file

Translation files are HJSON located at:

```
client/src/translations/<lang>.hjson
```

At minimum, add to `en.hjson` (English — required):

```hjson
{
    // ...existing...
    my_new_string: "My new string"
}
```

Add to as many other language files as you can. Files that don't have the key
will fall back to the English value automatically.

**Available languages:** `en`, `de`, `es`, `fr`, `it`, `nl`, `pl`, `ru`, `zh`,
`jp`, and others in `client/src/translations/`.

## 3. Use the string in code

```typescript
import { getTranslatedString } from "../utils/translations/translations";

// Simple lookup
const text = getTranslatedString("my_new_string");

// With replacements (use {{placeholder}} syntax in HJSON)
// HJSON: my_new_string: "Hello {{name}}, you have {{count}} kills"
const text = getTranslatedString("my_new_string", {
    name: "Player",
    count: "5",
});
```

## 4. Use in HTML (data-i18n attribute)

For static DOM elements, use the `data-i18n` attribute — the translation system
updates these automatically on language change:

```html
<span data-i18n="my_new_string"></span>
```

```svelte
<!-- In a Svelte component -->
<span data-i18n="my_new_string"></span>
```

`translateCurrentDOM()` is called after language change and updates all
`data-i18n` elements.

## 5. Test

```bash
bun dev
```

Switch languages in Settings → Language to verify fallback behavior.

## Notes

- Translations use **lazy loading** — non-mandatory languages are loaded on demand
- The `cv_language` cvar stores the selected language
- If a key is missing in the selected language, it falls back to English
- HJSON supports comments (`//`) and multiline strings — use them liberally

## Related

- [Translations Module](../subsystems/ui/modules/translations.md) — HJSON format, flow, lazy loading
- [Console CVars](../subsystems/console/modules/cvars.md) — `cv_language`
