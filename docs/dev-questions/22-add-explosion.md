# Q: How do I add a new explosion type with custom damage, radius, and shrapnel?

<!-- @tags: definitions, explosions, bullets, decals -->
<!-- @related: docs/subsystems/definitions/modules/explosions.md, docs/subsystems/definitions/modules/bullets.md -->

## 1. Add the explosion definition

Open `common/src/definitions/explosions.ts` and append:

```typescript
{
    idString: "my_explosion",
    name: "My Explosion",

    // Damage
    damage: 100,
    obstacleMultiplier: 2,      // damage × this vs obstacles

    // Radius (damage falloff)
    radius: {
        min: 4,   // full damage within this radius
        max: 12,  // zero damage at this radius (linear falloff)
    },

    // Visual & audio
    animation: {
        duration: 1000,
        tint: 0xffcc44,       // hex color
        scale: { start: 1, end: 2.5 },
    },
    cameraShake: {
        duration: 250,
        intensity: 5,
    },
    sound: "explosion_sound",   // optional — sound idString

    // Shrapnel (optional)
    shrapnelCount: 12,
    ballistics: {
        damage: 15,
        obstacleMultiplier: 1,
        speed: 0.18,
        range: 24,
        shrapnel: true,        // uses shrapnel bullet color
        tracer: { opacity: 0.8, width: 2 },
    },

    // Decal (optional ground mark)
    decal: "explosion_decal",  // references Decals collection
    decalFadeTime: 30000,      // ms until decal fades

    // Killfeed icon (optional)
    killfeedFrame: "frag_grenade",
}
```

## 2. A bullet definition is auto-generated

Each explosion automatically generates `{idString}_bullet` from its `ballistics`
field. You don't add anything to `bullets.ts` manually.

Shrapnel bullets use the tracer colors defined in `baseBullet.ts`.

## 3. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

Adding an explosion definition changes the `Explosions` serialization index.

## 4. Validate

```bash
bun validateDefinitions
```

## 5. Connect to a source (throwable, obstacle, or weapon)

**From a throwable** — set `explosionType` in `common/src/definitions/items/throwables.ts`:

```typescript
{
    idString: "my_grenade",
    explosionType: "my_explosion",
    // ...
}
```

**From an obstacle** (barrel) — set `explosion` in the obstacle definition:

```typescript
{
    idString: "my_barrel",
    explosion: "my_explosion",
    // ...
}
```

**From a gun** (frag round) — reference in the gun's bullet/special fire definition.

## Design Notes

- `radius.min` should be smaller than `radius.max`. Damage is 100% inside min,
  falling linearly to 0% at max.
- `obstacleMultiplier` > 1 makes the explosion more effective at destroying cover.
- Smoke grenade example: `damage: 0, radius: { min: 0, max: 0 }` — no damage,
  visual only.

## Related

- [Explosions Module](../subsystems/definitions/modules/explosions.md) — full field reference
- [Bullets Module](../subsystems/definitions/modules/bullets.md) — how shrapnel bullets are derived
- [Decals Module](../subsystems/definitions/modules/decals.md) — ground marks
- [Add Decal](27-add-decal.md) — if you need a custom decal for this explosion
