# 007 — Design

## Layout (LAY-01, LAY-02)

```
Shell <main h-dvh flex-col overflow-hidden>   (fullscreen = result loaded)
├── Navbar
├── <div grow min-h-0 flex-col> → <Outlet context={{ setFullscreen }}>
│     Results <div h-full min-h-0 flex-col>
│     ├── toolbar (Reset · wrap character · view select)        shrink-0
│     └── view area                                             flex-1 min-h-0
│           Table:   stats/filters (shrink-0) + scroll box (flex-1 overflow-auto, sticky thead)
│           Grouped: GroupPager
└── footer — not rendered while fullscreen
```

- `Shell` keeps a `fullscreen` flag and passes `setFullscreen` through the react-router
  `<Outlet context>`. While the flag is set:
  - `main` switches from `min-h-screen` to `h-dvh overflow-hidden` (dynamic viewport height, so the mobile URL bar does not hide the bottom navigation);
  - the content wrapper gets `min-h-0`;
  - the footer is not rendered.
- `Results` reads the context with `useOutletContext()`. The context is optional, because tests
  render `Results` without `Shell`. `Results` sets the flag while `blutilsResult` is non-null and
  clears it on unmount.
- Every flex ancestor of a scroll box needs `min-h-0`, otherwise the box grows instead of
  scrolling.

## Virtualization (TBL-02, GRP-02, GRP-05)

The project uses `@tanstack/react-virtual` ^3.14 (`useVirtualizer`) with
`getScrollElement = () => scrollRef.current`.

- **Table.** The view keeps the semantic `<table>`. The `<tbody>` renders a top spacer `<tr>`
  (`height = items[0].start`), the visible `Row`s (`ref={virtualizer.measureElement}` and
  `data-index`), and a bottom spacer `<tr>` (`getTotalSize() - lastItem.end`). This allows
  variable row heights, which `TaxonomyCell` produces. The header is `sticky top-0 z-10` with an
  opaque background. The scroll box is our own `div`, not flowbite's wrapper.
- **`components/VirtualList.tsx`.** A small generic list of absolutely positioned divs with
  `measureElement`. It is used for the records of the current group and for the sidebar. It
  exposes `scrollToIndex` through a ref prop (`listRef`), so the sidebar can keep the active
  entry in view.
- Filter or group changes reset the scroll position (`scrollToOffset(0)`).

## Group pager (GRP-01..05) — `components/GroupPager.tsx`

```tsx
interface GroupPagerProps {
  groups: { name: string; label: ReactNode; count: number }[];
  current: number;
  onNavigate: (index: number) => void;
  children: ReactNode;   // header + records of the current group
}
```

- **Sidebar.** A `VirtualList` of `<a role="link">` entries. The active entry carries
  `aria-current="page"` and the brand color, as mdbook's `.active` does.
- **Wide screens (`lg:`).** Two `<button>` bars, `aria-label="Previous group"` and
  `aria-label="Next group"`. They are absolutely positioned at the left and right of the
  content area, full height and about 90px wide, with a `FaAngleLeft`/`FaAngleRight` icon at
  2.5em. The icon is gray and turns darker on hover, and the bar gets a hover background. This
  mirrors mdbook's `.nav-chapters` / `.nav-wide-wrapper`. The content gets horizontal padding,
  so the bars never cover data.
- **Narrow screens.** A bottom bar holds two rounded 90px buttons with the same chevrons, as
  mdbook's `.mobile-nav-chapters` does.
- A chevron is not rendered at either end.
- **Keyboard.** A `keydown` listener on `document`:
  - It ignores events with `altKey`, `ctrlKey`, `metaKey` or `shiftKey`.
  - It ignores events whose target is `input`, `textarea`, `select` or `[contenteditable]`.
  - It ignores events while a `[role="dialog"]` is in the DOM (`ConsensusModal`).
  - Otherwise, ArrowLeft goes to the previous group and ArrowRight to the next one, as in
    mdbook's `book.js`.

The two `Grouped.tsx` files (taxonomy and subject) keep their own row renderers. Each one
renders `GroupPager`, the group header (moved out of `SectionScroller`), and a `VirtualList` of
rows. Both `SectionScroller.tsx` files and their tests are deleted.
`hooks/use-element-in-view` is deleted only if nothing else uses it.

## URL state (GRP-06, GRP-07) — `hooks/use-results-url-state.ts`

```ts
const { view, group, setView, setGroup } = useResultsUrlState();
// view:  ViewType parsed from ?v (unknown/missing → Table)
// group: string | null from ?g
// setView(v): sets v, deletes g, replace = false
// setGroup(name): sets g, keeps all other params, replace = false (back/forward walk groups)
```

- All writes use a functional update (`setSearchParams(prev => …)`), so `?p` is never dropped.
- `ResultLoader` changes from `setSearchParams({ p })` to a functional update that only sets `p`.
  That way, `?p=…&v=…&g=…` survives the load (AC-04).
- `Reset` still calls `setSearchParams({})`, which clears everything.
- The grouped view resolves `current = groups.findIndex(g => g.name === group)` and uses 0 when
  the group is not found (GRP-06, GRP-07). It does not rewrite the URL on a fallback, which
  avoids a history entry on every regroup.

## Table stats (TBL-04)

`ResultsPaginationStats` renders `"<total> records"` for the Table view and keeps
`"<total> in <groups> groups"` for the grouped views. The component name is kept to limit churn.

## Testing approach

jsdom has no layout: `clientHeight` is 0, so the virtualizer would render nothing.
`setupTests.ts` fakes `HTMLElement.prototype.clientHeight`/`clientWidth` (800/1200) and
`offsetHeight` = 40 for `[data-index]` items. Without a ResizeObserver entry, virtual-core
measures items with `offsetHeight`. Windowing therefore really happens in tests (about 20–40 items
mounted), and small fixtures still render fully. jsdom cannot reproduce the real layout. The
unbounded-first-render crash (see tasks.md notes) is covered only by the browser check.

## Alternatives considered

- **Keep the hand-rolled `SectionScroller`.** Rejected: it needs fixed row heights and has magic
  correction factors (D-03).
- **Use a `div` grid instead of `<table>` for the virtualized table.** Rejected: spacer rows keep
  the existing `Row` component, the semantics and the tests.
- **Store the group index in the URL.** Rejected: indexes shift when the wrap character
  regroups, while names are stable.
