# Input Management

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/input-management/modules/ -->
<!-- @source: client/src/scripts/managers/inputManager.ts -->

## Purpose

Maps hardware input (keyboard, mouse, touch, gyro, accelerometer) to game actions; buffers input state and queues `InputPacket`s to server once per client frame. Handles key binding customization and mobile-specific input modes (joysticks, gyro aiming).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/managers/inputManager.ts` | `InputManager` singleton — event listeners, state buffering, packet serialization, mobile joystick/gyro setup |
| `common/src/packets/inputPacket.ts` | `InputPacket` protocol definition — movement bitmask, actions array, turning, mobile data |
| `client/src/scripts/game.ts` | `Game.tick()` → calls `InputManager.update()` each frame |
| `common/src/constants.ts` | `InputActions` enum (EquipItem, DropWeapon, etc.) and `GameConstants.player.maxMouseDist` |
| `client/src/scripts/console/variables.ts` | `defaultBinds` — default key-to-action mappings |

## Architecture

The input pipeline is **event-driven buffering + polling-based sending**:

```
Hardware Input (keyboard, mouse, touch, gyro, accel.)
  ↓
InputManager event listeners (window.keydown/keyup, gameContainer.pointerdown/up/move, wheel)
  ↓
Update internal state:
  - pressed keys (movement flags: up/left/down/right)
  - mouse position / rotation angle
  - queued actions (EquipItem, DropWeapon, UseItem, etc.)
  - attacking flag
  ↓
InputManager.update() called every frame from Game.tick():
  - Assembles InputData from current state
  - Detects if state changed or timer expired (100 ms)
  - If changed OR timer expired: serialize + send InputPacket to server
  - Clear actions array (consumable per-packet)
  ↓
Server receives InputPacket in game.tick():
  - Player.handleInput(packet) parses movement bitmask
  - Player.secondUpdate() applies movement/attacking/item actions
  ↓
UpdatePacket sent to all clients:
  - Client receives new player position / state
  - PixiJS renders new frame
```

**Key insight:** Input is **not sent every frame**. It's sent when state changes or the 100 ms timer expires. This reduces bandwidth while maintaining feel for interactions (movement changes are visible within 25 ms server-side, but may queue if bundled with other actions).

## Input Categories

### Keyboard Input

Input is keybind-agnostic: the `InputMapper` system is bidirectional. Here are **default binds**:

| Action | Default Keys | Effect |
|--------|--------------|--------|
| Movement up | W, ↑ | Sets `movement.up` flag |
| Movement left | A, ← | Sets `movement.left` flag |
| Movement down | S, ↓ | Sets `movement.down` flag |
| Movement right | D, → | Sets `movement.right` flag |
| Attack | Mouse 0 (LMB) | Sets `attacking` flag |
| Reload | R | Queues `InputActions.Reload` |
| Interact | F | Queues `InputActions.Interact` |
| Use consumable (specific) | 6-9 | Queues `InputActions.UseItem` with item |
| Equip weapon slot 0 | 1 | Queues `InputActions.EquipItem` with slot=0 |
| Equip weapon slot 1 | 2 | Queues `InputActions.EquipItem` with slot=1 |
| Equip weapon slot 2 | 3, E | Queues `InputActions.EquipItem` with slot=2 |
| Equip throwable | 4 | Queues `InputActions.EquipOtherWeapon` |
| Equip last item | Q | Queues `InputActions.EquipLastItem` |
| Equip other weapon | Space | Queues `InputActions.EquipOtherWeapon` |
| Swap gun slots | T | Queues `InputActions.SwapGunSlots` |
| Cycle items (scroll) | Mouse Wheel Up/Down | Spawns one or more `InputActions.EquipItem` |
| Toggle slot lock | L | Queues `InputActions.ToggleSlotLock` with slot |
| Toggle map | G, M | Queues `InputActions.ViewMap` (state) |
| Map ping wheel | C | Spawns `InputActions.MapPing` |
| Explode C4 | Z | Queues `InputActions.ExplodeC4` |
| Cancel action | X | Queues `InputActions.Cancel` |

**Keybinds are fully customizable** via the in-game settings UI, backed by the `InputMapper` class.

### Mouse Input

| Input | Handler | Action |
|-------|---------|--------|
| Move (desktop) | `pointermove` event | Calculates rotation angle; if not game-over, updates `InputManager.rotation` and `distanceToMouse` |
| Move (mobile) | Disabled (uses joystick instead) | N/A |
| Left click (Mouse 0) | `pointerdown/up` events | Sets/clears `attacking` flag |
| Right click (Mouse 2) | Can be bound to actions (e.g., emote wheel) | Handled like any other input |
| Wheel up/down | `wheel` event + 50 ms debounce | Calls action bound to `cycle_items 1` or `-1` |

**Responsive rotation:** Mouse movement continuously updates player rotation (client-side) via `cv_responsive_rotation` CVar. This feels smooth even at 40 TPS server tick rate, because it's decoupled from server ticks.

### Touch Input (Mobile)

Mobile input uses **nipplejs virtual joysticks**. If `isMobile` is true OR `FORCE_MOBILE` is set:

- **Left joystick** (configurable left or right): Movement
  - Angle → sets `movementAngle`
  - Distance > threshold → sets `movement.moving` flag
  - Swappable with right joystick via `mb_switch_joysticks` CVar

- **Right joystick** (configurable): Aiming + attacking
  - Angle → sets `rotation`, enables `turning` flag
  - Distance > threshold → sets `attacking` flag
  - Special handling for shoot-on-release weapons (throwables, some guns)

**Joystick customization** (CVars):
- `mb_joystick_size` — pixel radius
- `mb_joystick_lock` — fixed position or dynamic
- `mb_joystick_transparency` — alpha 0–1
- `mb_left_joystick_color` — hex color
- `mb_right_joystick_color` — hex color
- `mb_switch_joysticks` — swap left/right roles

**Gyro aiming** (if `mb_gyro_angle > 0`):
- Device orientation (beta angle) can trigger `cycle_items` up/down for scope cycling

**Shake to reload** (if `mb_shake_to_reload` enabled):
- Accelerometer detects rapid acceleration changes
  - Counts shakes: when count > `mb_shake_count_threshold` → fires `reload` command
  - Resets on timeout (`mb_shake_delay`)
  - Force threshold: `mb_shake_force` (m/s²)

## InputPacket Structure

All fields in each packet:

| Field | Type | Sent Every Tick? | Packed As | Effect |
|-------|------|-----------------|-----------|--------|
| `pingSeq` | uint8 | Yes | uint8 (MSB = ping-only flag) | Used for network latency estimation; MSB=128 means "skip rest of packet" |
| `movement.up` | boolean | Yes | 1 bit in boolean group | Forward movement |
| `movement.down` | boolean | Yes | 1 bit | Backward movement |
| `movement.left` | boolean | Yes | 1 bit | Leftward movement |
| `movement.right` | boolean | Yes | 1 bit | Rightward movement |
| `attacking` | boolean | Yes | 1 bit | Fire weapon / activate throwable |
| `turning` | boolean | When mouse/joystick moves | 1 bit | Signals `rotation` + `distanceToMouse` present |
| `rotation` (if `turning`) | u16 → angle | When mouse/joystick moves | rotation2 (16-bit angle) | Aim direction (radians, 0–2π) |
| `distanceToMouse` (if `turning`) | float 0–256 | When mouse/joystick moves | quantized float (0–maxMouseDist, 2 decimals) | Distance from player to aim point |
| `isMobile` | boolean | Yes | 1 bit | Mobile vs desktop mode |
| `mobile.moving` (if `isMobile`) | boolean | When joystick moves | 1 bit | Mobile movement active |
| `mobile.angle` (if `isMobile`) | u16 → angle | When joystick moves | rotation2 | Mobile movement direction |
| `actions` | array of InputAction | Variable (max 7) | array of action records | Queued player actions this frame |

**Compression strategy:** Movement bitmask is 4 bits, boolean group is 8 bits per byte, angles use `rotation2()` quantization, floats use bounds-based compression (see `SuroiByteStream`).

**Send optimization:** `InputPacket` is only sent when:
1. `actions.length > 0` (always send if there are actions), OR
2. Any movement/attacking/turning/isMobile state changed since last packet, OR
3. 100 ms timer expires (guarantee server sees input even if buffered)

This is implemented in `InputManager.update()` via the `areDifferent()` function.

## InputManager API

### Properties

| Property | Type | Purpose |
|----------|------|---------|
| `movement` | `{up, down, left, right, moving}` | Keyboard movement flags |
| `movementAngle` | number | Mobile joystick movement angle (radians) |
| `rotation` | number | Current aim angle (set by mouse/joystick) |
| `mousePosition` | `Vector` | Current mouse screen position (client coords) |
| `gameMousePosition` | `Vector` | Current mouse aim position (game world coords) |
| `distanceToMouse` | number | Distance from player to aim point (used for aiming distance calculations) |
| `attacking` | boolean | Whether attacking this frame |
| `turning` | boolean | Whether aiming direction changed this frame |
| `isMobile` | boolean | Mobile mode (set during `init()`) |
| `binds` | `InputMapper` | Bidirectional input ↔ action mapping |
| `actions` | `InputAction[]` | Queued actions for this frame (max 7 per packet) |

### Main Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `getInstance()` | `static getInstance(): InputManager` | Singleton accessor |
| `init()` | `init(): void` | Set up event listeners, joysticks, gyro; called once by `Game.setup()` |
| `update()` | `update(): void` | Called each frame from `Game.tick()`; assembles + sends `InputPacket` if state changed or timer expired |
| `addAction()` | `addAction(action: InputAction \| SimpleInputActions): void` | Queue an action for next `InputPacket` (max 7 per packet; drops oldest if full) |
| `getMousePos()` | `getMousePos(): Vector` | Get current game-world mouse aim position |
| `cycleScope()` | `cycleScope(offset: number): void` | Cycle through available scopes; queues `InputActions.UseItem` if changed |
| `cycleThrowable()` | `cycleThrowable(offset: number): void` | Cycle through available throwables; queues `InputActions.UseItem` if changed |

### Key Internal Methods

| Method | Purpose |
|--------|---------|
| `handleInputEvent(down: boolean, event: KeyboardEvent \| MouseEvent \| WheelEvent)` | Raw event handler; filters for modifier keys, trusted events, page focus; calls `fireAllEventsAtKey()` |
| `fireAllEventsAtKey(input: string, down: boolean)` | Looks up actions bound to input; invokes them (with up/down variants for invertible actions) |
| `getKeyFromInputEvent(event)` | Converts raw event to input string (e.g., "W", "Mouse0", "MWheelUp") |
| `generateBindsConfigScreen()` | Renders keybinds UI in settings panel |

## Data Flow Examples

### Example 1: Player moves right

```
1. User presses D key
   → window.keydown event fires
   → handleInputEvent(true, event) called
   → fireAllEventsAtKey("D", true) called
   → finds action "+right" bound to "D"
   → invokes query "+right" via GameConsole
   → sets movement.right = true

2. Game.tick() fires next frame
   → InputManager.update() called
   → builds InputData: {movement: {right: true}, ...}
   → areDifferent() returns true (movement changed)
   → InputPacket.create() serializes to binary
   → sendPacket(packet) sends to server

3. Server receives InputPacket
   → Player.handleInput(packet) parses movement bitmask
   → sets player.movement.right = true
   → Player.update() recalculates velocity += rightDirection
   → next tick: player position updated, sent in UpdatePacket

4. Client receives UpdatePacket
   → Player.updateFromData() updates position
   → PixiJS renders new frame with player moved right
```

### Example 2: Player uses consumable key 8 (medikit)

```
1. User presses 8 key
   → handleInputEvent(true) called
   → fireAllEventsAtKey("8", true) called
   → finds action "use_consumable medikit" bound
   → invokes query via GameConsole
   → GameConsole service finds medikit definition
   → InputManager.addAction({type: InputActions.UseItem, item: medikit})

2. InputManager.update() in next game frame
   → builds InputData with actions: [InputActions.UseItem + medikit]
   → areDifferent() returns true (actions.length > 0)
   → serializes action in InputPacket
   → sends to server

3. Server receives InputPacket
   → Player.handleActions(packet.actions) called
   → processes InputActions.UseItem → Item.use() → player.health += medikit.healing

4. UpdatePacket sent to all clients
   → player health bar updated on all screens
```

### Example 3: Mouse move (aim direction)

```
1. User moves mouse
   → gameContainer.pointermove event fires
   → InputManager calculates rotation from center of screen
   → sets rotation, turning = true
   → updates distanceToMouse

2. Game frame 1:
   → InputManager.update()
   → builds InputData with turning: true, rotation, distanceToMouse
   → areDifferent() returns true
   → sends InputPacket

3. Game frames 2–N (no mouse move):
   → InputManager.update()
   → areDifferent() returns false (no state change)
   → BUT every 100 ms, timer expires
   → resends InputPacket to ensure server sees rotation

4. Client-side rotation is smooth because:
   → Game.activePlayer.container.rotation = InputManager.rotation every frame
   → decoupled from 40 TPS server tick
   → feels instant even if server tick is delayed
```

## Key Patterns

### Pattern: State Buffering + Polling Optimization

**When to use:** Any input system where you want to buffer state and send data in batches.

**Implementation:**
- Event listeners update state (keys pressed, mouse position)
- Polling function (`update()`) runs each frame
- Only serialize + send when state changes OR time threshold expires
- Clear per-packet data (actions) after sending

**Benefit:** Reduces bandwidth (especially over 100 ms timeout consolidation) while keeping input responsive.

**Used in suroi:**
- `movement`, `attacking` → always send if changed or 100 ms elapsed
- `actions` → clear each frame after sending (consumable per-packet)
- `turning` → only send if mouse/joystick moved

### Pattern: Bidirectional Input Mapping

**When to use:** Custom keybind systems where users expect "rebind any input to any action."

**Implementation:**
- `InputMapper.addActionsToInput()` — input string → set of actions
- `InputMapper.addInputsToAction()` — action string → set of inputs
- Keep both maps in sync when adding/removing binds
- Lookup: input → actions; or action → inputs

**Benefit:** Allows context-sensitive queries: "what key is bound to 'reload'?" or "what actions fire on 'R' key press?"

**Used in suroi:**
```typescript
// User presses D key
actions = inputMapper.getActionsBoundToInput("D")  // ["+right"]
// For each action, invoke it or its opposite
GameConsole.handleQuery("+right")  // movement.right = true
GameConsole.handleQuery("-right")  // movement.right = false (on key release)
```

### Pattern: Invertible Actions

**When to use:** Actions that have explicit on/off states (movement, holding button).

**Implementation:**
- Action string prefixed with `+` (on) or `-` (off)
- On keydown: execute `+action`
- On keyup: if action starts with `+`, execute `-action` instead
- For buttons without `-` prefix, do nothing on release

**Benefit:** Decouples bind definition from explicit state management. No need for a separate "stopMoving" action.

**Used in suroi:**
```typescript
bind "+up" ["W", "ArrowUp"]      // keydown → +up, keyup → -up
bind "reload" ["R"]             // keydown → reload, keyup → (no-op)
```

### Pattern: Shoot-On-Release for Throwables & Special Guns

**When to use:** Weapons where charging/aiming must happen before release.

**Implementation (mobile):**
- On joystick move, detect weapon type
- If gun with `shootOnRelease` or type is Throwable
- Store angle in `shootOnReleaseAngle`
- Set `shooting = true` only on joystick end (release)
- Clear via `resetAttacking` flag next frame

**Used in suroi:**
```typescript
// Mobile: user pulls right joystick (aiming)
if (def.defType === DefinitionType.Throwable && this.attacking) {
    shootOnRelease = true
    this.shootOnReleaseAngle = this.rotation  // store aim angle
} else {
    this.attacking = attacking  // regular gun fires on distance > threshold
}

// On joystick release:
this.attacking = shootOnRelease  // fire throwable with stored angle
this.resetAttacking = true  // clear next frame
```

## Dependencies

### Depends on:
- [Networking](../networking/) — `InputPacket` serialization via `SuroiByteStream`
- [Object Definitions](../object-definitions/) — `Loots`, `Emotes`, `MapPings` definitions for action payloads
- Config/Console system — `GameConsole`, `InputMapper` for keybind customization and CVars (mobile settings, cvs)

### Depended on by:
- [Game Loop](../game-loop/) — `Game.tick()` calls `InputManager.update()` each frame
- UI/Interaction system — Code calls `InputManager.addAction()` to queue manual interactions (loot, interact)
- Camera/Player rendering — `InputManager.rotation` used for player/camera facing direction

## Known Issues & Gotchas

### 1. **Hard 100 ms Send Rate Floor**
Even if nothing changes, `InputPacket` is resent every 100 ms. This can waste bandwidth if player is idle. Could optimize by deferring idle packets, but trade-off is server-side input lag detection (anti-cheat).

**Location:** `client/src/scripts/managers/inputManager.ts` line ~290

### 2. **Wheel Event No "Stop" Event**
Browser wheel events fire repeatedly while scrolling, but no "end" event. InputManager manually debounces with a 50 ms setTimeout to detect scroll stop.

**Gotcha:** Rapid scrolling within 50 ms may trigger multiple action cycles in one packet (max 7 actions).

**Location:** `client/src/scripts/managers/inputManager.ts` line ~600 (`this.mWheelStopTimer`)

### 3. **Action Queue Has Hard Limit (7 per packet)**
`addAction()` silently drops actions if queue already has 7. If user spams number keys 1-7, only first 7 will fire; rest dropped.

**Gotcha:** No warning to user. Test with rapid-fire consumables or multi-key sequences.

**Location:** `client/src/scripts/managers/inputManager.ts` line ~210

### 4. **Mobile Mode Detection Can Be Overridden**
`isMobile` is set during `init()` based on `isMobile.any` OR `FORCE_MOBILE` CVar. Once set, it's stuck for the session.

**Gotcha:** Plugging in a USB mouse after starting on mobile won't switch to desktop input mode.

### 5. **Responsive Rotation Decoupled from Server**
Client-side player rotation (via `cv_responsive_rotation`) updates every frame, but server only sees aim angle in InputPacket. If client sends outdated packet, server may aim with old angle.

**Effect:** Tiny "jitter" if client-server timestamp desync exists. Usually unnoticeable, but can manifest in high-latency scenarios.

### 6. **Gyro & Accelerometer Permissions**
On newer mobile OS (iOS 13+, Android 12+), gyro/accel access requires explicit user permission. If denied, gyro features silently fail.

**Gotcha:** Test on real devices. Emulators may not trigger permission flow.

**Location:** `client/src/scripts/managers/inputManager.ts` lines ~500–520

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — System components and dependencies
- [Data Model](../../datamodel.md) — Understand how `InputData` fits in overall data flow

### Tier 2 — Subsystems
- [Networking](../networking/) — `InputPacket` serialization and transmission; `PacketStream` multiplexing
- [Game Loop](../game-loop/) — Server-side input processing; `Player.handleInput()` called by game tick
- [Object Definitions](../object-definitions/) — `Loots`, `Emotes` registry for action payloads
- [Camera Management](../camera-management/) (if documented) — Uses `InputManager.rotation` for player facing
- [Network Graph / UI](../ui/) (if documented) — Displays input packet bandwidth stats

### Tier 3 — Modules
- *(None documented yet; would include: mobile joystick implementation, gyro handler, action queue, InputMapper internals)*

### Patterns
- [Input Management Patterns](patterns.md) — State buffering, invertible actions, shoot-on-release, bidirectional mapping

## References

- **Browser APIs:** Keyboard Symbols (`KeyboardEvent.key` vs `code`), Pointer Events, Wheel Events, DeviceOrientation, DeviceMotion
- **nipplejs:** Virtual joystick library (https://github.com/yoannimolive/nipplejs)
- **SuroiByteStream:** Binary serialization (see `common/src/utils/suroiByteStream.ts`)
- **GameConsole & CVars:** Console variable system for configuration (see `client/src/scripts/console/variables.ts`)
