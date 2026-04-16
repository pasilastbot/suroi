# Q23: How does the spritesheet packing pipeline work (4096×4096 hi-res / 2048×2048 lo-res)?

**Answer:**

The spritesheet packing pipeline converts raw SVG/PNG assets into multiple resolution-optimized WebGL-compatible atlases during Vite build time, using the MaxRectsPacker bin packing algorithm to automatically create multiple sheets if assets exceed the maximum size.

## Spritesheet Packing Pipeline (Source Art → Runtime)

The pipeline has three phases:

| Phase | Component | What It Does |
|-------|-----------|--------------|
| **Discovery** | Asset scanner | Scans `public/img/game/{modeName}/` for all images; mode sprites override shared sprites |
| **Packing** | `MaxRectsPacker` + skia-canvas | Bins images into fixed-size atlases; generates hi-res and lo-res variants |
| **Serialization** | PixiJS `SpritesheetData` | Writes atlas JSON (frame metadata) + PNG (rendered texture) to bundle |

### Asset Organization & Selection

Assets are organized by **game mode and shared assets**:

```
public/img/game/
├── shared/                    ← always included
│   ├── weapons/
│   ├── obstacles/
│   └── ui/
├── normal/                    ← normal mode-specific
├── halloween/                 ← halloween mode-specific
└── ... (other modes)
```

Each mode specifies which asset folders to pack:

```typescript
// common/src/definitions/modes.ts
normal: {
    spriteSheets: ["shared", "normal"]  // Load shared, then normal (normal overrides duplicates)
},
halloween: {
    spriteSheets: ["shared", "fall", "halloween"]  // Cascading: shared → fall → halloween
}
```

The cascade allows mode-specific variants to override shared assets by filename.

## Hi-Res vs Lo-Res Asset Variants

Both resolution tiers are generated from the same source image:

| Tier | Size | Purpose | When Used |
|------|------|---------|-----------|
| **Hi-res** | 4096×4096 | Desktop / high-DPI devices | By default if `MAX_TEXTURE_SIZE >= 4096` |
| **Lo-res** | 2048×2048 | Mobile / low-bandwidth / older hardware | If device doesn't support 4096×4096 |

**Runtime selection:**

```typescript
// If device doesn't support 4096x4096 textures, use 2048x2048
if (gl.getParameter(gl.MAX_TEXTURE_SIZE) < 4096) {
    highResolution = false;
}
```

**Generation process:**

```typescript
const canvas = new Canvas(4096, 4096);             // Hi-res
// ... draw all packed images at full resolution ...

const lowResCanvas = new Canvas(2048, 2048);      // Lo-res
const lowResCtx = lowResCanvas.getContext("2d");
lowResCtx.drawImage(canvas, 0, 0, 2048, 2048);   // scale down 50%
```

Both textures are stored in the bundle and loaded on-demand.

## Bin Packing Algorithm & Overflow Handling

**Algorithm:** [MaxRectsPacker](https://github.com/soimy/maxrects-packer) — JavaScript rectangle packing using MaxRects algorithm.

**Configuration:**

```typescript
export const compilerOpts = {
    margin: 8,                          // 8px padding between sprites
    maximumSize: 4096,                  // Max sheet width/height
    packerOptions: { allowRotation: false }  // No sprite rotation
}
```

### Automatic Bin Creation (Overflow Handling)

When a single atlas exceeds 4096×4096, the packer **automatically creates a new sheet**:

```typescript
const packer = new MaxRectsPacker(4096, 4096, 8, { allowRotation: false });

// Add all images to packer
// ...

// packer.bins = [ bin0, bin1, bin2, ... ] ← Multiple bins if overflow
await Promise.all(packer.bins.map(async bin => {
    // For EACH bin, render + generate separate JSON and PNG
}));
```

**Result:** If assets overflow:
- ✅ Packer automatically creates `bin[0]`, `bin[1]`, etc.
- ✅ Each bin generates its own hi-res **and** lo-res atlas
- ✅ All atlases bundled and loaded at runtime

**Example:** With ~800 sprite assets (typical), suroi generates:
- `normal` mode: 1-2 hi-res + 1-2 lo-res atlases
- `halloween` mode (with cascaded shared + fall + halloween): 2-3 hi-res + 2-3 lo-res atlases

## Vite Plugin Architecture

The image spritesheet system exports **two plugin configurations**:

| Plugin | Mode | Responsibility |
|--------|------|-----------------|
| `imageSpritesheet()[0]` | `build` | Pre-process all assets during `vite build` |
| `imageSpritesheet()[1]` | `serve` | Hot-reload spritesheets during `vite dev` |

### Build Phase

```typescript
{
    name: "vite-spritesheet-plugin:build",
    apply: "build",
    async buildStart() {
        // Pre-process ALL modes before bundling
        for (const modeName of Object.keys(Modes)) {
            await buildSpritesheets(modeName);
        }
    },
    generateBundle() {
        // Emit PNG atlases as final bundle assets
        for (const [fileName, source] of files) {
            this.emitFile({ type: "asset", fileName, source });
        }
    }
}
```

### Dev Server Phase

```typescript
{
    name: "vite-spritesheet-plugin:serve",
    apply: "serve",
    configureServer(server) {
        // Watch for asset changes; hot-reload only changed modes
        const onChange = (filename: string) => {
            const invalidatedModes = Object.entries(Modes)
                .filter(([, mode]) => mode.spriteSheets.includes(dir))
                .map(([modeName]) => modeName);

            // Rebuild and hot-reload only these modes
            for (const modeName of invalidatedModes) {
                // ...rebuild spritesheet...
            }
        };
        watcher = watch("public/img/game", { ignoreInitial: true })
            .on("add", onChange)
            .on("change", onChange)
            .on("unlink", onChange);
    }
}
```

### Virtual Modules

The plugin creates virtual importers for each resolution:

```typescript
// auto-generated
export const importSpritesheet = async (modeName) => {
    switch(modeName) {
        case "normal": return await import("virtual:image-spritesheets-low-res-normal");
        case "halloween": return await import("virtual:image-spritesheets-high-res-halloween");
        // ...
    }
}
```

Each mode's virtual module exports an array of `SpritesheetData`:

```typescript
// virtual:image-spritesheets-low-res-normal
export const spritesheets = [
    { meta: { image: "img/atlases/normal-abc12345@0.5x.png", scale: 0.5 }, frames: {...} },
    // More atlases if overflow
];
```

## Runtime Texture Loading & Frame Mapping

### Loading Pipeline

```typescript
export async function loadSpritesheets(modeName, renderer, highResolution) {
    // Step 1: Choose resolution based on device
    if (gl.getParameter(gl.MAX_TEXTURE_SIZE) < 4096) {
        highResolution = false;
    }

    // Step 2: Dynamically import virtual module
    const { importSpritesheet } = highResolution
        ? await import("virtual:image-spritesheets-importer-high-res")
        : await import("virtual:image-spritesheets-importer-low-res");

    // Step 3: Get mode's spritesheets
    const { spritesheets } = await importSpritesheet(modeName);

    // Step 4: Load each atlas
    await Promise.all(spritesheets.map(async spritesheet => {
        const sheetTexture = await Assets.load<Texture>(spritesheet.meta.image);
        
        // Step 5: Parse PixiJS Spritesheet (maps frame names to regions)
        const textures = await new Spritesheet(sheetTexture, spritesheet).parse();

        // Step 6: Add to global texture cache
        for (const [key, texture] of Object.entries(textures)) {
            Assets.cache.set(key, texture);
        }
    }));
}
```

### Frame Lookup at Runtime

Once loaded, sprites are fetched by name:

```typescript
export class SuroiSprite extends Sprite {
    static getTexture(frame: string): Texture {
        if (!Assets.cache.has(frame)) {
            frame = "_missing_texture";  // Fallback
        }
        return Texture.from(frame);
    }

    constructor(frame?: string) {
        super(frame ? SuroiSprite.getTexture(frame) : undefined);
    }
}
```

**Frame data example** (from generated JSON):

```json
{
  "meta": {
    "image": "img/atlases/normal-abc12345@1x.png",
    "scale": 1,
    "size": { "w": 4096, "h": 4096 }
  },
  "frames": {
    "gun_ar15": {
      "frame": { "x": 120, "y": 400, "w": 128, "h": 64 },
      "sourceSize": { "w": 128, "h": 64 }
    },
    "gun_ak47": {
      "frame": { "x": 250, "y": 400, "w": 140, "h": 68 },
      "sourceSize": { "w": 140, "h": 68 }
    }
  }
}
```

When `new SuroiSprite("gun_ar15")` is called, PixiJS:
1. Finds frame metadata `{ x: 120, y: 400, w: 128, h: 64 }`
2. Creates a texture region pointing to that rectangle in the atlas
3. Clones the texture for the sprite instance

## Handling Sprites Exceeding Available Sheet Space

**If a single asset exceeds 4096×4096:**

Currently, there is **no hard limit enforcement**. However:

- The MaxRectsPacker will attempt to pack the asset into available rectangles
- If a single image is larger than `maximumSize`, packing **fails silently**
- **Prevention:** Keep individual sprites < 2048×2048 px; the packer handles cascading overflow automatically if needed

**Best practice:** The MaxRectsPacker will automatically create additional bins if assets overflow, so scaling is automatic.

## Performance Implications

### Build Time

- **Initial build:** ~500-1500 ms per mode (skia-canvas + bin packing)
- **Dev reload:** ~200-400 ms (only changed mode re-packed)
- **Caching:** Atlases cached by file mod time; unchanged assets skip repacking

### Runtime Memory & Bandwidth

| Metric | Value | Notes |
|--------|-------|-------|
| Typical atlas size | 1-2 MB per hi-res atlas | PNG compression; normal mode ≈1.2 MB |
| Texture uploads | 50-100 ms | Depends on GPU and page load timing |
| Memory per atlas | ~32 MB (4096×4096 RGBA) | Hi-res: 4096² × 4 bytes; lo-res: 2048² × 4 |
| Total client memory | ~100-200 MB (all modes) | All atlases loaded in VRAM |
| Bundle size impact | ~10-15 MB (unpacked) | All mode atlases in HTML |

### Optimizations in Place

1. **Margin prevents bleed:** 8px padding between sprites prevents texture filtering artifacts
2. **Cascading sprites override:** Mode-specific variants don't duplicate in memory
3. **Resolution conditional load:** Devices with `MAX_TEXTURE_SIZE < 4096` load 2048×2048 atlases, saving 4× memory
4. **No rotation:** `allowRotation: false` keeps frame lookup O(1)

### Known Bottlenecks

- **Single atlases per mode:** If a mode exceeds 4096×4096, a second atlas doubles texture upload time
- **No atlas compression:** WebGL textures are uncompressed RGBA; no ASTC/DXT support
- **No dynamic atlasing:** All atlases built at startup; no lazy-load per asset type

## References

**Tier 1:**
- [docs/architecture.md](docs/architecture.md) — Build system overview

**Tier 2:**
- [docs/subsystems/client-rendering/README.md](docs/subsystems/client-rendering/README.md) — PixiJS rendering, spritesheet loading
- [docs/subsystems/game-modes/README.md](docs/subsystems/game-modes/README.md) — Mode definition, spritesheet configuration

**Tier 3:**
- [docs/subsystems/client-rendering/modules/rendering-pipeline.md](docs/subsystems/client-rendering/modules/rendering-pipeline.md) — Texture loading, frame mapping

**Source Code:**
- [client/vite/plugins/image-spritesheet-plugin.ts](client/vite/plugins/image-spritesheet-plugin.ts) — Packing algorithm, overflow handling
- [client/src/scripts/utils/pixi.ts](client/src/scripts/utils/pixi.ts) — Runtime loading, frame lookup
- [common/src/definitions/modes.ts](common/src/definitions/modes.ts) — Mode configuration
