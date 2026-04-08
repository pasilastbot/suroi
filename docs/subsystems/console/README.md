# Game Console Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: client/src/scripts/console/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Game Console provides an in-game debug console for commands, variables (cvars), key bindings, and developer tools. It is a floating window opened by a keybind (default: `~`).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/console/gameConsole.ts` | `GameConsole` — UI, input, output, command execution |
| `client/src/scripts/console/commands.ts` | Command definitions and handlers |
| `client/src/scripts/console/variables.ts` | `defaultBinds`, `defaultClientCVars`, ConVar system |
| `client/src/scripts/console/internals.ts` | `evalQuery`, `extractCommandsAndArgs` — query parsing |

## Architecture

```
GameConsole (FloatingWindow)
    ├── input — command/query input
    ├── output — log output
    └── autocomplete — command/variable suggestions

Commands — registered handlers (e.g. give, teleport)
Variables (cvars) — persisted settings (language, keybinds, etc.)
InputMapper — uses variables for key bindings
```

## Data Flow

```
User types in console
    → Parse as command or cvar query
    → Commands: execute handler
    → Cvars: get/set value, persist to localStorage
    → Output result to console
```

## Key Concepts

- **Commands:** Server-style commands (some require dev mode). Defined in `commands.ts`.
- **CVars:** Client variables — language, anonymize names, etc. Persisted across sessions.
- **Binds:** Key bindings stored as cvars; `InputMapper` reads them.
- **FloatingWindow:** Resizable, draggable console overlay.

## Module Index (Tier 3)

- [Commands](modules/commands.md) — Command class, registration, dev commands
- [CVars](modules/cvars.md) — ConVar system, flags, persistence, defaultClientCVars

## Protocol Considerations

- **Affects protocol:** No. Console is client-only; some commands may trigger server packets (e.g. give).

## Dependencies

- **Depends on:** InputManager (binds), Game (context), UIManager
- **Depended on by:** Input (key bindings), Development (debug commands)

## Related Documents

- **Tier 2:** [../input/](../input/) — Key bindings from console variables
- **Tier 2:** [../ui/](../ui/) — UI integration
