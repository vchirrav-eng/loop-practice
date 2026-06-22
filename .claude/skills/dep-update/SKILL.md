---
name: dep-update
description: Update a single outdated dependency, run tests, and prepare a PR. Use for routine dependency bumps in this repo.
---

# Dependency Update Procedure

## Conventions
- Bump exactly ONE dependency per run (never batch).
- Prefer the lowest-risk update available: patch > minor > major.
- Never bump a package on the DO-NOT-AUTO-UPDATE list below.
- Branch naming: `deps/<package>-<new-version>`.

## Steps
1. Run the audit command to list outdated packages: `npm outdated --json`.
2. Pick ONE package not already in loop-state.md "Done" or "Skipped".
3. Update only that package and its lockfile.
4. Run the full test suite: `npm test`.
5. Run the linter: `npm run lint`.

## Do-not-auto-update
- react

## Definition of done
- The single bump is committed on its own branch AND `npm test` exits 0 AND `npm run lint` exits 0.