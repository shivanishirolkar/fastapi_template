# Step 0 — Git & GitHub Setup

## 1. Configure Git
```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

## 2. Generate SSH Key and Add to GitHub
```bash
ssh-keygen -t ed25519 -C "your@email.com"
# hit enter for all defaults
cat ~/.ssh/id_ed25519.pub
# copy output, go to GitHub → Settings → SSH Keys → New SSH Key → paste
```

## 3. Test the Connection
```bash
ssh -T git@github.com
# should say: Hi username! You've successfully authenticated
```

## 4. Create the Repo
- Go to github.com → New repository
- Name it, set **private**, **do not initialize with README**
- Copy the SSH remote URL

## 5. Initialize and Push
```bash
mkdir project && cd project
git init
echo ".env
__pycache__
*.pyc
.pytest_cache
.coverage" > .gitignore
git add .gitignore
git commit -m "initial commit"
git branch -M main
git remote add origin git@github.com:yourusername/repo-name.git
git push -u origin main
```

# 1 — Docker Setup

## Prerequisite
Install Docker Desktop from https://docker.com and ensure it is running before proceeding.

## Files to Create

### `.env`
Create a `.env` file in the project root with these exact keys:
- `DATABASE_URL` — asyncpg connection string to a local postgres service named `postgres`, database `scheduler_db`
- `REDIS_URL` — redis connection string to a local service named `redis`
- `LOG_LEVEL=INFO`
- `API_KEY=changeme`

### `Dockerfile`
Create a standard Python 3.13-slim Dockerfile that installs from `requirements.txt` and starts the app with uvicorn on port 8000 with `--reload`.

### `docker-compose.yml`
Create a `docker-compose.yml` with exactly 3 services — `postgres`, `redis`, and `app` — with `restart: always` on all three.

- **postgres**: `postgres:16-alpine`, credentials matching `DATABASE_URL`, healthcheck using `pg_isready`
- **redis**: `redis:7-alpine`
- **app**: builds from the current directory, depends on postgres being healthy and redis being started, mounts the current directory into the container so code changes reflect without rebuilding, exposes port `8000`, reads env from `.env`

> Note: the app won't start yet — `requirements.txt` and `app/main.py` don't exist until prompt 3. Do not attempt to run or verify anything now.

---

# 2 — Project Hygiene

## `.gitignore`
Create a `.gitignore` excluding: `.env`, `__pycache__`, `*.pyc`, `.pytest_cache`, and `.coverage`.

## `.env.example`
Create a `.env.example` with the same keys as `.env` but placeholder values, so collaborators know what to configure:
- `DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/scheduler_db`
- `REDIS_URL=redis://redis:6379/0`
- `LOG_LEVEL=INFO`
- `API_KEY=your-api-key-here`

---

# 3 — Skeleton

## Folder Structure
Create the following folder structure under `app/`. Add an empty `__init__.py` to every folder. Do not write any application code yet — empty files only.
```
app/
├── __init__.py
├── main.py
├── config.py
├── models/
├── schemas/
├── repositories/
├── services/
├── api/
│   └── routes/
├── workers/
└── core/
scripts/
└── (empty for now)
```

## `requirements.txt`
Create a `requirements.txt` with these packages (no pinned versions yet):
```
fastapi
uvicorn
sqlalchemy
greenlet
asyncpg
pydantic
pydantic-settings
celery
psycopg2-binary
redis
httpx
pytest
pytest-asyncio
pytest-cov
python-json-logger
```

## `app/config.py`
Use `pydantic-settings` `BaseSettings` with exactly these fields:
- `DATABASE_URL: str`
- `REDIS_URL: str`
- `LOG_LEVEL: str` — defaults to `"INFO"`
- `API_KEY: str` — defaults to `"changeme"`

Export a `get_settings()` function decorated with `@lru_cache`.

## `app/main.py`
Create a minimal `app/main.py` containing only:
- An async `lifespan` context manager with placeholder comments for startup and shutdown logic
- A `FastAPI` app instance using that lifespan
- Placeholder comments for routers and middleware
- A `GET /` route returning `{"message": "ok"}`

## Documentation
Add docstrings to every function, class, and module created in this prompt and all subsequent prompts. Docstrings should explain what the function does, its parameters, and its return value.

## Scripts
Create two scripts in `scripts/` and make both executable with `chmod +x`:

**`scripts/rebuild.sh`** — full Docker rebuild. Use when changing `.env`, adding packages to `requirements.txt`, or making database schema changes. Runs `docker compose down`, then `docker compose up --build -d`, waits for the app to start, then runs `scripts/check.sh`.

**`scripts/check.sh`** — lightweight check without rebuilding. Use after code-only changes since uvicorn's `--reload` picks them up automatically. Curls `/health` and runs `pytest tests/ -v --cov=app`. Exits with a failure message if tests fail.

## Verification
```bash
docker compose up --build -d
docker compose exec app pip freeze > requirements.txt
./scripts/check.sh
# Expected: {"message": "ok"} and all tests pass
```

---

# 4 — Logging + Error Handling

## `app/core/logger.py`
Create a `setup_logging()` function that:
- Reads `LOG_LEVEL` from settings
- Configures the root logger to output structured JSON via `python-json-logger`'s `JsonFormatter`
- Includes exactly these fields on every line: `asctime`, `levelname`, `message`
- Outputs to stdout

## `app/main.py` — Logging
Call `setup_logging()` at the very top of `app/main.py`, before anything else.

## `app/main.py` — Request Logging Middleware
Add middleware that logs a single JSON line per request with exactly these fields:
- `method`, `path`, `status_code`, `latency` (seconds, 3 decimal places)

Log at `INFO` level. Example output:
```json
{"asctime": "2026-02-28T10:34:33", "levelname": "INFO", "message": "request", "method": "GET", "path": "/", "status_code": 200, "latency": 0.002}
```

## `app/main.py` — Global Exception Handlers
Register handlers for both `fastapi.HTTPException` and `starlette.exceptions.HTTPException` — FastAPI is built on Starlette and unknown routes raise Starlette's variant before FastAPI can intercept them. Both handlers must be identical and switch on `exc.status_code`:

| Status Code | Response Body |
|---|---|
| 401 | `{"error": "unauthorized"}` |
| 404 | `{"error": "not found"}` |
| 409 | `{"error": "job already exists"}` |
| 429 | `{"error": "too many requests"}` |
| any other | return `exc.detail` as-is |

Also register handlers for:
- `RequestValidationError` → 422 `{"error": "validation error", "details": <cleaned error list>}` — strip the `ctx` key from each error dict before returning, as `ctx` may contain raw Python exception objects that are not JSON serializable
- generic `Exception` → 500 `{"error": "internal server error"}` — log the full traceback at `ERROR` level using `logger.exception` with `method` and `path` before returning

All handlers must serialize the response body using `model_dump(exclude_none=True)` so optional fields set to `None` are never included in the response.

## Verification
```bash
curl http://localhost:8000/
docker compose logs app --tail=20
```
A JSON log line matching the format above must appear.

---

# 5 — Basic Endpoints

## `app/api/routes/health.py`
Create two endpoints. No database connection yet.

- `GET /health` → `{"status": "ok"}`
- `GET /metrics` → hardcoded placeholder with a `# TODO: replace with get_job_counts(db)` comment:
  ```json
  {"jobs_pending": 0, "jobs_running": 0, "jobs_completed": 0, "jobs_failed": 0}
  ```

Register both in `app/main.py`.

## Verification
```bash
curl http://localhost:8000/health
curl http://localhost:8000/metrics
```
Then open http://localhost:8000/docs and confirm all 3 endpoints (`/`, `/health`, `/metrics`) are listed.

---

# 6 — Test Setup

## Folder Structure
```
tests/
├── __init__.py
├── conftest.py
├── pytest.ini
└── integration/
    ├── __init__.py
    └── test_health.py
```

## `tests/pytest.ini`
Set `asyncio_mode = auto`.

## `tests/conftest.py`
Create a function-scoped `AsyncClient` fixture pointing at the FastAPI app from `app.main`. No database setup yet.

## `tests/integration/test_health.py`
Write async tests for these cases:

| Test | Request | Expected Status | Expected Body |
|---|---|---|---|
| `test_root_returns_200` | `GET /` | 200 | `{"message": "ok"}` |
| `test_health_returns_200` | `GET /health` | 200 | `{"status": "ok"}` |
| `test_metrics_returns_200` | `GET /metrics` | 200 | contains keys `jobs_pending`, `jobs_running`, `jobs_completed`, `jobs_failed` all with integer values |
| `test_unknown_route_returns_404` | `GET /unknown` | 404 | `{"error": "not found"}` — this is raised by Starlette before FastAPI intercepts it, so both exception handlers registered in prompt 4 must be in place for this test to pass |

## Verification
```bash
./scripts/check.sh
```

---

# 7 — First Deployment

## Prerequisites (Manual Steps — Do These Before Running Any Commands)

1. Create a DigitalOcean project and a Droplet with these exact specs:
   - **OS**: Ubuntu 24.04 LTS
   - **RAM**: 1GB, **CPU**: 1 vCPU, **SSD**: 25GB
   - **Region**: San Francisco

2. Copy the public key:
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
   Add it to the Droplet during creation under **Authentication → SSH Keys**.

3. Once created, copy the **public IPv4** from the DigitalOcean dashboard.

## Install Docker on the Droplet
SSH into the Droplet and install Docker:
```bash
ssh root@$DROPLET_IP

# Install Docker using the official convenience script
curl -fsSL https://get.docker.com | sh

# Verify both are installed
docker --version
docker compose version

exit
```

## Copy Project to the Droplet
From your local machine:
```bash
scp -r . root@$DROPLET_IP:/app
```

## Start the App
```bash
ssh root@$DROPLET_IP
cd /app
docker compose up --build -d
curl http://localhost:8000/health
exit
```

## `scripts/deploy.sh`
Create `scripts/deploy.sh` (executable) that:
- Sets `DROPLET_IP` at the top — replace the placeholder with your actual IP before running
- Uses `rsync` to sync the project to the Droplet, excluding `.git`, `__pycache__`, and `*.pyc`
- SSHs in to run `docker compose up --build -d`
- Waits a few seconds then curls `/health` to confirm the app is running

Do not include the `X-API-Key` header yet — authentication is added in prompt 17.

---

# 8 — API Contract

## `app/schemas/job_schema.py`
Schemas only — no database or business logic.

### Enums
- `JobStatus`: `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`
- `JobPriority`: `LOW`, `NORMAL`, `HIGH`

### `JobCreate`
| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | `str` | yes | |
| `payload` | `dict` | yes | |
| `priority` | `JobPriority` | no | defaults to `NORMAL` |
| `scheduled_at` | `datetime` | no | optional, timezone-aware |

Add a `@field_validator` on `payload` that rejects an empty dict with `ValueError("payload must not be empty")`.

### `JobResponse`
| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | `UUID` | yes | |
| `name` | `str` | yes | |
| `payload` | `dict` | yes | |
| `status` | `JobStatus` | yes | |
| `priority` | `JobPriority` | yes | |
| `created_at` | `datetime` | yes | |
| `updated_at` | `datetime` | yes | |
| `scheduled_at` | `datetime` | no | |
| `started_at` | `datetime` | no | |
| `completed_at` | `datetime` | no | |
| `result` | `dict` | no | |

Enable `from_attributes = True` so it can be constructed from a SQLAlchemy model instance.

### `JobListResponse`
| Field | Type | Notes |
|---|---|---|
| `items` | `list[JobResponse]` | |
| `total` | `int` | total jobs in database |
| `page` | `int` | 1-indexed |
| `page_size` | `int` | |

### `ErrorResponse`
| Field | Type | Notes |
|---|---|---|
| `error` | `str` | |
| `details` | `any` | optional, validation errors only |

Update all exception handlers in `app/main.py` to use `ErrorResponse` as the response body.

---

# 9 — Job Model

## `app/models/job_model.py`
SQLAlchemy model only — no database connection or business logic. Table name: `jobs`.

| Column | Type | Notes |
|---|---|---|
| `id` | `UUID` | primary key, auto-generated |
| `name` | `String` | not null |
| `status` | `Enum(JobStatus)` | not null, defaults to `PENDING` |
| `priority` | `Enum(JobPriority)` | not null, defaults to `NORMAL` |
| `payload` | `JSON` | not null |
| `scheduled_at` | `DateTime(timezone=True)` | nullable |
| `started_at` | `DateTime(timezone=True)` | nullable |
| `completed_at` | `DateTime(timezone=True)` | nullable |
| `result` | `JSON` | nullable |
| `created_at` | `DateTime(timezone=True)` | not null, auto-set on insert |
| `updated_at` | `DateTime(timezone=True)` | not null, auto-set on insert and update |

Import `JobStatus` and `JobPriority` from `app.schemas.job_schema` — do not redefine them.

---

# 10 — Database Connection

## `app/database.py`
Database connection only — no business logic.
- Async SQLAlchemy engine using `asyncpg` and `DATABASE_URL` from settings
- Async `SessionLocal` via `async_sessionmaker`
- `Base` declarative base for all models
- `get_db` async dependency that yields a session and closes it after the request

## `app/main.py` — Lifespan
Update the startup section to:
- Import all models so SQLAlchemy registers them before `create_all`
- Retry the database connection up to 5 times, with a 2-second sleep between attempts, using `SELECT 1` to verify liveness
- Log and re-raise if all attempts fail
- Call `Base.metadata.create_all` on success

## `app/api/routes/health.py` — Database Check
Update `GET /health` to test the real database on every request using `get_db`:
- Success: `{"status": "healthy", "database": "connected"}`
- Failure: `{"status": "unhealthy", "database": "unreachable"}` — log the exception before returning

Leave `GET /metrics` returning hardcoded zeros for now — it will be wired to real data in prompt 12 once the repository exists.

## Verification
Run a full rebuild since the database schema is being created:
```bash
./scripts/rebuild.sh
docker compose exec postgres psql -U postgres -d scheduler_db -c "\dt"
# Expected: jobs table visible
```

---

# 11 — Test Database

## `tests/conftest.py`
Replace the existing fixture with one that sets up a real Postgres test database. Rules:
- Use `DATABASE_URL` from settings with the database name replaced by `scheduler_test_db`
- Create `scheduler_test_db` at session start if it doesn't exist (connect to the default `postgres` database to issue `CREATE DATABASE`)
- Fresh async engine and session per test function
- `Base.metadata.create_all` before each test, `drop_all` after — clean schema every test
- Override the `get_db` dependency on the FastAPI app to use the test session
- Never use `aiosqlite` — always `asyncpg` against real Postgres
- `AsyncClient` fixture uses the app with the overridden dependency

## `tests/integration/test_health.py`
Add:

| Test | Request | Expected Status | Expected Body |
|---|---|---|---|
| `test_health_database_connected` | `GET /health` | 200 | `{"status": "healthy", "database": "connected"}` |

## `tests/integration/test_jobs.py`
Create the file. Add one test:

| Test | Description |
|---|---|
| `test_jobs_table_exists` | Query `information_schema.tables` where `table_name = 'jobs'` and assert exactly one row is returned |

## Verification
```bash
./scripts/rebuild.sh
./scripts/deploy.sh
```

---

# 12 — Repository

## `app/repositories/job_repository.py`
Raw database queries only — no business logic, no validation, no HTTP code. Implement exactly these five async functions:

### `create_job(db, job_data: JobCreate) -> Job`
Insert a new row from `job_data` and return the created instance.

### `get_job_by_id(db, job_id: UUID) -> Job | None`
Return the matching row or `None`.

### `get_active_job_by_name(db, name: str) -> Job | None`
Return the first row where `name` matches and `status` is `PENDING` or `RUNNING`. Filter in the database — do not fetch all rows and loop.

### `get_all_jobs(db, page: int, page_size: int) -> tuple[list[Job], int]`
Return a page of jobs (`LIMIT`/`OFFSET`, 1-indexed) plus a total `COUNT(*)`.

### `update_job_status(db, job_id: UUID, status: JobStatus, **kwargs) -> Job | None`
Update `status` and any additional columns passed as kwargs (e.g. `started_at`, `completed_at`, `result`). Return the updated instance or `None`.

### `update_job(db, job_id: UUID, updates: dict) -> Job | None`
Update any provided fields from the `updates` dict and return the updated instance, or `None` if not found.

### `delete_job(db, job_id: UUID) -> bool`
Delete the row matching `job_id`. Return `True` on success, `False` if not found.

### `get_job_counts(db) -> dict`
Run a single `GROUP BY status` query on the `jobs` table. Return a dict with exactly these keys: `jobs_pending`, `jobs_running`, `jobs_completed`, `jobs_failed`. Any status with no rows should default to `0`.

## `app/api/routes/health.py` — Wire Metrics
Now that the repository exists, update `GET /metrics` to replace the `# TODO` placeholder — inject `get_db` and call `get_job_counts(db)`. Return the result directly.

## Verification
```bash
./scripts/check.sh
./scripts/deploy.sh
```

---

# 13 — Service

## `app/services/job_service.py`
Business logic only — no direct queries, no HTTP code. All database access through the repository.

### `create_job(db, job_data: JobCreate) -> Job`
- Check for an active job with the same name via `get_active_job_by_name`
- If found, raise `ValueError("active job with this name already exists")`
- Otherwise call `create_job` from the repository and return the result

### `get_job(db, job_id: UUID) -> Job`
- Call `get_job_by_id`; if `None`, raise `ValueError("job not found")`
- Otherwise return the result

### `list_jobs(db, page: int, page_size: int) -> tuple[list[Job], int]`
Delegate directly to `get_all_jobs` — no additional logic.

### `update_job(db, job_id: UUID, updates: dict) -> Job`
- Call `get_job_by_id`; if `None`, raise `ValueError("job not found")`
- Otherwise call `update_job` from the repository and return the result

### `delete_job(db, job_id: UUID)`
- Call `get_job_by_id`; if `None`, raise `ValueError("job not found")`
- Otherwise call `delete_job` from the repository

## Verification
```bash
./scripts/check.sh
```

---

# 14 — Unit Test Service Layer

## `tests/unit/test_job_service.py`
Create `tests/unit/` with an `__init__.py` and `test_job_service.py`. Use `unittest.mock.AsyncMock` to mock the repository functions — do not touch the database. Test the service logic in isolation.

### Required Tests
| Test | Description | Expected Behaviour |
|---|---|---|
| `test_create_job_success` | Repository returns no active job, then creates one | Service returns the created job, repository `create_job` called once |
| `test_create_job_duplicate` | `get_active_job_by_name` returns an existing job | `ValueError("active job with this name already exists")` raised, repository `create_job` never called |
| `test_get_job_success` | `get_job_by_id` returns a job | Service returns it |
| `test_get_job_not_found` | `get_job_by_id` returns `None` | `ValueError("job not found")` raised |
| `test_list_jobs_delegates` | Call `list_jobs` | `get_all_jobs` called with correct `page` and `page_size`, result returned as-is |
| `test_update_job_success` | `get_job_by_id` returns a job, `update_job` returns updated job | Service returns updated job |
| `test_update_job_not_found` | `get_job_by_id` returns `None` | `ValueError("job not found")` raised, repository `update_job` never called |
| `test_delete_job_success` | `get_job_by_id` returns a job | Service calls repository `delete_job` |
| `test_delete_job_not_found` | `get_job_by_id` returns `None` | `ValueError("job not found")` raised, repository `delete_job` never called |

### Edge Cases
After writing the required tests, review `job_service.py` and add any additional edge cases not covered. Leave a comment on each explaining what it covers and why.

## Verification
```bash
./scripts/check.sh
```

---

# 15 — Routes

## `app/api/routes/job_routes.py`
Thin layer only — call the service, return the response. No business logic or direct queries.

### `POST /jobs`
- Accept `JobCreate` body
- Return `JobResponse` with status `201`
- On `ValueError("active job with this name already exists")` → `HTTPException(409)`

### `GET /jobs`
- Query params: `page=1`, `page_size=10`
- Return `JobListResponse` with status `200`

### `GET /jobs/{id}`
- Path param `id: UUID`
- Return `JobResponse` with status `200`
- On `ValueError("job not found")` → `HTTPException(404)`

### `PATCH /jobs/{id}`
- Path param `id: UUID`
- Accept an optional partial body — any subset of `JobCreate` fields
- Return updated `JobResponse` with status `200`
- On `ValueError("job not found")` → `HTTPException(404)`

### `DELETE /jobs/{id}`
- Path param `id: UUID`
- Return `204` with no body on success
- On `ValueError("job not found")` → `HTTPException(404)`

Register the router in `app/main.py`.

## Verification
```bash
./scripts/check.sh
```

---

# 16 — Verify Routes

## Test via Swagger UI
Open http://localhost:8000/docs and create a job via `POST /jobs`:
```json
{"name": "test-job", "payload": {"task": "test"}, "priority": "high"}
```

## Verify in the Database
```bash
docker compose exec postgres psql -U postgres -d scheduler_db -c "SELECT * FROM jobs;"
```

---

# 17 — Authentication

## `.env`
Generate a secure API key and replace `API_KEY=changeme`. Run this in your terminal and copy the output:
```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```
Paste the result as the value of `API_KEY` in `.env`. Do not print or log it anywhere in the code.

## `app/core/security.py`
Create a single FastAPI dependency `verify_api_key`:
- Read the `X-API-Key` header using `Header(default=None)` — using `Header(...)` would make it required and cause a 422 instead of 401 when the header is missing
- If the header is `None` or does not match `API_KEY` from settings (compared using `secrets.compare_digest`), raise `HTTPException(401)`
- Use `secrets.compare_digest` to prevent timing attacks

## `app/api/routes/job_routes.py`
Apply `verify_api_key` to all job routes via the router's `dependencies` parameter. Do not apply it to `/`, `/health`, `/metrics`, `/docs`, or `/openapi.json`.

## `scripts/deploy.sh`
- Read `API_KEY` from `.env` at the top of the script using `export $(grep -v '^#' .env | xargs)`
- Include the `X-API-Key` header in the health check curl

## Verification
Run a full rebuild since `.env` has changed:
```bash
./scripts/rebuild.sh
```

---

# 18 — Test Authentication

## `tests/conftest.py`
Update the `AsyncClient` fixture to include `X-API-Key: <API_KEY from settings>` by default on every request, so all future tests are written against a secured API from the start.

## `tests/integration/test_authentication.py`
Create the file with these tests:

### Required Tests
| Test | Description | Expected Status | Expected Body |
|---|---|---|---|
| `test_missing_api_key_returns_401` | `POST /jobs` with no key | 401 | `{"error": "unauthorized"}` |
| `test_invalid_api_key_returns_401` | `POST /jobs` with wrong key | 401 | `{"error": "unauthorized"}` |
| `test_valid_api_key_returns_201` | `POST /jobs` with correct key | 201 | valid `JobResponse` |
| `test_health_requires_no_api_key` | `GET /health` with no key | 200 | `{"status": "healthy", "database": "connected"}` |

For the 401 tests, override the fixture's default header by passing `headers={"X-API-Key": ""}` and `headers={"X-API-Key": "wrongkey"}` directly in those requests only.

### Edge Cases
Review `app/core/security.py` and add any additional edge cases not covered above. Leave a comment on each explaining what it covers and why.

## Verification
```bash
./scripts/check.sh
```

---

# 19 — Test Routes

## `tests/integration/test_jobs.py`
Add the following tests using the `AsyncClient` fixture. The fixture already includes the `X-API-Key` header by default — all requests are authenticated.

### Happy Path
| Test | Description | Expected Status | Expected Body |
|---|---|---|---|
| `test_create_job` | `POST /jobs` with valid body | 201 | `JobResponse` with `status: PENDING` |
| `test_get_job_by_id` | `GET /jobs/{id}` with existing id | 200 | matching `JobResponse` |
| `test_get_jobs_empty` | `GET /jobs` with no jobs | 200 | `{"items": [], "total": 0, "page": 1, "page_size": 10}` |
| `test_get_jobs_with_created_job` | `GET /jobs` after creating one | 200 | `total: 1`, job in `items` |

### Validation Errors
| Test | Description | Expected Status |
|---|---|---|
| `test_create_job_missing_name` | No `name` field | 422 |
| `test_create_job_invalid_priority` | `priority: "invalid"` | 422 |
| `test_create_job_empty_payload` | `payload: {}` | 422 — this triggers the `@field_validator` which raises a `ValueError`. The validation error handler must strip the `ctx` key from each error dict before returning, as `ctx` contains the raw `ValueError` object which is not JSON serializable |

### Business Logic Errors
| Test | Description | Expected Status | Expected Body |
|---|---|---|---|
| `test_create_duplicate_job` | Same `name` twice | 409 | `{"error": "job already exists"}` |
| `test_get_job_fake_uuid` | Valid UUID that doesn't exist | 404 | `{"error": "not found"}` |
| `test_update_job_not_found` | `PATCH /jobs/{id}` with fake UUID | 404 | `{"error": "not found"}` |
| `test_delete_job_not_found` | `DELETE /jobs/{id}` with fake UUID | 404 | `{"error": "not found"}` |

### PATCH and DELETE Happy Path
| Test | Description | Expected Status | Expected Body |
|---|---|---|---|
| `test_update_job` | `PATCH /jobs/{id}` with valid field update | 200 | updated `JobResponse` reflecting changes |
| `test_delete_job` | `DELETE /jobs/{id}` with valid id | 204 | no body |
| `test_delete_job_twice` | `DELETE /jobs/{id}` same id twice | 404 on second call | `{"error": "not found"}` |

### Edge Cases
| Test | Description | Expected Status |
|---|---|---|
| `test_create_job_missing_payload` | No `payload` field | 422 |
| `test_get_jobs_pagination` | `?page=1&page_size=1` after creating 2 jobs | 200, `total: 2`, 1 item |
| `test_get_job_invalid_uuid_format` | `GET /jobs/not-a-uuid` | 422 |

After writing the above, review `job_routes.py` and `job_service.py` and add any additional edge case tests you can identify. Leave a comment on each explaining what it covers and why it's worth testing.

## Verification
```bash
./scripts/check.sh
./scripts/deploy.sh
```

---

# 20 — Workers

## `docker-compose.yml`
Add a `worker` service:
- Builds from the current directory
- Command: `celery -A app.workers.job_tasks worker --loglevel=info`
- Depends on postgres (healthy) and redis (started)
- Reads from `.env`, no exposed ports, `restart: always`

## `app/workers/job_tasks.py`
Use `psycopg2-binary` for all database access — not asyncpg or SQLAlchemy. Celery workers are synchronous.

### Celery App
Connect to Redis (from `REDIS_URL` env var) as both broker and result backend.

### `process_job(job_id: str)` task
In order:
1. Connect to Postgres via `psycopg2` using `DATABASE_URL` from the environment
2. Set `status = RUNNING`, `started_at = now(UTC)`
3. `time.sleep(5)` to simulate work
4. Set `status = COMPLETED`, `completed_at = now(UTC)`, `result = {"message": "job completed successfully"}`
5. Close the connection

On any exception, before setting `status = FAILED` and re-raising, log at `ERROR` level using `logger.exception` with these fields: `job_id` and `error` (the exception message). This ensures worker failures are visible in `docker compose logs worker`.

## `app/services/job_service.py`
In `create_job`, after the repository commits the new job, dispatch it:
```python
process_job.delay(str(job.id))
```
This must happen after the job is committed, not before.

## Verification
Run a full rebuild since `docker-compose.yml` has changed:
```bash
./scripts/rebuild.sh
```

Create `scripts/test_worker.sh` (executable) that:
- Creates a job via `POST /jobs` and captures the returned `id`
- Polls `GET /jobs/{id}` every 2 seconds, printing the status each time
- Exits when status is `COMPLETED` or `FAILED`

---

# 21 — Test Workers

## `tests/conftest.py`
Add a function-scoped, `autouse=True` fixture that mocks `app.workers.job_tasks.process_job.delay` with a `MagicMock` for every test. Celery must never actually run during testing. The fixture should return the mock so individual tests can assert on it.

## `tests/integration/test_workers.py`
Create the file with these tests:

### Required Tests
| Test | Description | Expected Behaviour |
|---|---|---|
| `test_process_job_delay_called_on_create` | `POST /jobs` with valid body | `process_job.delay` called exactly once with the job's `id` as a string |
| `test_process_job_not_called_on_failed_create` | `POST /jobs` with invalid body | `process_job.delay` never called |
| `test_process_job_not_called_on_duplicate` | `POST /jobs` twice with same name | `process_job.delay` called exactly once total |

### Edge Cases
Review `job_tasks.py` and `job_service.py` and add any additional edge cases not covered above. Leave a comment on each explaining what it covers and why.

## Verification
```bash
./scripts/check.sh
./scripts/deploy.sh
```

---

# 22 — Final Cleanup & Deployment

## Uninstall Unused Packages
Review `requirements.txt` and remove any packages not imported anywhere in the codebase. Update the file after removing them. Run `./scripts/rebuild.sh` after changes.

## Run All Tests
```bash
./scripts/check.sh
```
Review the coverage report. For any file below 80%, add tests to bring it up. The generic 500 handler is difficult to trigger naturally — test it by adding a route in `conftest.py` that deliberately raises an unhandled `Exception`, then assert the response is `{"error": "internal server error"}` with status `500`. Remove the route after testing.

## Verify Live API on Droplet
Using your actual `DROPLET_IP` and `API_KEY`:

1. Hit `/health` and confirm `{"status": "healthy", "database": "connected"}`
2. Create a job and capture the returned `id`
3. Poll `GET /jobs/{id}` every 2 seconds until status is `COMPLETED` or `FAILED`

## Final Deploy
```bash
./scripts/deploy.sh
```

# 23 — Final Cleanup & Deployment

## ```README.md```
Create a detailed README.md covering: project overview, functional and non-functional requirements, full API reference (method, path, request/response, error codes), data model, local setup with Docker, deploying to DigitalOcean via scripts/deploy.sh, how to run tests, and future development. Write in clear technical prose.

---

# Notes

## Future Development

- **Alembic** — replace `create_all` with `alembic upgrade head` for migration history, rollback, and safe schema evolution
- **Rate limiting** — `slowapi` with Redis storage backend on `POST /jobs`
- **Horizontal scaling** — move rate limiter storage to Redis, tune Celery worker pool size
- **Graceful shutdown** — flush in-flight requests, wait for active Celery tasks, close DB connections cleanly
- **Multi-environment** — separate `.env.development` and `.env.production` configs, staging Droplet to verify before promoting to production

## CSV / File Ingestion

- Needs `python-multipart` in `requirements.txt`
- FastAPI uses `UploadFile` instead of a request body
