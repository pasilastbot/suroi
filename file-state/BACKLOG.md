# Backlog / gap flat-file state

- **Snapshot at:** 2026-10-06T10:15:00Z
- **Source store/revision:** git suroi@5f67ec7f8abd6ee84c39ad739f1f406d9f23e0dc (dirty)

## BACKLOG-ONBOARD-001 — Remaining subsystem reverse-engineering sweeps

- **Kind:** backlog
- **Disposition:** CLOSED — routed to full sweep publication 2026-10-06
- **Source:** USER:2026-10-06:full-codebase-reverse-engineer-sweep

## BACKLOG-ONBOARD-003 — Vite/editor/news entrypoints uncited

- **Kind:** backlog
- **Disposition:** OPEN
- **Observed:** 11 TypeScript files under `client/vite/**`, editor, and news remain uncited (build/auxiliary entrypoints).
- **Why unrouted:** Runtime-game disposition target met without them; cite in a build-pipeline pass or document as permanent exclusions.
- **Source:** file-state/coverage/SUROI-ONBOARD-source-disposition.json

## BACKLOG-ONBOARD-002 — Per-feature and edge-case SR depth

- **Kind:** backlog
- **Disposition:** OPEN
- **Observed:** Full sweep captured subsystem-level URs/SRs aligned to Tier-2 docs. Not every route, perk, map hook, or editor surface has its own SR (e.g. hitbox editor, editor/index.html, individual perks, emote/mapIndicator minor objects).
- **Why unrouted:** Baseline granularity is subsystem/architecture level; deeper rows belong to targeted reverse-engineer passes or normal delivery Epics.
- **Consequence:** Behavioral coverage denominator for “every public entry point” is not claimed.
- **Source:** DOC:docs/content-plan.md; DOC:docs/adding-content.md

## GAP-CHAR-001 — Niinistö spec vs implementation modifier mismatch

- **Kind:** gap
- **Gap-kind:** behavior / spec drift
- **Affected trace:** UR-CHAR-001, SR-CHAR-001
- **Observed:** `specs/features/character-selection.md` lists Niinistö `maxHealth ×0.92`; `common/src/definitions/items/skins.ts` implements `adrenDrain: 0.85` only.
- **Disposition:** OPEN
- **Consequence:** UR scenario S4 and product spec may disagree until reconciled in code or spec.
- **Source:** CODE:common/src/definitions/items/skins.ts; DOC:specs/features/character-selection.md

## GAP-VERIFY-001 — Full baseline not eligible for accept

- **Kind:** gap
- **Gap-kind:** evidence
- **Affected trace:** All 83 PENDING_VERIFICATION baseline rows
- **Observed:** GATE-VERIFY-TRACE-001 FAIL (packet rev 2) — 22/64 SR with unit/script runs; 0/19 UR upper proof; dirty tree.
- **Disposition:** OPEN
- **Consequence:** Use partial accept (5–7 test-backed SRs) with disclosed limits, or add E2E/integration tests then re-verify.
- **Source:** DOC:file-state/verify/VERIFY-SUROI-BASELINE-2026-10-06.json; DOC:file-state/GATES.md#GATE-VERIFY-TRACE-001

## GAP-TST-001 — stressTest not executed in onboarding evidence

- **Kind:** gap
- **Gap-kind:** evidence
- **Affected trace:** SR-TST-004
- **Observed:** stressTest requires a running server; no RUN captured during reverse-engineer pass.
- **Disposition:** OPEN
- **Source:** CODE:tests/src/stressTest.ts
