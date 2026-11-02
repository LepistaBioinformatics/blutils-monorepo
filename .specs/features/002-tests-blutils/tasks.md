# Test Suite — `modules/blutils` — Tasks

**Spec**: `spec.md` · **Design**: `design.md` · **Context**: `context.md` · **Status**: Draft

> Here **the test is the deliverable**: each task = one test module. Gate commands in
> `.specs/codebase/TESTING.md`. All tasks run inside `modules/blutils`; commits go to the
> submodule repository and the pointer is updated in the monorepo at the end of each phase.
> `[REFACTOR]` tasks change production code and depend on decision D-02.

---

## Execution Plan

### Phase 0 — Foundation (sequential)
```
T0.1 → T0.2 ─┬→ (Phases 1–5)
              └→ T0.3 (leaf)
```

### Phase 1 — Pure domain (parallel)
```
        ┌→ T1.1 utils
        ├→ T1.2 linnaean_ranks ──┐
T0.2 ───┼→ T1.3 taxon            ├→ (Phase 2)
        ├→ T1.4 blast_builder    │
        ├→ T1.5 blast_result ────┤
        ├→ T1.6 taxonomy_bean    │
        ├→ T1.7 consensus_result ┘
        └→ T1.8 dtos serde
```

### Phase 2 — Consensus core (parallel after T1.2/T1.5/T1.7)
```
T1.2,T1.5,T1.7 ──┬→ T2.1 single ─┐
                 ├→ T2.2 multi   ├→ T2.4 build_consensus_identities (IT)
                 └→ T2.3 build_blast_consensus_identity ┘
```

### Phase 3 — I/O and use cases (parallel)
```
T0.2 ──┬→ T3.1 file_or_stdin (IT)
       ├→ T3.2 stdin_source (IT, dedicated binary)
       ├→ T3.3 shared/
       ├→ T3.4 write_blutils_output (IT)
       ├→ T3.5 parse_consensus_as_tabular (IT)   [dep T3.4]
       └→ T3.6 run_blast_and_build_consensus (IT) [dep T2.4]
```

### Phase 4 — build-db (parallel)
```
T0.2 ──┬→ T4.1 .dmp loaders
       ├→ T4.2 build_taxonomy_database [dep T4.1]
       └→ T4.3 generate_taxonomies_file
```

### Phase 5 — CLI, subprocesses, CI (sequential where marked)
```
T0.2 → T5.1 clap parsing [P]
T2.4 → T5.2 cli.rs assert_cmd
T0.2 → T5.3 host_tools.rs (serial) → T5.4 proc adapter (serial) → T5.5 smoke #[ignore]
T5.2 → T5.6 CI
```

---

## Task Breakdown

### T0.1 — Dev-dependencies
**What**: add `[dev-dependencies]` `tempfile` to `core/Cargo.toml` and `ports/cli/Cargo.toml`; `assert_cmd`, `predicates`, `serial_test` to `ports/cli/Cargo.toml` and `adapters/proc/Cargo.toml` (`tempfile`, `serial_test`).
**Where**: `core/Cargo.toml`, `ports/cli/Cargo.toml`, `adapters/proc/Cargo.toml`
**Depends on**: —  **Requirement**: TB-FOUND-01
**Done when**: `cargo test` compiles (0 tests) · `cargo clippy --all-targets` with no new errors.
**Tests**: none · **Gate**: build · **Commit**: `test: add dev-dependencies for test suite`

### T0.2 — Helpers and fixtures
**What**: `core/tests/common/mod.rs` (`MockBlastn`, `fake_blast_db`, fixture writing); `#[cfg(test)] pub(crate) mod test_utils` in `core/src/lib.rs` (builders `row`, `lineage`, `interp(taxon, ranks)`); fixtures from design §4; shell fakes in `ports/cli/tests/bin/`.
**Where**: `core/tests/common/`, `core/tests/fixtures/`, `core/src/test_utils.rs`, `ports/cli/tests/bin/`
**Depends on**: T0.1 · **Requirement**: TB-FOUND-02
**Done when**: a smoke test `common_fixtures_load` reads `basic.tsv` and `basic.blutils.json` without error.
**Tests**: integration · **Gate**: full

### T0.3 — Zymo fixture sentinel test
**What**: `core/tests/fixture_zymo.rs`: R04 characterization (`json_content::<BlutilsOutput>` fails with `isConfig`) + `#[ignore = "BUG-R04"]` expecting success.
**Depends on**: T0.2 · **Requirement**: TB-DTO-01 · **Tests**: integration · **Gate**: full

### T1.1 — `domain/utils` [P]
Cases: design §3.1. **Where**: `core/src/domain/utils/mod.rs` · **Tests**: unit · **Gate**: quick · **Req**: TB-CORE-01

### T1.2 — `InterpolatedIdentity` and `LinnaeanRank` [P]
Cases: design §3.2 (includes R01, R05, R17, R18 with characterization/`#[ignore]` pair).
**Where**: `core/src/domain/dtos/linnaean_ranks.rs` · **Req**: TB-CORE-01 (FR-IDENT-01..04)
**Done when**: ≥ 25 tests; `1e-9` tolerance on floats; quick gate green.
**Tests**: unit · **Gate**: quick · **Commit**: `test(core): cover identity interpolation and rank parsing`

### T1.3 — `Taxon` / `CustomTaxon` [P]
Cases: design §3.3 (R19). **Where**: `core/src/domain/dtos/taxon.rs` · **Tests**: unit (tempdir) · **Gate**: quick · **Req**: TB-CORE-01

### T1.4 — `BlastBuilder` / `Strand` [P]
Cases: design §3.4. **Where**: `core/src/domain/dtos/blast_builder.rs` · **Tests**: unit · **Gate**: quick · **Req**: TB-DTO-01

### T1.5 — `parse_taxonomy` [P]
Cases: design §3.5. **Where**: `core/src/domain/dtos/blast_result.rs` · **Tests**: unit · **Gate**: quick · **Req**: TB-CORE-02

### T1.6 — `TaxonomyBean` [P]
Cases: design §3.6. **Where**: `core/src/domain/dtos/taxonomy_bean.rs` · **Tests**: unit · **Gate**: quick · **Req**: TB-CORE-02

### T1.7 — `ConsensusBean::fold_consensus_list` [P]
Cases: design §3.7 (FR-BEAN-01, R13). **Where**: `core/src/domain/dtos/consensus_result.rs` · **Tests**: unit · **Gate**: quick · **Req**: TB-CORE-03

### T1.8 — Serde of the remaining DTOs [P]
Cases: design §3.9. **Where**: files in `core/src/domain/dtos/` and `domain/entities/execute_blastn.rs` · **Tests**: unit · **Gate**: quick · **Req**: TB-DTO-01

### T2.1 — `find_single_query_consensus` [P]
Cases: design §3.11 (FR-SCORE-01..03, FR-SINGLE-*, AC-02, AC-03/R02, R08).
**Where**: `core/src/use_cases/build_consensus_identities/find_single_query_consensus.rs` (`mod tests`)
**Depends on**: T1.2, T1.5, T1.7 · **Req**: TB-CORE-02 · **Tests**: unit · **Gate**: quick
**Commit**: `test(core): cover single-hit consensus path`

### T2.2 — `find_multi_taxa_consensus` [P]
Cases: design §3.11 (FR-MULTI-01..05, AC-04, AC-05/R06, R03, NaN, `Literal`).
**Where**: `…/find_multi_taxa_consensus.rs` · **Depends on**: T1.2, T1.5, T1.7 · **Req**: TB-CORE-03 · **Tests**: unit · **Gate**: quick
**Commit**: `test(core): cover multi-hit LCA consensus`

### T2.3 — `build_blast_consensus_identity` [P]
Cases: design §3.11 (FR-MULTI-06, FR-IDENT-05, FR-BEAN-02, index panic).
**Where**: `…/build_blast_consensus_identity.rs` · **Depends on**: T1.2 · **Req**: TB-CORE-04 · **Tests**: unit · **Gate**: quick

### T2.4 — `build_consensus_identities` (integration)
Cases: design §3.11 "Integration" block (AC-01, FR-IN-05, FR-JOIN-01..04, FR-NOHIT-01/02, R07, R14).
**Where**: `core/tests/build_consensus_identities.rs` · **Depends on**: T0.2, T2.1, T2.2 · **Req**: TB-PIPE-01, TB-PIPE-02 · **Tests**: integration · **Gate**: full
**Done when**: 20 consecutive runs (`for i in $(seq 20); do cargo test -p blutils-core --test build_consensus_identities || break; done`) without failure.

### T3.1 — `FileOrStdin` (file) [P]
Cases: design §3.8 except stdin (R09). **Where**: `core/tests/file_or_stdin.rs` · **Tests**: integration · **Gate**: full · **Req**: TB-DTO-02

### T3.2 — `Source::Stdin` (dedicated binary) [P]
A single `#[test]`. **Where**: `core/tests/stdin_source.rs` · **Tests**: integration · **Gate**: full · **Req**: TB-DTO-02

### T3.3 — `use_cases/shared` [P]
Cases: design §3.13. **Where**: files in `core/src/use_cases/shared/` · **Tests**: unit (tempdir) · **Gate**: quick · **Req**: TB-PIPE-03

### T3.4 — `write_blutils_output` [P]
Cases: design §3.10. **Where**: `core/tests/write_blutils_output.rs` · **Tests**: integration · **Gate**: full · **Req**: TB-PIPE-03

### T3.5 — `parse_consensus_as_tabular`
Cases: design §3.14 (R10, R11). **Where**: `core/tests/parse_consensus_as_tabular.rs` · **Depends on**: T3.4 · **Tests**: integration · **Gate**: full · **Req**: TB-DTO-03

### T3.6 — `run_blast_and_build_consensus` with `MockBlastn`
Cases: design §3.12. **Where**: `core/tests/run_blast_and_build_consensus.rs` · **Depends on**: T2.4 · **Tests**: integration · **Gate**: full · **Req**: TB-PIPE-04

### T4.1 — `.dmp` loaders [P]
Cases: design §3.15 (loaders, R15). **Where**: `load_*.rs` files in `core/src/use_cases/build_blutils_db_from_ncbi_files/` · **Tests**: unit (tempdir) · **Gate**: quick · **Req**: TB-DB-02

### T4.2 — `build_taxonomy_database`
Cases: design §3.15 (R16). **Where**: `…/build_taxonomy_database.rs` · **Depends on**: T4.1 · **Tests**: unit (tempdir) · **Gate**: quick · **Req**: TB-DB-01

### T4.3 — `generate_taxonomies_file` [P]
Cases: design §3.16 (fs-only part). **Where**: `core/src/use_cases/build_kraken_db_from_ncbi_files/generate_taxonomies_file.rs` · **Tests**: unit · **Gate**: quick · **Req**: TB-DB-01

### T5.1 — Clap parsing [P]
Cases: design §3.17 "Parsing". **Where**: `ports/cli/src/main.rs` (`mod tests`) · **Tests**: unit · **Gate**: `cargo test -p blutils-cli --bins` · **Req**: TB-CLI-01

### T5.2 — `blu` via `assert_cmd`
Cases: design §3.17 "Process". **Where**: `ports/cli/tests/cli.rs` · **Depends on**: T0.2, T2.4 · **Tests**: e2e · **Gate**: full · **Req**: TB-CLI-02, TB-CLI-03

### T5.3 — Host tools via fake PATH
Cases: design §3.16 + §3.17 "Fake PATH" (R12). **Where**: `ports/cli/tests/host_tools.rs` · **Depends on**: T0.2 · **Tests**: e2e · **Gate**: full · **Req**: TB-PROC-01
**Done when**: `PATH` passed only to the child process via `assert_cmd` `.env("PATH", …)` — without mutating the test process environment (no need for `#[serial]`).

### T5.4 — `ExecuteBlastnProcRepository` (serial)
Cases: design §3.18 (fake). **Where**: `adapters/proc/tests/execute_blast.rs` (`#[serial]` — the test process itself resolves `blastn` via `PATH`) · **Depends on**: T5.3 · **Tests**: e2e · **Gate**: full · **Req**: TB-PROC-01

### T5.5 — Real BLAST+ smoke `#[ignore]`
Cases: design §3.18 (real). **Where**: `ports/cli/tests/smoke_blast.rs` · **Depends on**: T5.4 · **Tests**: e2e · **Gate**: `cargo test -- --ignored` (manual) · **Req**: TB-PROC-02

### T5.6 — CI
**What**: enable `push`/`pull_request` in `.github/workflows/rust.yml` in a separate `test` job (`cargo test`), keeping `publish-crate` only on `release`.
**Depends on**: T5.2 · **Tests**: none · **Gate**: build · **Req**: TB-CI-01

### `[REFACTOR]` (out of the first batch — D-02)
- **TR.1** trait `HostCommandRunner` in `core/src/domain/entities/` for `check_host_requirements`, `build_accessions_map`, `generate_fasta_file` and the qiime half → allows replacing T5.3 with in-process tests.
- **TR.2** convert input-reachable `panic!`/`process::exit` into `MappedErrors` (CONCERN-01) → turns `#[should_panic]` into `Err` asserts.

---

## Validation

### Check 1 — Granularity
| Task | A single test module/file? | Status |
|---|---|---|
| T0.1 | 3 `Cargo.toml`, one cohesive change (deps) | ✅ |
| T0.2 | helpers + fixtures (single infra) | ✅ |
| T1.1–T1.8, T2.1–T2.3, T3.3, T4.1–T4.3 | one source file each (T1.8/T3.3/T4.1: same directory, same pattern) | ✅ |
| T2.4, T3.1–T3.6, T5.1–T5.5 | one test file each | ✅ |
| T5.6 | one workflow | ✅ |

### Check 2 — Diagram × `Depends on`
| Task | Depends on | In the diagram | Status |
|---|---|---|---|
| T0.2 | T0.1 | T0.1 → T0.2 | ✅ |
| T0.3 | T0.2 | T0.2 → T0.3 (leaf) | ✅ |
| T1.*, T3.*, T4.*, T5.1 | T0.2 (implicit via phase) | T0.2 → phases | ✅ |
| T2.1, T2.2 | T1.2, T1.5, T1.7 | same | ✅ |
| T2.3 | T1.2 | included in T1.2,T1.5,T1.7 → T2.3 | ✅ |
| T2.4 | T0.2, T2.1, T2.2 | T2.1, T2.2 → T2.4 | ✅ |
| T3.5 | T3.4 | annotated | ✅ |
| T3.6 | T2.4 | annotated | ✅ |
| T4.2 | T4.1 | annotated | ✅ |
| T5.2 | T0.2, T2.4 | T2.4 → T5.2 | ✅ |
| T5.3 → T5.4 → T5.5 | sequential | same | ✅ |
| T5.6 | T5.2 | T5.2 → T5.6 | ✅ |

### Check 3 — Test type × TESTING.md matrix
| Task | Layer | Required type | Task type | `[P]` allowed? | Status |
|---|---|---|---|---|---|
| T1.*, T2.1–T2.3 | `domain/dtos`, consensus internals | unit | unit | yes | ✅ |
| T2.4, T3.1, T3.4–T3.6 | public use cases | integration | integration | yes (tempdir) | ✅ |
| T3.2 | stdin | integration | integration, dedicated binary | yes (isolated per binary) | ✅ |
| T3.3, T4.* | shared / build-db | unit (tempdir) | unit | yes | ✅ |
| T5.1 | cli parsing | unit | unit | yes | ✅ |
| T5.2 | cli process | e2e | e2e | yes | ✅ |
| T5.3 | fake PATH in the child (`assert_cmd .env`) | e2e | e2e | yes | ✅ |
| T5.4 | PATH of the process itself | e2e | e2e | **no** (serial) | ✅ without `[P]` |
| T5.5 | real BLAST+ | e2e `#[ignore]` | e2e | n/a | ✅ |

### Traceability
| Requirement | Tasks |
|---|---|
| TB-FOUND-01/02 | T0.1, T0.2 |
| TB-CORE-01 | T1.1, T1.2, T1.3 |
| TB-CORE-02 | T1.5, T1.6, T2.1 |
| TB-CORE-03 | T1.7, T2.2 |
| TB-CORE-04 | T2.3 |
| TB-PIPE-01/02 | T2.4 |
| TB-PIPE-03 | T3.3, T3.4 |
| TB-PIPE-04 | T3.6 |
| TB-DTO-01 | T0.3, T1.4, T1.8 |
| TB-DTO-02 | T3.1, T3.2 |
| TB-DTO-03 | T3.5 |
| TB-DB-01 | T4.2, T4.3 |
| TB-DB-02 | T4.1 |
| TB-CLI-01 | T5.1 |
| TB-CLI-02/03 | T5.2 |
| TB-PROC-01 | T5.3, T5.4 |
| TB-PROC-02 | T5.5 |
| TB-CI-01 | T5.6 |

**Coverage:** 21 requirements, 21 mapped, 0 without a task.

## Tools per task
To be confirmed with the user before execution (MCPs/skills). Default: no MCP; the skill
`superpowers:test-driven-development` does not apply (tests of existing code).
