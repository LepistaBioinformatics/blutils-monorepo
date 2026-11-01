# Contract Test `BlutilsOutput` ⇄ `BlutilsResult` — Tasks

> Spec: `spec.md` · Design: `design.md` · Status: **Done 2026-11-01** (local commits, not pushed)

## Execution Plan

```
002 T0.2, T3.4 ──→ C1 ──→ C3 ──┬→ C4 ──┐
                               └→ C5   ├→ C6
003 U0.2 ──────────────────────→ C4 ──┘
```

## Task Breakdown

| ID | What | Where | Depends on | Type | Gate | Status |
|---|---|---|---|---|---|---|
| C1 | `contract.rs`: generates normalized golden + key-set assertions (TC-01..04) | `modules/blutils/core/tests/contract.rs`, `…/fixtures/contract/` | 002-tests-blutils T0.2, T3.4 | integration | `cargo test -p blutils-core --test contract` | ✅ |
| C2 | Zymo characterization in Rust (TC-08) | covered by 002-tests-blutils **T0.3** (BUG-R04) | — | — | — | ✅ |
| C3 | Copy golden to the UI | `modules/blutils-ui/src/__fixtures__/blutils.contract.json` | C1 | none | `cmp` | ✅ |
| C4 | `BlutilsResult.contract.test.tsx` with `KNOWN_DRIFT` + render (TC-05, TC-06, TC-08 TS side) | `modules/blutils-ui/src/types/` | C3, 003-tests-blutils-ui U0.2 | integration | `CI=true npx craco test --watchAll=false` | ✅ |
| C5 | Drift workflow (TC-07) | `.github/workflows/contract.yml` (monorepo) | C3 | none | workflow run | ✅ |
| C6 | Update submodule pointers in the monorepo | root | C1, C4 | none | `git submodule status` | ✅ (local commit `102df36`) |

## Follow-ups

- Close the remaining `KNOWN_DRIFT` (`config.isConfig`, `config.runId`) by adding both to the TS
  `Config` interface.
