# Validation Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: tests/src/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Validation subsystem provides automated checks for game definitions and assets. Run before committing to catch schema errors, invalid references, and missing assets.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `tests/src/validateDefinitions.ts` | Validates all `ObjectDefinitions` — uniqueness, schema, cross-refs |
| `tests/src/validateSvgs.ts` | Validates SVG assets referenced by definitions exist and are well-formed |
| `tests/src/stressTest.ts` | Stress test — server load simulation |
| `tests/src/validationUtils.ts` | Shared validation helpers |
| `tests/src/math.test.ts` | Unit tests for `common/src/utils/math.ts` |

## Architecture

```
validateDefinitions
    → Iterate all definition collections
    → Check idString uniqueness
    → Check required fields
    → Check cross-references (e.g. gun.ammoType exists in Ammos)
    → Check for circular refs, invalid types

validateSvgs
    → For each definition referencing an SVG path
    → Verify file exists in client/public/img/
    → Validate SVG structure (optional)
```

## Commands

| Command | Purpose |
|---------|---------|
| `bun validateDefinitions` | Validate all definitions |
| `bun validateSvgs` | Validate SVG assets |
| `cd tests && bun stressTest` | Run stress test |
| `cd tests && bun test` | Run unit tests |

## Module Index (Tier 3)

- [validateDefinitions](modules/validate-definitions.md) — Schema, cross-refs, loot tables
- [validateSvgs](modules/validate-svgs.md) — SVG existence, size limits, structure

## When to Run

- **After definition changes:** Always run `bun validateDefinitions`
- **After adding SVGs:** Run `bun validateSvgs`
- **CI:** Both run on every PR via GitHub Actions

## Protocol Considerations

- **Affects protocol:** No. Validation does not modify protocol; it catches definition errors that could cause protocol issues.

## Dependencies

- **Depends on:** Definitions (all collections), common utils
- **Depended on by:** CI, Development workflow

## Related Documents

- **Tier 1:** [docs/development.md](../../development.md) — Validation commands
- **Tier 2:** [../definitions/](../definitions/) — What gets validated
