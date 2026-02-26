---
paths:
  - ".github/**"
  - "Dockerfile"
  - "docker-compose.yml"
  - "Makefile"
---

# CI/CD and Development Workflow

## Local Development

### Environment Setup

```bash
make venv          # Create virtual environment (pyenv if available, otherwise python3)
make install       # Install dependencies and pre-commit hooks
source venv/bin/activate
```

### Running the Application

```bash
make runtime       # Build and start all Docker services
make down          # Stop all containers
make clean         # Clean up Docker resources and venv
```

### Docker Services

| Service | Port | Purpose |
|---------|------|---------|
| runtime | 8000 | FastAPI API server (auto-reload in local) |
| db | 5432 | PostgreSQL 16 |
| redis | 6382 | Redis (session store, token blacklist) |
| test | - | Test runner (separate db/redis) |
| test_db | 5434 | PostgreSQL test database |
| test_redis | 6381 | Redis test instance |
| redisinsight | 8002 | Redis management UI |
| datadog | - | Datadog APM agent (optional) |

### Container Startup Flow

1. Docker Compose builds and starts db, redis
2. On first db init, `scripts/init_gateway_tables.sql` runs (creates gateway-owned tables)
3. Waits for health checks (pg_isready, redis-cli ping)
4. Starts runtime container
5. `run.sh` runs `alembic upgrade head` (creates dashboard-owned tables, aborts on failure, emits DogStatsD migration metrics)
6. Starts FastAPI via `ddtrace-run fastapi run` with `--reload` (local/loadtest) or `--workers $WEB_CONCURRENCY` (stagef/prod)

## Testing

```bash
make test          # Run pytest with coverage enforcement (70% branch minimum)
make coverage      # Run tests with coverage enforcement + HTML report
```

Tests run in an isolated Docker environment with separate database and Redis instances.

## Database Migrations

Migrations must run inside Docker (not local venv):

```bash
# Generate migration after model changes
docker compose exec runtime alembic revision --autogenerate -m "description"

# Or from outside the container
docker exec newton-dashboard-api alembic revision --autogenerate -m "description"

# Manually run migrations
docker compose exec runtime alembic upgrade head
```

## Code Quality

### Pre-Commit Hooks

Automatically run on `git commit`:

- `ruff` - Lint with auto-fix
- `ruff-format` - Code formatting
- `trailing-whitespace` - Cleanup
- `end-of-file-fixer`
- `check-yaml`, `check-json`
- `detect-private-key` - Prevent accidental key commits
- `codespell` - Spell checking

### Manual Quality Checks

```bash
# Format code
ruff format .

# Lint code
ruff check .

# Lint and auto-fix
ruff check --fix .
```

## Deployment Environments

| Environment | Variable | AWS Account | API Domain |
|-------------|----------|-------------|------------|
| LOCAL | `NEWTON_ENV=local` | N/A | `localhost:8000` |
| STAGEF | `NEWTON_ENV=stagef` | 701849097212 | `dashboard.api.stagef.newt.foundation` |
| PROD | `NEWTON_ENV=prod` | 574155753192 | `dashboard.api.newt.foundation` |

### CI/CD Workflows

- `ci.yml`: Triggers on push/PR to main. Publishes image, deploys to stagef, then deploys to prod.
- `prod-deploy.yml`: Manual workflow_dispatch for prod-only deploys.
- `reusable-cdk.yml`: Shared workflow. Runner selection: `CDK_DEPLOY_ENV == 'prod'` uses `prod` runner, otherwise `stagef` runner. Each runner has network access to its own environment's RDS. On deploy: checks gateway SSM migration marker (non-blocking), runs Alembic migrations, then CDK deploy. On diff: runs `alembic check` for pending migrations (non-blocking).

### CDK_DEPLOY_ENV

The `CDK_DEPLOY_ENV` input controls three things:
1. **Runner selection**: `prod` runner for prod (VPC access to prod RDS), `stagef` runner for stagef
2. **ACM certificate domain**: `dashboard.api.newt.foundation` (prod) or `dashboard.api.stagef.newt.foundation` (stagef)
3. **Container env vars**: `NEWTON_ENV` and `DD_ENV` are set to the deploy_env value

### ECS Task Definition Secrets

When manually updating ECS task definitions, ALL secrets required by the `Secrets` pydantic model in `app/core/config/secrets.py` must be present. Missing secrets cause `alembic upgrade head` to fail at import time (pydantic validation), crashing the container with exit code 1.

## Required Checks

Before merging:

- [ ] All tests pass (`make test`)
- [ ] ruff format and lint pass
- [ ] Pre-commit hooks pass
- [ ] Database migration generated (if models changed)
- [ ] No secrets in committed code

### CI Coverage Enforcement

Tests enforce a minimum coverage threshold via `pytest-cov`:

```bash
make test       # Runs pytest with --cov=app --cov-branch --cov-fail-under=70
make coverage   # Same + HTML report
```

Configuration is in `pyproject.toml` under `[tool.coverage.run]` and `[tool.coverage.report]`. The threshold should be raised incrementally as test coverage improves.

## Anti-patterns to Avoid

- Do not run migrations outside Docker
- Do not commit `.env` files
- Do not skip pre-commit hooks (`--no-verify`)
- Do not use `make clean` without stopping containers first
- Do not hardcode environment-specific values in code
