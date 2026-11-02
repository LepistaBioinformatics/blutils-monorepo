# 007 — Result viewer: virtual-scrolled table and mdbook-style group pager

Scope: **Large** (two views reworked, page layout, URL state; `modules/blutils-ui` only).

## Problem

- The **Table** view paginates (a "Page Size" dropdown plus flowbite `Pagination`). To see every
  record, the user has to click through pages, and the whole page scrolls.
- The **Grouped by taxonomy** and **Grouped by subject** views stack every group vertically. Each
  group has its own small scroller, so the user scrolls the page *and* each section. With many
  groups, it is hard to know where you are.

## Goals

- The results view fits in one screen. Only the table or list content scrolls, never the page.
- In the Table view, the user scrolls through **all** filtered records in one virtualized list.
- In the grouped views, the user steps through groups one at a time with previous/next
  navigation styled like mdbook (the Rust book).

## Requirements

### Layout

- **LAY-01**: While a result is loaded, the results view fills the viewport below the navbar.
  The document does not scroll vertically. Only the inner content areas scroll (table body,
  group list, sidebar).
- **LAY-02**: The site footer is hidden while a result is loaded. The loader screen, docs and
  error pages keep the normal layout with the footer (D-02).

### Table view

- **TBL-01**: Pagination and the "Page Size" control are removed. Every filtered record is
  reachable by scrolling one container.
- **TBL-02**: Rows are virtualized with `@tanstack/react-virtual` (D-03). Only the visible rows
  plus an overscan are in the DOM, and rows can have variable heights.
- **TBL-03**: The table header stays visible (sticky) while scrolling. Header search (Query,
  Proposed Taxon) and the "Unmatched" filter keep their current behavior. Changing a filter
  scrolls back to the top.
- **TBL-04**: The stats line shows the total number of filtered records (e.g. "6 records")
  instead of "N in M pages".

### Grouped views (taxonomy and subject)

- **GRP-01**: Only the **current** group is rendered: its header (name, rank, count, taxonomy)
  and its records.
- **GRP-02**: The records of the current group are virtualized and scroll inside the content
  area.
- **GRP-03**: The navigation follows mdbook:
  - On wide screens, a full-height chevron bar sits on the left edge (previous) and on the right
    edge (next) of the content area.
  - On narrow screens, previous/next chevron buttons sit below the content.
  - A chevron is not rendered when there is no previous or next group.
- **GRP-04**: The ArrowLeft and ArrowRight keys go to the previous and next group. The keys are
  ignored when a modifier key is held, when focus is in a text field or select, or when a modal
  is open.
- **GRP-05**: The left sidebar stays as an mdbook-style table of contents (D-01). It lists every
  group with its record count and highlights the current one. Clicking an entry jumps to that
  group, and the active entry is scrolled into view. The sidebar is virtualized, because the
  subject view can have one group per query.
- **GRP-06**: The current group is reflected in the URL as `?g=<group name>`, together with the
  view as `?v=<view type>` (D-04, D-05):
  - Reloading or sharing the URL restores the view and the group.
  - Browser back/forward steps through the visited groups.
  - An unknown `g` falls back to the first group.
  - Switching views drops `g`.
  - Reset clears every parameter.
  - `?p=` keeps working as before, and loading via `?p=` never drops `v` or `g`.
  - Caveat: an uploaded file sets `?p=<file name>`, which is not a URL, so reload and share
    restore `v`/`g` only for URL-loaded results. After a reload, an upload returns to the loader.
- **GRP-07**: The wrap character still regroups the subject view. The current `g` is kept if it
  still exists, otherwise navigation falls back to the first group.

## Acceptance criteria

- **AC-01**: With a loaded result in any view, `document.scrollingElement.scrollHeight` equals
  the viewport height. This is checked manually in a browser at 1280×800 and at 390×844.
- **AC-02**: The Table view of the zymo mock result (thousands of queries) scrolls smoothly to
  the last record without pagination. Fewer than about 100 rows are in the DOM at any time.
- **AC-03**: In the grouped views:
  - Clicking `›` and pressing ArrowRight both advance one group.
  - On the first group there is no `‹`, and on the last group there is no `›`.
  - Typing in the wrap-character field does not navigate.
- **AC-04**: Opening `…/?p=<url>&v=grouped-by-taxonomy&g=<name>` loads the result directly on
  that group. Back returns to the previously visited group.
- **AC-05**: Gates pass: `tsc --noEmit` is clean, `CI=true yarn test --watchAll=false` is green,
  and `CI=true yarn build` succeeds.

## Out of scope

- Changes to the data contract (`BlutilsResult`), the grouping algorithm, `ConsensusModal`,
  `Row` cell content, or the docs viewer.
- Search inside the sidebar table of contents. This is deferred.
