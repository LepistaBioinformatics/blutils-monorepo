# Contract Test `BlutilsOutput` ⇄ `BlutilsResult` — Design

> Spec: `spec.md` · Tasks: `tasks.md` · Status: Implemented 2026-11-01

## Architecture (D-04 default)

```
modules/blutils/core/tests/contract.rs
   └─ writes/compares → modules/blutils/core/tests/fixtures/contract/blutils.contract.json  (source of truth)
                                         │ manual copy on update
                                         ▼
modules/blutils-ui/src/__fixtures__/blutils.contract.json
   └─ read by src/types/BlutilsResult.contract.test.tsx
monorepo: .github/workflows/contract.yml → checkout --recursive → cmp of the two files
```

Why a copy and not a cross-reference: the submodules are independent repositories and CIs; the UI
cannot depend on `../blutils` in its own CI. The monorepo is the only place that sees both
pins and therefore hosts the drift check.

## Alternatives (for D-04)

- (a) Publish the golden as a `blutils` release asset and have the UI download it by version.
- (b) Generate the TS type from Rust (`ts-rs`/`typeshare`) — eliminates the bug class, but is a
  production change (out of this scope).

## Components

| Component | Location | Responsibility |
|---|---|---|
| Golden generator + key-set assertions | `modules/blutils/core/tests/contract.rs` | TC-01..TC-04 |
| Dedicated fixtures | `modules/blutils/core/tests/fixtures/contract/{blast.tsv,taxonomies.blutils.json,headers.txt}` | isolated from unit-test fixtures |
| UI copy | `modules/blutils-ui/src/__fixtures__/blutils.contract.json` | byte-identical to the golden |
| TS contract test | `modules/blutils-ui/src/types/BlutilsResult.contract.test.tsx` | TC-05, TC-06, TC-08 (TS side) |
| Drift workflow | `.github/workflows/contract.yml` (monorepo) | TC-07 |

TS key lists use `as const satisfies readonly (keyof T)[]` plus an exhaustiveness type check, so
adding a field to the interface without updating the list fails `tsc`.

## Implementation notes (2026-11-01)

- Golden results: `c1-single-strain-100` (`reachedRank` `strain`), `c2-multi-lca-family` (LCA,
  `maxAllowedRank` species), `c3-header-only` (`taxon: null`), `c4-single-98` (genus,
  `mutated: true`). `runId` → nil UUID, `blutilsVersion` → `0.0.0-contract`.
- TC-02 deviation: a found taxon with `maxAllowedRank: null` is no longer producible after
  BUG-R02/R23; the Rust test asserts the opposite invariant instead (fails if nulls come back).
- `KNOWN_DRIFT`: `config.isConfig`, `config.runId`.
- Regenerate: `UPDATE_GOLDEN=1 cargo test -p blutils-core --test contract` (in `modules/blutils`),
  then copy the golden to `modules/blutils-ui/src/__fixtures__/blutils.contract.json`.
