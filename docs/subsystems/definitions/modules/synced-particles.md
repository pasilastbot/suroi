# Definitions — SyncedParticles Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/syncedParticles.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents synced particle definitions: server-authoritative particles (muzzle flash, impacts, healing residue). Uses `ValueSpecifier`, `Animated`, and optional hitbox for scope-zoom effects (e.g. smoke grenade).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/definitions/syncedParticles.ts | `SyncedParticles`, `SyncedParticleDefinition` | High |

## SyncedParticleDefinition Fields

| Field | Purpose |
|-------|---------|
| `scale` | Animated or NumericSpecifier |
| `alpha` | Animated or NumericSpecifier |
| `lifetime` | Duration (ms) |
| `angularVelocity` | Rotation speed |
| `velocity` | VectorSpecifier, optional easing/duration |
| `zIndex` | Render layer |
| `frame` | Sprite frame name |
| `tint?` | Color override |
| `depletePerMs?` | Health/adrenaline drain (e.g. gas) |
| `spawner?` | Spawn multiple (count, radius, staggering) |
| `hitbox?` | CircleHitbox for scope zoom (smoke) |
| `scopeOutPreMs?` | Zoom out before particle ends |

## ValueSpecifier

- `T | { min: T, max: T }` — Single value or random range
- `resolveNumericSpecifier`, `resolveVectorSpecifier` — Resolve at spawn

## Animated

- `start`, `end` — ValueSpecifier
- `easing?` — Ease function name
- Interpolated over `lifetime`

## Business Rules

- Particles are server-spawned; client receives via UpdatePacket
- `hasCreatorID` — Include creator player ID for team-colored effects
- Hitbox particles (smoke) trigger scope zoom on client

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 2:** [../objects/](../../objects/) — SyncedParticle server object
- **Tier 2:** [../rendering/](../../rendering/) — Client particle rendering
