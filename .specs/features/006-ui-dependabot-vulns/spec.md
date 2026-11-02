# 006 — Fix blutils-ui Dependabot vulnerabilities

Scope: **Medium** (dependency-only change in `modules/blutils-ui`).

## Context

On 2026-11-02, 80 Dependabot alerts were open on LepistaBioinformatics/blutils-ui. All of them are
in `yarn.lock`. They have three sources:

1. **`flowbite-react@0.7.2`**: this release accidentally shipped its docs-site dependencies
   (`contentlayer`, `next-contentlayer`, `sharp`, `markdown-toc`/`gray-matter`). That pulls in
   `sharp`, `toml`, `zod`, `@grpc/grpc-js`, `@opentelemetry/*`, `protobufjs`, and part of
   `js-yaml`/`glob`.
2. **`react-router-dom@6.30.4`**: a direct dependency.
3. **`react-scripts@5` dev/build toolchain**: `webpack-dev-server@4`, `node-forge`, `svgo@1`,
   `deepmerge`, `shell-quote`, `proxy-addr`, `websocket-driver`, `qs`, and others.

## Requirements

- **VULN-01**: Upgrade `flowbite-react` within 0.7.x (to ^0.7.8) so the docs-site dependency tree
  is dropped. The Tailwind content glob must still match and the built CSS must not lose flowbite
  classes.
- **VULN-02**: Upgrade `react-router-dom`/`react-router` to a patched version: 7.x (>= 7.18.0) if
  the migration is clean, otherwise 6.30.6. Record any residual alert.
- **VULN-03**: Re-resolve every transitive package that has a patch within its current major.
  Prefer deleting lockfile entries and reinstalling. Use `resolutions` only where a parent pins an
  exact version, and never use a bare-name resolution across coexisting majors.
- **VULN-04**: Do not force major bumps that are incompatible with `react-scripts@5`
  (`webpack-dev-server@5`, `webpack-dev-middleware@7`, `svgo@2` under `@svgr`, `minimatch@11`,
  `glob@14`). Document them as residual, dev/build-only risk. Also document the alerts that have
  no upstream patch (`deepmerge`, `node-forge` GHSA-86w9-cpqp-85rv).
- **VULN-05**: Verification gates:
  - `tsc --noEmit` reports no errors.
  - `yarn test --watchAll=false` passes.
  - `yarn build` succeeds, and the built CSS size is comparable to the baseline.
  - `yarn start` boots and compiles.
  - A lockfile-vs-alert-range check lists exactly the remaining open alerts.

## Out of scope

- Migrating off CRA (`react-scripts`), e.g. to Vite. This is the only full fix for the VULN-04
  residuals and is deferred.
- Dismissing alerts on GitHub, pushing, or opening a PR. The user reviews locally.
