# 007 — Context (user decisions, 2026-11-02)

- **D-01 (sidebar):** Keep the grouped-view sidebar as an mdbook-style table of contents. The
  current group is highlighted, and clicking an entry jumps to it. The left/right chevrons and
  the keyboard arrows step through the groups.
- **D-02 (footer):** Hide the site footer while a result is loaded, so nothing outside the
  content scrolls.
- **D-03 (virtualizer):** Use `@tanstack/react-virtual` for the table rows, the group records and
  the sidebar. It replaces the hand-rolled `SectionScroller`.
- **D-04 (deep link):** Sync the current group to the URL (`?g=`). Back/forward steps through
  groups.
- **D-05 (derived from D-04, agent):** `?g=` alone cannot restore a group after a reload, because
  the view type lives in component state and group names are only unique within one view. For
  that reason, the view is synced too, as `?v=`. This decision is the agent's, not the user's;
  confirm during review.

Original request (verbatim):

> A cara registro, taxonomy ou subject o usuário só scrolla dentro dos registros, não a tela inteira

> vamos melhorar a interface de usuário do blutils-ui. Hoje quando o usuário carrega um resultado
> para visualizar no modo tabela ele pagina os resultados, entretanto quero que no lugar de
> paginar ele use rolagem virtual para que o usuário possa em uma única tela, sem precisar
> scrollar a tela, mas sim somente o conteúdo da tabela, rolar por todos os registros.
> Adicionalmente, quando nos modos grouped-by-* que inclue by taxonomy e subject, no lugar de
> fazer uma rolagem virtual de cima a baixo, o usuário poderá por exemplo, ir clicando para o
> lado direito ou esquerdo para ir passando por cada subject ou taxonomy. Use o mesmo estilo de
> botões e navegação adotado pela documentação do rust, que é o mdbook
