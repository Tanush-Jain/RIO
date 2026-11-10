# Contributing to RIO-MJP

## 1. Purpose

This guide explains how to understand, validate, and contribute to the RIO-MJP repository. RIO-MJP (REVA Intelligent Operations – Multi-Agent Journey Platform) supports academic administration and campus operations through a React web application, FastAPI services, PostgreSQL persistence, timetable optimization, and a LangGraph-based multi-agent assistant.

This document describes the repository's development workflow. It does not replace the project specification, technical documentation, or setup instructions.

## 2. Project areas

| Path | Responsibility |
|---|---|
| `frontend/` | React 19 and TypeScript single-page application, built with Vite |
| `backend/main.py` | FastAPI application creation, middleware, health endpoint, and router registration |
| `backend/routes/` | HTTP route modules for academic, administrative, timetable, resource, user, and other campus operations |
| `backend/models/` | SQLAlchemy ORM models representing the application's domain data |
| `backend/schemas/` | Pydantic request, response, and data-transfer schemas |
| `backend/optimizer/` | Timetable solver, independent validator, and timetable quality scorer |
| `backend/core/` | Shared backend security functionality |
| `backend/config.py` and `backend/database.py` | Environment configuration, SQLAlchemy engine/session setup, and database access |
| `agents/` | LangGraph supervisor, query, scheduling, and resource agents, shared state, registries, and campus tools |
| `database/migrations/` | Alembic migration environment and versioned database schema changes |
| `tests/` | Automated backend, API, agent, and domain tests |
| `scripts/` | Development, data preparation, verification, and maintenance utilities |
| `docs/` | Additional developer and operational documentation, when present |

The PostgreSQL database is the persistent source of truth. The agent layer is an orchestration and tool-calling layer; it should not be treated as an independent source of campus records. Timetable candidates are produced by the deterministic optimization workflow and checked by the validator before downstream use.

## 3. Development prerequisites

Check the repository manifests before installing tools. The checked-in configuration currently specifies:

- Python dependencies in `requirements.txt`.
- A PostgreSQL database for application data and a separate PostgreSQL test database.
- Node.js 22 in `.github/workflows/ci.yml`.
- Frontend dependencies and commands in `frontend/package.json`.
- Alembic migration configuration in `alembic.ini`.

Use a supported Python installation compatible with the dependency set and the Python version used by CI. The CI workflow currently configures Python 3.13.

## 4. Initial local setup

Run commands from the repository root unless a command explicitly changes directory.

### 4.1 Create a Python environment

Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

macOS/Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 4.2 Configure environment variables

Copy `.env.example` to `.env` and edit the local values. At minimum, configure:

- `DATABASE_URL`: PostgreSQL URL for the local application database.
- `TEST_DATABASE_URL`: PostgreSQL URL for a dedicated test database.
- `SECRET_KEY`: a private random signing secret of at least 32 bytes.
- `REDIS_URL`: local Redis URL if the Redis/Celery functionality is being used.
- `ANTHROPIC_API_KEY`: valid key only when exercising features that call the Anthropic service.
- `ENV`: local environment setting used by the checked-in environment template.

Generate a suitable local secret with:

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Never commit `.env`, API keys, database passwords, JWTs, or production credentials. The checked-in `.env.example` contains placeholders and examples, not deployable secrets.

### 4.3 Prepare PostgreSQL and apply migrations

Create the local application database and database user using your PostgreSQL administration tools. Set `DATABASE_URL` to that database. From the repository root, apply the Alembic migrations:

```bash
alembic upgrade head
```

`alembic.ini` points to `database/migrations`; the database URL is read by the migration environment from the runtime environment rather than being hard-coded in the INI file.

Follow `setup.md` if its staged setup procedure is the workflow being used for your checkout.

## 5. Running the application locally

### Backend

From the repository root, with the virtual environment active and `.env` configured:

```bash
uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000
```

The application registers its API routers in `backend/main.py`. A basic liveness/readiness endpoint is available at:

```text
GET http://127.0.0.1:8000/health
```

The current handler returns `{"status": "ok", "service": "rio-mjp-backend"}` when the application process is responding. This endpoint alone does not prove that every external integration or background worker is healthy.

### Frontend

In a second terminal:

```bash
cd frontend
npm ci
npm run dev
```

Use the local URL printed by Vite. The frontend's API base URL and environment-specific settings should be checked in the frontend source/configuration before changing environments.

## 6. Validation before committing

Run the checks relevant to the files changed. Do not report a check as passed unless it was actually executed successfully.

### Backend syntax compilation

From the repository root:

```bash
python -m compileall -q backend agents database
```

### Backend tests

The repository's tests require a dedicated PostgreSQL test database. Configure `TEST_DATABASE_URL` before running:

```bash
pytest -q
```

**Test database safety is mandatory:**

- The target database name must contain `test`.
- It must be different from the development database configured by `DATABASE_URL`.
- The test configuration is intended to use an isolated schema.
- Never point tests at production or at a database containing data that must be retained.
- Do not bypass the safeguards in the test setup or destructive database utilities.

The checked-in `.env.example` documents these requirements. The CI workflow also provisions PostgreSQL and creates a separate test database for the backend test job.

### Frontend lint and production build

From `frontend/`:

```bash
npm ci
npm run lint
npm run build
```

`npm run build` runs the TypeScript project build before creating the Vite production bundle.

### Review the final diff

Before committing:

```bash
git diff --check
git status --short
git diff
```

Check that the diff contains only the intended files and does not include `.env`, generated output, local database files, credentials, or unrelated formatting changes.

## 7. Git workflow

Start from the branch required by your team or course. A typical focused workflow is:

```bash
git switch main
git pull --ff-only
git switch -c docs/short-description
```

Use a separate branch for a distinct task. Keep each commit focused on one logical change. Avoid mixing unrelated documentation, formatting, and behavior changes in one commit.

Before committing, stage only the files belonging to that change:

```bash
git add path/to/changed-file
git diff --cached
git commit -m 'chore: "describe the maintenance change"'
```

The repository's maintenance commit examples may use the `chore:` prefix. Use a clear, specific subject that describes what was actually changed. Do not use vague subjects such as `changes`, `final update`, or `misc`.

Examples:

```text
chore: "add contributor workflow documentation"
chore: "clarify timetable optimizer module responsibilities"
chore: "document test database safety requirements"
```

Do not copy an example subject unless the staged diff matches it.

## 8. Change-specific guidance

### API and backend changes

- Keep HTTP concerns in the relevant route module and domain behavior in the appropriate service/module.
- Validate request and response data using the existing schema conventions.
- Preserve server-side authentication, role checks, and department-scope checks.
- Do not rely on frontend visibility rules as authorization.
- Add or update tests for behavior changes.

### Timetable optimization changes

- Keep the solver, validator, and scorer responsibilities distinct.
- Treat hard validation failures as blockers according to the existing workflow.
- Do not claim a timetable is valid solely because the solver returned a feasible result; inspect the validator result and the consuming workflow.
- Add regression coverage for constraint or scoring changes.

### Agent and tool changes

- Preserve the supervisor-to-specialist-agent routing conventions.
- Keep tool inputs and outputs consistent with the schemas and state definitions.
- Ensure campus facts are retrieved through authorized tools/data access rather than invented from model output.
- Test role restrictions and failure paths, not only successful responses.

### Database changes

- Use Alembic migrations for schema changes.
- Review generated migrations before applying them.
- Test migrations against a disposable database.
- Do not commit database dumps or real personal/institutional records.

### Frontend changes

- Follow existing React and TypeScript patterns.
- Keep API interaction consistent with the existing HTTP client and query/cache conventions.
- Run lint and production build checks.
- Include screenshots in a pull request only when a visible UI change is relevant.

## 9. Pull request checklist

A useful pull request description contains:

1. **Summary:** What changed?
2. **Reason:** Why is the change needed?
3. **Scope:** Which project areas and files are affected?
4. **Validation:** Which commands were run, and did they pass?
5. **Risk:** Does the change affect API contracts, database schema, authentication, optimization constraints, or user-visible behavior?
6. **Evidence:** Add screenshots or sanitized logs only when useful.

For a documentation-only or comment-only change, state explicitly that no executable logic, API contract, database schema, or user-visible behavior was changed, provided that this is true of the actual diff.

## 10. Before merging

- Review the complete diff and staged files.
- Confirm that no secrets, private data, or local artifacts are included.
- Confirm relevant automated checks pass.
- Resolve review comments.
- Ensure the branch targets the correct base branch.
- Follow the repository's required approval and merge process.

A clean, small, verifiable contribution is preferred over a large commit containing unrelated changes.
