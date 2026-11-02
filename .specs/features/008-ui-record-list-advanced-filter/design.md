# 008 — Design

## Data flow

```
blutilsResult.results
   │  parseFilter(?f)                       functions/record-filter.ts
   ▼
filteredResults = results.filter(r => matchesFilter(r, tokens))
   │  existing grouping (view + wrap character)       Results/index.tsx
   ▼
Table: VirtualList<RecordItem>     Grouped: GroupPager → VirtualList<RecordItem>
```

The filter is applied once, in `Results`, before grouping (FLT-06). `Table` stops filtering on
its own: it receives the records plus the unfiltered total, which it needs for the stats
(FLT-07).

## Modules

| Module | Responsibility |
|---|---|
| `functions/record-filter.ts` | `parseFilter(text) → FilterToken[]`, `serializeFilter(tokens)`, `matchesFilter(result, tokens)`, `readableLineage(taxonomy)`, `tokenLabel(token)`. Pure, unit tested. |
| `functions/taxonomy-tree.ts` | `buildTaxonomyTree(results) → TreeNode[]` (name, rank, path, count, children sorted by name). Pure. |
| `types/BeanSelectionRules.ts` | Adds `getRuleKind(beans) → "unanimous" \| "majority" \| "ambiguous" \| "unresolved"` and `RULE_INFO` (label, one-line summary, explanation). The numeric `rule` stays as is (existing callers and tests). |
| `components/RuleBadge.tsx` | A button showing the rule word. A click opens `RuleModal`, which lists all four rules with the current one highlighted. |
| `components/CompositionDonut.tsx` | SVG donut (96px): top 5 beans plus an "others" segment, each a `<path>` with a `<title>`. The center shows identity % and bit score side by side. No flowbite Tooltip, because there are too many per list. |
| `components/RecordItem.tsx` | One entry on the shared grid (`RECORD_GRID` class constant) and `RecordHeader` with the same grid. |
| `components/FilterBar.tsx` | Input, chips, token buttons, Matched/Unmatched toggles, Clear, and the tree button. Owns only the draft text; committed tokens come from props. |
| `components/TaxonomyTreeModal.tsx` | flowbite `Modal` with a collapsible tree. Clicking a name calls `onPick(path)`. |

## Token grammar (FLT-02)

```
filter := item (WS item)*
item   := key ":" value | metric op number | value
key    := q | t | rank | rule | has | no         (case-insensitive)
metric := id | bits ;  op := >= | <= | > | <
value  := "quoted text" | bare-word
```

- An unknown `key:` is treated as free text, which keeps the parser forgiving.
- `no:match` is normalized to the token `{ kind: "has", value: "no-match" }`.
- `serializeFilter` quotes any value that contains a space or a quote.
- `readableLineage("d__bacteria;p__bacillota")` returns `"bacteria > bacillota"`: lower case,
  hyphens as spaces, separated by `" > "`.
- `t:` and free text match it as a substring. A tree node at path `bacteria > bacillota` becomes
  `t:"bacteria > bacillota"`. Because lineages start at the root, this matches exactly the
  records under that node.

## Layout

- **Grid.** `RECORD_GRID = "grid grid-cols-1 gap-x-6 md:grid-cols-[minmax(14rem,2fr)_minmax(12rem,1.5fr)_7.5rem_6.5rem] items-center"`.
  The header and the list both use `[scrollbar-gutter:stable]`. The header is `overflow-hidden`
  and sits outside the scroll box; the gutter keeps the columns aligned with the scrollbar
  present.
- **Toolbar.** A three-column grid. Reset is on the left, the segmented `Button.Group` view
  selector is centered, and the wrap-character input with a help tooltip is on the right.
- **FilterBar.** It sits under the toolbar and spans every view (`shrink-0`).

## Removed

- `TableView/Row.tsx`, `TableView/SearchableHeadCell.tsx` and `components/MiniChart.tsx`, with
  their tests. They are replaced by `RecordItem`, `FilterBar` and `CompositionDonut`.
- Regression IDs that move:
  - BUG-U02 (null beans) is ported to the donut and `RecordItem`.
  - BUG-U04 (whitespace-only term) is ported to `FilterBar`.
  - BUG-U12 (rule 0 has no secondary span) is ported to `RecordItem`.
  - BUG-U14, U16 and U21 become obsolete, because their components were deleted.

## Alternatives considered

- **Separate fields per filter.** The user picked the single box (D-02).
- **Keep `<table>`.** Rejected: the multi-line entry, with the title, the info below it and the
  donut, fits a grid list better, and the grid keeps the columns aligned explicitly.
