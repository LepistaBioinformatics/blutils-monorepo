# 002 — Refresh the online viewer documentation

Scope: quick (`modules/blutils`: `docs/book/03_blutils_online_viewer.md` and its images).

## Request

> agora atualize a documentação da tela '/blutils-ui/docs' incluindo os screenshots

## Notes

- The `/docs` page of the viewer fetches the Markdown at runtime from the engine repo
  (`LepistaBioinformatics/blutils`, `docs/book/`, ref `main`). The new content appears on the
  site once it is merged into blutils `main`.
- The screenshots are regenerated from the current UI with the bundled ITS sample, light theme,
  1440×860, plus one dark-theme shot.
