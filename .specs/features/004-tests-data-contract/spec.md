# Contract Test `BlutilsOutput` ⇄ `BlutilsResult` — Specification

> Status: **Implemented 2026-11-01** · Design: `design.md` · Tasks: `tasks.md`
> Decision: **D-04** in `../002-tests-blutils/context.md` (golden location).
> Depends on: `002-tests-blutils` T0.2/T3.4 and `003-tests-blutils-ui` U0.2.

## Problem Statement

The only link between the two modules is the `BlutilsOutput` JSON (Rust,
`core/src/domain/dtos/blutils_output.rs`) mirrored by hand by `BlutilsResult`
(`blutils-ui/src/types/BlutilsResult.ts`). A divergence produces no compile error on either
side. The only real sample (`test/mock/output/zymo-mock/blutils.consensus.json`) is from **v7.1.3**,
while the engine is at **8.3.1** — and it was no longer accepted by Rust itself (BUG-R04, since fixed).

## Current drift (2026-11-01; updated after F4 fixes)

> Rows marked **resolved** were fixed by F4 (`005-test-suite-bugfixes`): TS types now mirror Rust `Option` fields.

| Field | Rust (serde camelCase) | TS | Status |
|---|---|---|---|
| `results[].runId` | `Option<Uuid>` → `uuid \| null` | absent | **resolved** (F4) |
| `results[].taxon` | `Option<TaxonomyBean>` → `null` | `taxon?: Taxon` | null × undefined (tolerated) |
| `taxon.taxonomy` | `Option<String>` | `string` required | **resolved** (F4) |
| `taxon.consensusBeans` | `Option<Vec<…>>` | `ConsensusBean[]` required | **resolved** (F4) |
| `taxon.maxAllowedRank` | `Option<LinnaeanRank>` | `maxAllowedRank?: string` | null × undefined |
| `consensusBeans[].taxonomy` | `Option<String>` | `string` | **resolved** (F4) |
| `config.isConfig`, `config.runId` | `bool`, `Uuid` required | absent | **drift** (UI does not read `config`) |
| `bitScore` | `f64` (`845.0`) | `number` | ok |
| formats | Json, Jsonl, Yaml | JSON only | by design (U11 won't fix) |

## Out of Scope

| Item | Reason |
|---|---|
| Generating TS types from Rust (`ts-rs`/`typeshare`) | production change; see `design.md` alternatives |
| YAML/JSONL in the viewer | by design (U11 won't fix) |

## Requirements

- **TC-01** — WHEN the Rust test `core/tests/contract.rs` runs THEN it SHALL generate, from **dedicated** fixtures in `core/tests/fixtures/contract/` (`blast.tsv`, `taxonomies.blutils.json`, `headers.txt`; see `002-tests-blutils/design.md` §4) and `MockBlastn`/`build_consensus_identities` + `write_blutils_output(Json)`, a golden `blutils.contract.json` with `runId` and `blutilsVersion` **normalized** (fixed values) and sorted results.
- **TC-02** — The golden SHALL contain at least: one single `ConsensusFound`, one multi with LCA, one `taxon: null`, one non-Linnaean `reachedRank` (`Other`), null and non-null `maxAllowedRank`.
- **TC-03** — WHEN the golden changes without `UPDATE_GOLDEN=1` THEN the Rust test SHALL fail showing the diff.
- **TC-04** — The Rust test SHALL assert the **exact key set** of each object (`BlutilsOutput`, `QueryWithConsensus`, `TaxonomyBean`, `ConsensusBean`, `BlastBuilder`) against a literal list — breaks when a serde field is added/removed.
- **TC-05** — WHEN the TS test `src/types/BlutilsResult.contract.test.ts` runs THEN it SHALL load the golden copy, type it as `BlutilsResult` and assert the same key set; keys present in the golden and absent from the TS type SHALL be in an explicit `KNOWN_DRIFT` list (today: `config.isConfig`, `config.runId`).
- **TC-06** — The TS test SHALL render `Results` with the golden (Table, page 1) without error — proof of real consumption.
- **TC-07** — WHEN the Rust golden and the UI copy differ THEN a check in the monorepo SHALL fail (`cmp`), in a monorepo workflow.
- **TC-08** — Characterization: the zymo v7.1.3 fixture is read by the UI (TS); on the Rust side it is covered by BUG-R04.

## Success Criteria

- [x] Adding a serde field to `TaxonomyBean` without touching the TS makes C1 **and** C5 fail.
- [ ] `KNOWN_DRIFT` documents each tolerated divergence; emptying it is the "contract aligned" criterion
  (open: `config.isConfig`, `config.runId`).
