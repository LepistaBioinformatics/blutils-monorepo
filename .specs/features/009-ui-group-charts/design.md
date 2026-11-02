# 009 — Design

## Data (`functions/chart-data.ts`, pure, unit tested)

- `readCount(query)`: reads `/size[_=](\d+)/i`; defaults to 1.
- `weightOf(result, weighting)`: 1 for records, `readCount(query)` for reads.
- `taxonAtRank(result, rank)`: the readable name of the lineage level whose prefix maps to the
  rank (`d`, `p`, `c`, `o`, `f`, `g`, `s`, or the full name). A lineage without that rank gives
  `"Unclassified <rank>"`; no taxon gives `"No match"`.
- `abundance(results, rank, weighting)` returns `[{ name, value, share }]` sorted by value.
- `weightedTree(results, weighting)` returns the lineage tree for the sunburst. It has the same
  shape as `buildTaxonomyTree`, but with weights.
- `confidence(groups, weighting)` returns, per group, the identities (sorted), their
  quartiles, and the rule counts.
- `composition(groups, rank, weighting, topN)` returns `{ taxa: topN names, rows: [{ subject,
  total, shares: number[], other }] }`. The taxa order is fixed by their global total, so a
  color follows its taxon.
- `subjectSummary(groups, weighting)` returns the records, reads, % matched, mean identity and
  rule counts.

## Components (`components/charts/`)

- `ChartPanel`: the selector (pill switch), Records/Reads toggle and rank select, then the
  chart. Tooltip: one shared `ChartTooltip` (absolutely positioned div) driven by
  `onPointerMove` and focus.
- `AbundanceBars`, `LineageSunburst`, `ConfidenceChart`, `StackedComposition`,
  `CompositionHeatmap`, `SubjectSummary`.
- Colors: CSS variables `--series-1..8`, `--series-other` and `--seq-*` in `index.css`, light
  under `:root` and dark under `.dark`.

## Wiring

- `use-results-url-state` gains `mode` (`list` | `charts`, `?m=`) and `chart` (`?c=`).
  `setView` drops `g` and `c`, and keeps `m`.
- Both `Grouped.tsx` views render a List/Charts switch above the content. In Charts mode they
  render `ChartPanel` with their groups. Opening a group from a chart sets `g` and `m=list`.
- `Results` passes `onFilterTaxonomy(path)` (adds a `t:` chip) down for the sunburst.
