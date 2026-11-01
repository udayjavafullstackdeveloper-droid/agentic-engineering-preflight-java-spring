# Developer README

## What this repository does

This is a preflight project for the assessment environment. It verifies a Java 21/Spring Boot API, PostgreSQL database, and React/TypeScript/Vite frontend can build and communicate. It is intentionally minimal; its main value is warming Docker, Maven, and Bun dependency caches before the assessment.

## Prerequisites

- Docker Desktop running
- Git

Local Java, Maven, Bun, and Python are optional because Docker Compose supplies the runtime tools.

## Start the stack

From the repository root:

```powershell
docker compose up
```

On the first run, Docker downloads images, Maven downloads backend dependencies, and Bun installs frontend packages. Leave the stack running until all services become healthy, then verify:

| Service | URL | Expected result |
|---|---|---|
| Frontend | `http://localhost:5173` | **Stack is up** and `database: ok` |
| API health | `http://localhost:8080/api/v1/utils/health-check/` | `true` |
| API schema/database check | `http://localhost:8080/api/v1/utils/db-check/` | `{"database":"ok"}` |
| OpenAPI document | `http://localhost:8080/v3/api-docs` | API specification JSON |

Use `docker compose down` to stop the stack while retaining caches and database data. Set `FRONTEND_PORT`, `API_PORT`, or `DB_PORT` before `docker compose up` if ports 5173, 8080, or 5432 are already occupied.

## Project layout

- `backend/` — Spring Boot application. Flyway migrations in `src/main/resources/db/migration/` own the database schema; JPA validates mappings against that schema at startup.
- `frontend/` — React application served by Vite. Browser requests under `/api` are proxied to the API service.
- `scripts/` — stdlib-only Python utilities for exporting Claude, Codex, and Copilot session logs.
- `compose.yaml` — starts PostgreSQL, the API, frontend, and builds the extractor image.

## Local validation commands

These are optional when using Compose, but useful when the relevant local tooling is installed:

```powershell
# Backend
cd backend
.\mvnw.cmd test
.\mvnw.cmd spotless:check

# Frontend
cd ..\frontend
bun run build

# One Python extractor check
cd ..
python scripts\test_extract_copilot_session_log.py
```

The Python checks are standalone assertion scripts. On Linux or macOS, use `./mvnw` instead of `.\mvnw.cmd`.

## Important implementation details

- Keep the trailing slash on `/api/v1/utils/health-check/` and `/api/v1/utils/db-check/`: Compose health checks and the frontend use those exact paths.
- Add database changes as new Flyway migrations; Hibernate is configured to validate the schema, not generate it.
- The API intentionally permits all requests and disables CSRF because it is a connectivity preflight, not an authentication implementation.
- Keep text files LF-terminated for Linux containers. `.gitattributes` enforces this, with Windows `.cmd`, `.bat`, and `.ps1` files as CRLF exceptions.

## Exporting session logs

Only run session-log exports when explicitly requested. Use the corresponding helper:

- Claude: `.claude/commands/extract-claude-session-logs.md`
- Codex: `.agents/skills/extract-codex-session-logs/`
- Copilot: `.agents/skills/extract-copilot-session-logs/`

Each export creates a readable Markdown log and a raw JSON envelope. Both generated files are ignored by Git.
