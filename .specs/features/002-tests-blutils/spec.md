# Test Suite — `modules/blutils` (Rust) — Specification

> Status: **Draft** · Date: 2026-11-01
> Decisions: `context.md` (D-01 bug policy, D-02 refactoring, D-03 as-built spec).
> Infra/gates: `.specs/codebase/TESTING.md`.

## Problem Statement

The `blutils` engine (≈5.2k lines, 3 crates) has no tests at all: zero `#[test]`, zero
`[dev-dependencies]`. The consensus algorithm — the project's scientific asset — has already been documented
as-built (`001-consensus-classification`), but no acceptance criterion (AC-01..AC-05) is verified
automatically, and the mapping found input-reachable panics and divergences between code and
spec. Any change today is made blindly.

## Goals

- [ ] All FR-*/AC-* of `001-consensus-classification/spec.md` covered by at least one traceable test.
- [ ] Each `BUG-Rxx` in the design with a characterization test + desired test `#[ignore]` (D-01).
- [ ] `cargo test` green without BLAST+ installed; smoke with real BLAST+ available via `--ignored`.
- [ ] Tests running on `push`/`pull_request` in the submodule's CI.

## Out of Scope

| Item | Reason |
|---|---|
| Fixing the bugs found | D-01: tests pin; fixing is its own feature |
| Benchmarks / performance (CONCERN-06) | not a functional correction |
| `stdin` tests beyond one case | global static `STDIN_HAS_BEEN_USED` |
| Fuzzing / property-based | may come later; there is no `proptest` in the project |
| Seams for subprocesses (`[REFACTOR]`) | D-02, awaiting decision |
| Dead code (`QuerySequence`, `settings.rs`, `ordered_iter`) | record only |

---

## User Stories

### P1: Verified classification core ⭐ MVP

**User Story**: As the algorithm maintainer, I want interpolated identity, per-taxon cutoffs,
the single path and the multi (LCA) path to be tested with synthetic lineages, so that I can change the
consensus knowing exactly what changes.

**Acceptance Criteria**:

1. WHEN `cargo test -p blutils-core --lib` runs THEN there SHALL be tests for each FR-IDENT-*, FR-SCORE-*, FR-SINGLE-*, FR-MULTI-*, FR-BEAN-* of the as-built spec.
2. WHEN a test depends on `HashMap`/`rayon` order THEN it SHALL sort before comparing (no flakiness over 20 consecutive runs).
3. WHEN the code diverges from the as-built spec (FR-IDENT-03, AC-03, AC-05) THEN the test SHALL pin the current behavior and reference the discrepancy (D-03).
4. WHEN a known input causes `panic!` (CONCERN-01) THEN there SHALL be a `#[should_panic(expected = …)]`.

**Independent Test**: `cargo test -p blutils-core --lib build_consensus_identities linnaean taxon`.

### P1: End-to-end consensus pipeline without BLAST+ ⭐ MVP

**User Story**: As a maintainer, I want to run `build_consensus_identities` and
`run_blast_and_build_consensus` (with `ExecuteBlastn` mocked) over synthetic fixtures, to validate
the taxonomy join, reconciliation of queries without hits, and output writing.

**Acceptance Criteria**:

1. WHEN `core/tests/` runs THEN it SHALL use only `tempdir()` and fixtures in `core/tests/fixtures/`.
2. WHEN `headers` contains a query with no row in the TSV THEN the result SHALL contain `NoConsensusFound` for it (AC-01).
3. WHEN `write_blutils_output` writes in Json/Jsonl/Yaml THEN the file SHALL be read back and have the expected fields (contract shape).
4. WHEN the `ExecuteBlastn` mock receives 120 sequences THEN it SHALL be called 3 times (chunks of 50).

### P1: Test foundation ⭐ MVP

**Acceptance Criteria**:

1. WHEN the `Cargo.toml` files are read THEN `core` and `ports/cli` SHALL have `[dev-dependencies]` with `tempfile` (+ `assert_cmd`, `predicates`, `serial_test` in cli).
2. WHEN a shared test helper exists THEN it SHALL be in `core/tests/common/mod.rs` (`ExecuteBlastn` mock, builders for `BlastResultRow`/`TaxonomiesMap`).

### P2: DTOs, serde and I/O

**User Story**: As a maintainer, I want to cover the DTOs' `FromStr`/`Display`/serde and the
`FileOrStdin` readers, because they define the contract with the UI and with the CLI itself (`build-tabular`).

**Acceptance Criteria**:

1. WHEN `LinnaeanRank`, `Taxon`, `Strand`, `OutputFormat` are (de)serialized THEN the values SHALL match the enum table in design §3.
2. WHEN `sequence_content` reads multi-line FASTA/empty lines/no header THEN it SHALL return the result documented in the design.
3. WHEN `parse_consensus_as_tabular` generates TSV THEN the test SHALL pin the current format (BUG-R10/R11).

### P2: Database builder (`build-db blu`)

**Acceptance Criteria**:

1. WHEN `build_taxonomy_database` receives 5 minimal `.dmp` + `accessions_map` THEN it SHALL generate `*.blutils.json` and `*.non-mapped.tsv` with verifiable lineages (deleted/merged/unknown, `replace_rank`, `drop_non_linnaean_taxonomies`, `ignore_taxids`).
2. WHEN each `.dmp` loader receives a valid/empty/short file THEN it SHALL have the documented behavior (including panics).

### P2: CLI

**Acceptance Criteria**:

1. WHEN `CliLauncher::<Opts>::command().debug_assert()` runs THEN it SHALL pass (no flag conflicts).
2. WHEN `blu blastn build-consensus` runs over fixtures THEN it SHALL produce JSON on stdout without BLAST+.
3. WHEN `--taxon custom` without `-c` THEN the process SHALL exit with 101 and stderr containing "Custom taxon values are required".

### P3: Subprocesses (fake PATH) and smoke with real BLAST+

**Acceptance Criteria**:

1. WHEN `PATH` points to fakes THEN `check_host_requirements`, `ExecuteBlastnProcRepository` and the `blastdbcmd` half of qiime/kraken SHALL be exercised (`#[serial]` tests).
2. WHEN `cargo test -- --ignored` runs with BLAST+ installed THEN it SHALL execute `makeblastdb` + `blu blastn run-with-consensus` over `test/mock/input/`.

### P3: CI

1. WHEN there is a `push`/`pull_request` in the submodule THEN `cargo test` SHALL run on GitHub Actions.

---

## Edge Cases (all become cases in the design)

- Identity below all cutoffs (single path) → panic (BUG-R08).
- Hits diverging already at the domain (index 0) → `usize` underflow (BUG-R03).
- `subject_taxid` missing from the taxonomy map → panic in a rayon thread (BUG-R07, hypothesis).
- Malformed lineage (`""`, `"d__2;"`, `"s__a__b"`, `";d__2"`) → `Err` in `parse_taxonomy`.
- `NaN` in `perc_identity` → panic in `partial_cmp().unwrap()`.
- Taxonomy with a single non-Linnaean rank → identity `NaN` (hypothesis).

## Requirement Traceability

| ID | Story | Status |
|---|---|---|
| TB-FOUND-01..02 | Foundation | Pending |
| TB-CORE-01..04 | Core (covers FR-IDENT/SCORE/SINGLE/MULTI/BEAN and AC-02..05) | Pending |
| TB-PIPE-01..04 | Pipeline (covers FR-IN, FR-JOIN, FR-NOHIT, FR-OUT, AC-01) | Pending |
| TB-DTO-01..03 | DTOs/serde/I/O | Pending |
| TB-DB-01..02 | build-db | Pending |
| TB-CLI-01..03 | CLI | Pending |
| TB-PROC-01..02 | Subprocess/smoke | Pending |
| TB-CI-01 | CI | Pending |

ID → task mapping in `tasks.md`.

## Success Criteria

- [ ] `cargo test` green on a machine without BLAST+, in < 60 s.
- [ ] 100% of AC-01..AC-05 with a test referencing the ID.
- [ ] All `BUG-R*` confirmed or refuted by a test (hypotheses resolved).
