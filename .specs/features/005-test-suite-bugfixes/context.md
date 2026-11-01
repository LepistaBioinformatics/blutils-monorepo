# Bug Fixes from the Test-Suite Mapping — Context (user decisions)

> Spec: `spec.md`

- **DF-01** — Algorithm bugs are fixed so the code matches the as-built spec
  `001-consensus-classification` (FR-IDENT-03, AC-03, AC-05) and the evident intent
  (user decision, 2026-11-01). Classification outputs change.
- **DF-02** — Crash paths reachable from input become `Err` or `NoConsensusFound`
  instead of `panic!`.
- **DF-03** — U11: won't fix, the UI is JSON-only by design (user, 2026-11-01). R31: fix (numeric
  leaf uses the merged target). R34: warn the user. R37: activate the CLI aliases.
- **DF-04** — YAML support **kept**; the deprecated `serde_yaml` was replaced by the maintained
  drop-in fork `serde_yaml_ng` 0.10 (user decision, 2026-11-01, after a removal was implemented and
  then reverted). Chosen over `serde_yml` (RUSTSEC-2025-0068) and `serde_norway` (repository
  unavailable).
- **DF-05** — OP-01: FR-IDENT-05 reworded to "rank demoted by identity truncation" (user, 2026-11-01).
- **DF-06** — OP-04: invalid/wrong-shape bodies loaded via URL show the upload message (user, 2026-11-01).
