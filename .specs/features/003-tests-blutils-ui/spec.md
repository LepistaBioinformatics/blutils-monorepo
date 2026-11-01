# Test Suite — `modules/blutils-ui` (React) — Specification

> Status: **Draft** · Date: 2026-11-01
> Shared decisions: `../002-tests-blutils/context.md` (D-01 bug policy, D-02 refactoring).
> Infra/gates: `.specs/codebase/TESTING.md`.

## Problem Statement

The viewer (`≈2,2k` TSX lines) has a single test, `src/App.test.tsx`, inherited from the CRA template
— and it **fails**: it looks for the text "learn react", which does not exist, and Jest cannot even resolve the `@/` alias
used in 11 files. The mapping found defects that affect the end user (results without a
taxon disappear from the taxonomy view; a search with a space hides all rows; a JSON file of the
wrong shape brings down the page with no error boundary), none of them detectable today.

## Goals

- [ ] `CI=true yarn test --watchAll=false` green, with infrastructure that supports `@/`, flowbite, IntersectionObserver, clipboard and upload.
- [ ] Pure logic (`src/types`, `src/functions`, `docs-config`) at 100% line coverage.
- [ ] Each `BUG-Uxx` in the design with a characterization + desired `test.skip` (D-01).
- [ ] Main flow (load file → switch views → filter → open modal → reset) covered by an integration test.
- [ ] Tests running in the submodule workflow before deploy.

## Out of Scope

| Item | Reason |
|---|---|
| E2E in a real browser (Playwright/Cypress) | no dependency today; RTL integration covers the flows |
| Visual regression / Tailwind snapshot | low value, brittle |
| YAML/JSONL support in the UI | it is a feature, not a test (recorded as BUG-U11/contract gap) |
| Fixing the bugs | D-01 |
| `index.tsx`, `reportWebVitals.ts`, `theme/`, `constants/` | no logic |

---

## User Stories

### P1: Working test infrastructure ⭐ MVP

**User Story**: As a developer, I want `yarn test` to actually run, so that I can write any other test.

**Acceptance Criteria**:
1. WHEN a test imports a module with `@/…` THEN Jest SHALL resolve it.
2. WHEN a component uses `IntersectionObserver`, `navigator.clipboard.writeText`, `File.prototype.text` or `window.scrollTo` THEN `setupTests.ts` SHALL provide controllable implementations.
3. WHEN `App.test.tsx` runs THEN it SHALL verify the real content of the `/` route (not "learn react").
4. WHEN the `pages/docs` tests run THEN `react-markdown`/`remark-gfm` SHALL be mocked (ESM-only).

**Independent Test**: `CI=true yarn test --watchAll=false` exits with code 0.

### P1: Verified pure logic ⭐ MVP

**Acceptance Criteria**:
1. WHEN `BeanSelectionRules.getMostProbableTaxon` receives `undefined`/`[]`/1 bean/[5,2]/[4,2]/[5,3]/[3,3]/[2,6] THEN it SHALL return the bean and rule documented in the design.
2. WHEN `kebabToPlain`/`kebabToSciname`/`kebabToScinameString` receive names and ranks THEN they SHALL produce correct capitalization and italics.
3. WHEN `docs-config` resolves slugs, assets and links THEN it SHALL follow the design table.

### P1: Result loading ⭐ MVP

**Acceptance Criteria**:
1. WHEN the user uploads a valid `blutils.consensus.json` THEN `resultSetter` SHALL receive the object and the URL SHALL gain `?p=<name>`.
2. WHEN the file is not valid JSON THEN a `role="alert"` with the invalid-file message SHALL appear.
3. WHEN the page opens with `?p=<absolute URL>` THEN it SHALL `fetch` that URL; WHEN `?p=` is only a file name THEN it SHALL NOT fetch.
4. WHEN "Load a sample dataset" is clicked THEN it SHALL fetch `EXAMPLE_RAW_RESULT_URL`; failure SHALL display an alert.

### P2: Result views

**Acceptance Criteria**:
1. WHEN the view is Table THEN it SHALL paginate, filter by query/subject/Unmatched and open `ConsensusModal`.
2. WHEN the view is Grouped by Subject with a break character THEN it SHALL group by prefix.
3. WHEN the view is Grouped by Taxonomy THEN the test SHALL pin that results without a taxon are discarded (BUG-U01) and have a `test.skip` expecting the "Unidentified" group.
4. WHEN Reset is clicked THEN the result SHALL be cleared and `?p` removed.

### P2: Components

**Acceptance Criteria**: each component in `src/components` and `Results/elements` with an RTL test covering the cases in design §3.

### P3: Routes, docs and CI

1. WHEN the route is unknown THEN it SHALL render `NotFound`.
2. WHEN `/docs/:page` with an invalid slug THEN it SHALL display an error without fetch.
3. WHEN there is a push to the submodule THEN the workflow SHALL run the tests before the build.

---

## Edge Cases

- `results: []` → Table with "0 in 0 pages"; empty grouped views.
- Valid JSON with the wrong shape (`{}`, `[]`, `{"results":null}`) → crash with no error boundary (BUG-U06).
- `consensusBeans: null` (allowed by Rust) → crash in 5 components (BUG-U02).
- Identifier/rank with an empty segment (`"a--b"`, `"g__"`) → `TypeError` (BUG-U05).
- Real fixture (3,6 MB, 3 626 results) → Table renders only page 1.

## Requirement Traceability

| ID | Story | Status |
|---|---|---|
| TU-INFRA-01..04 | Infrastructure | Pending |
| TU-PURE-01..03 | Pure logic | Pending |
| TU-LOAD-01..04 | Loading | Pending |
| TU-VIEW-01..04 | Views | Pending |
| TU-COMP-01 | Components | Pending |
| TU-ROUTE-01..02 | Routes/docs | Pending |
| TU-CI-01 | CI | Pending |

## Success Criteria

- [ ] Full suite < 30 s locally.
- [ ] Zero unexpected `console.error` (act/DOM nesting) during the tests.
- [ ] All `BUG-U*` confirmed or refuted.
