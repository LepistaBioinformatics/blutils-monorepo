# Spec — Consensus Taxonomic Classification (Consensus Classification)

> Status: **Documentation of an existing feature (as-built)** — reverse-engineered from
> the code in `modules/blutils/core`. The requirements below describe the current
> behavior of the algorithm, not a desired future state.
>
> Last updated: 2026-06-22

## 1. Overview

The functional heart of `blutils` is the **assignment of a consensus taxonomy to each
query sequence** from the alignments produced by NCBI BLAST+ `blastn`. The algorithm
does not blindly trust the best hit: it groups alignments by quality, reconciles
taxonomic divergences among multiple equivalent hits, and **demotes the assigned rank
according to the observed percent identity** (a 96% match cannot be called "species" if
the taxon's species cutoff is 97%).

The entry use case is
`core/src/use_cases/run_blast_and_build_consensus/` and the classification core is
`core/src/use_cases/build_consensus_identities/`.

### Glossary

| Term | Meaning |
|------|---------|
| **query** | Input sequence submitted to BLAST |
| **subject / hit** | Reference sequence aligned to the query |
| **rank** | Linnaean level (Domain → Species) or non-Linnaean rank (`Other`, e.g. strain, serovar) |
| **cutoff / backbone** | Minimum percent identity threshold for each rank of a taxon |
| **interpolated identity** | Cutoff computed for non-Linnaean ranks by linear interpolation between the neighboring default ranks |
| **consensus bean** | Aggregated occurrence of a taxon at a given rank (identifier + count + accessions) |
| **LCA** | *Lowest Common Ancestor* — deepest rank at which all equivalent hits agree |

## 2. Inputs and outputs

### Inputs (FR-IN)

- **FR-IN-01** — `blastn` tabular output file (TSV, 13 columns, no header):
  `query, subject_accession, subject_taxid, perc_identity, align_length, mismatches,
  gap_openings, q_start, q_end, s_start, s_end, e_value, bit_score`.
  Defined in `build_consensus_identities/mod.rs::get_results_dataframe`.
- **FR-IN-02** — Taxonomies file (JSON, `TaxonomiesMap`) mapping `taxid → lineage`.
  Each item exposes `numeric_lineage` and `text_lineage`.
- **FR-IN-03** — Selected `taxon` (`Fungi`, `Bacteria`, `Eukaryotes`, `Custom`) that
  determines the identity cutoffs per rank.
- **FR-IN-04** — Consensus `strategy` (`Cautious` or `Relaxed`).
- **FR-IN-05** — `use_taxid` flag: when `true`, the consensus uses the `numeric_lineage`
  (taxids); otherwise, it uses the `text_lineage`.
- **FR-IN-06** — Optional `custom_taxon_values` (YAML/JSON file), required when
  `taxon = Custom`.
- **FR-IN-07** — The list of input query headers (`blast_output.headers`) is used
  to reconcile queries with no hits.

### Output (FR-OUT)

- **FR-OUT-01** — A `Vec<ConsensusResult>`, one element per query, serialized as
  `BlutilsOutput` (JSON/JSONL/YAML, camelCase) — the **data contract** consumed by the UI.
- **FR-OUT-02** — Each result is `ConsensusFound(QueryWithConsensus)` or
  `NoConsensusFound(QueryWithoutConsensus)`.
- **FR-OUT-03** — The consensus taxon (`TaxonomyBean`) carries: `reached_rank`,
  `max_allowed_rank`, `identifier`, `perc_identity`, `bit_score`, `taxonomy` (lineage
  string), `mutated`, `single_match` and `consensus_beans`.

## 3. Functional requirements

### 3.1 Loading and joining (FR-JOIN)

- **FR-JOIN-01** — Load the BLAST output as a `DataFrame` (polars) with a fixed typed schema.
- **FR-JOIN-02** — Load the taxonomies and materialize a `(taxid, taxonomy)` `DataFrame`,
  choosing `numeric_lineage` or `text_lineage` according to `use_taxid` (FR-IN-05).
- **FR-JOIN-03** — Perform a **left join** `blast.subject_taxid = taxonomies.taxid`,
  preserving all BLAST hits even when the taxid does not exist in the taxonomies table.
- **FR-JOIN-04** — Group the resulting rows by `query` into `BlastQueryResult`
  (`fold_results_by_query`).

### 3.2 Reconciliation of queries with no hit (FR-NOHIT)

- **FR-NOHIT-01** — Every query present in `headers` (FR-IN-07) that does not appear among the
  BLAST results must produce a `BlastQueryResult` with `results = None`.
- **FR-NOHIT-02** — Queries with `results = None` resolve to `NoConsensusFound`.

### 3.3 Bit-score selection (FR-SCORE)

- **FR-SCORE-01** — For each query, group hits by `bit_score` (integer) and sort the
  groups in **descending** order (best first). See `find_single_query_consensus`.
- **FR-SCORE-02** — Only the **highest bit-score** group is actually evaluated: each
  branch (`len == 0`, `len == 1`, `len > 1`) returns from inside the loop. Lower bit-score
  groups are discarded. *(See evaluation CONCERN-04 in the design.)*
- **FR-SCORE-03** — Empty group → `NoConsensusFound`.

### 3.4 Single-hit path (FR-SINGLE)

- **FR-SINGLE-01** — When the highest bit-score group has exactly 1 hit, parse the
  literal taxonomy (`d__2;p__...;s__...`) into a `Vec<TaxonomyBean>` ordered from the most
  general to the most specific rank.
- **FR-SINGLE-02** — Build `InterpolatedIdentity` for the taxon and the lineage ranks.
- **FR-SINGLE-03** — Truncate the lineage to the deepest rank justified by the
  observed `perc_identity` via `get_adjusted_taxonomy_by_identity`.
- **FR-SINGLE-04** — Mark the result with `single_match = true` and produce a single
  `ConsensusBean`.

### 3.5 Multi-taxa / LCA path (FR-MULTI)

- **FR-MULTI-01** — When the highest bit-score group has >1 hit, sort the records by
  `(lineage length, perc_identity, align_length, subject_accession)`.
- **FR-MULTI-02** — Select the **reference lineage** according to the strategy:
  - `Cautious` → **shortest** lineage (first sorted record).
  - `Relaxed` → **longest** lineage (last sorted record).
- **FR-MULTI-03** — Walk the reference lineage from the most general to the most
  specific rank. At each rank, collect the set of `rank+identifier` present among the hits
  at the same level.
- **FR-MULTI-04** — While all hits agree at a rank (set of size 1),
  descend one level, recording that rank as the current consensus.
- **FR-MULTI-05** — At the first rank where the hits **diverge** (set > 1), step back
  to the last agreeing rank (`index - 1`) — the **LCA** — and stop descending (`break`).
- **FR-MULTI-06** — In both cases, delegate the assembly of the final taxon to
  `build_blast_consensus_identity`, aggregating the `consensus_beans` of that level.

### 3.6 Rank adjustment by identity (FR-IDENT)

- **FR-IDENT-01** — Each taxon has a **backbone** of cutoffs per default Linnaean rank:
  - Fungi / Eukaryotes: Species 97, Genus 95, Family 90, Order 85, Class 80, Phylum 75, Domain 60.
  - Bacteria: Species 99, Genus 97, Family 92, Order 85, Class 80, Phylum 75, Domain 60.
  - Custom: user-provided values (`CustomTaxon`).
- **FR-IDENT-02** — Non-Linnaean ranks (`Other`, e.g. strain/serovar) receive a cutoff by
  **linear interpolation** between the cutoffs of the default ranks that bound the window where
  the rank occurs (`interpolate_identities`).
- **FR-IDENT-03** — `get_rank_adjusted_by_identity(identity)` returns the deepest rank
  whose cutoff is satisfied by the observed identity (`max_allowed_rank`).
- **FR-IDENT-04** — `get_adjusted_taxonomy_by_identity(identity, lineage)` keeps only the
  ranks whose cutoff `<= identity`, truncating the lineage.
- **FR-IDENT-05** — `mutated = true` when identity truncation (FR-IDENT-04) **demoted the
  rank**, i.e. the rank the consensus reached on the reference lineage (before truncation) is
  deeper than the final `reached_rank`. A stop caused by hit divergence (LCA, FR-MULTI-05) does
  not set `mutated`. Applies to both the single and the multi path. *(Reworded 2026-11-01, OP-01:
  the previous wording "`reached_rank != max_allowed_rank`" contradicted AC-03 once FR-IDENT-03
  is honored, since after truncation both ranks coincide.)*

### 3.7 Consensus bean aggregation (FR-BEAN)

- **FR-BEAN-01** — `ConsensusBean::fold_consensus_list` groups beans by `rank__identifier`,
  summing `occurrences` and deduplicating `accessions`.
- **FR-BEAN-02** — Beans are sorted by `occurrences` (desc) and, on ties, by
  `identifier`.

## 4. Non-functional requirements

- **NFR-01 (Parallelism)** — Per-query classification runs in parallel via `rayon`
  (`into_par_iter`) in `build_consensus_identities`.
- **NFR-02 (Architecture)** — All consensus logic lives in `core/use_cases` (domain
  layer), with no external I/O effects beyond the adapters; it follows the hexagonal architecture
  described in `CLAUDE.md`.
- **NFR-03 (Data contract)** — The serialized DTOs (`#[serde(rename_all = "camelCase")]`)
  must remain mirrored in `blutils-ui/src/types/BlutilsResult.ts`.
- **NFR-04 (Determinism)** — Sorts use explicit tie-breakers (identifier, accession)
  for stable results despite parallelism.

## 5. Acceptance criteria (derived from current behavior)

- **AC-01** — Query with no BLAST hits but present in the headers → `NoConsensusFound`.
- **AC-02** — Query with a single best hit at 100% identity → full lineage,
  `single_match = true`, `reached_rank = Species` (if the taxon defines Species).
- **AC-03** — Query with a single best hit at an identity below the species cutoff →
  lineage truncated at the justified rank, `mutated = true`.
- **AC-04** — Query with multiple hits of the same bit-score that diverge at the genus rank →
  consensus returns at the family rank (LCA), with `consensus_beans` listing the divergent
  genera.
- **AC-05** — `Cautious` vs `Relaxed` strategy produces different reference lineages
  when the hits have different taxonomic depths.

## 6. Traceability (requirement → code)

| Requirement | File |
|-----------|---------|
| FR-IN, FR-JOIN, FR-NOHIT | `build_consensus_identities/mod.rs` |
| FR-SCORE, FR-SINGLE | `build_consensus_identities/find_single_query_consensus.rs` |
| FR-MULTI | `build_consensus_identities/find_multi_taxa_consensus.rs` |
| FR-IDENT | `domain/dtos/linnaean_ranks.rs` (`InterpolatedIdentity`), `domain/dtos/taxon.rs` |
| FR-BEAN | `domain/dtos/consensus_result.rs` (`ConsensusBean`) |
| FR-OUT | `domain/dtos/consensus_result.rs`, `domain/dtos/taxonomy_bean.rs`, `use_cases/write_blutils_output.rs` |
| Orchestration | `run_blast_and_build_consensus/mod.rs` |
