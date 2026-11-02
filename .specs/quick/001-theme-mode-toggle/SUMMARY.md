# 001 — Summary

Status: **Done** (2026-11-02). This work is uncommitted, in the `modules/blutils-ui` working tree
on the `feat/virtual-scroll-group-pager` branch, alongside `007-ui-virtual-scroll-group-pager`.

## Changes

- `src/components/ThemeModeToggle.tsx` (new). A three-state button built on flowbite-react's
  `useThemeMode`:
  - It cycles `auto` → `light` → `dark` → `auto`, with desktop, sun and moon icons.
  - Its `aria-label`/`title` read "Theme: <current> (switch to <next>)".
  - While in `auto`, it listens to `prefers-color-scheme` changes and re-applies `auto`.
    flowbite-react resolves `auto` only when the mode is set, so without this the page would
    not follow the OS.
- `src/components/Shell.tsx`: `DarkThemeToggle` is replaced by `ThemeModeToggle`. The old toggle
  only flipped between light and dark, so after the first click it never returned to the
  system theme.
- `src/components/ThemeModeToggle.test.tsx` (new, 5 tests):
  - the default is system;
  - the cycle order, with persistence and the `<html>.dark` class;
  - a persisted choice is restored;
  - OS changes are followed in system mode;
  - OS changes are ignored in explicit modes, and the listener is removed.
- The persistence key is unchanged (`flowbite-theme-mode`). The pre-paint script in
  `public/index.html` already handles `auto`, a missing value, `light` and `dark`.

## Verification

- `tsc --noEmit` is clean. `CI=true yarn test --watchAll=false` passes 26 suites and 322 tests.
  `CI=true yarn build` compiles.
- Mutation check: removing the `matchMedia` listener makes "follows OS theme changes while in
  system mode" fail.
- Headless Chromium, dev server:
  - with the OS set to dark, the page starts in System (dark);
  - when the OS switches to light, the page follows live;
  - clicking goes to Light, then Dark, then System, with localStorage and the background color
    changing at each step;
  - a reload keeps System.
