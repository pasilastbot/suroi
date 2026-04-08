# Validation — validateDefinitions Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/validation/README.md -->
<!-- @source: tests/src/validateDefinitions.ts, tests/src/validationUtils.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the definition validation pipeline: schema checks, cross-references, idString uniqueness, and loot table validation. Run before committing to catch definition errors.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file tests/src/validateDefinitions.ts | Main validation entry — gas, loot tables, definitions | High |
| @file tests/src/validationUtils.ts | `tester`, `validators`, `findDupes`, `safeString` | Medium |

## Validation Order

1. **Gas stages** — duration, oldRadius, newRadius, dps, summonAirdrop
2. **Loot tables** — Presence per mode, table references, weighted items
3. **Definitions** — All ObjectDefinitions collections:
   - idString uniqueness
   - Required fields
   - Cross-references (e.g. gun.ammoType exists in Ammos)
   - No circular refs, valid types

## Tester API

- `tester.assert(condition, message, path)` — Fail if false
- `tester.assertIsPositiveReal`, `assertIsRealNumber`, etc.
- `tester.assertNoPointlessValue` — Flag redundant defaults
- `tester.runTestOnArray` — Iterate with error path
- `logger.indent` — Nested log output

## Business Rules

- Each definition collection validated for schema + cross-refs
- Loot tables must reference valid item idStrings
- Run `bun validateDefinitions` after definition changes

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Validation overview
- **Tier 2:** [../definitions/](../../definitions/) — What gets validated
