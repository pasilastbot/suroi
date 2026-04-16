# Input Management — Mobile Input Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: client/src/scripts/managers/inputManager.ts -->

## Purpose
Encapsulates mobile-specific input handling: virtual joystick management via `nipplejs` library, touch event processing, device detection, and mobile-specific control mappings that differ from desktop keyboard/mouse input.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/inputManager.ts` | InputManager class with mobile initialization | High |
| `client/src/scripts/console/variables.ts` | Default key bindings (desktop vs. mobile) | Medium |
| `common/src/packets/inputPacket.ts` | InputData serialization (movement, rotation, action) | Low |
| `common/src/constants.ts` | InputActions enum (Attack, Reload, Interact) | Low |

## Mobile Detection

**Method:** Combine device-level detection with user override

```typescript
// client/src/scripts/managers/inputManager.ts:336
this.isMobile = (
  isMobile.any                                    // PixiJS device detector
  && GameConsole.getBuiltInCVar("mb_controls_enabled")  // User enabled mobile UI
) || FORCE_MOBILE;                                      // Dev override for testing
```

**Components:**
- `isMobile.any` — PixiJS library checks user agent, touch capability
- `mb_controls_enabled` — Console variable user can toggle (default: true on mobile browsers)
- `FORCE_MOBILE` — Dev constant to test mobile on desktop (@file client/src/utils/constants.ts)

**Impact:** Once `isMobile = true`, InputManager initializes virtual joysticks instead of keyboard listeners

## Virtual Joystick Initialization

**Library:** `nipplejs` — JavaScript virtual joystick for touch

```typescript
// Rough flow (simplified)
if (this.isMobile) {
  const manager = nipplejs.create({
    zone: canvasElement,           // Touch input area
    mode: "static",                // Fixed position, doesn't follow finger drag
    color: "rgba(255,255,255,0.5)", // Visual appearance
    size: 100,                     // Diameter in pixels
    // ... other options
  });

  manager.on("move", (event, data: JoystickOutputData) => {
    // data = { angle, distance, x, y, direction, pressure }
    this.mobileMovementVector = Vec(data.x, data.y);  // Normalized to [-1, 1]
  });

  manager.on("end", () => {
    this.mobileMovementVector = Vec(0, 0);  // Release joystick
  });
}
```

**Joystick output:**
- `angle` — Absolute angle in degrees (0–360)
- `distance` — Radial distance from center (0–100px, normalized)
- `x`, `y` — Cartesian components [-1, 1]
- `direction` — Cardinal direction (N, NE, E, SE, etc.)

**Layout:** Two joysticks typically positioned:
1. **Movement joystick** — Bottom-left corner
2. **Aiming joystick** — Bottom-right corner (or action buttons)

## Touch Event Handling

**Joystick generation:** `nipplejs` internally handles all touch events (no manual `touchstart`/`touchmove`/`touchend`)

**Custom touch logic** (if not using joystick):
```typescript
// Touch screen zones for action buttons
document.addEventListener("touchstart", (event) => {
  const touch = event.touches[0];
  const x = touch.clientX;
  const y = touch.clientY;
  
  // Map screen coordinates to action (fire, reload, etc.)
  if (isInFireButton(x, y)) {
    this.currentAction = PlayerActions.Attack;
  } else if (isInReloadButton(x, y)) {
    this.currentAction = PlayerActions.Reload;
  }
});

document.addEventListener("touchend", (event) => {
  this.currentAction = undefined;  // Release action
});
```

## Mobile vs. Desktop Control Differences

| Input Type | Desktop | Mobile |
|------------|---------|--------|
| **Movement** | WASD keys | Left virtual joystick |
| **Camera/Aim** | Mouse movement | Right virtual joystick (or gyroscope) |
| **Fire** | Left mouse button | Tap right side of screen OR button |
| **Reload** | R key | Reload button (always visible) |
| **Use Item** | E key | Item action button |
| **Swap Weapon** | Number keys or scroll | Weapon selection buttons |

**Key difference:** Mobile maps **spatial finger position** to directional axes, while desktop maps **distinct keys** to actions.

## Input Packet Format

**Both desktop and mobile** serialize input the same way (see @file common/src/packets/inputPacket.ts):

```typescript
InputData {
  movement: Vector,           // [-1, 1] per axis (WASD or joystick)
  rotation: number,           // Camera angle (mouse or gyro)
  action: InputAction | null, // Current action enum
  weaponIndex: number,        // Equipped weapon slot [0–4]
  // Data serialization handles desktop vs. mobile transparently
}
```

**Example mobile input:**
```typescript
// Left joystick moved up+left
InputData { movement: Vec(-0.8, -0.8), rotation: ... }

// Right joystick moved right (aiming)
InputData { rotation: 0 }  // 0 = facing right

// Fire button tapped
InputData { action: PlayerActions.Attack }
```

## Cross-Device Detection

**At game start:**
1. Detect device type: `isMobile.any`
2. Query user console settings
3. Load appropriate input mapper (`InputMapper` with desktop vs. mobile bindings)
4. Initialize visible UI (keyboard hints vs. on-screen buttons)

**Dynamic switching:** Can toggle via console:
```
> set mb_controls_enabled true
> reload()  // Reinitialize InputManager
```

**Use case:** Game works on any device; player can manually switch input mode if desired

## Mobile Optimization Patterns

### 1. Large Touch Targets
Action buttons padded and spaced to avoid mis-taps:
```typescript
const buttonSize = 60;  // px minimum (finger width ~45px)
const spacing = 10;     // px between buttons
```

### 2. Visual Feedback
Touch states are reflected immediately (no network latency):
```typescript
// Local
this.attackButtonPressed = true;  // UI shows pressed state

// Next tick
sendInputPacket({ action: PlayerActions.Attack });  // Send to server

// Server responds with state update
updatePacket.myAmmo = ...;  // Feedback that fire registered
```

### 3. Gesture Combinations
Multi-touch not used in Suroi (complex fine motor control not required):
- Single-touch: Movement or aiming
- Tap-and-drag: Alternative to joystick (not implemented)

### 4. Haptic Feedback (Optional)
Some mobile browsers support haptic feedback on action confirmation:
```typescript
navigator.vibrate?.(100);  // Vibrate for 100ms when fire triggers
```

## Mobile-Specific Issues & Workarounds

### Issue: Virtual Keyboard Occlusion
**Problem:** On-screen keyboard appears when typing (console), covers game
**Workaround:** Blur input when game in focus; only show keyboard when user explicitly opens console

### Issue: Momentum Scrolling
**Problem:** Swiping on joystick triggers page scroll on iOS
**Workaround:** CSS `touch-action: none;` on joystick container

### Issue: Notch / Safe Insets
**Problem:** Display notch on newer phones cuts off UI
**Workaround:** CSS `viewport-fit: cover; constant(safe-area-inset-*);` for button positioning

## Complex Functions

### `InputManager constructor` — @file client/src/scripts/managers/inputManager.ts:190
**Purpose:** Initialize input system based on device type
**Complexity:** ~150 lines
**Precondition:** Game canvas loaded
**Implicit behavior:**
- Detects device (`isMobile`)
- Sets up listeners (keyboard or touch)
- Creates input mapper (bindable keys/actions)
- Initializes virtual joysticks if mobile
- Caches default bindings from console variables

### `nipplejs.create()` setup — @file client/src/scripts/managers/inputManager.ts:386+
**Purpose:** Create virtual joystick instances
**Parameters:**
- `zone` — DOM element to attach joystick to
- `mode` — "static" (fixed position) or "dynamic" (follows finger)
- `color`, `size` — Appearance
- `dataOnly` — True for no visual display (custom rendering)
**Implicit behavior:**
- Registers touch listeners internally
- Emits "move" and "end" events
- Joystick persists for entire game session

### `InputManager.isMobile getter` — @file client/src/scripts/managers/inputManager.ts:187
**Purpose:** Read boolean mobile flag
**Return:** `true` if mobile controls active
**Usage:** Conditional rendering of UI buttons vs. keyboard hints

## Performance Considerations

**Touch latency:** ~8–17ms (one frame at 60 FPS)
**Joystick update rate:** Unlimited; fires on every touch move event
**Button rendering:** ~60 FPS (single canvas or DOM elements)

**Mobile optimization:**
- Minimize reflow/repaint (use fixed positioning for buttons)
- Use passive touch listeners (`addEventListener(..., { passive: true })`)
- Debounce input updates if battery critical

## Configuration & Tuning

**Default mobile settings:** @file client/src/scripts/console/variables.ts

```typescript
const defaultBinds = {
  mobile: {
    movement_forward: "W",     // Unbound on mobile (joystick instead)
    movement_left: "A",
    movement_backward: "S",
    movement_right: "D",
    // ... other keys ignored on mobile
  }
};
```

**Joystick appearance:** @file client/src/scripts/managers/inputManager.ts:290–320

```typescript
const joystickConfig = {
  zone: "#game-container",
  mode: "static",
  position: { left: "50px", bottom: "50px" },  // Bottom-left
  color: "rgba(255,255,255,0.5)",
  size: 100,
  multitouch: false,
  maxNumberOfNipples: 2  // Two joysticks (movement + aiming)
};
```

## Related Documents
- **Tier 2:** [Input Management](../README.md) — subsystem overview
- **Tier 2:** [Camera Management](../../camera-management/README.md) — input affects camera
- **Tier 3:** [Desktop Input](../modules/desktop-input.md) (when documented) — keyboard/mouse
- **Tier 1:** [Development Guide](../../../development.md) — testing mobile input
- **Patterns:** [../patterns.md](../patterns.md) — subsystem patterns
