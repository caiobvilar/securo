# Securo — AGENTS.md

Securo is a self-hosted personal finance manager: FastAPI backend (`backend/`), React 19 + Vite frontend (`frontend/`), PostgreSQL, Redis/Celery queue, Docker Compose. Optional AI Agents feature (LLM chat + RAG over your data, MCP tool server) lives in `backend/app/agents/` and `backend/mcp_server/`. Monorepo root managed with mise; web UI runs on port 3000.

## Dev environment

- Backend Python: `backend/.python-version` is 3.14 (single source of truth — CI and Dockerfile read it). Deps resolved by uv, locked in `backend/uv.lock`.
- Frontend: Node 22, npm. `frontend/package-lock.json` committed.
- mise tasks: root `mise.toml` declares config roots; `mise backend:install|test|lint` and `mise frontend:install|lint|build` wrap the commands below.
- Env reference is `.env.example`; `.env` and `secrets/` are gitignored (private keys for bank sync live in `secrets/`).

## Build & test

Backend (run from `backend/`):

```
uv sync --all-extras            # dev install (pytest, ruff, ty, aiosqlite)
ruff check .                    # lint
ty check .                      # type check
pytest                          # tests (README default)
pytest -n auto --dist loadfile --cov=app --cov-report=xml --cov-report=term-missing --cov-fail-under=60   # CI command
uv lock --check                 # CI: fails if pyproject.toml drifted from uv.lock
./scripts/lock.sh               # regenerate uv.lock after pyproject dep changes (pins uv==0.12.1)
prek run --all-files            # pre-commit hooks (ruff + ty), optional
```

Frontend (run from `frontend/`):

```
npm ci                          # CI install; npm install for local
npm run lint                    # eslint .
npm run build                   # tsc -b && vite build  (type check + build)
npm test                        # vitest run
npm run dev                     # vite dev server (proxies /api to backend)
```

Docker: `docker compose up --build` (deps: db, redis, backend, frontend, worker, beat). Agents feature is opt-in: `AGENTS_ENABLED=true COMPOSE_PROFILES=agents docker compose up -d` starts the `mcp-server` container. Bare-metal: `uvicorn mcp_server.main:app --host 127.0.0.1 --port 8765` in the same venv.

## Conventions

- Backend layout: `app/api/` (thin FastAPI routers) → `app/services/*_service.py` (business logic) → `app/models/` (SQLAlchemy) → `app/schemas/` (Pydantic). `app/tasks/` = Celery workers, `app/core/` = infra, `app/providers/` = bank-sync providers. Agents feature mirrors this split under `app/agents/`.
- Alembic migrations: `backend/alembic/versions/`, named `0XX_theme.py`, ≥70 committed. Hand-edit only via generated revisions.
- Money is `Decimal` everywhere in services; never float.
- Backend tests: pytest-asyncio `asyncio_mode = "auto"`, session-scoped loop; shared fixtures in `tests/conftest.py` (`client`, `session`, `test_user`, `clean_db`). SQLAlchemy async + aiosqlite.
- Frontend tests are unit tests in `src/lib/*.test.ts`, node environment (no jsdom, no component/DOM tests).
- i18n: every user-facing string goes through `src/locales/*.json` (en.json is the key source); `src/locales/i18n.test.ts` enforces key parity across locales.
- Commits: conventional, scoped — `feat(frontend):`, `fix:`, `feat:` (e.g. `feat(frontend): improve account detail layouts (#579)`).
- Bank sync providers auto-register when their creds are present in `.env`.

## Pitfalls

- **Lockfile is law.** CI runs `uv lock --check` and fails on drift; the Docker image installs from a `--frozen` export. Any `pyproject.toml` dep change must be followed by `./scripts/lock.sh` + commit of `uv.lock`.
- **Dep pins are deliberate:** `ruff>=0.1.0,<0.16` (0.16 reformats historical alembic import blocks) and `ty==0.0.66` (pre-1.0). Bump together with the required reformat/annotations, or CI turns red.
- **Backend tests run on SQLite, not Postgres.** `conftest.py` shims pgvector's Vector type to JSON; code relying on PG-only behavior (real vector similarity search, PG full-text) is not exercised by CI — those paths must be checked against a real Postgres.
- Coverage gate is 60% (`--cov-fail-under=60`) and conftest force-enables `AGENTS_ENABLED=true`, so agent routes are always mounted under test.
- `backend/celerybeat-schedule*` files are untracked runtime state — don't commit; `backend/data/` is gitignored.
- fastembed downloads a ~120 MB ONNX model to `/app/data/embedding_models` on first agents knowledge-base use; offline installs need it pre-seeded.