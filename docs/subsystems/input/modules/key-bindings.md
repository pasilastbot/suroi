# Input — Key Bindings Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/input/README.md -->
<!-- @source: client/src/scripts/managers/inputManager.ts, client/src/scripts/console/variables.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the key binding system: `InputMapper` bidirectional mapping between inputs (keys, buttons) and actions, `defaultBinds`, and how bindings are persisted via console cvars.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/managers/inputManager.ts | `InputMapper`, `InputManager` — mapping, packet build | High |
| @file client/src/scripts/console/variables.ts | `defaultBinds`, `defaultClientCVars` — default bindings | Medium |

## InputMapper

- **Bidirectional maps:** `_inputToAction` (key → actions), `_actionToInput` (action → keys)
- **addActionsToInput(key, ...actions)** — Bind actions to a key
- **addInputsToAction(action, ...inputs)** — Bind keys to an action
- **remove(input, action)** — Unbind one action from one input
- **unbindInput(key)** — Remove all actions from a key
- **unbindAction(action)** — Remove all keys from an action

## Actions

- **InputActions** — Game actions (EquipItem, Reload, Interact, UseItem, etc.)
- **Commands** — Console commands (e.g. `give`, `teleport`)
- **CompiledAction** — Special handlers (e.g. scope zoom)

## defaultBinds

- Defined in `variables.ts`
- Format: `{ action: string[] }` — action name → array of key strings
- Loaded at startup; `InputMapper` populated via `addInputsToAction`
- User bindings override via cvars (e.g. `bind "R" "reload"`)

## Persistence

- Key bindings stored as cvars; `archive` flag persists to localStorage
- Console `bind` command updates both InputMapper and cvar

## Business Rules

- One action can have multiple keys; one key can trigger multiple actions
- Mobile uses nipplejs joystick; `InputData.mobile` carries `{ moving, angle }`

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Input overview
- **Tier 2:** [../console/](../../console/) — CVars, bind command
- **Tier 3:** [../packets/modules/input-packet.md](../../packets/modules/input-packet.md) — InputPacket format
