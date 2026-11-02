# 008 — Result viewer: datatable-style record list and advanced filter

Scope: **Large** (`modules/blutils-ui`). This builds on `007-ui-virtual-scroll-group-pager`: the
virtual lists, the group pager and the `?v=`/`?g=` URL state.

## Problem

Records are shown as table rows. The proposed taxon, the rule (a bare number), the identity, the
bit score and a bar chart sit in loosely aligned cells, so values are hard to compare across
records. Filtering is split between two header searches and a dropdown, and only the Table view
has it. The wrap character is a bare input, and the view select sits on the right.

## Requirements

### Record entry (all views, D-04)

- **REC-01**: Each record is a list entry laid out on a grid that is shared by all entries in the
  list, so that the columns line up:
  - **Left:** the query as the main title, with copy and open-details actions.
  - **Under the title:** the secondary information (reached rank and taxonomy lineage).
  - **Next column:** the proposed taxon (the scientific name at the reached rank, plus the most
    probable bean when it differs).
  - **Next column:** the rule word.
  - **Right-aligned:** the composition donut.
- **REC-02**: A header row labels the columns (Query, Proposed taxon, Rule, Composition) and stays
  visible while the list scrolls.
- **REC-03**: A record without a taxon shows the title and "No significant match" in the
  proposed-taxon column, and leaves the other columns empty.

### Rule (D-01)

- **RULE-01**: The rule is shown as one word: Unanimous, Majority, Ambiguous or Unresolved.
  Ambiguous means the record has beans but none wins. Unresolved means it has no beans.
- **RULE-02**: Clicking the word opens a modal that explains all four rules, with the current
  rule highlighted.

### Composition donut

- **DON-01**: A circular (donut) chart of the consensus beans by occurrences. It shows the top 5
  beans; the remainder becomes one "others" segment. Each segment has an accessible title
  ("<sciname> ×<n>").
- **DON-02**: Inside the donut, the identity (%) and the bit score are shown side by side.

### Advanced filter (all views, D-02, D-03)

- **FLT-01**: One search box above the content, used by every view. Free text matches the query,
  the proposed taxon and the taxonomy (readable names), case-insensitively.
- **FLT-02**: Tokens:
  - `q:<text>` matches the query;
  - `t:<text>` matches the taxonomy. It accepts quotes, for example `t:"bacillus subtilis group"`.
  - `rank:<rank>` matches the reached rank;
  - `rule:<word>` matches the rule word;
  - `id>=` / `id<=` / `id>` / `id<` with a number filter on identity;
  - `bits` with the same operators filters on the bit score;
  - `has:match` / `no:match` keep records with or without a taxon.

  All tokens are combined with AND.
- **FLT-03**: Pressing Enter commits the typed text as chips. Each chip has a remove button. An
  empty or whitespace-only entry adds nothing.
- **FLT-04**: Buttons below the box insert a token prefix into the box and focus it: Query,
  Taxonomy, Rank, Rule, Identity, Bit score. Two toggle buttons, Matched and Unmatched, add or
  remove `has:match` / `no:match` directly. A "Clear" button removes every chip.
- **FLT-05**: A "Taxonomy tree" button opens a modal:
  - It holds a collapsible tree built from the lineages of the loaded result.
  - Each node shows its record count.
  - Clicking a node's name adds a chip that matches records whose lineage passes through that
    node (`t:` with the full path).
- **FLT-06**: The committed filter lives in `?f=`. It is kept when switching views, and Reset
  clears it. The filter applies before grouping, so the grouped views only show groups that
  still have records. The current group falls back to the first group when it is filtered out.
- **FLT-07**: The stats show "<shown> of <total> records" when a filter is active, and
  "<total> records" otherwise.

### Toolbar

- **TB-01**: The view selector is a centered, segmented button group with three options: Table,
  By taxonomy, By subject.
- **TB-02**: Reset stays on the left. The wrap character sits on the right. It can be edited in
  the grouped views and is disabled in the Table view. Its help text explains it, for example
  "." groups `SRR20752596.1_size_54` under `SRR20752596`.

## Acceptance criteria

- **AC-01**: In the Table view and in the grouped views of the zymo mock, the column positions are
  identical across entries. In the browser, the left edges of the proposed-taxon and rule columns
  are equal for all mounted entries.
- **AC-02**: `q:SRR20752596 t:bacillus id>=99` narrows the list. Switching to "By taxonomy" keeps
  the chips and shows only the matching groups. Reloading the URL restores the chips.
- **AC-03**: Clicking a tree node adds a `t:` chip, and the list only shows records under that
  node.
- **AC-04**: Clicking a rule word opens the rules modal with that rule highlighted.
- **AC-05**: Gates pass: `tsc` is clean, the tests are green and the build succeeds. A browser
  check passes at 1280×800 and 390×844.

## Out of scope

- Sorting by column. This is deferred.
- Saving named filters.
- Changes to `ConsensusModal`'s content.
