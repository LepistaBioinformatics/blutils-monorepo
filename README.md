# Blutils Monorepo

Aggregator repository for the **Blutils** project. It bundles the two Blutils
codebases as git submodules under [`modules/`](./modules) so they can be cloned,
versioned and worked on together, while each remains an independent, separately
released project.

| Module | Path | Stack | Role | Upstream |
| --- | --- | --- | --- | --- |
| **blutils** | [`modules/blutils`](./modules/blutils) | Rust (Cargo workspace) | The engine — `blu` CLI that runs parallel BLAST and builds consensus taxonomies | [LepistaBioinformatics/blutils](https://github.com/LepistaBioinformatics/blutils) |
| **blutils-ui** | [`modules/blutils-ui`](./modules/blutils-ui) | React + TypeScript + Tailwind | The viewer — a static web app that loads and explores Blutils result files | [LepistaBioinformatics/blutils-ui](https://github.com/LepistaBioinformatics/blutils-ui) |

## How the two modules relate

The modules are **complementary but decoupled**. They are not linked at build or
runtime, do not import one another, and share no dependencies. The relationship
is a one-way **producer → consumer** data flow:

```
            ┌──────────────────────────┐        Blutils output           ┌──────────────────────────┐
  BLAST  →  │  blutils (Rust CLI: blu) │  ──── (JSON / YAML, camelCase) ─→ │   blutils-ui (web app)   │
  + NCBI    │  run blast → consensus   │        result file               │  load → group → explore  │
  data      └──────────────────────────┘                                  └──────────────────────────┘
```

- **`blutils`** wraps NCBI BLAST+, parallelizes it, and condenses multi-hit
  results into consensus taxonomic identities. It writes a structured result
  file (`BlutilsOutput`).
- **`blutils-ui`** is a client-side-only viewer for that file. There is no
  backend: a user opens a Blutils output and the UI renders, groups and
  paginates the taxonomic results.

### The shared contract

The only coupling between the modules is the **serialization schema** of the
result file. The Rust `BlutilsOutput`
(`modules/blutils/core/src/domain/dtos/blutils_output.rs`, serialized with
`#[serde(rename_all = "camelCase")]`) is mirrored field-for-field by the
TypeScript `BlutilsResult` interface
(`modules/blutils-ui/src/types/BlutilsResult.ts`).

Changing a serialized field on one side **requires** the matching change on the
other — there is no compiler to catch a drift across the boundary.

## Getting started

```bash
# Clone the monorepo together with both submodules
git clone --recurse-submodules <this-repo-url>

# Or, if already cloned, fetch the submodule contents
git submodule update --init --recursive
```

Each module builds and runs independently with its own toolchain:

```bash
# Engine (requires NCBI BLAST+ installed on the host)
cd modules/blutils      && cargo build --release && cargo run -p blutils-cli -- --help

# Viewer
cd modules/blutils-ui   && yarn install && yarn start
```

See each module's own `README.md` for full usage. The Blutils book (user guide)
lives at
[blutils/docs/book](https://github.com/LepistaBioinformatics/blutils/blob/main/docs/book/README.md).

## Working with the submodules

The submodules are pinned to specific commits; the monorepo only tracks those
pointers. To advance a module:

```bash
git submodule update --remote modules/blutils-ui   # move it to its upstream tip
git add modules/blutils-ui && git commit            # record the new pinned commit
```

Code changes inside a submodule are committed and pushed in that module's own
repository first; only then commit the updated pointer here.
