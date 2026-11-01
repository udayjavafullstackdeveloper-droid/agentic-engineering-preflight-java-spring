# Repository Instructions

## Purpose and architecture

This repository is an environment preflight, not a feature application. It warms and verifies the same Java 21/Spring Boot/Postgres and React/TypeScript/Vite stack used by the assessment.

- `compose.yaml` is the integration entrypoint. It starts Postgres, runs the API from the bind-mounted `backend/` directory with Maven, and runs the frontend from the bind-mounted `frontend/` directory with Bun. Named volumes preserve database, Maven, and frontend dependency caches between runs.
- The Spring Boot API is in `backend/`. Flyway applies `src/main/resources/db/migration/` migrations; Hibernate is deliberately configured with `ddl-auto: validate`, so entity changes must remain consistent with migrations. `GET /api/v1/utils/health-check/` only verifies that the API responds, while `GET /api/v1/utils/db-check/` calls `PingRepository.count()` to verify the migrated Postgres schema and database connection.
- The Vite app in `frontend/` renders the preflight result. It calls the database check through the relative `/api/...` URL; Vite proxies that prefix to `VITE_API_PROXY_TARGET` or, in Compose, `http://api:8080`. Keep the Docker-oriented host allowlist and polling watcher configuration in `vite.config.ts`.
- `scripts/` contains stdlib-only Python extractors for Claude, Codex, and Copilot session logs. The harness-specific extractors share raw-envelope construction in `session_log_envelope.py` and local skill/command metadata parsing in `skill_metadata.py`.

## Commands

Run these from the repository root unless noted otherwise.

| Purpose | Command |
|---|---|
| Start the complete preflight stack | `docker compose up` |
| Stop the stack while preserving caches | `docker compose down` |
| Compile/package the backend | `cd backend; .\mvnw.cmd package` |
| Run backend tests | `cd backend; .\mvnw.cmd test` |
| Format-check backend Java | `cd backend; .\mvnw.cmd spotless:check` |
| Apply backend Java formatting | `cd backend; .\mvnw.cmd spotless:apply` |
| Build/type-check the frontend | `cd frontend; bun run build` |
| Run the frontend dev server | `cd frontend; bun run dev` |
| Run one extractor test file | `python scripts\test_extract_copilot_session_log.py` |
| Run one extractor test function | `python -c "import sys; sys.path.insert(0, 'scripts'); import test_extract_copilot_session_log as t; t.test_copilot_noise_cleaning()"` |

The Python checks are direct assertion scripts rather than a pytest suite. Run every `scripts\test_*.py` file when changing shared extractor behavior. Backend and frontend test source directories are not currently present; use the commands above for their available validation.

For Linux/macOS, replace `.\mvnw.cmd` with `./mvnw` and use `/` path separators in Python commands.

## Repository-specific conventions

- Preserve trailing slashes on the two utility endpoint mappings and frontend requests: Compose health checks and the browser probe use those exact paths.
- Schema evolution belongs in a new ordered Flyway migration; do not rely on Hibernate to create or alter tables.
- This project includes Spring Security only to warm assessment dependencies. `SecurityConfig` intentionally permits every request and disables CSRF for the unauthenticated preflight; do not treat it as an authentication pattern.
- Text files must retain LF endings so bind-mounted scripts run in Linux containers. `.gitattributes` enforces this, except Windows-executed `.cmd`, `.bat`, and `.ps1` files, which use CRLF.
- Session-log extraction is manual-only. Run the corresponding `.agents/skills/extract-*-session-logs` skill or `.claude` command only when explicitly requested. The Docker extraction workflow mounts only the applicable session-storage directory read-only; preserve that narrow mount scope. Generated session logs are intentionally ignored by Git.
