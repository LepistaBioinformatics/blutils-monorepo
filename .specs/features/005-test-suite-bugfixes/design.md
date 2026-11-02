# Bug Fixes from the Test-Suite Mapping — Design

> Spec: `spec.md` · Decisions: `context.md` · Tasks: `tasks.md`

## Regression-test rule

For every fixed bug, in the same change:
1. Delete the characterization test that pinned the buggy behavior.
2. Enable the desired-behavior test (`#[ignore = "BUG-…"]` / `test.skip("BUG-…")`) — it
   becomes the regression test. Keep the `BUG-xx` reference in a comment.
3. If no desired test existed, write one that fails on the old code and passes on the fix.
4. Mark the registry row `fixed` with the regression test name.

Exception: dead-code removals (R20, R25, R32) cannot be detected by a failing test; the registry
says so explicitly.

## Approach by area

| Area | Approach |
|---|---|
| Rank adjustment (R01) | deepest rank whose cutoff ≤ identity; `None` below every cutoff |
| Mutation flag (R02, R21) | computed after identity truncation on both paths; `mutated` = rank demoted by identity (DF-05) |
| Interpolation (R05, R17) | window bounded by previous/next default rank; Kingdom as a default rank at 67.5 in built-in backbones |
| Multi-hit LCA (R06, R22) | evaluate every co-best hit that has the rank; always truncate at the LCA index |
| Bit score (R14) | keep `f64`; group by exact value |
| Panics (R03, R07, R08, R23, R24, R29, R33, R35, R19) | `NoConsensusFound` for classification dead ends, `Err` for I/O/input errors (DF-02) |
| Output formats (R09, R10, R27) | JSONL without `null` config line; tabular rows newline-terminated, 12 columns; keep result `runId` |
| Taxdump (R15, R16, R30, R31, R34) | row-wise null filtering; no leading `;`; `replace_rank` on the leaf; merged leaf uses target taxid; warn on unparsed values |
| YAML library (DF-04) | `serde_yaml` → `serde_yaml_ng` (same API) |
| Leading non-Linnaean ranks (R38, R39) | cutoff of the next default rank; truncation with `take_while` (contiguous prefix) |
| Thread ceiling (R40) | dedicated pool of `min(threads, chunks)`; `blastn -num_threads threads / concurrency`; `limit_threads` caps the global rayon and Polars pools at startup |
| Panics (R42, R43) | CLI commands return `anyhow::Result` (exit 1, `Error: …`); core returns `Err` with path/line/chunk context |
| Viewer (U01–U22) | nullable TS types mirroring Rust `Option`; shape/status validation in the loader; pagination clamping; hook cleanup |

## Breaking changes

- Classification results change (DF-01).
- `CustomTaxon::from_file` returns `Result<CustomTaxon, MappedErrors>` (R19).
- Tabular output: one row per line, 12-column no-taxon rows (R10). JSONL omits the config line
  when there is no config (R09).
- `--threads` caps the whole run instead of each `blastn` call (R40).
- CLI errors exit with code 1 and `Error: <message>` instead of a panic (code 101) (R42).
- Results below the species cutoff keep leading non-Linnaean ranks in `taxonomy` (R38).
