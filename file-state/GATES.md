# Gate flat-file state

- **Snapshot at:** 2026-10-06T10:15:00Z
- **Source store/revision:** git suroi@5f67ec7f8abd6ee84c39ad739f1f406d9f23e0dc (dirty working tree)
- **Context / release:** Base (onboarding baseline; no delivery release selected)

## GATE-BASELINE-001 — Source-scoped baseline authorization (SUROI-ONBOARD-2026-10-06)

- **Kind:** human / baseline authorization
- **Transition / purpose:** Authorize publication of grounded as-built URs/SRs into Base as `PENDING_VERIFICATION` for run `SUROI-ONBOARD-2026-10-06`
- **Exact scope:** Repository key `suroi`; roots `common/`, `server/`, `client/`, `tests/`, `docs/`; full Tier-2 subsystem sweep (extended USER:2026-10-06:full-codebase-reverse-engineer-sweep — see `file-state/sources/SUROI-ONBOARD-2026-10-06.json`)
- **Prerequisites:** none
- **Fingerprint:** onboarding-inventory@5f67ec7+dirty26; corpus-empty-at-start
- **State:** ANSWERED
- **Verdict / answer:** Approve **baseline ready for use** (empty corpus → PENDING_VERIFICATION rows in Base)
- **Actor / evaluator:** USER:2026-10-06:file-backed-ledger-and-reverse-engineer-baseline
- **Sources:** USER:2026-10-06:file-backed-ledger-and-reverse-engineer-baseline; DOC:process/file-backed.md
- **Timestamps:** opened 2026-10-06T08:40:00Z; answered 2026-10-06T08:45:00Z; closed 2026-10-06T08:45:00Z
- **Application:** APPLIED at git suroi@5f67ec7
- **Predecessor / successor:** none / none

### Brief

**Brief:**
- What: Run reverse-engineering in file-backed mode and publish an initial as-built requirement baseline.
- Why now: No prior requirement corpus; adopt RDD with local `file-state/` ledger.
- Changes if approved: New UR/SR rows in `file-state/REQUIREMENTS.md` at `PENDING_VERIFICATION` with CODE/TEST citations.
- Risk if wrong: Over-broad or inaccurate statements until verify/accept passes; reversible by demotion or correction.
- Recommendation: Baseline mode (not DERIVED-only additions).

### Holds

- **Held items:** none
- **Applied transitions:** n/a (baseline publication, not lifecycle advance to DONE)

---

## GATE-VERIFY-TRACE-001 — Baseline as-built verification eligibility (full scope)

- **Kind:** trace / as-built verification
- **Transition / purpose:** Evaluate whether complete proof exists to open human baseline acceptance for all `PENDING_VERIFICATION` rows from SUROI-ONBOARD-2026-10-06
- **Exact scope:** 19 UR + 64 SR in `file-state/REQUIREMENTS.md` with GATE-BASELINE-001 provenance
- **Prerequisites:** GATE-BASELINE-001 PASS (applied)
- **Fingerprint:** proof_digest=571d1aeff59c67b9aa21e03b… (packet revision 3); VERIFY-SUROI-BASELINE-2026-10-06
- **State:** FAIL
- **Verdict / answer:** **Not eligible** for full-scope accept. Lower proof: **30/64 SR** tagged in unit tests, **131** `bun test` passes + validate scripts. Remaining: **0/19 UR** upper proof; **33 SR** without tests (UI/REN/CLI/COV disposition, LOOP-001/002, MAP-001, etc.); dirty tree; SR-TST-004; GAP-CHAR-001.
- **Actor / evaluator:** verify revision 3 — 2026-10-06
- **Sources:** RUN:bun test tests/src (131 pass); RUN:validate*; DOC:file-state/verify/VERIFY-SUROI-BASELINE-2026-10-06.json; DOC:tests/src/requirements/README.md
- **Timestamps:** evaluated 2026-10-06T11:10:00Z (revision 3)
- **Application:** NOT_APPLICABLE (trace gate)
- **Predecessor / successor:** none / GATE-ACCEPT-BASELINE-001 (may not OPEN until trace PASS or scoped subset)

### Holds

- **Held items:** all 83 baseline requirements remain `PENDING_VERIFICATION`
- **Applied transitions:** none

---

## GATE-ACCEPT-BASELINE-001 — Human acceptance PENDING_VERIFICATION → DONE (full baseline)

- **Kind:** human / existing-baseline acceptance
- **Transition / purpose:** One human decision to promote verified as-built scope to DONE per PROCESS.md §As-built verification and acceptance
- **Exact scope:** Full baseline (blocked — see GATE-VERIFY-TRACE-001)
- **Prerequisites:** GATE-VERIFY-TRACE-001 PASS
- **Fingerprint:** tied to VERIFY-SUROI-BASELINE-2026-10-06 proof_digest
- **State:** DRAFT
- **Verdict / answer:** not opened — prerequisites unmet
- **Actor / evaluator:** —
- **Sources:** DOC:file-state/verify/VERIFY-SUROI-BASELINE-2026-10-06.json
- **Timestamps:** —
- **Application:** NOT_APPLICABLE
- **Predecessor / successor:** GATE-VERIFY-TRACE-001 / none
