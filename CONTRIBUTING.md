# Contributing

## Workflow

1. Update `main` with `git pull --ff-only`.
2. Create one `feat/*`, `fix/*`, `docs/*`, `test/*`, or `chore/*` branch.
3. Keep each commit focused and use Conventional Commit prefixes.
4. Run the checks listed in the current taskbook unit.
5. Open a pull request, review the complete diff, and wait for required checks.
6. Squash merge, update local `main`, and delete the merged branch.

## Contract first

HTTP changes begin in `docs/api/openapi.yaml`. Regenerate Go and TypeScript artifacts and commit contract, generated code, implementation, and tests in reviewable commits. Generated files are never edited manually.

## Database changes

Published migration files are immutable. Add a new up/down pair, verify up/down/up locally, and keep production migrations forward-only.

## Definition of done

A change is done only when behavior, failure paths, tests, documentation, security impact, deployment impact, and rollback are reviewed.
