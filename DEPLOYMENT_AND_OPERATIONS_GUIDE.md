# RIO-MJP Deployment and Operations Guide

## 1. Purpose and deployment scope

RIO-MJP (REVA Intelligent Operations – Multi-Agent Journey Platform) is a campus operations and academic administration application. Its repository contains a React/TypeScript frontend, a FastAPI backend, PostgreSQL persistence managed through SQLAlchemy and Alembic, a timetable optimization module based on Google OR-Tools, and LangGraph-based agents with campus tools.

This guide describes the deployment preparation and operational checks supported by the repository configuration. It is not evidence that a production cloud environment, domain, TLS certificate, Redis worker, or monitoring service has already been provisioned. Those resources must be configured in the selected hosting environment.

## 2. Runtime components and responsibilities

| Component | Repository location | Operational responsibility |
|---|---|---|
| Web client | `frontend/` | React 19 and TypeScript user interface; Vite development and production build |
| HTTP application | `backend/main.py` | FastAPI app, CORS middleware, exception handling, `/health`, and API router registration |
| API modules | `backend/routes/` | HTTP endpoints for authentication and campus domains |
| Domain persistence | `backend/models/`, `backend/database.py` | SQLAlchemy models and database session access |
| Configuration | `backend/config.py`, `.env.example` | Environment loading, PostgreSQL URL validation, JWT secret validation, and engine/session setup |
| Schema migrations | `alembic.ini`, `database/migrations/` | Versioned PostgreSQL schema evolution |
| Timetable optimization | `backend/optimizer/` | OR-Tools solver, independent validator, and quality scorer |
| Agent orchestration | `agents/` | Supervisor and specialist agents, state, tool registry, and campus tools |
| Background task integration | `backend/celery_app.py` and Redis configuration | Celery application integration where the deployment enables it |

PostgreSQL is the persistent application data store. The API and authorized tools should read and update campus records through the application's data-access paths. The timetable solver generates candidate schedules; validation and the surrounding approval workflow remain important before a schedule is treated as ready for use.

## 3. Deployment prerequisites

Confirm the following before deployment:

- A supported Python runtime and installed packages from `requirements.txt`.
- Node.js 22 for the frontend build, matching the current GitHub Actions workflow.
- A PostgreSQL instance reachable from the backend.
- A dedicated application database and least-privilege database user.
- A secure `SECRET_KEY` generated for the target environment.
- Correct frontend API configuration for the target backend origin.
- A list of trusted frontend origins for CORS.
- Redis and Anthropic credentials only if the deployed features require them.
- A protected secret/configuration mechanism provided by the hosting platform.

The current CI workflow uses Python 3.13 and PostgreSQL 16 for its backend job. Check compatibility with the actual deployment runtime before selecting a different version.

## 4. Environment configuration

Create a deployment-specific environment configuration using `.env.example` as the variable reference. Do not deploy the example values unchanged.

### Required application settings

| Variable | Purpose | Deployment guidance |
|---|---|---|
| `DATABASE_URL` | Application PostgreSQL connection | Use the production database URL and a restricted database account |
| `SECRET_KEY` | JWT signing secret | Generate a unique random secret of at least 32 bytes; never reuse the placeholder |
| `ENV` | Environment label in the checked-in template | Set consistently with the target environment and verify which modules consume it |
| `ALLOWED_ORIGINS` | Trusted browser origins used by `backend/main.py` | Set to the exact frontend origin(s), comma-separated |

### Optional or feature-dependent settings

| Variable | Purpose | Deployment guidance |
|---|---|---|
| `REDIS_URL` | Redis connection for Celery-related functionality | Configure only when Redis-backed functionality is enabled |
| `ANTHROPIC_API_KEY` | Anthropic API access for agent functionality | Store as a secret; configure only when required |
| `JWT_ACCESS_EXPIRY_MIN` | Access-token expiry documented in `.env.example` | Confirm the active authentication implementation and configuration before changing |
| `JWT_REFRESH_EXPIRY_DAYS` | Refresh-token expiry documented in `.env.example` | Confirm the active authentication implementation and configuration before changing |
| `TEST_DATABASE_URL` | Isolated test database URL | Use only for test execution; never set it to the production database |
| `CONFIRM_DB_RESET` | Explicit confirmation for development reset utilities | Keep disabled in normal operation |
| `CONFIRM_DEV_SEED_RESET` | Explicit confirmation for development seed reset | Keep disabled in normal operation |

The repository's `backend/config.py` reads `.env` from the repository root, requires `DATABASE_URL` to be a valid PostgreSQL SQLAlchemy URL, and rejects missing, placeholder, or shorter-than-32-byte `SECRET_KEY` values. It derives secure-cookie behavior from `ENVIRONMENT=production` or the `COOKIE_SECURE` override. Because the sample file uses `ENV` while this module reads `ENVIRONMENT`, verify the exact variable expected by each deployed component; do not assume these names are interchangeable.

Generate a secret locally:

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Set secrets in the hosting provider's secret manager or protected environment configuration. Do not place real values in source control, Docker build arguments, frontend variables, logs, issue trackers, or screenshots.

## 5. Database preparation and migration

1. Provision PostgreSQL and create the application database and restricted database user.
2. Configure `DATABASE_URL` for the target database.
3. Confirm network access from the backend runtime to PostgreSQL.
4. Take a backup and verify the recovery procedure before a schema release.
5. From the repository root, with the intended environment loaded, run:

```bash
alembic upgrade head
```

The checked-in `alembic.ini` points to `database/migrations`; it intentionally does not hard-code a database URL. The migration environment obtains the URL at runtime.

6. Confirm the migration command completes successfully.
7. Record the deployed revision as part of the release record.

Do not run test, reset, truncate, or development seed utilities against production. The environment template documents explicit safeguards for destructive development database operations. A database rollback must follow a reviewed recovery plan and a verified backup; do not assume an application rollback automatically reverses schema changes.

## 6. Backend build and startup

Install dependencies in the backend deployment environment:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

A basic ASGI startup command, suitable for a controlled deployment environment, is:

```bash
uvicorn backend.main:app --host 0.0.0.0 --port 8000
```

For production, run Uvicorn under the process-management mechanism supported by the hosting platform. Configure restart behavior, worker/process sizing, resource limits, TLS termination, and inbound network rules according to the platform and expected load. Do not expose the database port publicly.

### Health endpoint

`backend/main.py` defines:

```text
GET /health
```

A successful response is:

```json
{
  "status": "ok",
  "service": "rio-mjp-backend"
}
```

This is a process-level liveness/readiness signal implemented by the application. It does not independently verify every downstream dependency, agent integration, Redis worker, or database operation. Add infrastructure-level dependency checks only through a deliberate implementation change; do not interpret this endpoint as proof that all services are operational.

## 7. Frontend build and serving

From the frontend directory:

```bash
cd frontend
npm ci
npm run lint
npm run build
```

The checked-in `package.json` defines `npm run build` as a TypeScript project build followed by a Vite production build. The resulting static assets are written to `frontend/dist/`.

Publish `frontend/dist/` using the selected static hosting platform or web server. Configure the frontend's API base URL using the actual configuration mechanism present in the frontend source. Ensure the configured backend URL matches the deployed API and that browser requests use HTTPS in production.

The repository's `netlify.toml` may be relevant if Netlify is the selected frontend host; review its redirects and build settings before relying on it. The presence of a hosting configuration file does not mean a site has already been deployed.

## 8. CORS, authentication, and security checks

`backend/main.py` reads `ALLOWED_ORIGINS` as a comma-separated list. If it is not configured, the application falls back to local development origins (`localhost` and `127.0.0.1` on ports 5173 and 3000). Set the exact trusted frontend origin(s) for a deployed environment.

Before release:

- Use HTTPS for browser-to-API traffic.
- Set a unique, strong `SECRET_KEY`.
- Confirm production cookie security settings and authentication behavior.
- Restrict database access to the backend and authorized operators.
- Apply role and department authorization on the server; frontend route visibility is not an access-control boundary.
- Protect administrative operations and credentials.
- Do not log passwords, access/refresh tokens, API keys, or sensitive student/faculty records.
- Store secrets outside the repository and rotate them through a controlled process.
- Use least-privilege database and service accounts.

## 9. Pre-release verification checklist

### Repository and build

- [ ] Confirm the release commit or tag.
- [ ] Install Python dependencies from `requirements.txt`.
- [ ] Run `python -m compileall -q backend agents database`.
- [ ] Run the backend test suite against a dedicated test database.
- [ ] Run `npm ci`, `npm run lint`, and `npm run build` in `frontend/`.
- [ ] Review CI results from `.github/workflows/ci.yml`.

### Database and configuration

- [ ] Confirm `DATABASE_URL` points to the intended PostgreSQL database.
- [ ] Confirm migrations are at the expected Alembic revision.
- [ ] Confirm the production signing secret is present and not a placeholder.
- [ ] Confirm the frontend origin list is correct.
- [ ] Confirm optional integration credentials are present only when required.
- [ ] Confirm backups and recovery procedures.

### Runtime smoke checks

- [ ] Confirm the backend process starts successfully.
- [ ] Request `GET /health` and verify the expected JSON response.
- [ ] Confirm the frontend loads and can reach the configured API.
- [ ] Exercise authentication using approved test accounts.
- [ ] Verify role-restricted routes reject unauthorized users.
- [ ] Test representative read-only campus queries.
- [ ] Exercise a controlled timetable-generation workflow in a non-production or approved test context.
- [ ] Confirm logs do not contain secrets or sensitive personal information.

The current GitHub Actions workflow runs frontend dependency installation, lint, and build checks. Its backend job provisions PostgreSQL, installs `requirements.txt`, compiles `backend`, `agents`, and `database`, and runs `pytest -q` against a separate test database. CI success is a useful release gate, but it does not replace deployment-specific smoke tests.

## 10. Operations and routine maintenance

### Monitoring

Monitor the signals available in the chosen hosting environment:

- Backend process availability and restart count.
- HTTP error rates and request latency.
- PostgreSQL connection availability, storage, and resource usage.
- Background task/Redis health if those features are enabled.
- Agent-provider connectivity and API errors if AI features are enabled.
- Timetable generation duration and validation outcomes.
- Frontend asset availability and browser-side API failures.

The repository does not define a complete production observability stack in the files reviewed for this guide. Select and configure logging, metrics, alerting, and tracing according to the deployment environment.

### Backups

- Schedule database backups according to the institution's recovery requirements.
- Restrict backup access and encrypt backups at rest and in transit.
- Retain backups according to the approved retention policy.
- Test restoration periodically in a non-production environment.
- Record the recovery owner and restoration procedure.

### Safe operational changes

- Apply schema changes through reviewed Alembic migrations.
- Test destructive scripts only against disposable development/test databases.
- Treat timetable approval/publication and administrative changes as controlled operations.
- Preserve audit records required by the application and institutional policy.
- Deploy a known commit and retain a rollback plan.

## 11. Troubleshooting

| Symptom | Checks |
|---|---|
| Backend fails during startup | Check that `DATABASE_URL` is present, valid, and targets PostgreSQL; verify `SECRET_KEY` is non-placeholder and at least 32 bytes |
| Database connection refused | Verify PostgreSQL is running, the hostname/port are reachable, and database credentials are correct |
| Alembic cannot connect | Confirm the environment is loaded and the migration command is run from the repository root |
| Frontend browser requests fail with CORS errors | Compare the browser's exact origin with `ALLOWED_ORIGINS`; check scheme, hostname, and port |
| Frontend loads but API calls fail | Verify the frontend API base URL, backend address, TLS configuration, and network rules |
| Authentication cookie is not retained | Check HTTPS and the active cookie security configuration; inspect browser cookie diagnostics without exposing token values |
| Backend tests cannot connect | Confirm PostgreSQL is available and `TEST_DATABASE_URL` targets a separate database whose name contains `test` |
| Agent feature fails | Check whether the feature requires a valid `ANTHROPIC_API_KEY`, provider availability, and relevant application logs |
| Redis-backed task does not run | Confirm Redis is reachable and the required worker process is actually deployed and running |

When reporting an incident, include the release identifier, environment name, failing endpoint or command, timestamp, and sanitized error details. Remove passwords, tokens, API keys, and personal data before sharing logs.

## 12. Deployment record

For each release, record:

| Field | Value to record |
|---|---|
| Release identifier | Git commit SHA or version tag |
| Environment | Development, staging, or production |
| Deployment date/time | Timestamp and timezone |
| Backend version | Deployed commit/build |
| Frontend version | Deployed commit/build |
| Database migration | Alembic revision applied |
| Configuration review | Confirmation without recording secret values |
| Verification | Health check, smoke tests, and CI result |
| Operator | Authorized person/team responsible |
| Rollback reference | Approved rollback or recovery procedure |

Do not store secret values in the deployment record.
