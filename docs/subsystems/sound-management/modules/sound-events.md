# Sound Management — Sound Events Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/sound-management/README.md -->
<!-- @source: client/src/scripts/managers/soundManager.ts -->

## Purpose

Handles sound playback lifecycle: trigger events from server packets or client UI, spatial audio (panning/falloff), dynamic volume updates, and audio sprite management.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/soundManager.ts` | SoundManager singleton, GameSound lifecycle, volume/pan updates | High |
| `client/src/scripts/managers/mapManager.ts` | Map feature sound triggers (building/obstacle destruction) | Medium |
| `vite/plugins/audio-spritesheet-plugin.ts` | Audio spritesheet packing (multiple sounds in one file) | Medium |
| `common/src/definitions/items/` | Weapon/item sound definitions | Low |

## Business Rules

- **Sound Playback:** API call `SoundManager.play(soundId, options)` returns `GameSound` instance
  - Sound must exist in audio spritesheet registry (via Vite plugin)
  - Play request queued until audio file loaded
  - Returns immediately with uninitialized GameSound object if load pending
  
- **Spatial Audio:** Sounds can be positional (3D-like audio in 2D world)
  - Position-based: Full volume at player position; decreases with distance
  - Falloff formula: `volume = 1 - (distance / maxRange)` (linear falloff)
  - Falloff constant: Steepness of volume curve (default: 1.0)
  - Pan (stereo): Computed from angle to sound source (left/right speaker)
  
- **Ambient vs. SFX:** Two volume tracks
  - **Ambient:** Background music, ambience (volume: `SoundManager.ambienceVolume`)
  - **SFX:** Sound effects, gunfire, impacts (volume: `SoundManager.sfxVolume`)
  
- **Dynamic Volume:** Sounds update volume/pan when camera moves (if `dynamic: true`)
  - Server-triggered sounds (e.g., distant gunfire) remain anchored at spawn position
  - Client-triggered sounds (e.g., UI click) update with camera (if enabled)
  
- **Looping:** Sounds can loop indefinitely or play once
  - Loop sounds must be manually stopped (not auto-terminated)
  
- **Layer Filtering:** Sounds can be restricted to specific layers
  - Cross-layer sounds muted (layer collision check via `adjacentOrEqualLayer()`)

## Data Lineage

```
Sound Trigger Event
├─ Trigger source: Server packet | Client UI
│
├─ SoundManager.play(soundId, options)
│  │
│  ├─ Load Audio (async)
│  │  └─ PixiSound.sound.play(id) → returns IMediaInstance
│  │
│  ├─ Create GameSound instance
│  │  ├─ Store: position, falloff, maxRange, layer
│  │  ├─ Dynamic: true → will update on camera move
│  │  ├─ Ambient: true → use ambienceVolume
│  │  └─ Parse options, set loop/speed
│  │
│  ├─ Register with active sounds Set
│  │
│  └─ Return GameSound to caller
│
└─ Update Loop (every frame):
   ├─ For each active sound:
   │  ├─ Check if expired (onEnd callback fired)
   │  ├─ If dynamic:
   │  │  ├─ Recalculate distance to camera
   │  │  ├─ Apply falloff: volume = managerVolume * this.volume * (1 - dist/maxRange)
   │  │  ├─ Clamp to [0, 1]
   │  │  ├─ Calculate pan (angle to speaker)
   │  │  └─ Update stereo filter & instance volume
   │  │
   │  └─ Remove if dead
   │
   └─ Cleanup expired sounds
```

## Complex Functions

### `SoundManager.play(name, options)` — @file client/src/scripts/managers/soundManager.ts:230

**Purpose:** Create and start playback of a sound effect or music track.

**Parameters:**
```typescript
interface SoundOptions {
    position?: Vector              // 2D world position (for spatial audio)
    falloff: number                // Volume curve steepness (0–1)
    layer: Layer | number          // Layer(s) this sound is on
    maxRange: number               // Maximum distance before mute
    loop: boolean                  // Loop indefinitely?
    speed?: number                 // Playback speed (1.0 = normal, 0.5 = half)
    dynamic: boolean               // Update volume on camera move?
    ambient: boolean               // Use ambient volume track?
    onEnd?: () => void             // Callback when sound ends
}
```

**Returns:** `GameSound` instance (may not be fully initialized if audio loading)

**Implicit Behavior:**
- If sound already queued in PixiSound registry, plays immediately
- Otherwise, queued; plays when file loaded (some delay)
- Stereo filter created and added to audio filters chain
- Position-based volume calculated on first update, then updated dynamically

### GameSound Constructor — @file client/src/scripts/managers/soundManager.ts:60

**Code:**
```typescript
constructor(id: string, name: string, options: SoundOptions) {
    this.id = id;
    this.name = name;
    this.position = options.position;
    this.falloff = options.falloff;
    this.maxRange = options.maxRange;
    this.layer = options.layer;
    this.speed = options.speed ?? 1;
    this.dynamic = options.dynamic;
    this.ambient = options.ambient;
    this.onEnd = options.onEnd;
    this.stereoFilter = new PixiSound.filters.StereoFilter(0);
    
    // Async play invitation to PixiSound
    const instanceOrPromise = PixiSound.sound.play(id, {
        loaded: (_err, _sound, instance) => {
            if (instance) this.init(instance);
        },
        filters: [this.stereoFilter],
        loop: options.loop,
        volume: this.managerVolume * this.volume,
        speed: this.speed
    });
    
    if (!(instanceOrPromise instanceof Promise)) {
        this.init(instanceOrPromise);  // Already loaded
    }
}
```

**Initialization Steps:**
1. Store parameters
2. Create stereo filter (pan control)
3. Request playback from PixiSound
4. If synchronously available, initialize immediately
5. If promise, wait for load callback

### Volume Calculation with Falloff — @file client/src/scripts/managers/soundManager.ts:85

**Spatial Audio Formula:**
```typescript
get managerVolume(): number {
    return this.ambient ? SoundManager.ambienceVolume : SoundManager.sfxVolume;
}

// In updateLayer():
const distanceToCamera = Vec.sub(SoundManager.position, this.position);
const distance = Vec.len(distanceToCamera);

const volumeFromDistance = distance > this.maxRange 
    ? 0 
    : Math.max(0, 1 - (distance / this.maxRange) * this.falloff);

const finalVolume = this.managerVolume * this.volume * volumeFromDistance * this.layerMult;
this.instance.volume = Math.max(0, Math.min(1, finalVolume));
```

**Logic:**
1. Distance from camera to sound position
2. Clamp distance against `maxRange`
3. Linear falloff: `(1 - distance/maxRange) * falloff` (falloff scales curve)
4. Layer multiplier: Attenuate if sound on different layer (crossfade distance)
5. Manager volume: Ambient or SFX master volume
6. Clamp final volume to [0, 1]

**Example (Gunshot 200 mm away):**
- `maxRange = 1000 mm`, `falloff = 1.0`, `managerVolume = 0.8`
- Distance falloff: `1 - (200/1000) * 1.0 = 0.8`
- Final: `0.8 * 1.0 * 0.8 = 0.64` (64% volume)

### Stereo Panning — @file client/src/scripts/managers/soundManager.ts:110

**Code:**
```typescript
const angleToSound = Angle.normalize(Angle.betweenPoints(SoundManager.position, this.position));
const panAmount = Math.sin(angleToSound - SoundManager.rotation);
this.stereoFilter.pan = Numeric.clamp(panAmount, -1, 1);
```

**Logic:**
1. Vector from camera to sound position
2. Angle difference: Sound angle minus camera rotation
3. Pan value: `sin(θ)` gives smooth -1 (full left) to +1 (full right)
4. Clamp to valid range

**Intuition:** Sine of angle gives lateral distance; sound directly left/right = ±1, directly ahead/behind = 0

### Layer-Based Volume Attenuation — @file client/src/scripts/managers/soundManager.ts:130–150

**Layer Multiplier Update:**
```typescript
updateLayer(force?: boolean): void {
    if (!force && !SoundManager.cameraLayer.value.has(this.layer as any)) {
        // Sound on different layer, tween volume down
        if (!this.layerVolumeTween) {
            this.layerVolumeTween = new Tween(this, { layerMult: 0 }, 200);
        }
    } else {
        // Sound on same layer, tween volume up
        if (this.layerVolumeTween) this.layerVolumeTween.kill();
        this.layerVolumeTween = new Tween(this, { layerMult: 1 }, 200);
    }
}
```

**Implicit Behavior:**
- Smooth volume crossfade when crossing layer boundaries
- 200 ms tween duration (audible fade, not abrupt clip)
- `layerMult` applied as multiplier in final volume calculation
- Prevents hearing sounds from layers player can't see

### Sound Event Triggers (Server-Based) — @file client/src/scripts/managers/mapManager.ts

**Example (Building destruction):**
```typescript
if (definition.sound !== undefined) {
    SoundManager.play(definition.sound, {
        position: this.position,
        falloff: 1.0,
        maxRange: 200,
        loop: false,
        dynamic: false,  // Anchored at destruction point
        ambient: false
    });
}
```

**Pattern:**
- Server sends "obstacle destroyed" packet
- Client unpacks position and definition
- Triggers client-side sound via definition.sound idString
- Sound plays once at destruction position, fades with distance

## Dependencies

**Internal:**
- [Camera Management](../../camera-management/) — Camera position/rotation for panning and distance calc
- [Game Loop](../../game-loop/) — Update ticker for per-frame volume refresh
- [Audio Spritesheet (Vite)](../../utilities-support/) — Audio file management

**External:**
- PixiJS Sound library (`@pixi/sound`)
- Tween system (smooth volume transitions)
- Date.now() for timing

## Configuration

| Setting | Effect | Type | Path |
|---------|--------|------|------|
| `SoundManager.sfxVolume` | SFX master volume (0–1) | number | `soundManager.ts` |
| `SoundManager.ambienceVolume` | Ambient/music volume (0–1) | number | `soundManager.ts` |
| `SoundDefinition.<soundId>.falloff` | Spatial falloff curve steepness | number | Per-sound definition |
| `SoundDefinition.<soundId>.maxRange` | Max distance before silent (mm) | number | Per-sound definition |

## Known Issues & Gotchas

- **Load Latency:** Audio files loaded asynchronously; play() may return before audio starts (especially on first playback)
- **Crossfade Abruptness:** Layer crossfade over 200 ms is fixed; very fast layer transitions may "pop"
- **Falloff Non-Physical:** Linear falloff doesn't match real-world inverse-square law (simplified for performance)
- **No 3D Positional Audio:** Panning is HRTF simulation via stereo filter; not true spatial audio API support
- **Memory Leak Risk:** Server sends many positional sounds per tick; ensure cleanup via `onEnd` callbacks and `ended` flag

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Sound Management subsystem overview
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Camera:** [../../camera-management/README.md](../../camera-management/README.md) — Camera position/rotation
- **Events:** [../../game-loop/README.md](../../game-loop/README.md) — Update loop integration
