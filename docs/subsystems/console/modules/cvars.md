# Game Console — CVars Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/console/README.md -->
<!-- @source: client/src/scripts/console/variables.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the ConVar (console variable) system: typed variables with flags (archive, readonly, cheat), persistence to localStorage, and casters for string↔value conversion.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/console/variables.ts | `ConVar`, `Casters`, `defaultClientCVars`, `defaultBinds` | Medium |

## ConVar

- **Value** — Typed (string, number, boolean)
- **Flags** — archive (persist), readonly, cheat, replicated
- **Caster** — String↔value conversion (toString, toNumber, toBoolean, etc.)
- **Change listeners** — `CVarChangeListener` on value change

## CVarFlags

| Flag | Purpose |
|------|---------|
| archive | Persist to localStorage |
| readonly | Cannot be changed at runtime |
| cheat | Dev-only |
| replicated | Server sync (if used) |

## defaultClientCVars

- **cv_language** — Selected language (affects translations)
- **cv_anonymize_player_names** — Replace names with defaultName_id
- Others — UI, debug, gameplay toggles

## Casters

- `toString`, `toNumber`, `toInt`, `toBoolean` — Parse string to value
- Return `Result<T, string>` — `{ res }` or `{ err }`

## Persistence

- `archive` flag → save to localStorage on change
- Load on init from `GameConsole.getBuiltInCVar` / stored values

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Console overview
- **Tier 3:** [commands.md](commands.md) — Commands can get/set cvars
- **Tier 2:** [../input/](../../input/) — defaultBinds, key bindings
