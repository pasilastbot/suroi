# Dev Questions — Gap Analysis & Improvement Plan

<!-- @updated: 2026-03-04 -->

## Summary

30 dev question files exist as stubs (title + tags only, no answers).
The documentation covers the system well at a **descriptive/reference level**
but almost entirely lacks **prescriptive how-to content**.

Developers can find what objects look like, but not how to add them.

---

## Gap Classification

### Tier B — Answers Already in Docs (just needs synthesizing into question file)

These questions are well-enough answered by existing documentation.
The question files just need to be filled in.

| Q# | Question | Primary Doc(s) |
|----|----------|----------------|
| Q04 | Protocol version bump | `docs/protocol.md` § Version Management, `docs/development.md` § Protocol Version |
| Q05 | Create server plugin | `docs/subsystems/plugins/modules/creating-plugins.md` (has code example) |
| Q09 | Dirty tracking & UpdatePacket | `docs/subsystems/game-loop/modules/tick-serialization.md` (comprehensive) |
| Q11 | Add / modify loot table | `docs/subsystems/map/modules/loot-tables.md` (data flow + types) |
| Q12 | Run locally | `docs/development.md` § Initial Setup (step-by-step) |
| Q21 | Layer system | `docs/datamodel.md` § Layer System + `docs/subsystems/rendering/modules/camera.md` |

### Tier A — Needs How-To Content (synthesize from 2–3 docs into a walkthrough)

These docs contain the relevant reference material but no step-by-step guide.
The question files need answers that synthesize across multiple documents.

| Q# | Question | Gap | Docs to Synthesize |
|----|----------|-----|--------------------|
| Q01 | Add a new gun | No step-by-step | `items.md`, `patterns.md`, `protocol.md` |
| Q02 | Add a new obstacle | No step-by-step | `obstacles.md`, `loot-tables.md`, `protocol.md` |
| Q07 | Add a new perk | No walkthrough | `items.md`, `patterns.md` (WearerAttributes), `inventory/README.md` |
| Q08 | Add a new throwable | No walkthrough | `items.md`, `explosions.md`, `weapons.md` |
| Q10 | Add / modify a map | Missing practical steps | `generation.md`, `modes.md`, `spritesheets.md` |
| Q13 | Add a new packet type | No step-by-step | `packets/README.md`, `protocol.md` |
| Q14 | Add a new timed player action | No walkthrough | `actions.md` |
| Q16 | Add a translation string | No step-by-step | `translations.md` |
| Q17 | Add a console command | No step-by-step | `commands.md` |
| Q18 | Add a cvar | No step-by-step | `cvars.md` |
| Q20 | Add a new emote | No step-by-step | `emotes.md` |
| Q22 | Add a new explosion type | No step-by-step | `explosions.md`, `bullets.md` |
| Q23 | Add a new melee weapon | No walkthrough | `items.md`, `weapons.md` |
| Q24 | Add a new skin | No walkthrough | `items.md`, `spritesheets.md` |
| Q27 | Add a new decal | No step-by-step | `decals.md`, `explosions.md` |
| Q30 | WearerAttributes | Concept exists, no usage guide | `patterns.md`, `datamodel.md`, `inventory/README.md` |

### Tier A+ — Needs New Documentation (significant gaps in existing Tier 3 docs)

These questions cannot be answered from existing docs alone. New Tier 3 docs
or major enrichment of existing docs is required.

| Q# | Question | Missing Doc | Notes |
|----|----------|-------------|-------|
| Q03 | Add a new building | Buildings doc is minimal; building definitions are complex (layers, walls, floors, doors, puzzles) | Needs a detailed walkthrough |
| Q06 | Add a new game mode | No end-to-end guide coordinating `modes.ts`, `maps.ts`, loot tables, gas stages, spritesheet variants | Cross-subsystem, high complexity |
| Q15 | Add a UI element to HUD | `hud.md` describes existing elements; missing: where to add new HTML/Svelte, hook into UpdatePacket, Svelte vs jQuery | Svelte 5 not documented anywhere |
| Q19 | Add a new player stat | `server-objects.md` is thin; missing: where to add field on `Player`, how to add to UpdatePacket PlayerData serialization, how to receive on client, display in UI | Touches 3 packages |
| Q25 | Airdrop system | Airdrop info is scattered (tick order step 2, gas stages `summonAirdrop`, `datamodel.md` constants); no dedicated doc covering the full flow | New T3 doc needed |
| Q26 | Bullet ballistics end-to-end | `bullets.md` covers definition fields; missing: fire → physics → hit detection → damage → tracer render | New T3 doc needed |
| Q28 | Debug rendering issues | No practical debugging guide; PixiJS devtools, texture name lookup, z-index, `_missing_texture` | New T3 doc needed |
| Q29 | Profile server performance | Only mentions `bun stressTest`; missing: Bun profiler, timing instrumentation, identifying slow ticks | New T3 doc needed |

---

## Improvement Roadmap

### Phase 1 — Fill Tier B (immediate, low effort)
Fill Q04, Q05, Q09, Q11, Q12, Q21 by synthesizing existing docs into the question files.
**Effort:** Low. Existing content, just needs writing.

### Phase 2 — Fill Tier A How-Tos (medium effort)
Fill Q01, Q02, Q07, Q08, Q10, Q13, Q14, Q16, Q17, Q18, Q20, Q22, Q23, Q24, Q27, Q30
by writing step-by-step walkthroughs that synthesize across the existing reference docs.
**Effort:** Medium. Requires reading source code for accurate file paths and field names.

### Phase 3 — Create Missing Tier 3 Docs (high effort)
For Q03, Q06, Q15, Q19, Q25, Q26, Q28, Q29:
1. Explore source code for each area
2. Create new Tier 3 docs:
   - `docs/subsystems/definitions/modules/buildings.md` → major enrichment
   - `docs/dev-questions/` or inline answers for Q06, Q15, Q19
   - `docs/subsystems/game-loop/modules/airdrops.md` → new doc for Q25
   - `docs/subsystems/objects/modules/bullet-system.md` → new doc for Q26
   - `docs/subsystems/rendering/modules/debug.md` → new doc for Q28
   - `docs/subsystems/game-loop/modules/profiling.md` → new doc for Q29
3. Update `content-plan.md`
**Effort:** High. Requires source code exploration.

---

## Coverage Scorecard

| Status | Count | Questions |
|--------|-------|-----------|
| Well covered (Tier B) | 6 | Q04, Q05, Q09, Q11, Q12, Q21 |
| Needs synthesizing (Tier A) | 16 | Q01, Q02, Q07, Q08, Q10, Q13, Q14, Q16, Q17, Q18, Q20, Q22, Q23, Q24, Q27, Q30 |
| Needs new docs (Tier A+) | 8 | Q03, Q06, Q15, Q19, Q25, Q26, Q28, Q29 |
| **Total** | **30** | |

**Answerable from existing docs today:** 6/30 (20%)
**Answerable after Phase 1+2:** 22/30 (73%)
**Fully answered after Phase 3:** 30/30 (100%)
