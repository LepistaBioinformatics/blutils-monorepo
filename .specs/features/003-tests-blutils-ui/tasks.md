# Test Suite — `modules/blutils-ui` — Tasks

**Spec**: `spec.md` · **Design**: `design.md` · **Status**: Draft

> Each task = one test file (co-located). Quick gate = `CI=true yarn test --watchAll=false`;
> build = `CI=true yarn test --watchAll=false && yarn build`. Run in `modules/blutils-ui`.
> Commits in the submodule repository; pointer updated in the monorepo at the end of each phase.

---

## Execution Plan

### Phase 0 — Infrastructure (sequential)
```
U0.1 → U0.2 → U0.3 → U0.4
```

### Phase 1 — Pure logic (parallel)
```
U0.4 ──┬→ U1.1 BeanSelectionRules
       ├→ U1.2 kebab-to-plain
       ├→ U1.3 kebab-to-scientific-name
       └→ U1.4 docs-config
```

### Phase 2 — Hook and leaf components (parallel)
```
U0.4 ──┬→ U2.1 useElementInView
       ├→ U2.2 CopyToClipBoard
       ├→ U2.3 ResultsPaginationStats
       ├→ U2.4 MiniChart          [dep U1.1, U1.3]
       ├→ U2.5 TaxonomyCell       [dep U1.2]
       └→ U2.6 SearchableHeadCell
```

### Phase 3 — Composite components (parallel)
```
Phase 2 ─┬→ U3.1 Row
         ├→ U3.2 ConsensusModal
         ├→ U3.3 SectionScroller (x2)
         └→ U3.4 ResultLoader
U3.1,U3.2,U2.6 → U3.5 Table
U3.3 → U3.6 Grouped (x2)
```

### Phase 4 — Pages, routes and CI (sequential)
```
U3.4,U3.5,U3.6 → U4.1 Results page → U4.2 Shell/App/404/5xx → U4.3 Docs page → U4.4 CI
```

---

## Task Breakdown

### U0.1 — `moduleNameMapper` for `@/`
**Where**: `package.json` · **Req**: TU-INFRA-01 · **Tests**: none · **Gate**: quick (after U0.3)
**Done when**: a test that imports `@/types/BeanSelectionRules` resolves.

### U0.2 — Polyfills in `setupTests.ts` + `src/test-utils/`
Controllable IO, clipboard, `Blob.text`, `scrollTo`; `renderWithRouter`; builders (design §1.2–1.3).
**Where**: `src/setupTests.ts`, `src/test-utils/*` · **Depends on**: U0.1 · **Req**: TU-INFRA-02 · **Tests**: none · **Gate**: quick

### U0.3 — Replace `App.test.tsx`
Remove the "learn react" assertion; verify the real content of `/` and unknown route → NotFound.
**Where**: `src/App.test.tsx` · **Depends on**: U0.2 · **Req**: TU-INFRA-03 · **Tests**: integration · **Gate**: quick
**Commit**: `test: make jest runnable and replace stale CRA test`

### U0.4 — Fixture `small.result.json`
Content per design §1.4. **Where**: `src/__fixtures__/small.result.json`, `src/__fixtures__/null-beans.result.json` · **Depends on**: U0.3 · **Tests**: none · **Gate**: quick

### U1.1 — `BeanSelectionRules.test.ts` [P]
Design §3.1 (U03). **Req**: TU-PURE-01 · **Tests**: unit · **Gate**: quick

### U1.2 — `kebab-to-plain.test.ts` [P]
Design §3.2 (U05). **Req**: TU-PURE-02 · **Tests**: unit · **Gate**: quick

### U1.3 — `kebab-to-scientific-name.test.tsx` [P]
Design §3.2. **Req**: TU-PURE-02 · **Tests**: unit · **Gate**: quick

### U1.4 — `docs-config.test.ts` [P]
Design §3.3. **Req**: TU-PURE-03 · **Tests**: unit · **Gate**: quick

### U2.1 — `use-element-in-view.test.tsx` [P]
Design §3.4 (U13). **Req**: TU-COMP-01 · **Tests**: unit · **Gate**: quick

### U2.2 — `CopyToClipBoard.test.tsx` [P]
### U2.3 — `ResultsPaginationStats.test.tsx` [P]
### U2.4 — `MiniChart.test.tsx` [P] (dep U1.1, U1.3; U02)
### U2.5 — `TaxonomyCell.test.tsx` [P] (dep U1.2; U05)
### U2.6 — `SearchableHeadCell.test.tsx` [P] (U04)
Cases: design §3.5/§3.6. **Req**: TU-COMP-01 · **Tests**: integration (RTL) · **Gate**: quick

### U3.1 — `Row.test.tsx` [P]
Design §3.6 (U12). **Req**: TU-COMP-01 · **Tests**: integration · **Gate**: quick

### U3.2 — `ConsensusModal.test.tsx` [P]
Design §3.6 (U02). **Req**: TU-COMP-01 · **Tests**: integration · **Gate**: quick

### U3.3 — `SectionScroller.test.tsx` (Taxonomy and Query) [P]
Design §3.6. **Depends on**: U2.1 · **Req**: TU-COMP-01 · **Tests**: integration · **Gate**: quick

### U3.4 — `ResultLoader.test.tsx` [P]
Design §3.6 (U07, U11). **Req**: TU-LOAD-01..04 · **Tests**: integration · **Gate**: quick
**Commit**: `test: cover result loading (upload, url, sample)`

### U3.5 — `Table.test.tsx`
Design §3.6 (U08). **Depends on**: U3.1, U3.2, U2.6 · **Req**: TU-VIEW-01 · **Tests**: integration · **Gate**: quick

### U3.6 — `Grouped.test.tsx` (Taxonomy and Query)
Design §3.6. **Depends on**: U3.3 · **Req**: TU-VIEW-02 · **Tests**: integration · **Gate**: quick

### U4.1 — `Results/index.test.tsx`
Design §3.7 (U01, U06, U09). **Depends on**: U3.4, U3.5, U3.6 · **Req**: TU-VIEW-02..04 · **Tests**: integration · **Gate**: quick

### U4.2 — `Shell.test.tsx`, `404.test.tsx`, `5xx.test.tsx`, extend `App.test.tsx`
Design §3.5 (Shell) and §3.8 (U10). **Depends on**: U4.1 · **Req**: TU-ROUTE-01 · **Tests**: integration · **Gate**: quick

### U4.3 — `pages/docs/index.test.tsx`
Design §3.8, with `jest.mock("react-markdown")`/`jest.mock("remark-gfm")`. **Depends on**: U4.2 · **Req**: TU-INFRA-04, TU-ROUTE-02 · **Tests**: integration · **Gate**: build

### U4.4 — CI
Add `CI=true yarn test --watchAll=false` before `yarn build` in `.github/workflows/deploy.yml` (and a job on `pull_request`).
**Depends on**: U4.3 · **Req**: TU-CI-01 · **Tests**: none · **Gate**: build

### `[REFACTOR]` (outside the first batch — D-02)
- **UR.1** extract `groupedResults` (`Results/index.tsx:42-85`) → `src/functions/group-results.ts` + unit test.
- **UR.2** extract `chunk` (`Table.tsx:43-47`) → `src/functions/chunk.ts` + unit test.
- **UR.3** export `BeanSelectionRules.Rules`, `translateRank` (`TaxonomyCell.tsx:38-59`), `markdownComponents` (`docs/index.tsx:20-40`).
- **UR.4** error boundary in `Results` (turns U06 from a crash into a message).

---

## Validation

### Check 1 — Granularity
Every task = one test file, except U0.2 (cohesive infra), U3.3/U3.6 (two near-identical components, same pattern) and U4.2 (trivial route pages). ✅

### Check 2 — Diagram × `Depends on`
| Task | Depends on | Diagram | Status |
|---|---|---|---|
| U0.2–U0.4 | previous | chain U0.1→U0.4 | ✅ |
| U2.4 | U1.1, U1.3 | annotated | ✅ |
| U2.5 | U1.2 | annotated | ✅ |
| U3.3 | U2.1 | Phase 2 → U3.3 | ✅ |
| U3.5 | U3.1, U3.2, U2.6 | annotated | ✅ |
| U3.6 | U3.3 | annotated | ✅ |
| U4.1 | U3.4, U3.5, U3.6 | annotated | ✅ |
| U4.2→U4.3→U4.4 | sequential | same | ✅ |

### Check 3 — Type × TESTING.md matrix
| Tasks | Layer | Required | Task | `[P]` | Status |
|---|---|---|---|---|---|
| U1.* | types/functions/docs-config | unit | unit | yes | ✅ |
| U2.1 | hooks | unit (renderHook) | unit | yes | ✅ |
| U2.2–U3.6 | components/elements | integration (RTL) | integration | yes (Jest isolates per file) | ✅ |
| U0.3, U4.1–U4.3 | pages/App | integration (MemoryRouter) | integration | not marked (phase sequence) | ✅ |

### Traceability
| Requirement | Tasks |
|---|---|
| TU-INFRA-01..04 | U0.1, U0.2, U0.3, U4.3 |
| TU-PURE-01..03 | U1.1, U1.2+U1.3, U1.4 |
| TU-LOAD-01..04 | U3.4 |
| TU-VIEW-01 | U3.5 |
| TU-VIEW-02..04 | U3.6, U4.1 |
| TU-COMP-01 | U2.*, U3.1–U3.3 |
| TU-ROUTE-01..02 | U4.2, U4.3 |
| TU-CI-01 | U4.4 |

**Coverage:** 19 requirements, 19 mapped.
