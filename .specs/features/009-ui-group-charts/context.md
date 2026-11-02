# 009 — Context (user decisions, 2026-11-02)

- **D-01:** The user asked for chart modes in the grouped views ("implemente modos graficos em
  by taxonomy e subject. Me dê sugestões") and picked every suggestion:
  - **By taxonomy:**
    - abundance bars (per taxon at a chosen rank, click to open the group);
    - lineage sunburst (click to zoom and to filter);
    - confidence per group (identity distribution plus rule mix).
  - **By subject:**
    - stacked composition (100% bar per subject, top N taxa plus Other);
    - heatmap subject × taxon;
    - summary per subject (records, reads, % matched, mean identity, rule mix).
- **D-02 (weighting):** The charts count records by default. A toggle switches to reads, using the
  `size_N` / `size=N` suffix of the query name (dereplicated data); a query without a size
  counts as 1.
- **D-03 (agent, dataviz method):** Identity charts use the validated reference categorical
  palette (8 hues, light and dark steps; `validate_palette.js` passes on `#ffffff` and on
  `#1f2937`). The soft row-bar palette fails the checks for identity use (normal-vision ΔE 8 <
  15, chroma below the floor), so it stays only on the per-record composition bar, where
  position and tooltips carry the identity. The heatmap uses the sequential blue ramp. Since the
  contrast check is a WARN, every chart ships direct labels or a legend plus tooltips, and the
  list stays one click away as the table view.
