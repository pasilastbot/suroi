# Audio Playback & Spatial Audio

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/sound-management/README.md -->
<!-- @source: client/src/scripts/managers/soundManager.ts -->

## Purpose

The Audio Playback module manages the complete audio lifecycle: loading sounds from spritesheets, playing instances with spatial positioning, updating volume based on distance/layer/camera movement, and handling cleanup. It bridges Web Audio API (via @pixi/sound) to the game's spatial coordinate system, enabling immersive 3D(ish) audio where weapon fire, footsteps, and environmental sounds respond realistically to player position and layer changes.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [client/src/scripts/managers/soundManager.ts](../../../../../client/src/scripts/managers/soundManager.ts) | `SoundManager` & `GameSound` classes; initialization, playback, per-frame updates | High |
| [client/src/scripts/game.ts](../../../../../client/src/scripts/game.ts) | Game object; owns `SoundManager`, calls `init()`, updates camera position, calls `SoundManager.update()` in game loop | High |
| [client/src/scripts/objects/gameObject.ts](../../../../../client/src/scripts/objects/gameObject.ts) | `playSound()` helper method — spatial wrapper for `SoundManager.play()` | Medium |
| [client/src/scripts/console/variables.ts](../../../../../client/src/scripts/console/variables.ts) | Console variables: `cv_sfx_volume`, `cv_ambience_volume`, `cv_music_volume` (0.0–1.0) | Low |
| [client/public/audio/game/](../../../../../client/public/audio/game/) | Audio asset directory; subdirs for spritesheet audio + no-preload audio | Low |

## Business Rules

- **Single Audio Context:** The Web Audio API state machine uses one `AudioContext` shared across all sounds via @pixi/sound.
- **Distance Attenuation:** Volume computed from sound position, camera position, `falloff` (exponent), and `maxRange` (cutoff distance). Exponent follows: `(1 - distance/maxRange) ^ (1 + falloff * 2)`, clamped.
- **Layer Attenuation:** Vertical layer boundaries introduce additional multipliers (same layer: 1.0×, +1 layer delta: 0.5×, 2+ layers: 0.15×). Ambient sounds only attenuate when going deeper underground; non-ambient sounds attenuate bidirectionally.
- **Spatial Panning:** Stereo pan computed from horizontal (X-axis) offset between camera and sound origin, normalized to [-1.0, 1.0]. No true 3D spatial audio (would require HRTF); pan is 1D left-right only.
- **Dynamic vs Static:** Sounds marked `dynamic: true` are tracked in `SoundManager.updatableSounds` and recalculated every frame when camera moves. Non-dynamic sounds are "fire and forget"—no per-frame overhead.
- **Layer Transitions:** When crossing layer boundaries, layer multiplier changes are animated via tween (500ms duration) to smooth volume jumps.
- **Volume Hierarchy:** Final volume = `managerVolume * objectVolume * layerMultiplier`, where `managerVolume` depends on sound category (SFX vs Ambience).

## Data Lineage

```
User Gesture (click, touch)
  ↓
Game.init() → SoundManager.init()
  ↓
Fetch binary spritesheet file (concatenated .mp3 segments)
  ↓
Parse spritesheet manifest (ID → offset/length map)
  ↓
Register all sound IDs (PixiSound.sound.add(id, buffer))
  ↓
Optional: Load no-preload sounds from /audio/game/no-preload/<id>.mp3
  ↓
[PLAYBACK READY]
  ↓
Game object calls this.playSound(name, options)
  ↓
SoundManager.play(name, options) creates new GameSound
  ↓
GameSound constructor calls PixiSound.sound.play(id, config)
  ↓
Audio buffer streams to Web Audio API (decoded or pre-decoded)
  ↓
PixiSound.IMediaInstance callback fires; GameSound.init(instance)
  ↓
Attach stereo filter; call update() to compute initial volume
  ↓
If dynamic or ambient: add to SoundManager.updatableSounds Set
  ↓
[PLAYING]
  ↓
Every game frame: SoundManager.update() → iterate updatableSounds
  ↓
For each GameSound.update(): recompute volume & pan from camera position
  ↓
Sound ends or stopped: instance emits "end" or "stop" event
  ↓
GameSound.ended = true; removed from updatableSounds at next update
  ↓
[CLEANUP]
```

## Complex Functions

### `GameSound.constructor(id: string, name: string, options: SoundOptions)` — @file client/src/scripts/managers/soundManager.ts:32

**Purpose:** Create a sound instance, start playback, attach spatial audio filters, initialize event listeners.

**Implicit behavior:**
- Stores all spatial parameters (`position`, `falloff`, `maxRange`, `layer`) for later updates
- Creates `StereoFilter` for left-right panning; no channel mixing or HRTF
- Calls `PixiSound.sound.play(id, config)` with `loaded` callback; playback may not start immediately if buffer still streaming
- If buffer is **already decoded**, `init(instance)` fires synchronously
- If buffer is **still loading**, `loaded` callback queued; `init(instance)` fires asynchronously
- Logs console warning if sound ID not found in registry
- **Critical:** @pixi/sound has a quirk where calling `stop()` on an already-ended sound stops a random sound instead; `GameSound` prevents this via `ended` flag guard in `stop()`

**Called by:** `SoundManager.play()` as constructor call, or directly by advanced callers.

---

### `GameSound.update(): void` — @file client/src/scripts/managers/soundManager.ts:103

**Purpose:** Recalculate volume and stereo pan based on current camera position. Called every frame if `dynamic: true` or `ambient: true`.

**Algorithm:**
```typescript
if (this.position) {
    // Compute distance from camera to sound origin
    const diff = Vec.sub(SoundManager.position, this.position);
    
    // Distance falloff: (1 - normalized_distance) ^ (1 + falloff * 2)
    this.instance.volume = (
        1 - Numeric.clamp(Math.abs(Vec.len(diff) / this.maxRange), 0, 1)
    ) ** (1 + this.falloff * 2) * this.managerVolume * this.volume * this.layerMult;
    
    // Stereo pan from horizontal offset
    this.stereoFilter.pan = Numeric.clamp(-diff.x / this.maxRange, -1, 1);
} else {
    // Non-spatial sound: no panning, constant volume
    this.instance.volume = this.managerVolume * this.volume * this.layerMult;
}
```

**Implicit behavior:**
- `Vec.len(diff)` computes Euclidean distance (2D); vertical (Y) distance doesn't affect volume (only horizontal spread matters for panning)
- Pan sign is **negated** (`-diff.x`), so sounds to camera's right appear in right channel
- `normalize(distance / maxRange)` means at `maxRange`, volume → 0; beyond `maxRange`, still 0 (effectively infinite falloff past cutoff)
- `1 + falloff * 2` exponent means: falloff=0 → linear decay; falloff=1 → cubic decay
- Volume is multiplied by three factors: manager volume (global SFX/ambience slider), object volume (sound-specific multiplier), and layer multiplier (transition-animated)
- **No-op if `instance` not yet initialized** (buffer still loading)

**Called by:** `SoundManager.update()` once per frame for all sounds in `updatableSounds`.

---

### `GameSound.updateLayer(initial?: boolean): void` — @file client/src/scripts/managers/soundManager.ts:128

**Purpose:** Recalculate layer attenuation factor when the player crosses layer boundaries (surface ↔ water ↔ underground).

**Algorithm:**
```typescript
let layerMult = 1;
const layerDelta = this.ambient 
    ? this.layer - Game.layer
    : Math.abs(this.layer - Game.layer);

if (layerDelta === 1) {
    layerMult = 0.5;
} else if (layerDelta >= 2) {
    layerMult = 0.15;
}

if (initial) {
    // First call: set immediately, no tween
    this.layerMult = layerMult;
    return;
}

// Subsequent calls: animate the transition
this.layerVolumeTween?.kill();  // Cancel previous tween if still running
this.layerVolumeTween = Game.addTween<GameSound>({
    target: this,
    to: { layerMult },
    duration: 500,  // 500ms smooth transition
    onComplete: () => { this.layerVolumeTween = undefined; }
});
```

**Implicit behavior:**
- **Ambient sounds** (e.g., wind, ambience music) only decrease underground (`this.layer - Game.layer`), not when surfacing from water
- **Non-ambient sounds** (weapons, footsteps) attenuate bidirectionally (`Math.abs(delta)`)
- Layer delta of exactly 1 → 50% volume; delta ≥ 2 → 15% volume (cumulative)
- First call (`initial: true`) sets `layerMult` synchronously; subsequent calls tween over 500ms
- This creates smooth transitions when jumping between layers; multiple rapid layer changes queue tweens (only final state matters)
- Tween uses game's own tween system (`Game.addTween<GameSound>`), not external library

**Called by:** `GameSound.init()` (initial call), and `Game.onLayerChange()` event (subsequent calls when player changes layers).

---

### `SoundManager.init(): Promise<void>` — @file client/src/scripts/managers/soundManager.ts:182

**Purpose:** Load audio spritesheet and register all sound IDs at game start. Called once; throws error on repeat calls.

**Algorithm:**
```typescript
async init(): Promise<void> {
    if (this._initialized) throw new Error("SoundManager has already been initialized");
    this._initialized = true;

    // 1. Read manager volume settings from console variables
    this.sfxVolume = GameConsole.getBuiltInCVar("cv_sfx_volume");
    this.ambienceVolume = GameConsole.getBuiltInCVar("cv_ambience_volume");

    // 2. Import dynamic spritesheet (mode-specific)
    const { importSpritesheet } = await import("virtual:audio-spritesheet-importer");
    const modeName = Modes[Game.modeName].similarTo ?? Game.modeName;
    const { filename, spritesheet } = await importSpritesheet(modeName);
    
    // 3. Fetch binary audio data
    const audio = await (await fetch(filename)).arrayBuffer();
    
    // 4. Register each sound slice from spritesheet
    let offset = 0;
    for (const [id, length] of Object.entries(spritesheet)) {
        this.addSound(id, { source: audio.slice(offset, offset + length) });
        offset += length;
    }

    // 5. Load no-preload (lazy) sounds
    const { noPreloadSounds } = await import("virtual:audio-spritesheet-no-preload");
    for (const id of noPreloadSounds) {
        this.addSound(id, { url: `./audio/game/no-preload/${id}.mp3`, preload: false });
    }
}
```

**Implicit behavior:**
- **Lazy initialization:** Audio context only starts on first `SoundManager.init()` call, respecting browser autoplay restrictions
- **Mode-specific spritesheet:** Different game modes (normal, fall, hunted, halloween) have separate audio assets; `Modes[Game.modeName].similarTo` allows fallback to base mode
- **Virtual imports:** `virtual:audio-spritesheet-importer` and `virtual:audio-spritesheet-no-preload` are Vite virtual modules generated at build time by the audio spritesheet plugin
- **Spritesheet format:** Binary concatenation of .mp3 segments + manifest mapping sound ID → (offset, length). No decompression needed; segments are standalone MP3 frames
- **No-preload sounds:** Lazy-loaded .mp3 files (e.g., long ambience tracks or rarely-used sounds); loaded on-demand, not bundled in main spritesheet
- **Error loading:** If a sound fails to load, @pixi/sound logs a warning but doesn't throw; playback continues with best-effort
- **Console variables:** `cv_sfx_volume` and `cv_ambience_volume` cached at init time; dynamic changes to these CVars also update `this.sfxVolume` and `this.ambienceVolume` (possibly via console callback, not shown)

**Called by:** `Game.init()` during game startup.

---

### `SoundManager.play(name: string, options?: Partial<SoundOptions>): GameSound` — @file client/src/scripts/managers/soundManager.ts:244

**Purpose:** Create and start a new sound instance. Returns handle for control.

**Algorithm:**
```typescript
play(name: string, options?: Partial<SoundOptions>): GameSound {
    const sound = new GameSound(name, name, {
        falloff: 1,
        maxRange: 256,
        dynamic: false,
        ambient: false,
        layer: Game.layer,
        loop: false,
        ...options  // Caller-provided overrides
    });

    // Add to update set if dynamic or ambient
    if (sound.dynamic || sound.ambient) {
        this.updatableSounds.add(sound);
    }

    return sound;
}
```

**Implicit behavior:**
- **Default parameter values** balance performance with realism: dynamic=false (no per-frame update unless needed), ambient=false (apply layer attenuation), layer=Game.layer (sound originates at current player layer)
- **maxRange=256:** Most sounds inaudible beyond 256 units (about 8 grid cells at 32 units/cell). Can override for far-reaching sounds (e.g., explosions)
- **falloff=1:** Cubic distance decay is realistic default; falloff=0 is linear (rarer, sounds carry farther)
- **Tracking:** Only `dynamic: true` or `ambient: true` sounds are tracked for per-frame updates. Fire-and-forget (non-dynamic, non-looping) sounds have zero per-frame CPU cost
- **Immediate return:** Returns `GameSound` immediately, even if audio buffer still loading; playback will start when buffer ready

**Called by:** Game objects, UI, ambient system, anywhere sound playback needed.

---

### `SoundManager.update(): void` — @file client/src/scripts/managers/soundManager.ts:232

**Purpose:** Update all dynamic/ambient sounds, clean up ended ones.

**Algorithm:**
```typescript
update(): void {
    for (const sound of this.updatableSounds) {
        if (sound.ended) {
            this.updatableSounds.delete(sound);
            continue;
        }
        sound.update();  // Recalculate volume/pan
    }
}
```

**Implicit behavior:**
- O(N) iteration over active dynamic sounds; set automatically removes ended sounds
- Deleted sounds are garbage-collected afterward (no dangling references)
- Order of iteration is unspecified (Set is unordered); sounds update in any order
- **No-op for fire-and-forget sounds:** Non-dynamic, non-looping sounds finish or stop without per-frame updates
- Called once per game tick (every frame at 40 TPS → 25ms per tick)

**Called by:** `Game.update()` during main loop.

---

## Audio System Architecture

### Initialization Flow

```
User clicks "Play" button (gesture)
  ↓
Game.init()
  ↓
SoundManager.init()
  ├─→ Fetch spritesheet binary (single HTTP request)
  ├─→ Parse manifest (ID → offset/length)
  ├─→ Register all IDs with @pixi/sound
  └─→ Lazy-load no-preload sounds (async, background)
  ↓
Audio context unlocked (first gesture fires context.resume())
  ↓
[Ready for playback]
```

### Runtime Audio Update

```
Game main loop (every frame at 40 TPS)
  ↓
Game.update()
  ├─→ Update camera position
  ├─→ SoundManager.position = camera.position
  ├─→ SoundManager.update()
  │   └─→ For each dynamic/ambient sound:
  │       ├─→ Compute distance from camera to sound
  │       ├─→ Apply distance falloff (exponent-based)
  │       ├─→ Apply layer attenuation
  │       ├─→ Compute stereo pan (X offset)
  │       ├─→ Set instance.volume and stereoFilter.pan
  │       └─→ Remove if ended
  └─→ Continue game logic
  ↓
Audio playback continues independently (Web Audio API thread)
```

## Audio Asset Management

### Spritesheet Format

The Vite audio spritesheet plugin concatenates mode-specific audio files:

```
Input: client/public/audio/game/<mode>/
  ├─ shared/ (all modes)
  │   ├─ weapon_ak47_fire.mp3 (1.2 MB)
  │   ├─ weapon_shotgun_fire.mp3 (950 KB)
  │   └─ ...
  ├─ normal/ (normal mode only)
  │   ├─ ambience_wind.mp3 (2 MB)
  │   └─ ...
  └─ fall/ (fall mode sounds)
      ├─ fall_warning.mp3 (400 KB)
      └─ ...

Plugin processes:
  ↓
Concatenate in order: shared/*.mp3 + mode/*.mp3
  ↓
Output binary file: client/dist/assets/audio_spritesheet_<mode>.bin (~15–40 MB)

Manifest:
{
  "weapon_ak47_fire": { "offset": 0, "length": 1257472 },
  "weapon_shotgun_fire": { "offset": 1257472, "length": 995328 },
  "ambience_wind": { "offset": 2252800, "length": 2097152 },
  ...
}
```

### No-Preload Storage

Rarely-used or long-duration sounds stored separately to reduce bundle size:

```
client/public/audio/game/no-preload/
  ├─ generator_running.mp3  (long loop, only loaded when generator active)
  ├─ boss_music_intense.mp3 (boss music, loaded on-demand)
  └─ ...
```

Loaded dynamically via `fetch("./audio/game/no-preload/<id>.mp3")` with slight latency (~100–200ms depending on file size).

### Sound Categories

| Category | Examples | Preload | Falloff | Max Range | Loop | Spatial |
|----------|----------|---------|---------|-----------|------|---------|
| Weapon Fire | `weapon_ak47_fire`, `weapon_shotgun_fire` | Yes | 1–2 | 512 | No | Yes |
| Impact | `hits/wood`, `swings/metal` | Yes | 0.5–1 | 256 | No | Yes |
| Footsteps | `footsteps/grass`, `footsteps/concrete` | Yes | 0.8 | 256 | No | Yes |
| Ambience | `ambience_wind`, `river_ambience`, `ocean_ambience` | Yes | 0 | ∞ | Yes | No |
| UI | `button_press`, `pickup` | Yes | N/A | N/A | No | No |
| Explosions | `explosion_1`, `explosion_2` | Yes | 2 | 1024 | No | Yes |
| Music | `menu_music`, `menu_music_fall` | No (lazy) | N/A | N/A | Yes | No |
| Misc | `join_notification`, `death_marker` | Yes | Varies | Varies | Varies | Varies |

## Spatial Audio Implementation

### Distance Falloff Formula

Audio volume decays with distance using an **exponential model**:

```
normalizedDistance = clamp(distance / maxRange, 0, 1)
volumeAttenuation = (1 - normalizedDistance) ^ (1 + falloff * 2)
finalVolume = volumeAttenuation * managerVolume * objectVolume * layerMult
```

**Examples:**
- At distance = 0: attenuation = 1.0 (full volume)
- At distance = maxRange: attenuation = 0.0 (silence)
- At distance = maxRange/2 with falloff=1: attenuation = 0.5^3 = 0.125 (12.5%)
- At distance = 3×maxRange: attenuation = 0.0 (beyond cutoff)

**Falloff Exponent:**
- falloff=0 → `(1 - dist)^1` = linear decay
- falloff=1 → `(1 - dist)^3` = cubic decay (realistic for inverse-square-ish)
- falloff=2 → `(1 - dist)^5` = steep falloff (far sounds disappear fast)

### Stereo Panning

Pan computed from **horizontal (X-axis) offset** between camera and sound origin:

```
diff = cameraPosition - soundPosition
pan = clamp(-diff.x / maxRange, -1, 1)
stereoFilter.pan = pan  // -1 = full left, 0 = center, 1 = full right
```

**Why negated?** Sound to camera's right has positive `diff.x`; negating makes it map to positive pan (right channel).

**Example:**
- Sound 100 units to camera's right, maxRange=256: pan = -100/256 ≈ -0.39 (slightly right)
- Sound 256 units to camera's left, maxRange=256: pan = -(-256)/256 = 1.0 (full left)

### Layer Attenuation Model

Vertical layer boundaries create acoustic discontinuities:

| Scenario | Ambient Sound | Non-Ambient Sound |
|----------|---------------|-------------------|
| Same layer (underwater to underwater) | 1.0× | 1.0× |
| +1 layer (above to underwater) | 0.5× | 0.5× |
| +2 layers (above to underground) | 0.15× | 0.15× |
| -1 layer (underwater to above) | 0.5× | 0.5× |
| -2 layers (underground to above) | 0.15× | 0.15× |

**Ambient sound exception:** If marked `ambient: true`, underwater/underground sounds **only decrease when diving deeper**. Surfacing keeps ambient volume constant. This prevents background music from disappearing abruptly when surfacing.

**Transition animation:** Layer changes are smoothed via tween (500ms), creating natural fade-in/fade-out during layer crossing.

## Volume Control

### Manager Volume (Global Sliders)

Two master volume categories controlled by console variables:

```typescript
sfxVolume: number = GameConsole.getBuiltInCVar("cv_sfx_volume");      // 0.0–1.0, mutable
ambienceVolume: number = GameConsole.getBuiltInCVar("cv_ambience_volume");  // 0.0–1.0, mutable
```

**Console Variables:**
| Variable | Default | Range | Purpose |
|----------|---------|-------|---------|
| `cv_sfx_volume` | 1.0 | 0.0–1.0 | Master volume for all SFX (weapons, impacts, UI) |
| `cv_ambience_volume` | 1.0 | 0.0–1.0 | Master volume for environment/music |
| `cv_music_volume` | 1.0 | 0.0–1.0 | Music-specific (separate from ambience) |

**GameSound determines category:**
```typescript
get managerVolume(): number {
    return this.ambient ? SoundManager.ambienceVolume : SoundManager.sfxVolume;
}
```

### Instance Volume Multipliers

Each `GameSound` has independent multipliers:

```typescript
volume: number = 1.0;      // Base volume for this sound (0.0–2.0 typical)
layerMult: number = 1.0;   // Layer attenuation factor (animated)
```

**Final volume calculation:**
```
finalVolume = managerVolume                       // Global SFX or Ambience slider
            * this.volume                          // Per-sound base volume
            * this.layerMult                       // Layer attenuation (animated 0–1)
            * distanceAttenuation                  // Computed from position/falloff/range
```

### Volume Fading

Sounds can be manually faded by modifying `volume`:

```typescript
const sound = SoundManager.play("ambience_wind", { loop: true });
// Fade out over 1 second
Game.addTween({
    target: sound,
    to: { volume: 0 },
    duration: 1000,
    onComplete: () => sound.stop()
});
```

No built-in fade API; use game's tween system.

## Sound Triggers

Sounds are triggered from diverse sources throughout the game:

### Weapon Fire

```typescript
// @file client/src/scripts/objects/bullet.ts:143
SoundManager.play(`${gunIdString}_fire${this.lastShot ? "_last" : ""}`, {
    position: this.position,
    falloff: 1,
    maxRange: 512,
    dynamic: false,
    layer: this.layer
});
```

Sound ID pattern: `weapon_<gun_idString>_fire` or `weapon_<gun_idString>_fire_last`

### Impact Sounds

```typescript
// @file client/src/scripts/objects/projectile.ts:216
this.hitSound = SoundManager.play(
    `hits/${definition.hitSound || "rock"}`,
    { position: this.position, layer: this.layer, ... }
);
```

Categories: `hits/wood`, `hits/rock`, `hits/sand`, `hits/metal`, `hits/glass`, etc.

### Ambience (Looping)

```typescript
// @file client/src/scripts/game.ts:1063
this.ambience = SoundManager.play(ambience, { loop: true, ambient: true });
```

Triggered once at game start; runs continuously until game ends.

### Building/Object Interactions

```typescript
// @file client/src/scripts/objects/building.ts:255
this.sound = SoundManager.play(sounds.normal, soundOptions);
```

Examples: door open/close, generator hum, generator start, locker unlock, etc. Sound ID defined in building definition.

### UI Feedback

```typescript
// @file client/src/scripts/managers/inputManager.ts:252
if (playSound) SoundManager.play("pickup");
```

Examples: `button_press`, `pickup`, `puzzle_solved`, `puzzle_error`, `speaker_start`, etc. Non-spatial, global sounds.

### Network Events (Server-Triggered)

```typescript
// @file client/src/scripts/game.ts:1040
SoundManager.play(soundID);  // Server specifies sound to play
```

Triggered by server via `SoundRequest` packet; allows server-controlled audio (kill notifications, system alerts).

## Sound Queueing & Polyphony

### Polyphony Limits

No explicit per-sound limit; Web Audio API has **hardware constraints**:
- Typical browsers: 32–64 simultaneous sample playback contexts
- With 40 TPS (25ms/tick) gameplay, maximum ~3-5 significant sounds per frame
- If limit exceeded, oldest sounds are culled by browser

### Priority System

No built-in priority; all sounds treated equally. If polyphony limit hit, oldest audio nodes are killed.

**Best practice:** Avoid spawning >5 loud, simultaneous sounds (e.g., don't play weapon fire + impact + footstep all at once per object).

### Overlap Handling

Repeated sounds (e.g., rapid weapon fire) spawn new `GameSound` instances:

```typescript
// Multiple rapid shots spawn multiple instances
for (let i = 0; i < shotCount; i++) {
    SoundManager.play("weapon_ak47_fire", { ... });  // Each call creates new GameSound
}
```

No deduplication; each `play()` call creates a new instance, allowing overlapping playback.

## Music System

### Menu Music

Loaded at menu startup; distinct track per game mode:

```typescript
// @file client/src/scripts/game.ts:780
this.music = sound.add("menu_music", {
    url: `./audio/music/menu_music${menuMusicSuffix}.mp3`,
    autoPlay: false
});

// Play when menu becomes active
this.music.play({
    loop: true,
    volume: GameConsole.getBuiltInCVar("cv_music_volume")
});
```

**Music files by mode:**
```
client/public/audio/music/
├─ menu_music.mp3           (default)
├─ menu_music_old.mp3       (legacy option, if cv_use_old_menu_music =true)
├─ menu_music_fall.mp3      (fall mode)
├─ menu_music_halloween.mp3 (halloween event)
├─ menu_music_winter.mp3    (winter event)
├─ menu_music_hunted.mp3    (hunted mode)
└─ ...
```

**Mode Music Routing:**
```typescript
const _modeName = Modes[Game.modeName].similarTo ?? Game.modeName;
// Load "menu_music_${modeName}.mp3"; fallback to default if not found
```

### In-Game Music

Currently placeholder; no dynamic in-game music implemented. Ambience tracks (`ambience_wind`, `river_ambience`, `ocean_ambience`) serve as background.

Future: Could add danger-phase music, boss music, etc. following same pattern.

### Volume & Transitions

Music volume controlled by `cv_music_volume` console variable:

```typescript
sound.volume = GameConsole.getBuiltInCVar("cv_music_volume");
```

No built-in crossfade between tracks; would require manual tween of volume.

## Network Events

Server can trigger client-side sounds via packets:

### `SoundRequest` Packet

When server sends `SoundPacket` with sound ID:

```typescript
// @file client/src/scripts/game.ts:1040
SoundManager.play(soundID);
```

Examples:
- Kill leader assigned: `SoundManager.play("kill_leader_assigned")` triggered by server
- Kill leader dead: `SoundManager.play("kill_leader_dead")` triggered by server
- Player death notification (if unfocused): `SoundManager.play("join_notification")` triggered by server

Server event → Packet → Client receives → Sound plays (non-spatial).

## Performance Considerations

### Audio Context Limits

Web Audio API provides:
- **Single `AudioContext` per window** (shared by all @pixi/sound instances)
- **32–256 audio nodes** depending on browser (nodes = source + filter + destination chains)
- **CPU overhead:** ~0.5% per active sound node; Web Workers can offload mixing on some browsers

### Memory Usage

```
Spritesheet: ~20–40 MB (compressed, uncompressed in browser memory after decode)
Per-sound instance: ~50 KB (node refs + parameter storage)
With 64 simultaneous sounds: ~3.2 MB overhead
```

### Game Loop Impact

**Per-frame cost:**
```
for each dynamic/ambient sound (typical: 2–5 sounds):
    - Vec.sub() + Vec.len(): ~10 μs (distance calculation)
    - Math operations (attenuation, layer tween): ~5 μs
    - Filter pan update: ~2 μs
    Total per sound: ~20 μs
    Total per frame: ~20–100 μs (negligible)
```

Low cost because only dynamic/ambient sounds updated; most fire-and-forget sounds have zero per-frame overhead.

### Browser Optimization

- **Mute on unfocus:** Audio context can pause if window loses focus; implement manual pause:
  ```typescript
  window.addEventListener("blur", () => { 
      // Pause all sounds or lower volume to avoid background drain
  });
  ```
- **Battery drain:** Audio playback drains battery on mobile; limit simultaneous sounds on mobile platforms.
- **Autoplay restrictions:** Most browsers require gesture before playback; @pixi/sound handles resume automatically on gesture.

## Known Gotchas & Constraints

### Browser Autoplay Policy

**Symptom:** First few sounds don't play or play silently.

**Cause:** Web Audio API requires explicit user gesture (click, touch, key) to start playback. Calling `SoundManager.play()` before first gesture queues sounds but doesn't start them.

**Solution:** @pixi/sound handles this via `AudioContext.resume()` on first click/touch. Ensure user interacts before critical sounds (e.g., game start).

### iOS Audio Muting

**Symptom:** Non-musical audio muted even though game volume is high.

**Cause:** iOS mute switch affects non-musical audio globally. Music apps (Apple Music) bypass this; game audio is subject to it.

**Solution:** No workaround; documented platform limitation. Advise users to unmute device if audio is silent.

### Mobile Silence Mode

**Symptom:** Android users report no sound despite global volume on.

**Cause:** Android has separate "ringer" and "media" volume streams. Ringer mute affects in-app audio if routed incorrectly.

**Solution:** No fix from browser; platform limitation. Test audio on actual devices.

### Web Audio Latency

**Symptom:** Weapon fire sound lags noticeably after trigger (50–200ms).

**Cause:** Buffer loading, Web Audio scheduling, browser GC pauses.

**Solution:** Preload all critical sounds in spritesheet. Use `dynamic: false` for one-off sounds (no per-frame overhead = faster playback).

### Pitch Shifting Glitches

**Symptom:** Speed-shifting a sound causes crackling or phase issues.

**Cause:** Web Audio's playback rate change is not perfectly smooth; resampling can introduce artifacts.

**Solution:** Set `speed` at creation, not during playback. For pitch variety, include multiple .mp3 files at different pitches instead of runtime shifting.

### Pan Clipping (Unnatural Stereo)

**Symptom:** Pan values > ±1.0 create unnatural stereo (one channel louder).

**Cause:** StereoFilter pan must be clamped to [-1, 1]; beyond that, behavior is undefined.

**Solution:** Pan is already clamped in `GameSound.update()`: `Numeric.clamp(-diff.x / maxRange, -1, 1)`. Issue unlikely if maxRange ≥ audio origin distance.

### Rapid Layer Changes

**Symptom:** Volume "stutters" or jumps when crossing multiple layers rapidly.

**Cause:** Layer multiplier tweens overlap. Only the final tween state matters.

**Solution:** Tweens are automatically cancelled on new `updateLayer()` call; final state converges correctly.

### @pixi/sound Bug: stop() on Ended Sound

**Symptom:** Calling `sound.stop()` on an already-ended sound stops a different, unrelated sound.

**Cause:** Upstream @pixi/sound bug; stopped sound lookups are fragile.

**Solution:** `GameSound.stop()` guards with `if (this.ended) return;` to prevent double-stop.

### Audio File Compatibility

**Symptom:** Some audio files don't play on Safari or older browsers.

**Cause:** Browser codec support varies. MP3 is widely supported; WAV/OGG less so.

**Solution:** All game audio is MP3 (universal support). Vite audio spritesheet plugin handles encoding.

### Mixing Artifacts with Category Volume Changes

**Symptom:** Changing `cv_sfx_volume` mid-sound causes abrupt volume jump.

**Cause:** Volume change is immediate; no smooth fade.

**Solution:** No built-in fade on CVar change. For smooth transition, manually tween affected sounds.

## Dependencies

### Depends on:
- **Client Rendering** ([../client-rendering/](../client-rendering/)) — PixiJS provides @pixi/sound library for audio
- **Game Loop** ([../game-loop/](../game-loop/)) — SoundManager.update() called in main game tick
- **Camera Management** ([../camera-management/](../camera-management/)) — Camera position used for spatial calculations (though currently SoundManager.position updated manually, not from camera manager)
- **Console/Debug System** — cv_sfx_volume, cv_ambience_volume, cv_music_volume read at init and dynamically

### Depended on by:
- **Game Objects (Client)** ([../game-objects-client/](../game-objects-client/)) — All objects call `this.playSound()`
- **Input Management** ([../input-management/](../input-management/)) — UI events trigger SoundManager.play()
- **UI Management** ([../ui-management/](../ui-management/)) — Menu music, UI feedback sounds

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Client/server division; PixiJS rendering
- [Data Model](../../datamodel.md) — Game object types, layers, spatial coordinates

### Tier 2 — Subsystems
- [Sound Management](../README.md) — Parent subsystem overview
- [Client Rendering](../client-rendering/) — PixiJS and @pixi/sound library
- [Game Loop](../game-loop/) — 40 TPS tick timing, update order
- [Camera Management](../camera-management/) — Camera position for spatial audio
- [Game Objects (Client)](../game-objects-client/) — GameObject.playSound() helper

### Tier 3 — Modules
- **Audio Playback** (this document) — Audio lifecycle, spatial audio
- *Sound Effects Library* (future) — Catalog of all sound IDs, falloff parameters
- *Music & Ambience* (future) — Menu music, game phase music

## See Also

- [@pixi/sound API Docs](https://pixijs.com/docs/components/sound/) — Official reference
- [Web Audio API MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API) — Underlying Web Audio standard
- [content-plan.md](../../../content-plan.md) — Documentation roadmap
