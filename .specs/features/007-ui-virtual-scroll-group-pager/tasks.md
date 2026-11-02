# 007 — Tasks

Branch: `feat/virtual-scroll-group-pager` in `modules/blutils-ui`. The gate for every task is
`npx tsc --noEmit` plus `CI=true yarn test --watchAll=false`.

| ID | Task | Reqs | Depends on | Status |
|---|---|---|---|---|
| T1 | Add `@tanstack/react-virtual`; stub `clientHeight`/`clientWidth` in `setupTests.ts` | TBL-02 | - | Done |
| T2 | `hooks/use-results-url-state.ts` + tests; `ResultLoader` keeps other params when setting `p` | GRP-06 | - | Done |
| T3 | `Shell` fullscreen via Outlet context (hide footer, `h-screen`); `Results` layout `h-full min-h-0`, view from URL, drop `pageSize` | LAY-01, LAY-02 | T2 | Done |
| T4 | Table: remove pagination/page size; virtualize `tbody` with spacer rows; sticky header; stats "N records"; tests | TBL-01..04 | T1, T3 | Done |
| T5 | `components/VirtualList.tsx` + `components/GroupPager.tsx` (sidebar TOC, mdbook chevrons, keyboard) + tests | GRP-03..05 | T1 | Done |
| T6 | Rewire both `Grouped.tsx` views on `GroupPager` + `VirtualList`, URL group; delete `SectionScroller`s (and the in-view hook if unused); tests | GRP-01, GRP-02, GRP-06, GRP-07 | T2, T5 | Done |
| T7 | Results integration tests: deep link `?p&v&g`, back/forward, view switch drops `g`, Reset clears | AC-03, AC-04 | T4, T6 | Done |
| T8 | Gates: `tsc`, tests, `CI=true yarn build`; manual browser check of AC-01/AC-02 (zymo mock) | AC-01..05 | T7 | Done |

## Results (2026-11-02)

- Gates: `tsc --noEmit` is clean. `CI=true yarn test --watchAll=false` passes 25 suites and
  317 tests (it was 263 tests before), in two consecutive runs. `CI=true yarn build` compiles.
- Browser check (headless Chromium, dev server, zymo mock with 3626 records, 1280×800 and
  390×844):
  - In every view, `scrollHeight === innerHeight` and no footer is rendered (AC-01).
  - The Table reaches record 3625 with at most 25 rows mounted, and the `thead` stays pinned at
    offset 0 (AC-02).
  - ArrowRight and Next advance `?g=`, and history back returns to the previous group (AC-03,
    AC-04).

## Implementation notes

- **Initial unbounded render (found in the browser, not in jsdom).** `Results` used to set the
  Shell fullscreen flag in an effect. For one commit, the table's scroll box sat in an unbounded
  parent and measured the whole list (about 260k px), so all 3626 rows mounted and the renderer
  died. The fix: the Shell context now exposes `fullscreen`, and `Results` renders a view only
  when it is outside a Shell (tests) or when `shell.fullscreen` is true.
- **Test infrastructure.** `setupTests.ts` fakes `clientHeight`/`clientWidth` (800/1200) for all
  elements, and `offsetHeight` = 40 for `[data-index]` items. virtual-core measures items with
  `offsetHeight` when no ResizeObserver entry is available.
- **minimatch.** The bare `"minimatch": "^9.0.7"` resolution (from an earlier Dependabot round)
  forced minimatch 9 onto `fork-ts-checker-webpack-plugin`, which calls the v3 default export.
  `yarn start` crashed (`minimatch_1.default is not a function`) as soon as there was any type
  issue to report. The bare resolution was removed. Each range now resolves within its own major
  (3.1.5 / 5.1.9 / 9.0.9), and all of them are patched for the earlier 3.x/5.x/9.x advisories.
  The lockfile check against all 203 historical alerts still matches only the 13 known
  residuals.
- **Removed tests.** The pagination, page-size, `containerHeight` and IntersectionObserver-gating
  tests were removed, because their subject no longer exists. They were replaced by windowing,
  scroll-to-top-on-filter and pager tests. Obsolete regression IDs: BUG-U08, BUG-U17 (pagination)
  and BUG-U13, BUG-U15 (deleted `use-element-in-view` hook). See `003-tests-blutils-ui/design.md`
  §2.

## Open points

- OP-01: D-05 (`?v=` added to the URL) is the agent's decision. Confirm it in review.
- OP-02: GRP-05 is met only on `md`+ screens. Below `md`, the table of contents is hidden and
  only the chevrons are available. mdbook exposes its sidebar on phones through a menu toggle,
  which is a candidate follow-up.
- OP-03: In the grouped views at about 1280px, the record row is wider than the content area.
  The Composition chart is reachable only by scrolling sideways inside the list.
