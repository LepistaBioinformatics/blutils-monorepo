# Bug Fixes from the Test-Suite Mapping — Specification (+ tasks inline)

> Status: **Done (except F5 deferred items)** · Date: 2026-11-01
> Source registries: `002-tests-blutils/design.md` §2 (BUG-R01..R34) and
> `003-tests-blutils-ui/design.md` §2 (BUG-U01..U21).
> Supersedes D-01 for the bugs listed here: the user authorized fixing them (2026-11-01).

## Decisions

- **DF-01** — Algorithm bugs are fixed so the code matches the as-built spec
  `001-consensus-classification` (FR-IDENT-03, AC-03, AC-05) and the evident intent
  (user decision, 2026-11-01). Classification outputs change.
- **DF-02** — Crash paths reachable from input become `Err` or `NoConsensusFound`
  instead of `panic!`.
- **DF-03** — U11: won't fix, the UI is JSON-only by design (user, 2026-11-01). R31: fix (numeric leaf uses the merged target). R34: warn the user. R37: activate the CLI aliases.
- **DF-04** — YAML support **kept**; the deprecated `serde_yaml` was replaced by the maintained drop-in fork `serde_yaml_ng` 0.10 (user decision, 2026-11-01, after a removal was implemented and then reverted). Chosen over `serde_yml` (RUSTSEC-2025-0068) and `serde_norway` (repository unavailable).

## Regression-test rule

For every fixed bug, in the same change:
1. Delete the characterization test that pinned the buggy behavior.
2. Enable the desired-behavior test (`#[ignore = "BUG-…"]` / `test.skip("BUG-…")`) — it
   becomes the regression test. Keep the `BUG-xx` reference in a comment.
3. If no desired test existed, write one that fails on the old code and passes on the fix.
4. Mark the registry row `fixed` with the regression test name.

## Work packages

| WP | Scope | Bugs | Depends on |
|---|---|---|---|
| F1 ✅ | Consensus algorithm (`core/src/domain/dtos/linnaean_ranks.rs`, `taxon.rs`, `consensus_result.rs`, `use_cases/build_consensus_identities/*`) | R01, R02, R03, R05, R06, R07, R08, R13, R14, R17, R18, R21, R22, R23, R24, R25 | — |
| F2 ✅ | Engine I/O and build-db (`core/src/use_cases/{write_blutils_output,parse_consensus_as_tabular,shared,run_blast_and_build_consensus,build_blutils_db_from_ncbi_files}`, `domain/dtos/{blast_builder,file_or_stdin}.rs`) | R04, R09, R10, R11, R15, R16, R26, R27, R28, R29, R30, R32, R33 | — |
| F3 ✅ | CLI-facing (`taxon.rs` `CustomTaxon::from_file` API, `build_kraken_db_from_ncbi_files`, `ports/cli`) | R12 ✅, R20 ✅, R35 ✅, R36 ✅, R19 ✅ | — |
| F4 ✅ | UI (`modules/blutils-ui/src`) incl. TS contract nullability | U01–U10, U12–U22 | — (UI suite complete) |
| F5 ✅ | Former deferred items | U11 (won't fix), R31 ✅, R34 ✅ (warn), R37 ✅ | — |

## Success Criteria

- [x] `cargo test` green with zero `#[ignore = "BUG-…"]` left for fixed bugs.
- [x] `CI=true npx craco test --watchAll=false` green with zero `test.skip("BUG-…")` left for fixed bugs.
- [x] `cargo clippy --all-targets` passes without `-A clippy::unused_io_amount` (R26).
- [ ] As-built spec `001-consensus-classification` and STATE discrepancies marked resolved.

## Open points raised by the fixes

- **OP-01 — FR-IDENT-05 vs AC-03 (spec inconsistency). RESOLVED 2026-11-01:** FR-IDENT-05 reworded in `001` spec/design to the implemented definition. After R01, the literal FR-IDENT-05 rule
  ("`mutated` when `reached_rank != max_allowed_rank`") yields `mutated: false` for a single hit
  at 98% (Genus == Genus), contradicting AC-03. Implemented: `mutated` = "rank demoted by identity
  truncation" on both paths. The as-built spec text must be reconciled (user decision).
- **OP-02 — Kingdom cutoff (R17).** Built-in backbones now carry Kingdom = 67.5 (midpoint of
  Domain 60 / Phylum 75). This is a new scientific parameter without a cited source (see CONCERN-02).
- **OP-03 — Stale golden output.** `modules/blutils/test/mock/output/zymo-mock/blutils.consensus.json`
  was produced by the old algorithm; its `maxAllowedRank`/`mutated` values no longer match.
- **OP-04 — Error text on the fetch path (U06/U07). RESOLVED 2026-11-01:** invalid/wrong-shape bodies via `?p=`/sample now show the upload message; network/non-ok keep the load error.
  Original note: A wrong-shape or non-ok body loaded via
  `?p=`/sample shows the load error ("Couldn't load …"); "isn't a valid Blutils result" is shown
  only for uploads. One-line change if the upload wording is preferred everywhere.
