# File-backed process store

This workspace uses the **file-backed** requirement store. Authoritative process
records live under `file-state/` and are versioned in git.

- **Store root:** `file-state/`
- **Canonical shapes:** `.modernpath/rdd/file-state/` (templates only; not the store)
- **Store-backed mode:** not active (`process/store-backed.md` is absent)

Agents read and write ledger files directly. Do not recreate retired
store-backed projections or use `modernpath author` as the write path for
requirement lifecycle here.
