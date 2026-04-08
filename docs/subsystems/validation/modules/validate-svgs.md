# Validation — validateSvgs Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/validation/README.md -->
<!-- @source: tests/src/validateSvgs.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents SVG asset validation: file existence, size limits, structure checks. Run after adding or modifying SVG assets to catch missing or oversized files.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file tests/src/validateSvgs.ts | SVG validation entry | Medium |

## Validation

- **Paths:** `client/public/img/game/`, `client/public/img/killfeed/` — all `.svg` files
- **Existence:** Definitions referencing SVG paths must have files present
- **Size limits:** Per-category `MAX_SIZES` (buildings 300k, weapons 150k, etc.)
- **Path length:** `MAX_PATH_LENGTH` (100k)
- **Structure:** svg-parser validates SVG structure

## MAX_SIZES (bytes)

| Category | Max |
|----------|-----|
| buildings | 300,000 |
| airdrop, equipment, weapons, obstacles | 150–200k |
| badges, emotes, skins, loot | 100k |
| particles, perks, explosions | 25k |

## Commands

- `bun validateSvgs` — Validate all
- `bun validateSvgs --fix` — Optional fixes (e.g. unminified files)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Validation overview
- **Tier 2:** [../definitions/](../../definitions/) — Definitions reference SVG paths
