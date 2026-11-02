# 002 — Summary

Status: **Done** (2026-11-02).

- `03_blutils_online_viewer.md` was rewritten for the current viewer: opening a result
  (including the sample dataset and links), the record list, sorting, filtering (token table,
  suggestions, taxonomy tree), the three views, the charts, the rules, the record details and
  the theme.
- 13 screenshots (`images/blutils-ui-01..13.png`; 01–08 replaced, 09–13 new) were captured with
  Playwright against the dev server.
- The page was previewed in the real viewer by intercepting the raw GitHub requests with local
  files. All 14 sections render, the token table renders, and all 13 images load.
- In-page `#anchor` links were avoided, because react-markdown renders headings without ids in
  the viewer.
