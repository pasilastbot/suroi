# Q: How do I add a new console variable (cvar) that persists across sessions?

<!-- @tags: console, cvars, persistence, localStorage -->
<!-- @related: docs/subsystems/console/modules/cvars.md -->

## 1. Add to defaultClientCVars

Open `client/src/scripts/console/variables.ts` and add your cvar to the
`defaultClientCVars` object:

```typescript
export const defaultClientCVars: SimpleCVarMapping = Object.freeze({
    // ...existing cvars...
    cv_my_setting: false,      // boolean with default
    // OR
    cv_my_volume: 1.0,         // number with default
    // OR
    cv_my_mode: "normal",      // string with default
});
```

The `cv_` prefix is a convention for client variables. Use it consistently.

**Name rules:**
- `cv_*` — regular client variable
- `dv_*` — dev/debug variable
- `pf_*` — performance variable

## 2. Make it persistent (archived)

To persist across sessions (saved to localStorage), use the object form with flags:

```typescript
export const defaultClientCVars: SimpleCVarMapping = Object.freeze({
    cv_my_setting: {
        value: false,        // default value
        flags: {
            archive: true,   // save to localStorage
            readonly: false, // can be changed at runtime
            cheat: false,    // not a cheat code
        }
    },
});
```

The simple form (just the value) automatically uses `archive: false`.

## 3. Use the cvar in code

```typescript
import { GameConsole } from "./gameConsole";

// Read
const value = GameConsole.getBuiltInCVar("cv_my_setting"); // typed correctly

// Write
GameConsole.setBuiltInCVar("cv_my_setting", true);

// Listen for changes
GameConsole.builtInCVars.get("cv_my_setting")?.addChangeListener(
    (newValue, oldValue) => {
        console.log("cv_my_setting changed:", oldValue, "→", newValue);
        // Apply the effect here
    }
);
```

## 4. Expose in the Settings UI (optional)

If the cvar should be configurable through the Settings menu, add a checkbox,
slider, or select to the appropriate Svelte component in
`client/src/components/` and bind it to the cvar via `getBuiltInCVar` /
`setBuiltInCVar`.

## 5. Test

```bash
bun dev
```

Open the console (`` ` ``) and type `cv_my_setting` to read the value, or
`cv_my_setting true` to set it.

## Type Safety

`defaultClientCVars` uses a `SimpleCVarMapping` type that infers the TypeScript
type of each cvar from its default value. `getBuiltInCVar("cv_my_setting")`
returns `boolean` (or the correct type) automatically.

## CVarFlags Reference

| Flag | Effect |
|------|--------|
| `archive: true` | Persisted to localStorage (survives page reload) |
| `readonly: true` | Cannot be changed from console at runtime |
| `cheat: true` | Dev-only; blocked in normal play |
| `replicated: true` | Synced to server (if applicable) |

## Related

- [CVars Module](../subsystems/console/modules/cvars.md) — full ConVar reference
- [Console Commands](17-add-console-command.md) — commands can read/write cvars
