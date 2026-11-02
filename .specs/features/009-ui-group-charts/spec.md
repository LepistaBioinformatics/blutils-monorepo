# 009 — Charts in the grouped views

Scope: **Large** (`modules/blutils-ui`). This builds on 007 (pager) and 008 (filter, record
list).

## Requirements

- **CH-01:** "By taxonomy" and "By subject" get a **List / Charts** switch. List is the current
  pager. Charts replaces it with a chart panel over all the groups of the view. The filter
  applies, as everywhere. The mode and the selected chart live in the URL (`?m=charts&c=<id>`).
- **CH-02:** The chart panel has:
  - a chart selector;
  - a **Records / Reads** toggle (D-02);
  - a **rank** selector where the chart aggregates by rank (domain → species, default genus).
- **CH-03 (By taxonomy):**
  - **Abundance bars:** one horizontal bar per taxon at the rank, sorted by value and labeled
    with value and share. The top 30 are shown and the rest fold into "Other". Clicking a bar
    opens that group in List mode when it exists at the view's grouping.
  - **Lineage sunburst:** rings from domain outwards, sized by the weight. Clicking a segment
    zooms into it, and a breadcrumb zooms back. A "Filter by …" action adds the `t:` chip for
    the zoomed lineage.
  - **Confidence:** for the top groups by size, an identity strip (min / quartiles / max, with
    the median marked) and a 100% bar of the rule mix (Unanimous / Majority / Ambiguous /
    Unresolved).
- **CH-04 (By subject):**
  - **Stacked composition:** one 100% bar per subject (after the wrap character). The top 8
    taxa overall at the rank get fixed colors; the rest is "Other" (gray). There is a legend,
    and a tooltip per segment.
  - **Heatmap:** subjects × the top 20 taxa. Each cell's shade is the share of the taxon in the
    subject, on a sequential blue scale with a legend. The tooltip shows the value.
  - **Summary:** a table per subject with records, reads, % matched, mean identity, and a small
    rule-mix bar.
- **CH-05 (accessibility):**
  - Every mark has a tooltip (hover and focus) and an accessible label.
  - Identity is never conveyed by color alone (legend or direct labels).
  - Dark mode uses its own validated steps.
  - The charts have no dependency beyond React: plain SVG and HTML.

## Acceptance

- **AC-01:** On the ITS sample, Abundance bars at genus list *Saccharomyces* (or the top genus)
  first, and Reads changes the values.
- **AC-02:** The sunburst zooms and filters. The stacked composition shows one bar per subject
  with the wrap character `.` (7 subjects on ITS).
- **AC-03:** `tsc` is clean, the tests are green, the build succeeds, and a browser check passes
  in light and dark mode.
