# Bug Fixes from the Test-Suite Mapping — Specification

> Status: **Done 2026-11-01** · Design: `design.md` · Tasks: `tasks.md` · Decisions: `context.md`
> Source registries: `002-tests-blutils/design.md` §2 (BUG-R01..R43) and
> `003-tests-blutils-ui/design.md` §2 (BUG-U01..U22).
> Supersedes D-01 for the bugs listed here: the user authorized fixing them (2026-11-01).

## Problem Statement

The test suites built in `002-tests-blutils` and `003-tests-blutils-ui` pinned 59 defects
(37 in the engine, 22 in the viewer) with characterization tests and disabled desired-behavior
tests. Several make classification results diverge from the as-built spec
(`001-consensus-classification`), crash the run on valid-looking input, corrupt output formats, or
break the viewer on outputs the engine legitimately produces.

## Goals

- [x] Fix every registered bug the user approved, each with a regression test that fails on the old code.
- [x] Make classification behavior match the as-built spec (FR-IDENT-03, AC-03, AC-05).
- [x] Remove input-reachable panics from the engine.
- [x] Leave the suites green with no disabled bug tests.

## Out of Scope

| Item | Reason |
|---|---|
| U11 (YAML/JSONL in the viewer) | won't fix — viewer is JSON-only by design (DF-03) |
| Scientific review of the built-in cutoffs | OP-02 / CONCERN-02 |
| Regenerating the legacy zymo golden output | OP-03, requires BLAST+ |

## Requirements

- **BF-01** — Every bug in scope SHALL be fixed in production code with a regression test, following
  the regression-test rule in `design.md`.
- **BF-02** — Algorithm fixes SHALL follow the as-built spec (DF-01).
- **BF-03** — Inputs that previously panicked SHALL yield `Err` or `NoConsensusFound` (DF-02).
- **BF-04** — Each fixed registry row SHALL be marked `fixed` with its regression test name.
- **BF-05** — Breaking changes SHALL be flagged in commit messages (`!` + `BREAKING CHANGE`).

## Success Criteria

- [x] `cargo test --workspace` green with zero `#[ignore = "BUG-…"]` left.
- [x] `CI=true npx craco test --watchAll=false` green with zero `test.skip("BUG-…")` left.
- [x] `cargo clippy --all-targets` passes without `-A clippy::unused_io_amount` (R26).
- [x] As-built spec `001-consensus-classification` and STATE discrepancies marked resolved.
