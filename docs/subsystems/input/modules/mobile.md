# Input — Mobile Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/input/README.md -->
<!-- @source: client/src/scripts/managers/inputManager.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents mobile/touch input: nipplejs virtual joysticks for movement and aim, `InputData.mobile`, `mb_controls_enabled` cvar, and `FORCE_MOBILE` for testing.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/managers/inputManager.ts | nipplejs setup, mobile InputData | Medium |
| @file client/src/scripts/utils/constants.ts | `FORCE_MOBILE` | Low |

## Detection

- **isMobile:** From `pixi.js` — `isMobile.any`
- **mb_controls_enabled:** Cvar — user can disable mobile controls
- **FORCE_MOBILE:** Constant — force mobile UI/controls for desktop testing
- **InputManager.isMobile** — `(isMobile.any && mb_controls_enabled) || FORCE_MOBILE`

## Joysticks

- **nipplejs** — Virtual joystick library
- **Left joystick** — Movement (up/down/left/right)
- **Right joystick** — Aim (angle)
- Created when `isMobile` and game started; destroyed on game end

## InputData.mobile

- When mobile: `{ moving: boolean, angle: number }`
- **moving** — From left joystick (any direction pressed)
- **angle** — From right joystick (aim direction)
- Replaces keyboard movement and mouse aim in InputPacket

## Business Rules

- Mobile disables some pointer events (e.g. map ping via click) to avoid conflicts
- MapManager uses `isMobile.any` for resolution and visibility
- `cv_blur_splash` defaults off on mobile (performance)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Input overview
- **Tier 3:** [key-bindings.md](key-bindings.md) — Key bindings (desktop)
- **Tier 2:** [../console/](../../console/) — mb_controls_enabled cvar
