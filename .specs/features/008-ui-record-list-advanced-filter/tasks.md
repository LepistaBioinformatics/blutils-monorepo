# 008 — Tasks

Branch: `feat/virtual-scroll-group-pager` (the working tree is shared with 007 and
`quick/001-theme-mode-toggle`). The gate is `tsc --noEmit` plus the full test suite.

| ID | Task | Reqs | Depends on | Status |
|---|---|---|---|---|
| T1 | `functions/record-filter.ts` + tests (parse, serialize, match, lineage) | FLT-01, FLT-02 | - | Done (unit tests) |
| T2 | `functions/taxonomy-tree.ts` + tests | FLT-05 | T1 | Done (unit tests) |
| T3 | `BeanSelectionRules.getRuleKind` + `RULE_INFO`; `RuleBadge` + modal | RULE-01, RULE-02 | - | Done |
| T4 | `CompositionDonut` | DON-01, DON-02 | - | Done |
| T5 | `RecordItem` + `RecordHeader`; adopt in Table and both grouped views; delete Row, SearchableHeadCell, MiniChart | REC-01..03 | T3, T4 | Done |
| T6 | `FilterBar` + `TaxonomyTreeModal`; `?f=` in `use-results-url-state`; filtering in `Results` before grouping; stats | FLT-01..07 | T1, T2 | Done |
| T7 | Toolbar: centered segmented view selector, wrap character with help | TB-01, TB-02 | T6 | Done |
| T8 | Port and add the UI tests (component + Results integration) | all | T5..T7 | Done |
| T9 | Gates + browser check (alignment, filter across views, tree, rule modal) | AC-01..05 | T8 | Done |

## Implementation notes

- **Browser check** (dev server, zymo mock, 1280×800 and 390×844):
  - In the Table view, the proposed-taxon and rule columns start at the same x for every
    mounted entry and the header (577 / 988 px).
  - `q:SRR20752596 t:bacillus id>=99` gives "302 of 3626 records". The chips survive the switch
    to "By taxonomy" (2 groups) and a reload, via `?f=`.
  - Picking a tree node adds `Taxonomy: bacteria > terrabacteria group` (16 groups).
  - A rule click opens the modal with the current rule highlighted.
  - No page errors.
- **Fixes found during the check.**
  - The query title wrapped badly in the narrower grouped content, so it now breaks with
    `break-all` instead of truncating.
  - On phones, the entry is two columns: the text stacks on the left and the donut spans the
    rows on the right. The view selector gets its own centered row below `md`.
- `TaxonomyCell.translateRank` is exported so the tree can show readable ranks (`g` → genus).

## Iterations with the user (2026-11-02, after the first browser pass)

Each item is a user request, applied and verified in the browser.

- **Neutral palette.** Controls, the navbar (badges, active link), the view selector, chips, the
  sidebar and accents are gray. Color is kept only for the rule words and the composition.
  `theme/flowbite-theme.ts` remaps the `info` button and the navbar active link.
- **Composition.** The donut was replaced by the previous bar style (`CompositionBar`):
  - identity % and bit score side by side, with the bar below them;
  - the top 5 beans with widths by occurrences, a native `title` on each, and "+N" for the rest.

  This supersedes DON-01/DON-02 (the donut).
- **Proposed taxon** uses the query's typography (`text-lg font-semibold`).
- **Wrap character.**
  - It is shown only in "By subject", above the subjects list, through the `GroupPager`
    `sidebarHeader` prop (above the content below `md`).
  - It comes with automatic suggestions (`functions/suggest-wrap-characters.ts`): non-alphanumeric
    characters present in ≥ 50% of the queries that produce more than one group and fewer groups
    than there are queries, best coverage first, then fewest groups. On zymo it suggests `.`
    (9 groups).
  - It supersedes TB-02.
- **ConsensusModal** no longer forces `dark`, so it follows the theme.
- **Widths.**
  - The Table view is centered at `max-w-6xl`. The grouped views have no width cap (the user
    reverted a cap).
  - The record grid minimums were lowered, and the side chevrons start at `xl`, so rows fit
    from 1024px up (checked at 1024, 1280, 1440 and 1920).
- **Search bar.**
  - It is larger and rounded (`rounded-2xl`), with the token buttons inside the same box.
  - The view selector sits to its right; on phones it moves to its own row and the buttons
    scroll horizontally.
  - The toolbar row was removed. Reset moved to the navbar through a `Shell` outlet slot
    (`navbarActions`, React portal); without a Shell (tests), it renders in place.
- **Tests.** The subagent's suite plus mine: 32 suites and 435 tests, green warm and with
  `--no-cache`. `tsc` is clean and `CI=true yarn build` compiles.
  - Obsolete regression IDs: BUG-U14, U16 and U21 (`Row`/`SearchableHeadCell` deleted).
  - Ported: U02 (RecordItem, CompositionBar, both grouped views), U04 (FilterBar), U12
    (RecordItem), U09/U22 (subject-view wrap control).
  - A ResultLoader timing race in a test (`?p=` read before the router update) was fixed with
    `waitFor`. It was not a product bug.

## Iterations, part 2 (2026-11-02)

- **Cleaner layout.**
  - The boxes around the lists, the header background, the sidebar separator and the group
    accent bar were removed.
  - Records keep only a thin bottom line between them (requested), with a hover background.
- **One module for search and view.**
  - The view selector is a pill switch rendered inside the search box (`FilterBar` `trailing`
    prop), at the end of the input row.
  - The token buttons are borderless pills in a second row of the same box.
- **Value suggestions.**
  - After `q:`, `t:`, `rank:` or `rule:` (typed or inserted by a button), a listbox under the
    input offers values from `functions/filter-suggestions.ts`: queries, lineage names, readable
    ranks and rule words. Values have counts and are sorted by frequency, prefix matches first.
  - The index is built from the already-filtered records, so a suggestion never leads to an
    empty list.
  - Keyboard: ↑/↓, Enter picks, and Escape closes the list and keeps the text. The input is an
    ARIA combobox.
- **Details modal.**
  - The modal shows everything from the row: proposed taxon and rank, the most probable bean,
    the composition bar, taxonomy (one rank per line), details (run, reached and max allowed
    rank, single match, mutated) and the rule (word, summary, explanation), placed right above
    the consensus beans.
  - The beans share the modal background and are separated by horizontal lines.
  - It opens from a click anywhere on the row, except the rule (its own modal) and the copy
    icon. A click that ends a text selection does not open it.
  - The `stopPropagation` wrappers also stop clicks bubbling from the rules modal (a React
    portal) to the row.
- **Rows.** Values align to the top. The composition bar uses a soft, muted palette
  (`constants/sequencial-colors.ts`).
- **Gates.** `tsc` is clean, 34 suites and 459 tests pass, warm and with `--no-cache`, and
  `CI=true yarn build` compiles.

## Iterations, part 3 (2026-11-02)

- **Search box focus.** `ring-1` was stacked with `shadow-sm`, which made the top and bottom
  edges look thicker. The box now uses a real 1px border and drops the shadow on focus, so the
  edges are uniform (measured: 1px on all four sides).
- **Wrap card ("Group by prefix").**
  - It has a title and a one-line purpose, a visible "Wrap character" label, a filled input and
    suggestion pills showing the character plus "N groups".
  - A Before/After preview on two lines shows the effect on a real query.
  - It uses fills instead of borders.
- **Composition palette in dark mode.** The bar reads `--composition-<n>` CSS variables
  (`index.css`): the soft light palette under `:root`, deeper tones of the same hues under
  `.dark`.
- **Taxonomy tree.** The first three levels open by default (`DEFAULT_EXPANDED_DEPTH`).
- **View selector.** It is the last row inside the search box, below the advanced filter
  buttons (`FilterBar` `footer` prop, renamed from `trailing`).
- **Gates.** `tsc` is clean, 34 suites and 461 tests pass (with `--no-cache`), and the build
  compiles.

## Iterations, part 4 (2026-11-02)

- **Taxonomy tree:**
  - two levels open by default;
  - larger, filled expand buttons;
  - animated open/close: grid-rows height transition and chevron rotation, `motion-reduce`
    respected. Branches mount on first open and stay mounted, with `aria-hidden` and `inert`
    (set via a ref, because React 18 drops `inert`).
- **Sample dataset:** the ZymoBIOMICS ITS run classified with the relaxed strategy, from
  blutils-benchmarks `output/blutils/ITS/relaxed.json` (sha256 matches the benchmark
  MANIFEST). It is served from `public/examples/zymo-its-relaxed.json`, with an absolute URL
  so `?p=` survives a reload, and a contract test checks it field-for-field.
- **Reset:** a dark button with an icon and a tooltip, in the navbar.
- **Sorting** (Table and both grouped views):
  - Query, Taxonomy and Proposed taxon headers, in natural order;
  - the order cycles asc → desc → original (file) order;
  - "Sorted by …" plus a prominent "Original order" pill (`SortStatus`);
  - `aria-sort` on the headers;
  - records without a value go last.
- **Keyboard:**
  - ↑/↓ highlight a record (`aria-current`) and keep it in view; Enter opens its details;
  - ←/→ pulse the matching pager chevron (on a click too).
  - A shared `isNavigationKey` guard ignores keys with a modifier, while typing, and while a
    dialog is open.
- **Details modal:** `dismissible`, so it closes on Escape and on a click outside.
