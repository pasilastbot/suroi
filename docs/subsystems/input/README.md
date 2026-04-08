# Input & Controls Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: client/src/scripts/managers/inputManager.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Input subsystem captures keyboard, mouse, touch, and mobile joystick input, maps them to game actions, and produces `InputPacket` data sent to the server every frame (when changed).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/managers/inputManager.ts` | `InputManager` — input mapping, packet creation |
| `client/src/scripts/console/variables.ts` | `defaultBinds` — default key bindings |
| `client/src/scripts/utils/crosshairs.ts` | Crosshair definitions |
| `common/src/packets/inputPacket.ts` | `InputData`, `InputAction`, `areDifferent` |

## Architecture

```
InputMapper (keyboard/mouse → actions)
    ├── _inputToAction — key/button → Set<action>
    └── _actionToInput — action → Set<key/button>

InputManager
    ├── Movement (WASD / mobile joystick)
    ├── Aim (mouse / mobile stick)
    ├── Actions (reload, interact, etc.)
    └── Build InputPacket
            └── areDifferent() — skip if unchanged
```

## Input Sources

| Source | Usage |
|--------|-------|
| **Keyboard** | Movement (WASD), actions (R, F, E, etc.) |
| **Mouse** | Aim direction, attack (click), map ping |
| **Touch** | Mobile: nipplejs joystick for movement + aim |
| **Gamepad** | Optional (if supported) |

## Key Bindings

- Defined in `defaultBinds` (variables.ts)
- `InputMapper.addActionsToInput(key, ...actions)` — bind key to actions
- `InputMapper.addInputsToAction(action, ...inputs)` — bind action to keys
- Actions map to `InputActions` enum (EquipItem, Reload, Interact, etc.)

## Data Flow

```
Frame (requestAnimationFrame or game loop)
    → InputManager.update()
    → Read movement (up/down/left/right)
    → Read aim (rotation, distanceToMouse)
    → Read attacking
    → Collect actions (reload, interact, use item, etc.)
    → Build InputData
    → areDifferent(lastPacket, newPacket)?
    → If different: serialize InputPacket, send to server
```

## Mobile Support

- `isMobile` / `FORCE_MOBILE` — enables touch UI
- `nipplejs` — virtual joystick for movement and aim
- `InputData.mobile` — `{ moving, angle }` when mobile

## Ping Sequence

- `pingSeq` — 7-bit sequence number in InputPacket
- High bit (128) — "skip" flag; server ignores packet (e.g. no input change)

## Module Index (Tier 3)

- [Key Bindings](modules/key-bindings.md) — InputMapper, defaultBinds, action mapping
- [Mobile](modules/mobile.md) — nipplejs joysticks, InputData.mobile, FORCE_MOBILE

## Protocol Considerations

- **Affects protocol:** Yes. InputPacket format changes require protocol bump.

## Dependencies

- **Depends on:** Packets (InputPacket), CameraManager (aim direction), Game
- **Depended on by:** Game (sends packets), UI (keybind config)

## Related Documents

- **Tier 1:** [docs/protocol.md](../../protocol.md) — InputPacket, InputActions
- **Tier 2:** [../packets/](../packets/) — InputPacket module
- **Tier 2:** [../rendering/](../rendering/) — CameraManager for aim
