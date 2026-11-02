# Bug Fixes from the Test-Suite Mapping — Tasks

> Spec: `spec.md` · Design: `design.md` · Status: **Done 2026-11-01** (local commits, not pushed)

## Work packages

| WP | Scope | Bugs | Depends on | Status |
|---|---|---|---|---|
| F1 | Consensus algorithm (`core/src/domain/dtos/linnaean_ranks.rs`, `taxon.rs`, `consensus_result.rs`, `use_cases/build_consensus_identities/*`) | R01, R02, R03, R05, R06, R07, R08, R13, R14, R17, R18, R21, R22, R23, R24, R25 | — | ✅ |
| F2 | Engine I/O and build-db (`core/src/use_cases/{write_blutils_output,parse_consensus_as_tabular,shared,run_blast_and_build_consensus,build_blutils_db_from_ncbi_files}`, `domain/dtos/{blast_builder,file_or_stdin}.rs`) | R04, R09, R10, R11, R15, R16, R26, R27, R28, R29, R30, R32, R33 | — | ✅ |
| F3 | CLI-facing (`taxon.rs` `CustomTaxon::from_file` API, `build_kraken_db_from_ncbi_files`, `ports/cli`) | R12, R19, R20, R35, R36 | F1 (`taxon.rs`), F2 (`run_parallel_blast.rs`, `shared/`) | ✅ |
| F4 | UI (`modules/blutils-ui/src`) incl. TS contract nullability | U01–U10, U12–U22 | UI suite complete | ✅ |
| F5 | Former deferred items | U11 (won't fix), R31, R34 (warn), R37 | DF-03 | ✅ |
| F6 | YAML library swap | DF-04 | — | ✅ |
| F7 | End-to-end findings (zymo + NCBI 16S run in `e2e/`) and panic sweep | R38, R39, R40, R41, R42, R43 | DF-07 | ✅ |

Gates: `cargo test --workspace`, `cargo clippy --all-targets`, `CI=true npx craco test --watchAll=false`,
`npx tsc --noEmit`, `CI=true npx craco build`.

## Open points / follow-ups

- **OP-01 — RESOLVED** (DF-05): FR-IDENT-05 reworded in `001` spec/design. After R01, the literal
  rule ("`mutated` when `reached_rank != max_allowed_rank`") yielded `mutated: false` for a single
  hit at 98%, contradicting AC-03.
- **OP-02 — open.** Built-in backbones now carry Kingdom = 67.5 (midpoint of Domain 60 / Phylum 75),
  a new scientific parameter without a cited source (see CONCERN-02).
- **OP-03 — open.** `modules/blutils/test/mock/output/zymo-mock/blutils.consensus.json` was produced
  by the old algorithm; its `maxAllowedRank`/`mutated` values no longer match. Regenerating needs BLAST+.
- **OP-05 — RESOLVED, won't implement** (DF-08). The e2e run (2026-11-01) shows 316/2283 ASVs
  assigned outside the mock, all where a non-mock reference scores higher (293
  `bacillus-mojavensis-subgroup`, ~1 nt from *B. subtilis/spizizenii*). Marker/reference limitation,
  not an algorithm bug; no bit-score margin is added.
- **OP-04 — RESOLVED** (DF-06): invalid/wrong-shape bodies via `?p=`/sample show the upload message;
  network/non-ok responses keep the load error.
