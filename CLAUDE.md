# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Environment Setup
- `make venv` - Create virtual environment (uses pyenv if available, otherwise python3)
- `make install` - Install dependencies and setup pre-commit hooks
- `source venv/bin/activate` - Activate virtual environment

### Running the Application
- `make dev` - Create venv, install deps, and start Docker (full setup)
- `make runtime` - Build and start the API service with Docker
- `make test` - Run tests with coverage enforcement (70% branch minimum)
- `make coverage` - Run tests with coverage enforcement + HTML report
- `make init EMAIL=user@example.com` - Initialize a user account
- `make init-agent ADDRESS=0x... [NAME=my-agent]` - Create agent user with API key (headless)
- `make datadog` - Start Datadog agent
- `make down` - Stop all Docker containers
- `make clean` - Clean up Docker resources and virtual environment

## Database Migrations

**IMPORTANT**: Database migrations must be run inside the Docker environment, NOT in the local virtual environment.

**Ownership model:** Dashboard and gateway share a single `newton_gateway` database. Alembic manages dashboard tables (`user`, `user_factor`, `user_key`, `cli_session`, `project`). Gateway tables (`api_keys`, `policy_client_secret`, `encrypted_data_refs`) are managed by `newton-prover-avs` sqlx migrations and excluded from `--autogenerate`. For local dev, gateway tables are bootstrapped by `scripts/init_gateway_tables.sql` (auto-generated from gateway migrations, do not edit manually).

### Creating a New Migration

1. Make your changes to the database models in `app/db/models/`
2. Generate the migration from inside the Docker container:
   ```bash
   docker compose exec runtime bash
   alembic revision --autogenerate -m "description of your changes"
   ```
   or
   ```bash
   docker exec newton-dashboard-api alembic revision --autogenerate -m "description of your changes"
   ```

### Running Migrations

Migrations are automatically run on container startup. To manually run migrations:

```bash
docker compose exec runtime bash
alembic upgrade head
```


### Code Quality
- Uses `ruff` for linting and formatting (configured in pyproject.toml)
- Uses `pre-commit` hooks for code quality enforcement
- Line length limit: 100 characters
- Python version: 3.13

## Architecture Overview

### Tech Stack
- **Framework**: FastAPI with SQLAlchemy ORM
- **Database**: PostgreSQL with Alembic migrations
- **Cache**: Redis for token blacklisting and session storage
- **Authentication**: JWT tokens with RS256 signing
- **Blockchain**: web3.py (AsyncWeb3) for on-chain contract interactions
- **Containerization**: Docker with docker-compose for local development

### Application Structure
- `app/main.py` - FastAPI application entry point with middleware setup
- `app/api/v1/` - API routes organized by resource type
- `app/core/` - Core application logic (config, clients, exceptions)
- `app/core/blockchain/` - Async Web3 providers, ABI registry, type-safe contract wrappers
- `app/db/models/` - SQLAlchemy database models
- `app/services/` - Business logic services
- `app/middleware/` - Request/response middleware
- `app/utils/` - Utility functions and constants
- `scripts/` - Maintenance scripts (ABI sync, DB init, agent bootstrap, Datadog monitors, E2E tests)

### Key Components

#### Authentication System
- Multi-factor authentication with email, passkey, and SIWE (Sign-in with Ethereum) support
- API key → JWT token exchange for headless agent authentication (`POST /v1/auth/token`)
- JWT token management with refresh token rotation
- Token blacklisting using Redis for secure logout
- JWKS endpoint for public key distribution

#### Service Architecture
- **TokenManager** (`app/services/tokens/token_manager.py`) - Handles JWT creation, validation, and rotation
- **FactorVerifyFlow** (`app/services/authn/factor_verify_flow/`) - Multi-factor authentication flows

#### Blockchain Layer
- Async Web3 providers with per-chain singletons and retry configuration
- ABI and deployment address registry synced from newton-prover-avs contracts
- Type-safe contract wrappers for NewtonPolicy, NewtonPolicyClient, PolicyClientRegistry
- Multichain support: Ethereum Mainnet, Sepolia, Base, Base Sepolia, Anvil
- Sync ABIs via `python scripts/sync_abis.py`

#### Database Models
- User management with multi-chain support
- API key management
- User factors (email, passkey, SIWE) with verification tracking
- Policy client secrets (read/delete, creates/updates via Newton Prover AVS Gateway)
- Projects (off-chain metadata like names, keyed by policy client address + chain ID)

### Environment Configuration
- Uses `.env` file for local development secrets
- Environment-specific URLs and configuration via `app/core/config/env.py`
- Supports LOCAL, STAGEF, and PROD environments

### Testing
- Separate test database and Redis instance
- Integration and unit tests in `tests/` directory
- Uses pytest with async support
- Test fixtures for authentication and passkey credentials

### Docker Services
- `runtime` - Main API service (port 8000)
- `db` - PostgreSQL database (port 5432)
- `redis` - Redis cache (port 6382)
- `test` - Test runner service
- `test_db` - PostgreSQL test database (port 5434)
- `test_redis` - Redis test instance (port 6381)
- `datadog` - Monitoring agent
- `redisinsight` - Redis management UI (port 8002)

## Key Principles

1. **Never run migrations locally** — always inside Docker (`docker compose exec runtime bash`)
2. **Never edit auto-generated files** — `scripts/init_gateway_tables.sql` and `app/core/blockchain/abis/` are generated
3. **Never leak internal details in error responses** — use generic error messages for clients
4. **Always validate input with Pydantic** — no raw dict access from request bodies
5. **Always use `Depends()` for service injection** — match existing FastAPI patterns
6. **Shared database awareness** — this API shares `newton_gateway` DB with the gateway; only modify dashboard-owned tables

## Common Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| Migration runs locally but fails in Docker | Different Python/DB versions or missing gateway tables | Always generate and run migrations inside Docker |
| `init_gateway_tables.sql` out of date | Gateway added new tables/columns | Re-run `make generate-init-sql` in newton-prover-avs repo |
| ABI mismatch with on-chain contract | Stale ABIs in `app/core/blockchain/abis/` | Run `python scripts/sync_abis.py` |
| Test DB has stale schema | Migration not applied to test DB | Run `make test` (auto-applies migrations to test DB) |
| Auth endpoint returns 500 | JWT signing key not configured in `.env` | Check `RS256_PRIVATE_KEY` and `RS256_PUBLIC_KEY` in `.env` |

## Modular Rules

Detailed guidelines are in `.claude/rules/`:

| File              | Contents                                      |
|-------------------|-----------------------------------------------|
| `architecture.md` | System components, service interactions        |
| `blockchain.md`   | Web3 providers, ABI registry, contract patterns |
| `database.md`     | SQLAlchemy models, Alembic migrations          |
| `code-style.md`   | Formatting, naming, import conventions         |
| `python-style.md` | Python-specific patterns, async, type hints    |
| `testing.md`      | pytest patterns, fixtures, Docker test setup   |
| `security.md`     | Auth, JWT, secrets, input validation           |
| `performance.md`  | Async patterns, connection pooling             |
| `ci-cd.md`        | Docker, GitHub Actions, deployment             |
| `agent-guide.md`  | Key files, layer architecture, agent directives |
| `lessons.md`      | Recurring mistakes and prevention rules        |
