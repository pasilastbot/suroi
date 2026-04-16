# Client Rendering — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/client-rendering/README.md -->

## Pattern: Renderable Game Object (Pool-Managed)

**When to use:** Adding a new game object type that is server-tracked, sent in `UpdatePacket`, and needs persistent visual representation.

**Implementation:**

1. Add an entry to `ObjectCategory` in `common/src/constants.ts` and define the network serialization in `common/src/utils/objectsSerializations.ts`.
2. Create a class in `client/src/scripts/objects/` that extends `GameObject.derive(ObjectCategory.YourCategory)`.
3. Store visual sprites as `readonly` properties (typically `SuroiSprite` instances).
4. In the constructor:
   - Call `super(id)`.
   - Create all `SuroiSprite` instances and set their `zIndex`.
   - Add sprites to `this.container`.
   - Call `this.updateFromData(data, true)` to apply the initial server state.
5. Override `updateFromData(data: ObjectsNetData[ObjectCategory.YourCategory], isNew = false)`:
   - Check `data.full` for full-data-only fields (sent on creation and after entering view).
   - Apply position: `this.position = data.position` (triggers interpolation tracking).
   - Call `this.updateLayer()` to register with the correct `layerContainer`.
6. Override `destroy()` to stop sounds, kill tweens/timeouts, and call `super.destroy()`.
7. Register the class in `ObjectClassMapping` in `game.ts`:
   ```typescript
   const ObjectClassMapping = Object.freeze({
       // ...existing entries...
       [ObjectCategory.YourCategory]: YourClass,
   });
   ```

**Full vs Partial updates:**

`updateFromData` receives `isNew = true` on construction and whenever the server sends a full-dirty entry (e.g. object re-entering view distance). It receives `isNew = false` for partial-dirty entries. Use the pattern:

```typescript
override updateFromData(data: ObjectsNetData[ObjectCategory.Foo], isNew = false): void {
    if (data.full) {
        // Full data: initialise definition, create sprites, set static properties
        this.definition = data.full.definition;
        this.image.setFrame(this.definition.idString);
    }
    // Partial data: position and runtime state are always present
    this.position = data.position;
    this.rotation = data.rotation;
    this.updateLayer();
}
```

**Layer registration:**

After setting `this.layer`, call `this.updateLayer()` (inherited from `GameObject`). This calls `CameraManager.getContainer(this.layer).addChild(this.container)` — moving the container to the correct per-floor sub-container.

**Example files:**
- `@file client/src/scripts/objects/decal.ts` — simplest pooled object (position + definition only, no partial state)
- `@file client/src/scripts/objects/loot.ts` — full/partial split, multiple child sprites
- `@file client/src/scripts/objects/gameObject.ts` — base class with interpolation and layer management

---

## Pattern: SuroiSprite Usage

**When to use:** Any time you need a sprite in the game world (inside a `GameObject.container` or a particle).

**Implementation:**

```typescript
import { SuroiSprite, toPixiCoords } from "../utils/pixi";
import { PIXI_SCALE } from "../utils/constants";

// Create (sets anchor to (0.5, 0.5) automatically)
const sprite = new SuroiSprite("some_texture_name");

// Fluent configuration — all setters return `this`
sprite
    .setFrame("different_texture")   // swap texture
    .setPos(100, 200)                // pixels (already in canvas space)
    .setVPos(toPixiCoords(gamePos))  // convert from game units → pixels first
    .setScale(0.75)                  // uniform scale
    .setScale(1.5, 0.8)              // non-uniform scale
    .setRotation(Math.PI / 4)        // radians
    .setAngle(45)                    // degrees
    .setTint(0xff0000)               // red tint
    .setZIndex(ZIndexes.Players)
    .setAlpha(0.5);

// Add to a container
this.container.addChild(sprite);

// Swap texture at runtime
sprite.setFrame("updated_texture");

// Null-safe texture check: if frame is missing from Assets.cache,
// getTexture() falls back to "_missing_texture" and logs a warning.
const tex = SuroiSprite.getTexture("my_frame");
```

**Key rule:** `setPos` / `setVPos` expect **pixel** coordinates. Always call `toPixiCoords(pos)` when you have a game-unit `Vector`, or manually multiply by `PIXI_SCALE`.

**Example files:** `@file client/src/scripts/utils/pixi.ts`

---

## Pattern: Camera Follow (Active Player)

**When to use:** Keeping the viewport centered on the local player with movement smoothing.

**Implementation (from `game.ts` `render()`):**

```typescript
// Called each animation frame inside Game.render():
if (GameConsole.getBuiltInCVar("cv_movement_smoothing")) {
    CameraManager.position = this.activePlayer.container.position;
    // container.position is already in pixel space (set by updateContainerPosition())
}
```

`GameObject.updateContainerPosition()` (called via `updateInterpolation()`) lerps between `_oldPosition` and `position`, converting to pixels via `toPixiCoords`, and stores the result in `this.container.position`. Passing that pixel vector to `CameraManager.position` means the camera follows the interpolated, smoothed visual position — not the raw server position (which jumps once per tick).

`CameraManager.update()` then converts camera position to a container offset:
```typescript
const cameraPos = Vec.add(
    Vec.scale(position, this.container.scale.x),
    Vec(-this.width / 2, -this.height / 2)
);
this.container.position.set(-cameraPos.x, -cameraPos.y);
```

**Example files:** `@file client/src/scripts/managers/cameraManager.ts`

---

## Pattern: Screen Shake

**When to use:** Communicating impact — explosions, heavy damage, landing.

**Implementation:**

```typescript
import { CameraManager } from "../managers/cameraManager";

// duration in ms, intensity in game units
CameraManager.shake(300, 1.5);
```

Each frame while `shaking = true`, `CameraManager.update()` adds a random offset within a circle of radius `shakeIntensity` to the camera position. Shaking stops automatically after `shakeDuration` ms.

Respects the `cv_camera_shake_fx` CVar — if disabled, `shake()` is a no-op.

**Example files:** `@file client/src/scripts/managers/cameraManager.ts`

---

## Pattern: Spawning Particles

**When to use:** Visual feedback for hits, deaths, environment (footsteps, smoke, blood, ambient effects).

**Implementation:**

```typescript
import { ParticleManager, type ParticleOptions } from "../managers/particleManager";
import { ZIndexes, Layer } from "@common/constants";

// Single particle
ParticleManager.spawnParticle({
    frames: ["blood_particle_1", "blood_particle_2"],  // random selection
    position: hitPosition,
    speed: Vec(randomFloat(-5, 5), randomFloat(-5, 5)),
    lifetime: 800,                   // ms
    zIndex: ZIndexes.Players + 0.5,
    layer: Layer.Ground,
    scale: { start: 0.8, end: 0.3, ease: EaseFunctions.sineIn },
    alpha: { start: 1, end: 0 },
    rotation: randomRotation()
});

// Multiple particles with varied options
ParticleManager.spawnParticles(5, () => ({
    frames: "spark",
    position: explosionCenter,
    speed: randomVector(-20, 20, -20, 20),
    lifetime: 500,
    zIndex: ZIndexes.Explosions
}));

// Continuous emitter (e.g. smoke from a burning obstacle)
const emitter = ParticleManager.addEmitter({
    delay: 200,    // ms between spawns
    active: true,
    spawnOptions: () => ({
        frames: "smoke_particle",
        position: obstaclePosition,
        speed: Vec(0, -3),
        lifetime: 1500,
        zIndex: ZIndexes.ObstaclesLayer3
    })
});

// Stop the emitter later
emitter.destroy();
```

**Example files:** `@file client/src/scripts/managers/particleManager.ts`

---

## Pattern: Playing a Positional Sound

**When to use:** Any in-world audio — gunshots, explosions, footsteps, ambient sounds.

**Implementation:**

```typescript
import { SoundManager } from "../managers/soundManager";

// One-shot sound at a world position
SoundManager.play("gun_shot", {
    position: muzzlePosition,   // game-unit Vector
    falloff: 1,
    maxRange: 96,
    layer: this.layer,
    loop: false,
    dynamic: false,
    ambient: false
});

// Dynamic sound that updates as the camera moves (e.g. looping fire)
const fireSound = SoundManager.play("fire_loop", {
    position: this.position,
    falloff: 0.8,
    maxRange: 48,
    layer: this.layer,
    loop: true,
    dynamic: true,   // pan/volume recalculated each frame
    ambient: false
});

// Stop it later
fireSound?.stop();
```

Sounds on a layer different from the active player's layer are automatically muffled (see `SOUND_FILTER_FOR_LAYERS` in `constants.ts`).

**Example files:** `@file client/src/scripts/managers/soundManager.ts`

---

## Pattern: Tween Animation

**When to use:** Smooth property transitions — fade in/out, scale pop, position slide, zoom.

**Implementation:**

```typescript
import { Game } from "../game";
import { EaseFunctions } from "@common/utils/math";

// Animate a sprite's alpha from current to 0 over 300 ms
const tween = Game.addTween({
    target: this.image,
    to: { alpha: 0 },
    duration: 300,
    ease: EaseFunctions.sineIn,
    onComplete: () => {
        this.image.destroy();
    }
});

// Cancel before completion (e.g. object was destroyed early)
tween.kill();
```

`Game.addTween()` registers the tween in `Game.tweens`. All tweens are advanced in `Game.render()` each frame. Using `Game.addTween` (rather than a raw `Tween`) ensures the tween is cleaned up when the game ends.

**Within a `GameObject`**, use `this.addTimeout` / `this.timeouts` for cleanup — all registered timeouts are cancelled when `destroy()` is called.

**Example files:** `@file client/src/scripts/utils/tween.ts`

---

## Pattern: Layer-Aware Rendering

**When to use:** Any object that can exist in the basement, ground, or upstairs floors.

**Implementation:**

After updating `this.layer` in `updateFromData`, call:

```typescript
this.updateLayer();
```

`updateLayer()` (provided by `GameObject`) moves `this.container` to the container returned by `CameraManager.getContainer(this.layer)`. The camera manager computes the appropriate sub-container index from the local player's current layer via `getLayerContainerIndex(objectLayer, Game.layer)`.

During a layer transition, `getContainer` may return `tempLayerContainer` to prevent visual flickering as the destination sub-container fades in (see the "Layer container zIndex hack" in the README gotchas).

**Building ceilings** use a separate `ceilingContainer` that is faded out when the local player enters the building, so the interior becomes visible.

**Example files:** `@file client/src/scripts/objects/building.ts`, `@file client/src/scripts/managers/cameraManager.ts`
