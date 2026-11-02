# Testing Infrastructure

> State as of **2026-11-01**: **neither module has functional tests.** This document
> describes the available infrastructure and the target infrastructure defined by the features
> `002-tests-blutils`, `003-tests-blutils-ui` and `004-tests-data-contract` (`.specs/features/`).

## Test Frameworks

| Module | Unit/Integration | E2E / process | Coverage |
|---|---|---|---|
| `modules/blutils` | `cargo test` (libtest), Rust 2021 | `assert_cmd` + `predicates` over the `blu` binary (to be added) | none (optional: `cargo-llvm-cov`) |
| `modules/blutils-ui` | Jest 27.5 + jsdom 16.7 (via `react-scripts` 5.0.1 / `craco test`), Testing Library React 13.4 (`renderHook` included), user-event 13.5 (synchronous API, no `setup()`), jest-dom 5.17 | none (out of scope) | `jest --coverage` built into CRA, no threshold |

**Rust dev-dependencies:** none today in any `Cargo.toml`. Target: `tempfile` (core, cli),
`assert_cmd` + `predicates` (cli), `serial_test` (cli). `serde_json`, `serde_yaml_ng` and `uuid` are already
regular deps of core and serve the tests.

## Test Organization

### blutils (Rust)

| Where | What for | Reason |
|---|---|---|
| `#[cfg(test)] mod tests` in the file itself | `pub(crate)`/`pub(super)`/private items (almost all of the consensus core, `InterpolatedIdentity`, cutoffs, `parse_taxonomy`, `.dmp` loaders) | visibility: external `tests/` cannot reach them |
| `core/tests/*.rs` | re-exported `pub` use cases: `build_consensus_identities`, `write_blutils_output`, `run_blast_and_build_consensus` (with `ExecuteBlastn` mock), `parse_consensus_as_tabular`, serde contract | black box over the crate's public API |
| `core/tests/fixtures/` | synthetic 13-column blast TSV + `TaxonomiesMap` JSON, minimal `.dmp` | `test/mock/` has no fixtures usable by the consensus |
| `ports/cli/src/main.rs` `mod tests` | clap parsing (`CliLauncher::<Opts>::try_parse_from`) | `Opts` is private and the crate has no lib target |
| `ports/cli/tests/cli.rs` | `assert_cmd` over `blu` | stdout, `process::exit`, panics → exit 101 |
| `ports/cli/tests/host_tools.rs` | fakes of `which`/`blastn`/`blastdbcmd` via the child process `PATH` (`assert_cmd .env`) | does not mutate the test environment |
| `#[ignore = "requires ncbi-blast+"]` | smoke with real BLAST+ (`test/mock/input/*`) | does not run in default CI |

**Naming:** `fn <unit>_<condition>_<result>()`; tests that pin a bug: `BUG-Rxx` prefix/comment.

### blutils-ui (TypeScript)

| Where | What for |
|---|---|
| `src/**/<File>.test.ts(x)` (co-located) | CRA default (`testMatch` already covers it) |
| `src/test-utils/` | `renderWithRouter`, controllable `IntersectionObserver` mock, builders for `Result`/`Taxon`/`ConsensusBean` |
| `src/__fixtures__/` | small hand-made fixture + copy of the contract golden |

## Testing Patterns

### Unit
- **Rust:** pure functions with `assert_eq!`; `HashMap`/`rayon` results **sorted before comparing**
  (non-deterministic order in `fold_consensus_list`, `build_consensus_identities`, `build_taxonomy_database`);
  `f64` compared with tolerance; input panics with `#[should_panic(expected = …)]` (CONCERN-01).
- **UI:** pure functions (`BeanSelectionRules`, `kebab-*`, `docs-config`) with `expect`; hooks with `renderHook`.

### Integration
- **Rust:** filesystem via `tempfile::tempdir()` (never fixed paths); `ExecuteBlastn` is the **only existing
  seam** for subprocesses — mock in `core/tests/common/mod.rs`.
- **UI:** components with Testing Library; pages inside `MemoryRouter`; `fetch` via
  `jest.spyOn(global, "fetch")`; flowbite modals are portaled → query with `screen`, not `container`.

### E2E
- **Rust:** `blu blastn build-consensus` does not call `check_host_requirements` → real E2E without BLAST+.
- **UI:** no browser E2E (out of scope, see spec `003-tests-blutils-ui`).

## Test Execution

| Module | Command |
|---|---|
| blutils | `cargo test` / `cargo test -p blutils-core` / `cargo test <name>` / `cargo test -- --ignored` (requires BLAST+) |
| blutils-ui | `CI=true yarn test --watchAll=false` / `yarn test -- --watchAll=false --coverage` |

**Configuration required in the UI (currently missing — the current `App.test.tsx` test fails):**
- `"jest": { "moduleNameMapper": { "^@/(.*)$": "<rootDir>/src/$1" } }` in `package.json`
  (the `@/` alias only exists in webpack via `craco.config.js`).
- `setupTests.ts`: polyfills for `IntersectionObserver`, `navigator.clipboard`, `Blob/File.prototype.text`, `window.scrollTo`.
- `react-markdown@10` and `remark-gfm@4` are ESM-only → `jest.mock` in the `pages/docs` tests.
- `--coverage` with the default istanbul provider fails (`minimatch is not a function`): `package.json` `resolutions` pins `minimatch@^9`, but `test-exclude@6` needs `minimatch@^3`. Use `--coverageProvider=v8` (or scope the resolution).
- `yarn test` fails in the local environment (yarn cache dir not writable); `npx craco test` is equivalent.
- CRA sets `resetMocks: true` → mock implementations must be declared in `beforeEach`/in the test.

## CI

| Module | Current situation | Target |
|---|---|---|
| blutils | `.github/workflows/rust.yml` runs `cargo test --verbose` **only on `release: published`** (`push`/`pull_request` triggers commented out) | test job on `push`/`pull_request` |
| blutils-ui | `.github/workflows/deploy.yml` only does `yarn build` + deploy; **does not run tests** | `CI=true yarn test --watchAll=false` before the build |
| monorepo | no CI | contract drift check (feature `004-tests-data-contract`) |

## Coverage Targets

**Current:** 0% in both modules. **Goal (Draft):** consensus core (`build_consensus_identities/`,
`linnaean_ranks.rs`, `taxon.rs`) ≥ 90% of lines; UI `src/types`, `src/functions` 100%.
**Enforcement:** none until the goals are confirmed.

## Test Coverage Matrix

| Module | Code Layer | Required Test Type | Location Pattern | Run Command |
|---|---|---|---|---|
| blutils | `core/src/domain/dtos` (pure) | unit | in-file `mod tests` | `cargo test -p blutils-core --lib` |
| blutils | `core/src/use_cases/build_consensus_identities` internals | unit | in-file `mod tests` | `cargo test -p blutils-core --lib` |
| blutils | public `core/src/use_cases` (fs, mock blast) | integration | `core/tests/*.rs` | `cargo test -p blutils-core --tests` |
| blutils | `core/src/use_cases/build_blutils_db_from_ncbi_files` | unit + integration (tempdir) | in-file `mod tests` | `cargo test -p blutils-core --lib` |
| blutils | use cases with subprocess and no seam (`check_host_requirements`, `blastdbcmd`) | e2e (fake PATH) | `ports/cli/tests/host_tools.rs` | `cargo test -p blutils-cli --test host_tools` |
| blutils | `adapters/proc` | e2e (fake PATH) + real `#[ignore]` | `adapters/proc/tests/` | `cargo test -p blutils-proc-adapter` |
| blutils | `ports/cli` parsing | unit | `ports/cli/src/main.rs` `mod tests` | `cargo test -p blutils-cli --bins` |
| blutils | `ports/cli` process | e2e | `ports/cli/tests/cli.rs` | `cargo test -p blutils-cli --tests` |
| ui | `src/types`, `src/functions`, `pages/docs/docs-config.ts` | unit | co-located `*.test.ts(x)` | `CI=true yarn test --watchAll=false` |
| ui | `src/hooks` | unit (`renderHook`) | co-located | same |
| ui | `src/components`, `Results/elements/*` | integration (RTL) | co-located | same |
| ui | `pages/**`, `App.tsx` | integration (MemoryRouter) | co-located | same |
| ui | `theme/`, `constants/`, `index.tsx`, `reportWebVitals.ts` | none | — | — |
| both | `BlutilsOutput` ⇄ `BlutilsResult` contract | contract | `core/tests/contract.rs` + `src/types/BlutilsResult.contract.test.ts` | see feature `004-tests-data-contract` |

## Parallelism Assessment

| Test Type | Parallel-Safe? | Isolation Model | Evidence |
|---|---|---|---|
| Rust unit | Yes | pure, no global state | — |
| Rust integration (fs) | Yes | one `tempdir()` per test | — |
| Rust with `Source::Stdin` / `"-"` | **No** | `STDIN_HAS_BEEN_USED` (`file_or_stdin.rs:17`) is a global static | dedicated test binary, a single test |
| Rust fake PATH in the child (`assert_cmd .env("PATH")`) | Yes | environment of the child process only | `ports/cli/tests/host_tools.rs` |
| Rust PATH of the process itself (`adapters/proc`) | **No** | `PATH` is process-global | `#[serial]` in `adapters/proc/tests/execute_blast.rs` |
| Rust clap parsing (`main.rs` `mod tests`) | Yes **if** each parse passes an explicit query | the positional's `"-"` default sets `STDIN_HAS_BEEN_USED` | design `002-tests-blutils` §3.17 |
| Rust `#[ignore]` BLAST+ | Yes | own tempdir | — |
| Jest (all) | Yes | Jest isolates per file; mocks reset (`resetMocks: true`) | `react-scripts/scripts/utils/createJestConfig.js` |

## Gate Check Commands

| Gate | Module | When | Command |
|---|---|---|---|
| Quick | blutils | unit-only tasks | `cargo test -p blutils-core --lib` |
| Quick | ui | UI tasks | `CI=true yarn test --watchAll=false` |
| Full | blutils | tasks with integration/e2e | `cargo test` |
| Full | ui | same | `CI=true yarn test --watchAll=false --coverage --coverageProvider=v8` |
| Build | blutils | end of phase | `cargo build --release && cargo clippy --all-targets && cargo test` — **note:** `cargo clippy` already fails on the baseline (3 `written amount is not handled` errors in `blutils-core` lib, pre-existing); until fixed, the gate is "no new clippy findings in test modules" |
| Build | ui | end of phase | `CI=true yarn test --watchAll=false && yarn build` |
