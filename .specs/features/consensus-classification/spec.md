# Spec — Classificação Taxonômica por Consenso (Consensus Classification)

> Status: **Documentação de feature existente (as-built)** — reverse-engineered a partir
> do código em `modules/blutils/core`. As requisições abaixo descrevem o comportamento
> atual do algoritmo, não um estado desejado futuro.
>
> Última atualização: 2026-06-22

## 1. Visão geral

O coração funcional do `blutils` é a **atribuição de uma taxonomia de consenso a cada
sequência de consulta (query)** a partir dos alinhamentos produzidos pelo `blastn` do
NCBI BLAST+. O algoritmo não confia cegamente no melhor hit: ele agrupa os alinhamentos
por qualidade, reconcilia divergências taxonômicas entre múltiplos hits equivalentes e
**rebaixa o rank atribuído de acordo com a identidade percentual observada** (um match de
96% não pode ser chamado de "espécie" se o cutoff de espécie do táxon é 97%).

O use case de entrada é
`core/src/use_cases/run_blast_and_build_consensus/` e o núcleo de classificação é
`core/src/use_cases/build_consensus_identities/`.

### Glossário

| Termo | Significado |
|-------|-------------|
| **query** | Sequência de entrada submetida ao BLAST |
| **subject / hit** | Sequência de referência alinhada à query |
| **rank** | Nível Lineano (Domain → Species) ou rank não-Lineano (`Other`, ex. strain, serovar) |
| **cutoff / backbone** | Limiar de identidade percentual mínimo para cada rank de um táxon |
| **interpolated identity** | Cutoff calculado para ranks não-Lineanos por interpolação linear entre os ranks default vizinhos |
| **consensus bean** | Ocorrência agregada de um táxon num dado rank (identifier + contagem + accessions) |
| **LCA** | *Lowest Common Ancestor* — rank mais profundo em que todos os hits equivalentes concordam |

## 2. Entradas e saídas

### Entradas (FR-IN)

- **FR-IN-01** — Arquivo de saída tabular do `blastn` (TSV, 13 colunas, sem header):
  `query, subject_accession, subject_taxid, perc_identity, align_length, mismatches,
  gap_openings, q_start, q_end, s_start, s_end, e_value, bit_score`.
  Definido em `build_consensus_identities/mod.rs::get_results_dataframe`.
- **FR-IN-02** — Arquivo de taxonomias (JSON, `TaxonomiesMap`) mapeando `taxid → lineage`.
  Cada item expõe `numeric_lineage` e `text_lineage`.
- **FR-IN-03** — `taxon` selecionado (`Fungi`, `Bacteria`, `Eukaryotes`, `Custom`) que
  determina os cutoffs de identidade por rank.
- **FR-IN-04** — `strategy` de consenso (`Cautious` ou `Relaxed`).
- **FR-IN-05** — Flag `use_taxid`: quando `true`, o consenso usa o `numeric_lineage`
  (taxids); caso contrário, usa o `text_lineage`.
- **FR-IN-06** — `custom_taxon_values` opcional (arquivo YAML/JSON) obrigatório quando
  `taxon = Custom`.
- **FR-IN-07** — A lista de headers das queries de entrada (`blast_output.headers`) é usada
  para reconciliar queries sem nenhum hit.

### Saída (FR-OUT)

- **FR-OUT-01** — Um `Vec<ConsensusResult>`, um elemento por query, serializado como
  `BlutilsOutput` (JSON/YAML, camelCase) — o **contrato de dados** consumido pela UI.
- **FR-OUT-02** — Cada resultado é `ConsensusFound(QueryWithConsensus)` ou
  `NoConsensusFound(QueryWithoutConsensus)`.
- **FR-OUT-03** — O táxon de consenso (`TaxonomyBean`) carrega: `reached_rank`,
  `max_allowed_rank`, `identifier`, `perc_identity`, `bit_score`, `taxonomy` (string da
  linhagem), `mutated`, `single_match` e `consensus_beans`.

## 3. Requisitos funcionais

### 3.1 Carga e junção (FR-JOIN)

- **FR-JOIN-01** — Carregar a saída do BLAST como `DataFrame` (polars) com schema tipado fixo.
- **FR-JOIN-02** — Carregar as taxonomias e materializar um `DataFrame` `(taxid, taxonomy)`,
  escolhendo `numeric_lineage` ou `text_lineage` conforme `use_taxid` (FR-IN-05).
- **FR-JOIN-03** — Executar um **left join** `blast.subject_taxid = taxonomies.taxid`,
  preservando todos os hits do BLAST mesmo quando o taxid não existe na tabela de taxonomias.
- **FR-JOIN-04** — Agrupar as linhas resultantes por `query` em `BlastQueryResult`
  (`fold_results_by_query`).

### 3.2 Reconciliação de queries sem hit (FR-NOHIT)

- **FR-NOHIT-01** — Toda query presente em `headers` (FR-IN-07) que não apareça entre os
  resultados do BLAST deve gerar um `BlastQueryResult` com `results = None`.
- **FR-NOHIT-02** — Queries com `results = None` resolvem para `NoConsensusFound`.

### 3.3 Seleção por bit-score (FR-SCORE)

- **FR-SCORE-01** — Para cada query, agrupar os hits por `bit_score` (inteiro) e ordenar os
  grupos em ordem **decrescente** (melhor primeiro). Ver `find_single_query_consensus`.
- **FR-SCORE-02** — Apenas o grupo de **maior bit-score** é efetivamente avaliado: cada
  ramo (`len == 0`, `len == 1`, `len > 1`) retorna de dentro do laço. Grupos de bit-score
  inferiores são descartados. *(Ver evaluação CONCERN-04 no design.)*
- **FR-SCORE-03** — Grupo vazio → `NoConsensusFound`.

### 3.4 Caminho de hit único (FR-SINGLE)

- **FR-SINGLE-01** — Quando o grupo de maior bit-score tem exatamente 1 hit, fazer parse da
  taxonomia literal (`d__2;p__...;s__...`) em `Vec<TaxonomyBean>` ordenado do rank mais
  geral para o mais específico.
- **FR-SINGLE-02** — Construir `InterpolatedIdentity` para o táxon e ranks da linhagem.
- **FR-SINGLE-03** — Truncar a linhagem para o rank mais profundo justificado pela
  `perc_identity` observada via `get_adjusted_taxonomy_by_identity`.
- **FR-SINGLE-04** — Marcar o resultado com `single_match = true` e produzir um único
  `ConsensusBean`.

### 3.5 Caminho multi-táxon / LCA (FR-MULTI)

- **FR-MULTI-01** — Quando o grupo de maior bit-score tem >1 hit, ordenar os registros por
  `(comprimento da linhagem, perc_identity, align_length, subject_accession)`.
- **FR-MULTI-02** — Selecionar a **linhagem de referência** conforme a estratégia:
  - `Cautious` → linhagem mais **curta** (primeiro registro ordenado).
  - `Relaxed` → linhagem mais **longa** (último registro ordenado).
- **FR-MULTI-03** — Percorrer a linhagem de referência do rank mais geral para o mais
  específico. Em cada rank, coletar o conjunto de `rank+identifier` presentes entre os hits
  do mesmo nível.
- **FR-MULTI-04** — Enquanto todos os hits concordam num rank (conjunto de tamanho 1),
  descer um nível, registrando esse rank como o consenso corrente.
- **FR-MULTI-05** — No primeiro rank em que os hits **divergem** (conjunto > 1), retroceder
  ao último rank concordante (`index - 1`) — o **LCA** — e encerrar a descida (`break`).
- **FR-MULTI-06** — Em ambos os casos, delegar a montagem do táxon final a
  `build_blast_consensus_identity`, agregando os `consensus_beans` daquele nível.

### 3.6 Ajuste de rank por identidade (FR-IDENT)

- **FR-IDENT-01** — Cada táxon possui um **backbone** de cutoffs por rank Lineano default:
  - Fungi / Eukaryotes: Species 97, Genus 95, Family 90, Order 85, Class 80, Phylum 75, Domain 60.
  - Bacteria: Species 99, Genus 97, Family 92, Order 85, Class 80, Phylum 75, Domain 60.
  - Custom: valores fornecidos pelo usuário (`CustomTaxon`).
- **FR-IDENT-02** — Ranks não-Lineanos (`Other`, ex. strain/serovar) recebem cutoff por
  **interpolação linear** entre os cutoffs dos ranks default que delimitam a janela onde o
  rank ocorre (`interpolate_identities`).
- **FR-IDENT-03** — `get_rank_adjusted_by_identity(identity)` retorna o rank mais profundo
  cujo cutoff é satisfeito pela identidade observada (`max_allowed_rank`).
- **FR-IDENT-04** — `get_adjusted_taxonomy_by_identity(identity, lineage)` mantém apenas os
  ranks cujo cutoff `<= identity`, truncando a linhagem.
- **FR-IDENT-05** — Quando `reached_rank != max_allowed_rank`, o bean é marcado como
  `mutated = true` (o rank foi rebaixado em relação à anotação original).

### 3.7 Agregação de consensus beans (FR-BEAN)

- **FR-BEAN-01** — `ConsensusBean::fold_consensus_list` agrupa beans por `rank__identifier`,
  somando `occurrences` e deduplicando `accessions`.
- **FR-BEAN-02** — Os beans são ordenados por `occurrences` (desc) e, em empate, por
  `identifier`.

## 4. Requisitos não-funcionais

- **NFR-01 (Paralelismo)** — A classificação por query roda em paralelo via `rayon`
  (`into_par_iter`) em `build_consensus_identities`.
- **NFR-02 (Arquitetura)** — Toda a lógica de consenso vive em `core/use_cases` (camada de
  domínio), sem efeitos de I/O externos além dos adapters; respeita a arquitetura hexagonal
  descrita no `CLAUDE.md`.
- **NFR-03 (Contrato de dados)** — Os DTOs serializados (`#[serde(rename_all = "camelCase")]`)
  devem permanecer espelhados em `blutils-ui/src/types/BlutilsResult.ts`.
- **NFR-04 (Determinismo)** — Ordenações usam desempates explícitos (identifier, accession)
  para resultados estáveis apesar do paralelismo.

## 5. Critérios de aceite (derivados do comportamento atual)

- **AC-01** — Query sem hits no BLAST mas presente nos headers → `NoConsensusFound`.
- **AC-02** — Query com um único melhor hit a 100% de identidade → linhagem completa,
  `single_match = true`, `reached_rank = Species` (se o táxon define Species).
- **AC-03** — Query com um único melhor hit a identidade abaixo do cutoff de espécie →
  linhagem truncada no rank justificado, `mutated = true`.
- **AC-04** — Query com múltiplos hits de mesmo bit-score que divergem no rank de gênero →
  consenso retorna no rank de família (LCA), com `consensus_beans` listando os gêneros
  divergentes.
- **AC-05** — Estratégia `Cautious` vs `Relaxed` produz linhagens de referência diferentes
  quando os hits têm profundidades taxonômicas distintas.

## 6. Rastreabilidade (requisito → código)

| Requisito | Arquivo |
|-----------|---------|
| FR-IN, FR-JOIN, FR-NOHIT | `build_consensus_identities/mod.rs` |
| FR-SCORE, FR-SINGLE | `build_consensus_identities/find_single_query_consensus.rs` |
| FR-MULTI | `build_consensus_identities/find_multi_taxa_consensus.rs` |
| FR-IDENT | `domain/dtos/linnaean_ranks.rs` (`InterpolatedIdentity`), `domain/dtos/taxon.rs` |
| FR-BEAN | `domain/dtos/consensus_result.rs` (`ConsensusBean`) |
| FR-OUT | `domain/dtos/consensus_result.rs`, `domain/dtos/taxonomy_bean.rs`, `use_cases/write_blutils_output.rs` |
| Orquestração | `run_blast_and_build_consensus/mod.rs` |
