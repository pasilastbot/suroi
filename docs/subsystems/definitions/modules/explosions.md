# Definitions — Explosions Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/explosions.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents explosion definitions: damage, radius, camera shake, animation, shrapnel, and decals. Explosions are triggered by obstacles (barrels), throwables (grenades), and weapons (USAS, M590M).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/definitions/explosions.ts | `Explosions`, `ExplosionDefinition` | Medium |

## ExplosionDefinition Fields

| Field | Purpose |
|-------|---------|
| `damage` | Base damage at center |
| `obstacleMultiplier` | Damage multiplier vs obstacles |
| `radius` | `{ min, max }` — damage falloff range |
| `cameraShake` | `{ duration, intensity }` — client effect |
| `animation` | `{ duration, tint, scale }` — visual |
| `shrapnelCount` | Number of shrapnel projectiles |
| `ballistics` | BaseBulletDefinition for shrapnel |
| `decal?` | Ground decal after explosion |
| `decalFadeTime?` | Decal fade duration |
| `sound?` | Explosion sound |
| `killfeedFrame?` | Kill feed icon |

## Explosion Sources

- **Obstacles:** Barrel, super barrel, stove, propane tank, silo, refinery barrels
- **Throwables:** Frag grenade, smoke, C4, pumpkin, S.E.E.D., M202-F
- **Weapons:** USAS-12, M590M (frag rounds), firework launcher, seedshot

## Ballistics

- Shrapnel uses `ballistics` (damage, speed, range, tracer)
- `shrapnel: true` in ballistics — uses shrapnel bullet color
- Each explosion contributes to `Bullets` (see [bullets.md](bullets.md))

## Business Rules

- `radius.min`/`radius.max` — damage scales with distance
- Smoke grenade: damage 0, radius 0 — visual only
- Decals reference `Decals` definitions

## Related Documents

- **Tier 3:** [bullets.md](bullets.md) — Explosions contribute bullet definitions
- **Tier 2:** [../objects/](../../objects/) — Explosion server object
- **Tier 2:** [../inventory/](../../inventory/) — ThrowableItem triggers explosions
