# Game Console — Commands Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/console/README.md -->
<!-- @source: client/src/scripts/console/commands.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the console command system: `Command` class, registration, execution, and dev-only commands. Commands are invoked from the in-game console.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/console/commands.ts | `Command`, `CommandExecutor`, `setUpCommands()` | High |

## Command Structure

- **Command** — `name`, `executor`, `info` (signatures, allowOnlyWhenGameStarted)
- **CommandExecutor** — `(...args) => void | PossibleError`
- **CommandInfo** — `short`, `long` descriptions, `signatures` (args, optional, rest, noexcept)

## Execution

- `command.run(args)` — Runs executor; returns error if any
- `allowOnlyWhenGameStarted` — If true, command only runs when `Game.gameStarted`
- Dev commands — Require dev mode; some send packets to server (e.g. `give`, `teleport`)

## Example Commands

| Command | Purpose |
|---------|---------|
| `give` | Spawn item (dev) |
| `teleport` | Move player (dev) |
| `bind` | Set key binding |
| `cv_language` | Set language cvar |
| `clear` | Clear console output |

## Business Rules

- Commands are registered in `setUpCommands()` at startup
- Autocomplete uses command names and cvars
- Server commands (give, etc.) require WebSocket connection and dev mode

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Console overview
- **Tier 2:** [../input/](../../input/) — bind command, InputMapper
