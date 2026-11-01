# Contract Test `BlutilsOutput` ⇄ `BlutilsResult` — Specification (+ inline design)

> Status: **Implemented 2026-11-01 (C1–C6; C6 = local commits, not pushed)** · Date: 2026-11-01 · Scope: Medium (design and tasks inline).
> Open decision: **D-04** in `../002-tests-blutils/context.md` (golden location).
> Depends on: `002-tests-blutils` T0.2/T3.4 and `003-tests-blutils-ui` U0.2.

## Problem Statement

The only link between the two modules is the `BlutilsOutput` JSON (Rust,
`core/src/domain/dtos/blutils_output.rs`) mirrored by hand by `BlutilsResult`
(`blutils-ui/src/types/BlutilsResult.ts`). A divergence produces no compile error on either
side. The only real sample (`test/mock/output/zymo-mock/blutils.consensus.json`) is from **v7.1.3**,
while the engine is at **8.3.1** — and it is no longer accepted by Rust itself (BUG-R04).

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

## Requirements

- **TC-01** — WHEN the Rust test `core/tests/contract.rs` runs THEN it SHALL generate, from **dedicated** fixtures in `core/tests/fixtures/contract/` (`blast.tsv`, `taxonomies.blutils.json`, `headers.txt`; see `002-tests-blutils/design.md` §4) and `MockBlastn`/`build_consensus_identities` + `write_blutils_output(Json)`, a golden `blutils.contract.json` with `runId` and `blutilsVersion` **normalized** (fixed values) and sorted results.
- **TC-02** — The golden SHALL contain at least: one single `ConsensusFound`, one multi with LCA, one `taxon: null`, one non-Linnaean `reachedRank` (`Other`), null and non-null `maxAllowedRank`.
- **TC-03** — WHEN the golden changes without `UPDATE_GOLDEN=1` THEN the Rust test SHALL fail showing the diff.
- **TC-04** — The Rust test SHALL assert the **exact key set** of each object (`BlutilsOutput`, `QueryWithConsensus`, `TaxonomyBean`, `ConsensusBean`, `BlastBuilder`) against a literal list — breaks when a serde field is added/removed.
- **TC-05** — WHEN the TS test `src/types/BlutilsResult.contract.test.ts` runs THEN it SHALL load the golden copy, type it as `BlutilsResult` and assert the same key set; keys present in the golden and absent from the TS type SHALL be in an explicit `KNOWN_DRIFT` list (today: `config.isConfig`, `config.runId`).
- **TC-06** — The TS test SHALL render `Results` with the golden (Table, page 1) without error — proof of real consumption.
- **TC-07** — WHEN the Rust golden and the UI copy differ THEN a check in the monorepo SHALL fail (`cmp`), in a monorepo workflow.
- **TC-08** — Characterization: the zymo v7.1.3 fixture is read by the UI (TS) but rejected by Rust (BUG-R04).

## Design (D-04 default)

```
modules/blutils/core/tests/contract.rs
   └─ writes/compares → modules/blutils/core/tests/fixtures/contract/blutils.contract.json  (source of truth)
                                         │ manual copy (script) on update
                                         ▼
modules/blutils-ui/src/__fixtures__/blutils.contract.json
   └─ read by src/types/BlutilsResult.contract.test.ts
monorepo: .github/workflows/contract.yml → checkout --recursive → cmp of the two files
```

Why a copy and not a cross-reference: the submodules are independent repositories and CIs; the UI
cannot depend on `../blutils` in its own CI. The monorepo is the only place that sees both
pins and therefore hosts the drift check.

Alternatives (for D-04): (a) publish the golden as a `blutils` release asset and have the UI download it
by version; (b) generate the TS type from Rust (`ts-rs`/`typeshare`) — eliminates the bug
class, but is a production change (out of this scope).

## Tasks

| ID | What | Where | Depends on | Type | Gate |
|---|---|---|---|---|---|
| C1 | `contract.rs`: generates normalized golden + key-set assertions (TC-01..04) | `modules/blutils/core/tests/contract.rs`, `…/fixtures/contract/` | 002-tests-blutils T0.2, T3.4 | integration | `cargo test -p blutils-core --test contract` |
| C2 | Zymo characterization in Rust (TC-08) | already covered by 002-tests-blutils **T0.3** | — | — | — |
| C3 | Copy golden to the UI | `modules/blutils-ui/src/__fixtures__/blutils.contract.json` | C1 | none | — |
| C4 | `BlutilsResult.contract.test.ts` with `KNOWN_DRIFT` + render (TC-05, TC-06, TC-08 TS side) | `modules/blutils-ui/src/types/` | C3, 003-tests-blutils-ui U0.2 | integration | `CI=true yarn test --watchAll=false` |
| C5 | Drift workflow (TC-07) | `.github/workflows/contract.yml` (monorepo) | C3 | none | workflow run |
| C6 | Update submodule pointers in the monorepo | root | C1, C4 | none | `git submodule status` |

## Success Criteria

- [ ] Adding a serde field to `TaxonomyBean` without touching the TS makes C1 **and** C5 fail.
- [ ] `KNOWN_DRIFT` documents each tolerated divergence; emptying it is the "contract aligned" criterion.

## Implementation notes (2026-11-01)

- Golden: `c1-single-strain-100` (`reachedRank` `strain`), `c2-multi-lca-family` (LCA, `maxAllowedRank`
  species), `c3-header-only` (`taxon: null`), `c4-single-98` (genus, `mutated: true`). `runId` →
  nil UUID, `blutilsVersion` → `0.0.0-contract`.
- TC-02 deviation: a found taxon with `maxAllowedRank: null` is no longer producible after
  BUG-R02/R23; the Rust test asserts the opposite invariant instead (fails if nulls come back).
- `KNOWN_DRIFT`: `config.isConfig`, `config.runId`.
- Regenerate: `UPDATE_GOLDEN=1 cargo test -p blutils-core --test contract` (in `modules/blutils`),
  then copy the golden to `modules/blutils-ui/src/__fixtures__/blutils.contract.json`.
- Drift check: `.github/workflows/contract.yml` (monorepo, `cmp` of both copies).
