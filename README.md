# Personal Site

A content-rich personal website built as a modular monolith with a statically exported Next.js public site, a Go API, PostgreSQL, and an authenticated admin interface.

## Status

The repository is being built from zero by following `docs/taskbook/README.md`.

## Architecture

- `web/`: Next.js public site and admin UI
- `server/`: Go API and database migrations
- `docs/api/openapi.yaml`: source of truth for the HTTP contract
- `docs/`: decisions, taskbook, design, and operations
- `nginx/`: production edge configuration
- `scripts/`: repeatable development and operations commands

## Local prerequisites

- Node.js 24.14.0
- pnpm 11.19.0
- Go 1.25.3
- Docker Engine with Compose

Do not invent setup steps from this short README. Use the taskbook.
