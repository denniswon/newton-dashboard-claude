---
paths:
  - "app/db/**"
  - "alembic/**"
  - "scripts/init_gateway_tables.sql"
---

# Database Guidelines

## SQLAlchemy Patterns

### Async Engine and Session

All database operations use async SQLAlchemy with asyncpg (`app/db/session.py`):

```python
engine = create_async_engine(
    database_url,
    pool_size=20,
    max_overflow=40,
)
async_session = async_sessionmaker(engine, expire_on_commit=False)
```

For automated testing, `NullPool` is used when `DATABASE__NULL_POOL=true` to avoid async pytest limitations.

### Model Definition

Models inherit from `Base` (declarative base) and use `Mapped` type annotations:

```python
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class User(Base):
    __tablename__ = "user"

    id: Mapped[UUID] = mapped_column(Uuid, primary_key=True, default=uuid4)
    is_active: Mapped[bool] = mapped_column(nullable=False, default=True)
    created_at: Mapped[datetime.datetime] = mapped_column(
        TIMESTAMP(timezone=True), nullable=False, server_default=func.now()
    )
```

### Column Type Conventions

| Python Type | SQLAlchemy Type | PostgreSQL Type | Usage |
|-------------|----------------|-----------------|-------|
| `UUID` | `Uuid` | `UUID` | Primary keys, foreign keys |
| `str` | `Text` | `TEXT` | Variable-length strings |
| `bytes` | `LargeBinary` | `BYTEA` | Ethereum addresses, secrets |
| `bool` | (default) | `BOOLEAN` | Flags |
| `int` | (default) | `INTEGER` | Counts, limits |
| `list` | `JSONB` | `JSONB` | Permissions, metadata |
| `dict` | `JSONB` | `JSONB` | Structured metadata |
| `datetime` | `TIMESTAMP(timezone=True)` | `TIMESTAMPTZ` | Timestamps |

### Custom Column Types

- **EthereumAddress** (`app/db/types.py`) - Stores 20-byte Ethereum addresses as BYTEA, auto-converts "0x..." hex strings
- **Secrets** (`app/db/types.py`) - Stores secrets as BYTEA, converts to/from UTF-8 strings (encoding only, not encryption)

### Relationships

```python
# One-to-Many
user: Mapped["User"] = relationship("User", back_populates="api_keys")

# Use TYPE_CHECKING to avoid circular imports
if TYPE_CHECKING:
    from app.db.models.user import User
```

### Table Arguments

Define constraints and indexes in `__table_args__`:

```python
__table_args__ = (
    CheckConstraint("octet_length(address) = 20", name="api_keys_address_len"),
    Index("idx_api_keys_address", "address"),
    Index("idx_api_keys_active", "is_active"),
    Index("idx_api_keys_permissions", "permissions", postgresql_using="gin"),
)
```

### Timestamps Pattern

All models with timestamps should use:

```python
created_at: Mapped[datetime.datetime] = mapped_column(
    TIMESTAMP(timezone=True), nullable=False, server_default=func.now()
)
updated_at: Mapped[datetime.datetime] = mapped_column(
    TIMESTAMP(timezone=True), nullable=False, server_default=func.now(), onupdate=func.now()
)
```

For reliable `updated_at` behavior, also create a PostgreSQL trigger (see SQL schema comments on models).

## Database Ownership

The dashboard and gateway share a single `newton_gateway` database with split migration ownership:

| Owner | Tables | Migration Tool |
|-------|--------|----------------|
| Dashboard (this repo) | `user`, `user_factor`, `user_key`, `cli_session`, `project` | Alembic |
| Gateway (`newton-prover-avs`) | `api_keys`, `policy_client_secret`, `encrypted_data_refs` | sqlx |

Alembic `--autogenerate` excludes gateway tables via `include_name` (filters DB reflection) and `include_object` (filters model metadata comparison) in `alembic/env.py`. Both are required: `include_name` prevents reflecting gateway tables from the DB, and `include_object` prevents SQLAlchemy models for gateway tables from appearing as pending additions. For local dev, gateway tables are bootstrapped by `scripts/init_gateway_tables.sql` (mounted as `docker-entrypoint-initdb.d/01_gateway_tables.sql`).

To reinitialize the local database (e.g., after schema changes to the init script):

```bash
make down && docker volume rm newton-dashboard-api_postgres_data && make runtime
```

### Cross-Repo Init SQL Sync

`scripts/init_gateway_tables.sql` is **auto-generated** from gateway sqlx migrations by `newton-prover-avs/scripts/generate-init-sql.sh`. Do not edit it manually.

**Automated sync workflow:**

1. Gateway migrations merge to `main` in `newton-prover-avs`
2. `sync-init-sql.yml` generates fresh init SQL and dispatches `repository_dispatch` (`update-init-sql`) to this repo
3. `.github/workflows/update-init-sql.yml` receives the event, regenerates the init SQL, and opens a PR

**Manual regeneration** (requires Docker and sqlx-cli):

```bash
# From the gateway repo
cd /path/to/newton-prover-avs
./scripts/generate-init-sql.sh -o /path/to/newton-dashboard-api/scripts/init_gateway_tables.sql

# Validate existing init SQL matches current migrations
./scripts/generate-init-sql.sh --validate /path/to/newton-dashboard-api/scripts/init_gateway_tables.sql
```

## Alembic Migrations

### Creating Migrations

Migrations must run inside Docker. Only dashboard-owned tables are managed by Alembic:

```bash
# Generate migration after model changes
docker compose exec runtime alembic revision --autogenerate -m "description"

# Or from outside
docker exec newton-dashboard-api alembic revision --autogenerate -m "description"
```

### Running Migrations

Migrations auto-run on container startup via `run.sh`. To run manually:

```bash
docker compose exec runtime alembic upgrade head
```

### Migration Best Practices

- Always review auto-generated migrations before committing
- Add SQL schema comments to model files for reference
- Use descriptive migration messages: "add address column to api_keys"
- Test migrations by running `make runtime` and verifying `/health`
- For data migrations, write explicit `op.execute()` SQL statements

### Downgrade Safety

- Include working `downgrade()` functions in migrations
- Test downgrade path: `alembic downgrade -1`
- For destructive changes (drop column), consider a two-phase migration

### Production Rollback Strategy

See `docs/ROLLBACK.md` for the full rollback playbook covering:
- ECS circuit breaker (automatic rollback on health check failure)
- Alembic manual downgrade via ECS Exec
- Pre-migration RDS snapshots
- Two-phase destructive change policy
- Failed deploy runbook with common causes and fixes

## Query Patterns

### Service Layer Queries

```python
# Select with filter
result = await session.execute(
    select(ApiKey).where(
        ApiKey.user_id == user_id,
        ApiKey.is_active == True,
    )
)
keys = result.scalars().all()

# Select one or none
result = await session.execute(
    select(User).where(User.id == user_id)
)
user = result.scalar_one_or_none()

# Insert
new_key = ApiKey(user_id=user_id, name=name, api_key=generated_key)
session.add(new_key)
await session.commit()
await session.refresh(new_key)

# Update
key.name = new_name
key.updated_at = func.now()
await session.commit()

# Soft delete (deactivate)
key.is_active = False
await session.commit()
```

### Avoid Raw SQL

Always use SQLAlchemy query builders:

```python
# Good: Parameterized query
select(ApiKey).where(ApiKey.api_key == provided_key)

# Bad: Raw SQL with string interpolation
text(f"SELECT * FROM api_keys WHERE api_key = '{provided_key}'")
```

## Anti-patterns to Avoid

- Do not run migrations outside Docker
- Do not use raw SQL string interpolation
- Do not skip `await session.commit()` after writes
- Do not load entire tables without pagination
- Do not use synchronous SQLAlchemy in async code
- Do not forget to add indexes for frequently queried columns
- Do not use `String` type when `Text` is more appropriate for PostgreSQL
