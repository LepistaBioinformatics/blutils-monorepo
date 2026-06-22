# STATE — Memória do Projeto

> Memória persistente entre sessões: decisões, concerns e ideias diferidas.
> Datas em formato absoluto.

## Decisões

- **2026-06-22** — Algoritmo de classificação por consenso documentado *as-built* em
  `.specs/features/consensus-classification/{spec.md,design.md}`, via engenharia reversa do
  código em `modules/blutils/core`. Nenhum código foi alterado — apenas documentação.

## Concerns abertos (do design as-built)

Detalhe completo em `.specs/features/consensus-classification/design.md` §8.2.

- **CONCERN-01** — Caminhos de erro da classificação usam `panic!`/`unwrap` em vez de
  propagar `MappedErrors`; um panic numa thread paralela pode derrubar o processo.
- **CONCERN-02** — Cutoffs de identidade por táxon (`taxon.rs`) marcados com `TODO: review`
  e sem fonte científica citada — parâmetro mais sensível do algoritmo.
- **CONCERN-03** — `bit_score` truncado de `Float64` para `i64`; afeta o agrupamento por
  bit-score.
- **CONCERN-04** — `find_single_query_consensus` avalia apenas o grupo de maior bit-score
  (sempre retorna na 1ª iteração do laço); confirmar se é intencional.
- **CONCERN-05** — Mapeamento posicional de colunas em `fold_results_by_query` acoplado à
  ordem do schema, sem proteção de compilação.
- **CONCERN-06** — `fold_results_by_query` materializa o DataFrame inteiro e itera fora do
  motor lazy do polars.

## Todos diferidos

- Cobrir AC-01..AC-05 (`spec.md`) com testes unitários, em especial LCA (FR-MULTI-05) e
  truncamento por identidade (FR-IDENT-04).
- Avaliar mitigações dos CONCERNs acima (ver `design.md` §8.3).

## Preferências

- Idioma de trabalho: Português (termos técnicos e identificadores preservados em inglês).
