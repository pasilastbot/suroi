# Visual Effects Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/client-rendering/README.md -->
<!-- @source: client/src/scripts/game.ts, client/src/scripts/managers/gasManager.ts -->

## Purpose

Renders temporary visual overlays and screen effects: screen shake (bullet impact, explosions), gas damage vignette (red tint at screen edge), color overlays (gas/poison), blur/distortion effects, and effect stacking behavior.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [client/src/scripts/game.ts](../../../../../client/src/scripts/game.ts) | Screen shake, color overlay application, vignette rendering, effect cleanup | High |
| [client/src/scripts/managers/gasManager.ts](../../../../../client/src/scripts/managers/gasManager.ts) | Gas/poison visual effect (green tint), damage vignette state | High |
| [client/src/scripts/game.ts](../../../../../client/src/scripts/game.ts) (Tween system) | Animation tweens for effect fade-in/fade-out | Medium |

## Business Rules

- **Screen Shake**: Triggered by bullet impacts (own weapon recoil), nearby explosions, vehicle collisions
  - Amplitude: Decreases over time (easing function applied)
  - Duration: Configurable per event (e.g., explosion larger shake than single bullet)
  - Offset: Applied to camera position, not to individual sprites
- **Damage Vignette**: Red/orange tint at screen edges when player taking damage
  - Triggered by: Damage event (bullet hit, melee hit, explosion, gas damage)
  - Intensity: Scales with damage amount (larger damage = darker vignette)
  - Fade: Vignette fades out over time; multiple damages stack (resets timer)
- **Gas Effect**: Green/poison tint when in gas zone, blue tint for other hazards
  - Opacity: Scales with gas stage (early stages faint, late stages dark)
  - Color: Configurable per mode (defined in mode definition)
- **Color Overlay**: Full-screen color tint applied via `ColorMatrixFilter` on root stage
  - Stack: Multiple overlays compose (gas + damage + effect = multi-layer tint)
  - Performance: Single filter applied to `pixi.stage`, not individual sprites
- **Blur/Distortion** (if implemented): Gaussian blur when underwater or in special zones
  - Applied via `BlurFilter` on root stage or game container
  - Temporarily hidden if framerate drops below threshold
- **Effect Cleanup**: Overlays fade out when trigger ends (no lingering); instances reused for next effect

## Data Lineage

```
Game Server sends DamagePacket or GasPacket
  ↓
Game.onUpdate() processes packet
  ↓
Determine effect type (damage vignette, gas tint, shake, etc.)
  ↓
Queue effect in game.screenEffects or immediate application:
  
IMMEDIATE EFFECT (Screen Shake):
  1. Calculate shake amplitude, direction, duration
  2. Apply offset to CameraManager.container position
  3. Tween amplitude → 0 over duration
  4. On complete: reset camera position

TINTED OVERLAY (Gas/Damage):
  1. Create/update ColorMatrixFilter parameters
  2. Set color (red for damage, green for gas)
  3. Set opacity/intensity
  4. Apply filter to pixi.stage
  5. Tween opacity → 0 over fade duration
  6. On complete: remove filter

VIGNETTE (Red Edge):
  1. Render vignette graphics (radial gradient from transparent center → red edge)
  2. Apply to separate container above game layer
  3. Fade opacity over time
  4. Remove when opacity → 0
```

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| `shakeIntensity` | Base amplitude for screen shake | Damage amount or explosion definition |
| `shakeDuration` | How long shake lasts (milliseconds) | Server packet or hardcoded default |
| `damageVignette.maxIntensity` | Vignette opacity at max damage in one frame | `GameConstants` or mode definition |
| `damageVignette.fadeTime` | Time for vignette to fade out (milliseconds) | Hardcoded in effect handler |
| `gasColor` | Tint color applied in gas zone | Mode definition (`colors.gas`) |
| `gasOpacity` | Max opacity of gas tint | Scales with gas stage (0–1) |

## Complex Functions

### `Game.applyScreenShake(intensity, duration)` — @file client/src/scripts/game.ts
**Purpose:** Apply and tween screen shake effect to camera.

**Implicit behavior:**
- Calculates random offset direction from intensity
- Applies offset to `CameraManager.container.position`
- Creates tween that reduces offset over duration (uses easing function for natural decay)
- Stacks with existing shake (if another explosion occurs, intensities add)
- On complete: resets camera position to exact player position (no residual shake)

### `Game.applyDamageVignette(damageAmount)` — @file client/src/scripts/game.ts
**Purpose:** Render red tint at screen edges proportional to damage taken.

**Implicit behavior:**
- Calculates vignette intensity from damage (clamped to [0, 1])
- Creates/updates vignette Graphics container with radial gradient
- If new damage arrives before vignette fades → intensity resets (refreshes fade timer)
- Vignette sprite positioned at camera center, follows screen movement
- Fades out over configurable time (e.g., 600ms)

### `GasManager.setGasEffect(stage, isInGas)` — @file client/src/scripts/managers/gasManager.ts
**Purpose:** Apply or remove gas tint from screen based on player's gas status.

**Implicit behavior:**
- Reads stage number from GasPacket (determines gas opacity)
- Creates ColorMatrixFilter with stage-based green/blue color
- Applies filter to `pixi.stage` (affects entire render)
- No fade: immediately applies/removes (sharp on entry to gas zone)
- Multiple clouds → only strongest tint rendered (not stacked)

### Effect Stacking Behavior — @file client/src/scripts/game.ts
**Purpose:** Ensure multiple simultaneous effects don't overwhelm screen.

**Implicit behavior:**
- **Priority**: Gas effect lowest, damage vignette medium, screen shake highest
- **Layering**: Screen shake (offset) + vignette (graphics) + color overlay (filter) rendered in order
- **Clipping**: Multiple damage hits don't increase vignette intensity beyond 1.0
- **Fade synchronization**: Each effect has independent fade timer; they don't interfere

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Client rendering subsystem overview
- **Tier 2:** [../../camera-management/README.md](../../camera-management/README.md) — Camera system (shake applied to viewport)
- **Tier 2:** [../../gas/README.md](../../gas/README.md) — Server-side gas logic (triggers gas effect)
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — Client/server data flow
