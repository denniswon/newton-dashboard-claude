# Lessons Learned

Living document. When a mistake recurs or a non-obvious pattern causes confusion, add an entry.

## Format

Each entry:
- **What happened**: Brief description
- **Why**: Root cause
- **Rule**: What to do differently

---

## Entries

### 1. Migration squash broke staging/prod deployments

- **What happened**: Squashed 24 Alembic migrations into `0001_initial_schema.py`. Staging and prod deployments broke because `alembic_version` still referenced the old revision `add_address_to_api_keys`, which no longer exists in the codebase.
- **Why**: Migration squashes require updating `alembic_version` in existing databases via `alembic stamp`. No stamp step was included and no deployment runbook was created.
- **Rule**: When squashing migrations, either (a) add recovery logic to the migration runner (now done — `run.sh` and CI auto-stamp on "Can't locate revision"), or (b) include a manual `alembic stamp` step in the deployment runbook. Always verify CI deploys pass after a squash.
