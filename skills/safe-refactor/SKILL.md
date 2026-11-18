---
name: safe-refactor
description: Restructure code without changing behaviour, using worktrees, a green test baseline, and the knowledge graph to find every caller.
---

# Safe refactor

Restructure code without changing behaviour, using worktrees and the knowledge
graph to stay safe.

## Steps

- Map first (see the *Map this codebase* skill) so you know every caller of the
  code you're about to move. Use the knowledge graph to find references — don't
  rely on grep alone.
- Open an isolated worktree: `baton new "refactor: <area>"`. Never refactor on
  a branch another agent is using.
- Establish a green baseline: run the build + tests **before** touching
  anything. If they aren't green, stop — fix or report that first.
- Make the change in small, behaviour-preserving steps. Re-run tests after each
  step. Check edit signals before touching shared files.
- Keep the public API identical unless the task says otherwise. If you must
  change a signature, update every caller the graph found.
- Record any non-obvious decision with `baton memory add`, then `baton pass`
  or `baton merge` the worktree once tests pass.
