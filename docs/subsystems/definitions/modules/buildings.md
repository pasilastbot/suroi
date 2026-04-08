# Definitions — Buildings Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/buildings.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

Buildings are multi-part structures (walls, floors, ceilings) that define map layout and provide cover. They support layers (basement, ground, upstairs) and can have doors, puzzles, and interactive elements.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `buildings.ts` | Building definitions, floor types, container tints | High |

## Business Rules

- Buildings define hitboxes for collision
- Buildings can have `orientation` (0 or 1) for rotation
- Floor types affect movement (normal, ice, water, etc.)
- Container tints (e.g. `ContainerTints`) customize building appearance per map variant

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 1:** [../../../datamodel.md](../../../datamodel.md) — Layer system
