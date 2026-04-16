# Sound Management

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/sound-management/modules/ -->
<!-- @source: client/src/scripts/managers/soundManager.ts -->

## Purpose

Sound Management handles all audio playback in the game client: weapon fire, footsteps, impacts, ambience, and UI feedback. Built on [@pixi/sound](https://www.npmjs.com/package/@pixi/sound) v6, the system provides spatial audio (volume & panning based on object position), layer-aware attenuation, dynamic updates tied to camera movement, and efficient audio spritesheet bundling via Vite.

## Key Files & Entry Points

| File | Purpose | Responsibility |
|------|---------|-----------------|
| [client/src/scripts/managers/soundManager.ts](../../../../client/src/scripts/managers/soundManager.ts) | `SoundManager` class, `GameSound` class | Audio playback, volume control, spatial audio, instance pooling |
| [client/src/scripts/game.ts](../../../../client/src/scripts/game.ts) | `Game` class owns SoundManager | Initialization, update loop, environment ambience lifecycle |
| [client/src/scripts/objects/gameObject.ts](../../../../client/src/scripts/objects/gameObject.ts) | `GameObject.playSound()` helper | Wrapper method for spatial sound playback from game objects |
| [client/package.json](../../../../client/package.json) | `@pixi/sound: ^6.0.1` | Audio library dependency |
| [client/public/audio/](../../../../client/public/audio/) | Audio asset directory | All game audio: `.mp3` files organized by category |

## Architecture

### Audio Loading

Audio assets are bundled via **Vite audio spritesheet plugin** during build:

```
client/vite.config.ts (Vite plugin: audio-spritesheet-plugin)
  ↓
Concatenates all mode-specific audio files (shared/ + normal/ + fall/ + etc.)
  ↓
Single binary audio file + spritesheet manifest (offset map)
  ↓
At runtime: SoundManager.init() loads spritesheet, registers sound IDs
```

The spritesheet approach enables:
- Single HTTP request for all preloaded audio
- O(1) lookup by sound ID (maps to offset in binary buffer)
- Fallback to lazy-loaded individual files for "no-preload" sounds

### Sound Playback Flow

```
Game object calls playSound(name, options)
  ↓
GameSound constructor → PixiSound.sound.play(id, config)
  ↓
Audio buffer loads (from spritesheet or individual file)
  ↓
Web Audio API: source node created, stereo filter + pan applied
  ↓
If dynamic=true or ambient=true → added to SoundManager.updatableSounds
  ↓
Game.update() loop → SoundManager.update() → GameSound.update()
  ↓
Computes falloff distance, applies stereo pan, updates volume
  ↓
Audio plays at computed volume & pan angle
```

### Volume & Distance Model

Volume is computed from **falloff** and **maxRange** parameters:

```typescript
const distance = Vec.len(Vec.sub(cameraPosition, soundPosition))
const attenuated = (1 - clamp(distance / maxRange, 0, 1)) ^ (1 + falloff * 2)
finalVolume = attenuated * managerVolume * objectVolume * layerMultiplier
```

- **falloff**: Exponent controlling attenuation curve (0 = linear, 1 = quadratic, default=1)
- **maxRange**: Distance at which volume reaches 0 (default=256 units)
- Stereo pan is X-component of diff vector, normalized to [-1, 1]

### Layer Attenuation

Sounds attenuate when crossing layer boundaries (underground/water layers):

| Layer Delta | Attenuation |
|-------------|-------------|
| Same layer | 1.0× (no attenuation) |
| Adjacent layer | 0.5× |
| 2+ layers away | 0.15× |

**Ambient sounds only decrease underground** (e.g., music fades when player dives underwater). Non-ambient sounds attenuate bidirectionally (both going underground and surfacing).

Layer transitions are smoothed via **tween animation** (500ms duration) to avoid abrupt volume jumps.

## SoundManager API

### Class: `SoundManagerClass`

#### Static Properties

| Property | Type | Purpose |
|----------|------|---------|
| `sfxVolume` | `number` 0.0–1.0 | Master SFX volume (weapon fire, impacts, etc.) |
| `ambienceVolume` | `number` 0.0–1.0 | Master ambience volume (music, wind, water) |
| `position` | `Vector` | Camera position for spatial audio calculations |
| `updatableSounds` | `Set<GameSound>` | Sounds with `dynamic=true` or `ambient=true` requiring per-frame updates |

#### Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `init()` | `async init(): Promise<void>` | Load audio spritesheet, register all sound IDs. Called once at game start. |
| `addSound()` | `addSound(id: string, opts: Partial<PixiSound.Options>): void` | Register a single sound (used internally during `init()`). |
| `play()` | `play(name: string, options?: Partial<SoundOptions>): GameSound` | Create & play a sound; return `GameSound` handle for later control. |
| `has()` | `has(name: string): boolean` | Check if sound ID is registered. |
| `update()` | `update(): void` | Update all dynamic/ambient sounds (called every game frame). Removes ended sounds. |
| `stopAll()` | `stopAll(): void` | Stop all playing sounds and clear updatable sounds set. Used on game end. |

### Interface: `SoundOptions`

```typescript
interface SoundOptions {
    position?: Vector          // World position for spatial audio (omitted = non-spatial)
    falloff: number            // Distance attenuation exponent (0 = linear, 1 = quadratic)
    layer: Layer | number      // Layer the sound originates from (0=ground, 1=water, 2=underground)
    maxRange: number           // Distance at which volume becomes 0
    loop: boolean              // Loop playback when sound ends
    speed?: number             // Playback speed multiplier (default 1.0)
    dynamic: boolean           // Update volume & pan every frame when camera moves
    ambient: boolean           // Layer attenuation affects the sound
    onEnd?: () => void         // Callback when sound finishes (before stop or loop)
}
```

**Default values** when calling `SoundManager.play()`:
```typescript
{
    falloff: 1,
    maxRange: 256,
    dynamic: false,
    ambient: false,
    layer: Game.layer,
    loop: false
}
```

### Class: `GameSound`

Represents a single playing sound instance. Created by `SoundManager.play()`, controlled via returned handle.

#### Properties

| Property | Type | Purpose |
|----------|------|---------|
| `id` | `string` | Sound identifier (e.g., "weapon_ak47_fire") |
| `name` | `string` | Display name |
| `position` | `Vector \| undefined` | World position; `undefined` = non-spatial sound |
| `volume` | `number` 0.0–1.0 | Base volume for this sound instance (multiplied into final volume) |
| `instance` | `PixiSound.IMediaInstance \| undefined` | @pixi/sound instance; `undefined` until audio loads |
| `ended` | `boolean` | Set when sound finishes or was stopped |

#### Methods

| Method | Purpose |
|--------|---------|
| `update(): void` | Recalculate volume & pan based on current camera position. Called by `SoundManager.update()`. |
| `updateLayer(initial?: boolean): void` | Recalculate layer attenuation. Animated via tween (500ms) if not initial. |
| `setPaused(paused: boolean): void` | Pause or resume playback. |
| `stop(): void` | Stop playback and mark as ended. (No-op if already ended to avoid @pixi/sound bug.) |

#### Events

When `instance` is initialized, two listeners are attached:
- `instance.on("end", () => {...})` — Fires when sound completes or loops
- `instance.on("stop", () => {...})` — Fires when `stop()` is called

## Sound Categories

### Weapon Fire

Sound IDs follow the pattern `weapon_<type>_<action>`:

```
weapon_ak47_fire           Automatic rifle
weapon_scout_fire          Sniper rifle
weapon_m16_fire            Burst rifle
weapon_shotgun_fire        Shotgun (loud)
weapon_deagle_fire         Pistol
weapon_mosin_fire          Heavy rifle
weapon_sword_swing         Melee swing
weapon_grenade_pull        Grenade pin
weapon_heal_item           Medical item use
```

### Impact & Destruction

```
detection                  Player detected by USAS/Radar
bleed                      Bleeding effect
swings/<surface>           Melee impact on wood/rock/sand
hits/<surface>             Bullet impact on wood/rock/metal/sand
explosions/explosion_<n>   Explosion blast
```

### Player Feedback

```
bush_rustle_1, bush_rustle_2       Walk through bush
footsteps/<surface>                Step sound (varies by layer/floor type)
death_marker                       Death marker placed
join_notification                  Player joins (if unfocused)
kill_leader_assigned               Self promoted to kill leader
kill_leader_dead                   Kill leader eliminated
```

### Airdrop & Environment

```
airdrop_landing            Airdrop descending
airdrop_land               Airdrop lands
airdrop_land_water         Airdrop lands in water
flare                      Flare activates
generator_starting         Generator power-up
generator_running          Generator running (looping)
River/Ocean ambience       Environmental ambience
```

### UI Sounds

```
button_press               Menu click
puzzle_solved              Puzzle/tutorial completion
puzzle_error               Puzzle error
speaker_start              Loudspeaker notification
vault_door_open            Vault opens
door/door_close            Door shuts
door/door_open             Door opens
emote                      Player emote action
```

### Music

Menu music tracks by mode:

```
menu_music                 Default menu track
menu_music_fall            Fall event variant
menu_music_halloween       Halloween event
menu_music_hunted          Hunted mode
menu_music_winter          Winter event
```

Loaded in `client/public/audio/music/` as separate `.mp3` files (not in spritesheet).

## Usage Patterns

### Playing a Sound from a Game Object

```typescript
// From any GameObject subclass (Player, Obstacle, Projectile, etc.)
this.playSound("weapon_ak47_fire");  // Spatial sound at this.position

this.playSound("weapon_ak47_fire", {
    loop: false,
    speed: 0.9,           // Slightly pitch-shifted
    falloff: 0.5          // Less distance attenuation
});

// onEnd callback fired when sound completes
this.playSound("ambience_wind", {
    ambient: true,
    loop: true,
    onEnd: () => { console.log("sound ended"); }
});
```

### Playing a Global (Non-Spatial) Sound

```typescript
// From Game or anywhere else
SoundManager.play("join_notification");  // No position = no spatial audio

SoundManager.play("detection", {
    falloff: 2,           // Steeper attenuation
    maxRange: 512         // Farther audible range
});
```

### Managing Looping Sounds

```typescript
// Store handle to stop later
const ambience = SoundManager.play("generator_running", { loop: true, ambient: true });

// Stop when generator destroyed
ambience.stop();
```

### Pausing Sounds

```typescript
const sound = SoundManager.play("weapon_fire");
sound.instance?.paused = true;    // Pause
sound.instance?.paused = false;   // Resume
```

Or via the helper:

```typescript
sound.setPaused(true);
sound.setPaused(false);
```

## Performance Optimization

### Audio Spritesheet Bundling

All preload sounds are concatenated into a single binary file + manifest:
- **Pros:** Single HTTP request, O(1) ID lookup, fast initial load
- **Cons:** All sounds loaded into memory even if not used yet

**No-preload sounds** (rare, long-duration sounds) are loaded individually from `client/public/audio/game/no-preload/`:
- Avoids bloating the main spritesheet
- Loaded on-demand with slight latency

### Instance Pooling

`SoundManager.updatableSounds` only tracks dynamic/ambient sounds. Non-spatial, non-looping sounds are "fire-and-forget" — no per-frame overhead.

### Lazy Initialization

Audio context only initializes when `SoundManager.init()` is called (typically at game start), not on module load. This respects browser autoplay restrictions.

## Gotchas & Known Issues

### Audio Context Initialization

Web Audio API requires a **user gesture** (click, touch, key press) to unlock playback. @pixi/sound handles this, but sounds may play silently on first load without interaction.

### Mobile Audio Muting

**iOS limitation:** All non-musical audio is muted until the user physically unmutes via the device mute switch. Music plays unless muted. This is a platform restriction, not a bug.

### Buffer Loading Latency

The first time a sound is played, its audio buffer may still be loading. @pixi/sound handles this gracefully:
- If buffer loaded: plays immediately
- If loading: `loaded` callback fires when ready; playback starts then

For critical sounds (e.g., weapon fire), sounds are preloaded in the spritesheet.

### Pitch Shifting Artifacts

Changing `speed` (pitch) on-the-fly can introduce glitches in Web Audio API. Best practice: set speed at creation, not during playback.

### Layer Transitions

When crossing layer boundaries, volume transitions are animated (500ms tween). During rapid layer changes (e.g., falling through multiple layers), tweens queue; only the final state matters.

### @pixi/sound Quirks

- Calling `stop()` on an already-ended sound may stop a **different** random sound (upstream @pixi/sound bug). `GameSound` guards against this via `ended` flag.
- Audio context may disconnect on mobile if the app loses focus for extended periods.
- Pan is 1D stereo; no true 3D spatial audio (requires Web Audio HRTF, unsupported).

## Dependencies

### Depends on:
- **Client Rendering** ([docs/subsystems/client-rendering/](../client-rendering/)) — Game runs in PixiJS canvas; SoundManager uses PixiJS sound library
- **Game Loop** ([docs/subsystems/game-loop/](../game-loop/)) — `SoundManager.update()` called in game tick; sounds bound to object lifetime
- **Input Management** — User interactions (weapon fire, footsteps) trigger sounds
- **Camera Management** ([docs/subsystems/camera-management/](../camera-management/)) — Sound position updated from camera position; spatial distance calculated from camera to sound origin

### Depended on by:
- **Game Objects (Client)** ([docs/subsystems/game-objects-client/](../game-objects-client/)) — All game objects call `this.playSound()` for effects
- **UI Management** ([docs/subsystems/input-management/](../input-management/)) — UI events trigger feedback sounds

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Component map, tech stack, client side overview
- [Data Model](../../datamodel.md) — Game object types, layer definitions
- [API Reference](../../api-reference.md) — If audio is mentioned in protocol

### Tier 2 — Subsystems
- [Client Rendering](../client-rendering/) — PixiJS renderer, layers, containers
- [Game Loop](../game-loop/) — Tick update, frame timing
- [Game Objects (Client)](../game-objects-client/) — GameObject hierarchy, playSound() calls
- [Camera Management](../camera-management/) — Camera position used for spatial calculations
- [Input Management](../input-management/) — User actions trigger sounds

### Tier 3 — Modules
- (Patterns coming soon) — Common sound playback patterns

## See Also

- [@pixi/sound Docs](https://pixijs.com/docs/components/sound/) — Official API reference
- Web Audio API — MDN docs on Web Audio for audio buffer internals
- [content-plan.md](../../../content-plan.md) — Documentation roadmap
