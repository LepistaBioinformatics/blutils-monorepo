# STATE — Project Memory

> Persistent memory across sessions: decisions, concerns and deferred ideas.
> Dates in absolute format.

## Decisions

- **2026-06-22** — Consensus classification algorithm documented *as-built* in
  `.specs/features/001-consensus-classification/{spec.md,design.md}`, via reverse engineering of the
  code in `modules/blutils/core`. No code was changed — documentation only.
- **2026-11-01** — Mapped the test chain of both modules (neither had functional tests).
  Artifacts: `.specs/codebase/TESTING.md` and the features `002-tests-blutils`, `003-tests-blutils-ui`,
  `004-tests-data-contract` (spec/design/tasks). No code changed.
- **2026-11-01 (default, TO BE CONFIRMED)** — Decisions D-01..D-04 in
  `.specs/features/002-tests-blutils/context.md`: bugs pinned by a characterization test + a disabled
  desired-behavior test; `[REFACTOR]` tasks out of the first wave; as-built spec not edited;
  contract golden generated in Rust and copied to the UI with a drift check in the monorepo.

## Open concerns (from the as-built design)

Full detail in `.specs/features/001-consensus-classification/design.md` §8.2.

- **CONCERN-01** — Classification error paths use `panic!`/`unwrap` instead of
  propagating `MappedErrors`; a panic in a parallel thread can take down the process.
  *Mostly addressed 2026-11-01 (DF-02: R03, R07, R08, R19, R23, R24, R29, R33, R35); CLI `cmds`
  still turn `Err` into `panic!` (exit 101 with message).*
- **CONCERN-02** — Identity cutoffs per taxon (`taxon.rs`) marked with `TODO: review`
  and with no cited scientific source — the most sensitive parameter of the algorithm.
  *Still open; R17 added Kingdom = 67.5 (midpoint Domain/Phylum), also unsourced (OP-02).*
- **CONCERN-03** — `bit_score` truncated from `Float64` to `i64`; affects grouping by
  bit-score. *Resolved 2026-11-01 (BUG-R14).*
- **CONCERN-04** — `find_single_query_consensus` evaluates only the highest bit-score group
  (always returns on the 1st loop iteration); confirm whether this is intentional.
  *Made explicit 2026-11-01 (BUG-R25): only the best group is processed, by design.*
- **CONCERN-05** — Positional column mapping in `fold_results_by_query` coupled to the
  schema order, with no compile-time protection.
- **CONCERN-06** — `fold_results_by_query` materializes the entire DataFrame and iterates outside the
  polars lazy engine.

## Code × as-built spec discrepancies (resolved 2026-11-01 — code fixed to match spec, see 005 DF-01; OP-01 resolved via DF-05)

- **FR-IDENT-03** × BUG-R01 — `get_rank_adjusted_by_identity` returns the 1st rank with cutoff ≥
  identity (one rank deeper). Confirmed by test.
- **AC-03** × BUG-R02 — single path never sets `mutated`/`maxAllowedRank`. Confirmed by test.
- **AC-05** × BUG-R06/R22 — `Relaxed` never evaluates ranks past the shortest lineage, then reports the reference record's full species via the single-match shortcut. Confirmed by test.

## High-severity bugs found during mapping

All fixed on 2026-11-01 with regression tests (R03, R04, R07, R08, R12, U01, U02, U06 and the rest).
Full registry with regression-test names: `002-tests-blutils/design.md` §2 (BUG-R01..R43) and
`003-tests-blutils-ui/design.md` §2 (BUG-U01..U22). All closed (U11 won't fix).

## Deferred todos

- Regenerate the stale zymo golden output with the fixed algorithm (OP-03; needs BLAST+).
- OP-02: source for the Kingdom cutoff (67.5) and the other built-in cutoffs (CONCERN-02).
- Evaluate mitigations for remaining CONCERN-02/05/06 (`001-consensus-classification/design.md` §8.3).

## Status 2026-11-01 — test suites and bug fixes (committed locally on `test/foundation`, not pushed)

- Test suites built per `002-tests-blutils` (phases 0–5) and `003-tests-blutils-ui` (phases 0–4),
  on local branch `test/foundation` in each submodule and in the monorepo. Committed locally, NOT pushed (user reviews locally).
- Results: Rust workspace 345 passed / 0 failed / 3 ignored (BLAST+ smoke); UI 269 passed / 1 skipped
  (BUG-U11); `cargo clippy --all-targets` 0 errors; `tsc` clean; `craco build` OK.
- Bugs fixed with regression tests: BUG-R01..R30, R32, R33, R35, R36 (34) and BUG-U01..U10, U12..U22 (21).
  Plan: `.specs/features/005-test-suite-bugfixes/` (decisions in `context.md`, open points OP-01..OP-04 in `tasks.md`).
- Follow-up round (same day): R31, R34 (warn), R37 fixed; U11 won't fix (UI JSON-only); OP-01
  (FR-IDENT-05 reworded) and OP-04 resolved; CI test jobs added (T5.6 `rust.yml`, U4.4 `deploy.yml`);
  contract feature 004 implemented (C1–C5) incl. monorepo `.github/workflows/contract.yml`.
- Latest results: Rust 350 passed / 0 failed / 3 ignored; UI 286 passed / 0 skipped.
- 2026-11-01: YAML support kept; `serde_yaml` (deprecated) replaced by `serde_yaml_ng` 0.10 (DF-04).
- 2026-11-01: first end-to-end run with real BLAST+ 2.17 (NCBI 16S + current taxdump + 9 zymo SRA
  runs, local untracked `e2e/`). Composition matches the ZymoBIOMICS mock; the smoke tests ran for the
  first time (test helper fixed: mock lineages may stop above species). Findings R38–R43 fixed (F7,
  DF-07). Rust 356 passed / 0 failed / 3 ignored; smoke 3/3 with BLAST+. Rebuilt taxonomy DB identical
  to the previous one; full zymo rerun identical except for the R38 `taxonomy` prefix.

## Preferences

- Artifact language: English (see `.claude/rules/english-only.md`, 2026-11-01). All existing `.specs` docs translated on 2026-11-01.
