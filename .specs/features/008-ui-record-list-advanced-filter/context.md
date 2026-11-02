# 008 — Context (user decisions, 2026-11-02)

- **D-01 (rule names):** Rules are shown as one word. 1 = **Unanimous** (a single consensus bean),
  2 = **Majority** (the top bean has ≥ 2× the occurrences of the second). Today's rule 0 is split
  into **Ambiguous** (beans exist but none wins) and **Unresolved** (no beans). Clicking the word
  opens a modal that explains every rule and highlights the current one.
- **D-02 (filter UX):** A single search box. It accepts free text (matched against the query, the
  proposed taxon and the taxonomy) and tokens: `q:` `t:` `rank:` `rule:` `id>=` `bits>=`
  `has:match` / `no:match`. Each token becomes a removable chip. Buttons below the box insert
  tokens, and a "Taxonomy tree" button opens a collapsible tree with counts; clicking a node adds
  a `t:` chip for that lineage. The box replaces the table header searches and the "Unmatched"
  dropdown.
- **D-03 (filter in URL):** The committed filter lives in the URL as `?f=`, next to `?v=`/`?g=`.
  It survives reloads, links and view switches.
- **D-04 (shared entry):** The Table view and both grouped views use the same datatable-style
  record entry, with the same aligned columns.

Original request (verbatim):

> vamos converter os registros da tabela em elementos de lista como uma datatable. Todas as
> informações principais devem ficar em destaque nas entradas da lista, a query ainda deve ser o
> titulo principal a esquerda seguida pelo proposed taxon e a rule utilizada pade propor o taxon.
> Vamos transformar a rule de algo numérico em algo textual que seja uma unica palavra, porém se
> o usuário clicar na rule ele abrirá um modal explicando como funciona a rule. As demais
> informações de composição, identidade, bitscore devem ficar alinhadas a direita, entretanto o
> grafico da composição agora deve ser um grafico circular com os valores de identidade e
> bitscore dentro dele, um ao lado do outro. As demais informações devem ficar logo abaixo do
> titulo principal. As colunas principais devem ser alinhadas, de modo que o usuário possa fazer
> uma omparação direta entre valores do proposed taxon por exemplo. Isso facilitará a
> visualização. Vamos também implementar um filtro avançado, que deve servir para todos os modos
> de tela, apesar de ser um filtro avançado a aparencia deve ser simples, ele poderá usar os
> filtros clicando em botões abaixo que permitem por exemplo filtrar por /q=<valor da query> e ao
> mesmo tempo /t=<taxonomia com espaços e tudo mais>, adicionalmente ele poderá ver a taxonomia
> real num formato de árvore que permitirá ele ir clicando e abrindo as taxonomias e filtrando.
> Além disso, o usuário tem a opção de definir o 'wrap character', que permite que o usuário
> selecione um caractere para quebrar o texto em partes e agrupar registros, por exemplo
> 'SRR20752596.1_size_54' se usar o ponto como caractere delimitador ele consegue agrupar os
> resultados nas telas de grouped-by-* esse deve ser também disponível para o usuário. Caso
> encontrar uma maneira melhor de filtrar os registros me sugira. Uma maneira melhora seria
> permitir uma caixa de filtragem só que ele pode digitar qualquer valor, acho uma ideia boa
> também. A seleção entre os modos de tela deve ficar centralizado e caso o usuário alterar entre
> eles os filtros continuam aplicados.
