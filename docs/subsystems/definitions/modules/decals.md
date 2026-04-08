# Definitions — Decals Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/decals.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents decal definitions: ground marks left by explosions, impacts, healing residue. Decals are placed on the map and fade over time. Referenced by `ExplosionDefinition.decal`.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/definitions/decals.ts | `Decals`, `DecalDefinition` | Low |

## DecalDefinition Fields

| Field | Purpose |
|-------|---------|
| `image?` | Sprite path (default from idString) |
| `scale?` | Scale (default 1) |
| `rotationMode` | Full, Limited, Binary, None |
| `zIndex?` | Render layer |
| `alpha?` | Opacity |

## Decal Types

- **Explosion decals** — explosion_decal, frag_explosion_decal, smoke_explosion_decal
- **Seed decals** — seed_decal, seed_explosion_decal, seed_explosion_decal_infected
- **Healing residue** — Auto-generated from HealingItems (`{idString}_residue`)
- **Other** — vomit_pool, used_flare_decal

## Business Rules

- Explosions reference decals via `ExplosionDefinition.decal`
- `decalFadeTime` on explosion — how long decal persists
- Healing items auto-generate residue decals

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 3:** [explosions.md](explosions.md) — Explosions reference decals
- **Tier 2:** [../objects/](../../objects/) — Decal server object
