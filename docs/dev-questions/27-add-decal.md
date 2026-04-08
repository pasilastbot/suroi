# Q: How do I add a new decal (ground mark) and attach it to an explosion or obstacle?

<!-- @tags: definitions, decals, explosions, rendering -->
<!-- @related: docs/subsystems/definitions/modules/decals.md, docs/subsystems/definitions/modules/explosions.md -->

## 1. Add the SVG asset

```
client/public/img/game/shared/<idString>.svg
```

Decals are ground textures — keep them flat, avoid excessive detail.
A good size is 64×64 or 128×128 logical pixels.

## 2. Validate the SVG

```bash
bun validateSvgs
```

## 3. Add the decal definition

Open `common/src/definitions/decals.ts` and append:

```typescript
{
    idString: "my_decal",
    // image defaults to idString — only override if filename differs:
    // image: "some_other_frame",
    scale: 1.2,
    rotationMode: RotationMode.Full,  // or Limited, Binary, None
    zIndex: ZIndexes.Decals,          // usually default
    alpha: 0.8,
}
```

## 4. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

## 5. Validate

```bash
bun validateDefinitions
```

## 6. Attach to an explosion

In `common/src/definitions/explosions.ts`, reference your decal:

```typescript
{
    idString: "my_explosion",
    // ...
    decal: "my_decal",
    decalFadeTime: 20000,   // ms — how long before decal fades
}
```

## 7. Attach to an obstacle (optional)

Obstacles can spawn a decal on destruction. Reference in the obstacle definition:

```typescript
{
    idString: "my_obstacle",
    // ...
    decal: "my_decal",
}
```

## Auto-Generated Decals

Healing items (bandages, medikits, etc.) automatically generate residue decals
named `{idString}_residue`. You don't need to add these manually.

## Notes

- `rotationMode: RotationMode.Full` → random rotation each time (good for
  explosion scorch marks — avoids repetition)
- `rotationMode: RotationMode.None` → fixed 0° (good for directional marks)
- `zIndex` controls whether the decal appears above or below other ground objects
- Decals use the `Decal` `ObjectCategory` and are sent in `UpdatePacket`
  `GameObjects` split

## Related

- [Decals Module](../subsystems/definitions/modules/decals.md) — full field reference
- [Explosions Module](../subsystems/definitions/modules/explosions.md) — `decal` + `decalFadeTime` fields
- [Add Explosion](22-add-explosion.md) — attaching a decal to a new explosion
