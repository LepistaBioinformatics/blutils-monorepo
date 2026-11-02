# English-only artifacts

Everything written into the repository or its history MUST be in **English**:

- Code comments and docstrings.
- Commit messages (subject and body), branch names, PR titles/descriptions,
  issue and review comments posted via `gh`.
- Markdown and documentation files (`.specs/**`, `README.md`, `CHANGELOG.md`,
  `docs/**`, `.claude/**`).
- Identifiers, log/error messages, test names, and fixture descriptions.
- Configuration comments (`Cargo.toml`, `package.json`, workflows, YAML).

This applies to both submodules (`modules/blutils`, `modules/blutils-ui`) and to
the monorepo itself.

## What is not covered

- Conversational replies to the user may follow the user's language.
- Quoting existing non-English text verbatim (e.g. a user message or an external
  source) is allowed, but the surrounding prose stays in English.

## Existing non-English content

When editing a file that already contains non-English text, write new content in
English. Translate the existing text only when explicitly asked, so diffs stay
focused.
