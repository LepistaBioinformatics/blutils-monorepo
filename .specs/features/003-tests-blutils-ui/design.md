# Test Suite — `modules/blutils-ui` — Design

**Spec**: `spec.md` · **Context**: `../002-tests-blutils/context.md` · **Status**: Draft · 2026-11-01

> Paths relative to `modules/blutils-ui/`; lines at commit `39a9fc3`.
> **[H]** = static-reading hypothesis, to be confirmed by the test. No mark = checked in the code.

---

## 1. Infrastructure

### 1.1 Jest configuration (no eject)
- `package.json` → `"jest": { "moduleNameMapper": { "^@/(.*)$": "<rootDir>/src/$1" } }`
  (`moduleNameMapper` is in CRA's `supportedKeys`; the current alias only exists in `craco.config.js → webpack.alias`).
- `pages/docs` tests use `jest.mock("react-markdown")` / `jest.mock("remark-gfm")` — the alternative
  `transformIgnorePatterns` would require an allow-list of ~60 ESM packages.
- CRA uses `resetMocks: true`: mock implementations in `beforeEach`.

### 1.2 `src/setupTests.ts`
| Polyfill | Why | Form |
|---|---|---|
| `IntersectionObserver` | `hooks/use-element-in-view.tsx:8`, both `SectionScroller` | mock class that registers instances and exposes `trigger(isIntersecting)` via `src/test-utils/intersection-observer.ts` |
| `navigator.clipboard.writeText` | `components/CopyToClipBoard.tsx:15` | `jest.fn().mockResolvedValue(undefined)` in `beforeEach` |
| `Blob.prototype.text` | jsdom 16.7 does not implement it; `ResultLoader.tsx:73` | via `FileReader` |
| `window.scrollTo` | `pages/docs/index.tsx:82-84` | `jest.fn()` |

`matchMedia` and `ResizeObserver` are **not** needed (flowbite uses `matchMedia?.`; floating-ui checks `typeof ResizeObserver`).

### 1.3 `src/test-utils/`
- `renderWithRouter(ui, { route })` → `MemoryRouter` + `initialEntries`.
- builders: `makeBean(overrides)`, `makeTaxon(overrides)`, `makeResult(overrides)`.
- `intersection-observer.ts` (above).

### 1.4 Fixtures (`src/__fixtures__/`)
- `small.result.json` — handmade, 8–10 results, **all renderable**: `taxon: null`; 1 bean (rule 1); [4,2] (rule 2); [3,3] (rule 0); >5 beans (FaPlus); non-italic rank (`species-group`); two queries with the same prefix (`S1_a`, `S1_b`).
- `null-beans.result.json` — 1–2 results with `taxon` present and `consensusBeans: null`; used **only** in the BUG-U02 cases (MiniChart, Row, ConsensusModal, Grouped). Kept separate so it does not break the page-1 happy-path tests.
- `blutils.contract.json` — copy of the Rust golden (feature `004-tests-data-contract`).
- The real zymo (3,6 MB) is **not** copied; an optional smoke test reads it from `../blutils/test/mock/...` only when it exists (`describe.skip` if absent) — the UI submodule cannot depend on the monorepo path in CI.

---

## 2. Bug register (`BUG-U*`)

| ID | Sev | Location | Description | State |
|---|---|---|---|---|
| U01 | H | `pages/@anon/Results/index.tsx:58-59` | `key = unidentifiedKey` overwritten by `key = item.taxon?.identifier` → results without a taxon discarded in the taxonomy view (1 343/3 626 in the zymo fixture) | **fixed** (regression: `BUG-U01: taxon-less results are grouped under 'Unidentified'`) |
| U02 | H | `MiniChart.tsx:18`, `Row.tsx:19`, `ConsensusModal.tsx:19`, `GroupedByTaxonomyView/Grouped.tsx:62`, `GroupedByQueryView/Grouped.tsx:61` | `consensusBeans` typed non-null in TS but `Option` in Rust → crash when `taxon` exists and `consensusBeans` is `null` (the `?.` only guards `taxon`) | **fixed** (regression: `BUG-U02` tests in MiniChart, Row, ConsensusModal, both Grouped (TS types now nullable)) |
| U03 | M | `types/BeanSelectionRules.ts:70` | `getMostProbableTaxon` sorts the caller's array **in place** (React state) | **fixed** (regression: `BUG-U03: does not mutate the caller's array`) |
| U04 | M | `TableView/SearchableHeadCell.tsx:27` + `Table.tsx:64` | whitespace-only term → `""` (not `undefined`) → all rows filtered out | **fixed** (regression: `BUG-U04` in SearchableHeadCell and Table) |
| U05 | M | `functions/kebab-to-plain.ts:5`, `kebab-to-scientific-name.tsx:16,36` | empty segment (`""`, `"a--b"`, `"-a"`) → `TypeError` | **fixed** (regression: `BUG-U05` in kebab helpers, TaxonomyCell, GroupedByTaxonomy) |
| U06 | H | `Results/index.tsx:46` | valid JSON with the wrong shape → `TypeError` on render; no error boundary | **fixed** (regression: `BUG-U06` in ResultLoader (upload + `?p=`) and Results) |
| U07 | M | `ResultLoader.tsx:53` | `response.ok` not checked; 404 HTML falls into the JSON error | **fixed** (regression: `BUG-U07: a non-ok response…`) |
| U08 | L | `TableView/Table.tsx:99-111,150` | `currentPage` not reset on filter: content falls back to the last page, but no page is marked `aria-current` and Previous/Next both stay enabled | **fixed** (regression: `BUG-U08: filtering resets to page 1`, `…changing the page size resets to page 1`) |
| U09 | M | `Results/index.tsx:51-52,65` | query starting with the break character → key `""` → discarded | **fixed** (regression: `BUG-U09: … keeps its own group`) |
| U10 | L | `pages/5xx.tsx:21` | `alt="Page not found"` on the 500 error page | **fixed** (regression: `BUG-U10`) |
| U11 | M | `ResultLoader.tsx:102` | UI only accepts JSON; CLI also emits YAML/JSONL | **won't fix — JSON-only by design** (2026-11-01); pinned by `U11: the file input only accepts JSON` + YAML/JSONL upload tests |
| U12 | L | `TableView/Row.tsx:79-87` | rule 0 → bean `undefined` → condition always true → empty `<span>` | **fixed** (regression: `BUG-U12: rule 0 renders no secondary span`) |
| U13 | L | `hooks/use-element-in-view.tsx:22` | deps `[options]` with a new object literal on every render → observer recreated | **fixed** (regression: `BUG-U13`) |
| U14 | L | `Row.tsx:26,57` | `showChildren` state never read (dead code) | **fixed** (regression: `BUG-U14: clicking the row itself does not re-render it`) |
| U15 | L | `hooks/use-element-in-view.tsx:12-20` | Unmount never calls `unobserve`: React 18 clears `ref.current` before effect cleanup, so the guard skips it and the element stays observed | **fixed** (regression: `BUG-U15: unmount unobserves the element`) |
| U16 | L | `TableView/SearchableHeadCell.tsx:75-83` | Clicking the clear icon bubbles to the header `onClick`, reopening an empty search input | **fixed** (regression: `BUG-U16: clear icon … keeps the input closed`) |
| U17 | M | `TableView/Table.tsx:106-107,151` | With zero results, Next stays enabled; clicking it sets page 0 → table reads `paginatedResults[-1]` and stays empty even after the filter is cleared | **fixed** (regression: `BUG-U17: …'Next' is disabled…`) |
| U18 | M | `GroupedByQueryView/Grouped.tsx:42`, `GroupedByQueryView/SectionScroller.tsx:73` | Query names are formatted as scientific names with the first record's rank (`S1_no-hit` → "S1_no Hit") | **fixed** (regression: `BUG-U18: query names are shown verbatim`) |
| U19 | M | `ResultLoader.tsx:47-66` | After a successful load `setSearchParams({p})` re-triggers the `?p=` effect → `fetch` and `resultSetter` run twice (to be rechecked on the Results page in U4.1) | **fixed** (regression: `BUG-U19: ?p=<unencoded URL> is fetched once…`, `…a successful sample load fetches and sets the result once`) |
| U20 | L | `ResultLoader.tsx:57` | A failing user-supplied `?p=` URL shows "Couldn't load the sample dataset" | **fixed** (regression: `BUG-U20: a failing ?p= URL shows a generic error message`) |
| U21 | L | `TableView/Row.tsx:33` | `ConsensusTule` is declared inside render → remounts every Row render (tooltip id changes) | **fixed** (regression: `BUG-U21: re-rendering the row keeps the rule cell mounted`) |
| U22 | L | `pages/@anon/Results/index.tsx:32` | Leftover debug `console.log(character)` on every wrap-character submit | **fixed** (regression: `BUG-U22: submitting a wrap character does not log to the console`) |

---

**Interaction U01 × U05:** fixing U01 alone ("Unidentified" group with rank `""`) would crash the taxonomy view at `kebabToPlain("")` (`GroupedByTaxonomyView/SectionScroller.tsx:80`); both must be fixed together.

## 3. Case catalog by file

### 3.1 `src/types/BeanSelectionRules.ts` — unit (priority 1)
- `getRuleDescription(0|1|2)` → texts; `3` and `-1` → `throw "Invalid rule"`. (`Rules` not exported → use numbers; exporting it is `[REFACTOR]`.)
- `getMostProbableTaxon`: `undefined`→`{undefined,0}`; `[]`→0; 1 bean→rule 1; [5,2]→5, rule 2; [4,2]→rule 2 (`>=` boundary, `:77`); [5,3]→0; [3,3]→0; [2,6]→bean 6; `null`→0.
- U03: `Object.freeze` on the array → characterization (throws) + `test.skip` (does not mutate).

### 3.2 `src/functions/` — unit
- `kebabToPlain`: `"show-also"`→`"Show Also"`, `"no-rank"`→`"No Rank"`, `"species"`→`"Species"`; U05.
- `kebabToSciname` (render): `("bacillus-subtilis","species")` → `<span class="italic">Bacillus subtilis</span>`; genus/subspecies italic; `species-subgroup` not italic and capitalizes all words; `"Species"` not italic (case-sensitive); `props.children` is a string in both branches (contract used by `ConsensusModal.tsx:32`).
- `kebabToScinameString`: same logic as a string; U05.

### 3.3 `src/pages/docs/docs-config.ts` — unit
- `getDocPage()` → overview; `"run-blast"` → page; `"nope"` → `undefined`.
- `resolveDocAsset`: `undefined`/`""` unchanged; `http(s):`, `data:`, `mailto:` (case-insensitive) unchanged; `"./images/x.png"`, `"/images/x.png"`, `"images/x.png"` → `DOCS_RAW_BASE + "images/x.png"`.
- `slugForDocLink`: `"./02_run_blast_and_generate_consensus_identities.md#sec"` → `"run-blast"`; `"README.md?x"` → `"overview"`; external URL/`undefined` → `undefined`.

### 3.4 `src/hooks/use-element-in-view.tsx` — `renderHook`
- initial `inView` `false`; trigger `true`/`false`; `observe` only with `ref.current`; unmount → `unobserve`; U13 (count constructions on rerender with a new object).

### 3.5 `src/components/`
- **CopyToClipBoard**: click on the svg → `writeText(text)`; class ends in `"undefined"` without `className` [H].
- **MiniChart**: ≤5 bars; width `occ_i/occ*200px`; color index 0 = `#00A087`; `FaPlus` only with >5 beans; tooltip `"<sciname> x<occ>"`; `occurrences=0` → width `NaNpx`/`Infinity` [H]; U02.
- **ResultsPaginationStats**: `"N in M pages"` (Table) / `"groups"` (grouped).
- **Shell** (MemoryRouter): `/` → "Explorer" active; `/docs/x` → "Docs" active; external links `target=_blank rel=noreferrer`; current year in the footer (regex); "Loading…" fallback.

### 3.6 `src/pages/@anon/Results/elements/`
- **ResultLoader** (integration, `fetch` mock + `Blob.text`):
  - valid upload → `resultSetter(obj)` + `?p=<name>`; invalid/YAML/JSONL → "isn't a valid Blutils result" alert; changing the file clears the error; "View results" disabled without a file; labels change after selection.
  - `?p=https://x/y.json` → `fetch` called; `?p=foo` and `?p=blutils.consensus.json` → not called.
  - sample button → `fetch(EXAMPLE_RAW_RESULT_URL)`; spinner + disabled button during load; failure → "Couldn't load the sample dataset"; U07 (404 HTML response).
  - wrong-shape JSON is passed to `resultSetter` (the crash is in the parent — see Results).
- **TaxonomyCell**: split on `;`; segment `x__y` → `">" + kebabToPlain(y)` + tooltip `translateRank(x).toUpperCase()`; map `superkingdom→kingdom, d→domain, p, c, o, f, g, s`, others pass through; `""` → empty span; `"a__b__c"` → raw; `"g__"` → U05.
- **ConsensusModal**: `result undefined` → nothing; header = query; per bean: sciname, rank, `x<occ>`, accessions; copy buttons (`sciName` and `accessions.join(" ")`); `taxon null` → empty body; U02; closing calls `setOpenModal`. Query via `screen` (portal).
- **TableView/Row** (inside `<table><tbody>`): `taxon null` → "No significant match" `colSpan 7`, no taxonomy row; `percIdentity 99.356` → `"99.4"`; rule colors/number; extra row for the most probable bean; click on the query → `handleQueryDetails(record)`; U12.
- **TableView/SearchableHeadCell** (inside `<table><thead><tr>`): click shows the input (placeholder "Search..."); typing → `handleSearch(term.trim())`, empty → `undefined`; blur hides it; badge + clear → `handleSearch(undefined)`; U04.
- **TableView/Table**: pageSize counts groups; dropdown 10/15/25/50/100 → `handlePageSize`; Unmatched SHOW_ALSO/SHOW_ONLY/OMIT; case-insensitive query and subject search (AND when both); empty groups removed; U08; modal opens/closes. `chunk` tested indirectly (extraction = `[REFACTOR]`).
- **Grouped (Taxonomy/Query)** + **SectionScroller**: `#<name>` anchors with `x<n>`; `containerHeight` (`pageSize·40` / 80 / `n·40`); content only after the IO trigger; scroll `scrollTop 400` → items 10..20 and `translateY(400px)`; header with rank and TaxonomyCell only in the Taxonomy version.

### 3.7 `src/pages/@anon/Results/index.tsx` — integration
- no result → `ResultLoader`; after uploading `small.result.json` → Table.
- switching view resets pageSize to 10; break input disabled in Table; break `"_"` in Grouped by Subject groups `S1_a`/`S1_b`.
- U01 (Grouped by Taxonomy without "Unidentified"), U06 (`{}` → error), U09.
- Reset → back to the loader and removes `?p`.
- `results: []` → "0 in 0 pages".
- Optional smoke test with zymo: Table renders 10 rows on page 1.

### 3.8 Routes and pages
- **App.tsx**: uses a hardcoded `BrowserRouter` → `window.history.pushState` before `render`; `/` → "Open your results"/loader [H text]; `/anything` → `NotFound`; `/docs` → lazy `Loading…` fallback.
- **404/5xx**: texts and link to home; U10.
- **pages/docs/index.tsx** (react-markdown mocked): unknown slug → error without fetch; fetch `DOCS_RAW_BASE+file`; `!ok` → error; success → rendered markdown; `aria-current="page"` on the active link; `scrollTo` on slug change; unmount during fetch does not update state (`cancelled` guard). `markdownComponents` (img → `resolveDocAsset` + `loading=lazy`; internal link → `<Link>`; external → `_blank`) tested via a mock that renders the `components` map.

---

## 4. Design decisions
- **user-event 13** (already installed, synchronous API) — do not upgrade to v14 in this feature.
- **No MSW**: `jest.spyOn(global, "fetch")` is enough for 3 callers.
- **Co-located tests** (CRA convention), not `__tests__/`.
