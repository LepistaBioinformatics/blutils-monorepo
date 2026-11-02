# 006 — Design

## Approach

The change touches dependencies only. No application code changes except the minimum needed for
API compatibility.

1. **flowbite-react 0.7.2 → ^0.7.8.** The 0.7.3+ releases removed the leaked docs dependencies.
   Stay on 0.7.x, because 0.8+ changes the theming and Tailwind integration. Check that the
   `tailwind.config.js` content glob (`node_modules/flowbite-react/lib/esm/**/*.js`) still exists.
   Compare the built CSS size against the baseline (`main.*.css`, 120626 bytes on `main`).
2. **react-router-dom.** Try v7 (>= 7.18.0). Usage is limited to BrowserRouter/MemoryRouter,
   Routes/Route, Link, Outlet, useLocation, useParams and useSearchParams. Known friction:
   - jsdom 16 lacks `TextEncoder` (polyfill in `setupTests.ts`).
   - TS 4.9 compatibility of the v7 typings.
   - v7 needs Node >= 20 (CI already uses `node-version: 20`).

   Fallback: 6.30.6, which leaves GHSA-wrjc-x8rr-h8h6 and GHSA-337j-9hxr-rhxg (SSR-only) as
   residual.
3. **Transitive deps.** Delete the vulnerable `yarn.lock` entries and run `yarn install`, so that
   the caret ranges re-resolve to the patched versions. Add or adjust `resolutions` only where a
   parent pins a range that excludes the patch. Use path-scoped resolutions (`**/parent/pkg`)
   where several majors coexist.
4. **Residuals (not forced).** Each of these needs a major that `react-scripts@5` cannot load, or
   has no upstream patch. They affect only the local dev server and the build toolchain, not the
   static GitHub Pages bundle:
   - `webpack-dev-server@4`
   - `webpack-dev-middleware@5`
   - `svgo@1`
   - `minimatch`/`glob` majors
   - `deepmerge`
   - `node-forge` GHSA-86w9

## Alternatives considered

- **Force `webpack-dev-server@5` via resolutions.** Rejected: `react-scripts@5` passes options
  that WDS 5 removed (`onBeforeSetupMiddleware`, `https`), so `yarn start` breaks. Adapting would
  mean a CRACO devServer shim, which is a CRA migration in disguise.
- **Migrate off CRA (Vite).** This is the only complete fix, but it is out of scope and deferred.
