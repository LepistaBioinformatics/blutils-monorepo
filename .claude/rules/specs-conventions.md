---
paths:
  - ".specs/**"
---

# Spec conventions

## Folder numbering

Every folder that holds a definition under `.specs/` MUST be prefixed with a
zero-padded, three-digit sequence number followed by a kebab-case slug:

```
.specs/features/001-consensus-classification/
.specs/features/002-tests-blutils/
.specs/quick/001-fix-null-beans/
```

### Scope

Applies to every definition container created under `.specs/`, including but not
limited to `.specs/features/` and `.specs/quick/`, and to any new sibling folder
that groups per-item definitions (one folder per feature, task, ADR, etc.).

It does NOT apply to fixed structural folders/files: `.specs/project/`,
`.specs/codebase/`, and the files inside a definition folder (`spec.md`,
`design.md`, `tasks.md`, `context.md`, `TASK.md`, `SUMMARY.md`).

### How to pick the number

1. List the parent directory (e.g. `ls .specs/features/`).
2. Take the highest existing prefix and add 1. Start at `001` in an empty parent.
3. The sequence is **per parent directory** (`features/` and `quick/` count independently).
4. Never reuse, fill gaps in, or renumber existing folders — numbers are stable IDs
   referenced from other documents and commits.

## Files inside a feature folder

Every `.specs/features/NNN-slug/` folder MUST contain these three files, regardless of
scope size. Do not inline design or tasks into `spec.md`; keep each concern in its file.

| File | Required | Content |
|---|---|---|
| `spec.md` | yes | WHAT: problem, goals, out of scope, requirements with IDs, acceptance criteria, success criteria |
| `design.md` | yes | HOW: architecture, components, data, alternatives, decisions taken while designing, implementation notes |
| `tasks.md` | yes | Atomic tasks with dependencies, status, gates; follow-ups and open points |
| `context.md` | optional | User decisions on gray areas (decision IDs), only when they exist |

- For a small feature, `design.md`/`tasks.md` may be short, but they must exist.
- For documentation-only work with nothing to execute (e.g. an as-built description),
  `tasks.md` states that there are no implementation tasks and links to the
  features that carry the follow-up work.
- `.specs/quick/NNN-slug/` folders keep the quick-mode pair `TASK.md` + `SUMMARY.md`.

## References

- Refer to definitions by their full folder name (`002-tests-blutils`), never the
  bare slug.
- When renaming is unavoidable, update every reference in `.specs/`, `.claude/`
  and `CLAUDE.md` in the same change, and use `git mv` for tracked folders.
