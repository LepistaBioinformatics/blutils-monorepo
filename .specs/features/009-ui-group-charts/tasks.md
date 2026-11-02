# 009 — Tasks

| ID | Task | Status |
|---|---|---|
| T1 | `functions/chart-data.ts` + unit tests | Done |
| T2 | Palette CSS variables (validated) + `ChartTooltip` | Done |
| T3 | `AbundanceBars`, `LineageSunburst`, `ConfidenceChart` | Done |
| T4 | `StackedComposition`, `CompositionHeatmap`, `SubjectSummary` | Done |
| T5 | `ChartPanel` + URL state (`m`, `c`) + List/Charts switch in both grouped views | Done |
| T6 | Component/integration tests; gates; browser check light/dark | Done |

## Results (2026-11-02)

- Gates: `tsc` is clean, 37 suites and 517 tests pass (warm and with `--no-cache`), and
  `CI=true yarn build` compiles.
- Tests: 12 chart-data unit tests, 11 ChartPanel tests, and 5 Results integration tests covering
  the `?m=charts` deep link, a bar opening a group, the sunburst filter, and view switches.
- Browser check on the ITS sample, in light and dark: all six charts render with no errors.
  - By records, *Cryptococcus* leads (392 records, 29%); by reads, *Saccharomyces* leads (51,123
    reads, 52%).
  - A bar click opens the group in List mode.
  - The sunburst zoom and its "Filter by" add a `t:` chip.
- Fixes found while looking at the charts:
  - The sunburst painted every ring one hue when the lineage had a single root (one kingdom).
    Single-child chains are now neutral, and colors start at the first branching level.
  - The tooltip ignores pointer events without coordinates.
- The `k__` rank is now read as "kingdom" (UNITE-style ITS lineages) in `translateRank`, the
  tree and the modal.
- The composition palette check: the user's soft row palette fails the dataviz checks for
  identity use, so it is kept only on the per-record bar. The charts use the validated
  reference palette (D-03).
