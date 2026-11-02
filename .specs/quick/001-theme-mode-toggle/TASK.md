# 001 — Three-state theme toggle (system / light / dark)

Scope: quick (`modules/blutils-ui`, about 3 files).

## Request

> revise o thema para que ele use o thema do sistema por padrão entretanto se o usuário ir
> clicando ele pode selecionar entre dark, light ou tema do sistema

## What

- **Default.** The app follows the OS theme (`prefers-color-scheme`) and tracks it live while in
  system mode.
- **Cycle.** The navbar button cycles **System → Light → Dark → System**. The icon shows the
  current mode (desktop / sun / moon). The `aria-label`/`title` name the current mode and the
  next one.
- **Persistence.** The choice is persisted under the existing `flowbite-theme-mode` key
  (`auto` | `light` | `dark`). `public/index.html` already applies this key before first paint.

## Where

- `src/components/ThemeModeToggle.tsx` (new). It uses flowbite-react's `useThemeMode`.
- `src/components/Shell.tsx` replaces `DarkThemeToggle` with the new component.
- `src/components/ThemeModeToggle.test.tsx` (new).

## Done when

- The tests cover: the default is system; the cycle order; persistence; the `dark` class on
  `<html>`; live OS changes in system mode; OS changes ignored in explicit modes.
- `tsc` is clean, the full test suite is green, and the build succeeds.
