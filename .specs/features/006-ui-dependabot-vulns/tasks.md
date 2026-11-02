# 006 — Tasks

| ID | Task | Depends on | Status |
|---|---|---|---|
| T1 | Baseline: build on `main`, record CSS size | - | Done (120626 B) |
| T2 | Bump `flowbite-react` to ^0.7.8; verify tailwind glob + CSS size | T1 | Done (0.7.8; CSS 120359 B) |
| T3 | Bump `react-router-dom` (v7 >= 7.18.0, fallback 6.30.6) | - | Done (7.18.5) |
| T4 | Re-resolve transitive deps (lockfile entry removal; scoped resolutions) | T2, T3 | Done |
| T5 | Gates: `tsc --noEmit`, `yarn test --watchAll=false`, `yarn build`, `yarn start` boot | T4 | Done (286/286; build + dev server OK) |
| T6 | Lockfile-vs-alert-range check; record residuals | T5 | Done (72 of 85 cleared) |
| T7 | Commit, PR and merge (user-approved) | T6 | Done (PR #9, merge cb183b2) |

## Implementation notes

- **react-router v7.** Jest 27 does not resolve `exports` subpaths, so `react-router/dom` is
  mapped in `package.json` `jest.moduleNameMapper`. jsdom 16 lacks `TextEncoder`/`TextDecoder`,
  so these are polyfilled from `util` in `setupTests.ts`. By default, v7 wraps router state
  updates in `React.startTransition`. That made the URL lag behind the urgent state updates in
  `ResultLoader`/`Results`: `?p` was lost after an upload, and the file was fetched twice on Reset.
  Three tests caught this. `BrowserRouter` and the test `MemoryRouter` now pass
  `useTransitions={false}`, which restores the v6 synchronous behavior.
- **flowbite-react 0.7.8.** It ships source maps that reference an unpublished `src/`, which
  produced many "Failed to parse source map" warnings. These are filtered via
  `ignoreWarnings` in `craco.config.js`.
- **Transitive deps.** The vulnerable `yarn.lock` entries were removed and re-resolved. New
  resolutions: `qs` ^6.16.2 (express pins `~6.15.1`) and `postcss-selector-parser` ^6.1.5 (pinned
  at `6.0.10`). Both exist in only one major here.

## Residual alerts (13 of 85; dev server / build toolchain only)

These can only be fixed by a major that `react-scripts@5` cannot load, or have no upstream patch.
None of them ships in the static GitHub Pages bundle.

| Alerts | Package | Locked | Fix |
|---|---|---|---|
| #35 #36 #104 #122 #132 #133 | webpack-dev-server | 4.15.1 | 5.2.6 (major) |
| #184 | webpack-dev-middleware | 5.3.4 | 7.4.6 (major) |
| #137 #165 #166 | svgo (via @svgr) | 1.3.2 | 2.8.4 (major) |
| #210 | glob | 10.5.0 | 14.0.2 (major) |
| #211 | minimatch | 9.0.9 | 11.0.1 (major) |
| #197 | deepmerge | 4.3.1 | none |

## Follow-ups

- Migrate off CRA to remove the residual dev-toolchain alerts (VULN-04).
