# Dashboard API Agent Guide

Rules and heuristics specific to AI agents working in this FastAPI codebase.

## Key Files

| File | Purpose |
|------|---------|
| `app/main.py` | FastAPI app entry point, middleware setup |
| `app/api/v1/routes/` | API route handlers |
| `app/services/` | Business logic layer |
| `app/db/models/` | SQLAlchemy models |
| `app/core/config/` | Environment configuration |
| `app/core/blockchain/` | Web3 providers, ABI registry, contract wrappers |
| `alembic/` | Database migration scripts |
| `scripts/sync_abis.py` | Sync ABIs from newton-prover-avs contracts |

## Layer Architecture

When adding a feature, work through layers in this order:

1. **Model** (`app/db/models/`) — define or update SQLAlchemy models
2. **Service** (`app/services/`) — implement business logic
3. **Route** (`app/api/v1/routes/`) — expose via FastAPI endpoint
4. **Schema** (inline Pydantic or `app/schemas/`) — request/response validation
5. **Migration** — generate Alembic migration **inside Docker** (never locally)
6. **Tests** (`tests/`) — cover the new path

## Critical Workflows

### Database Migrations

ALWAYS run inside Docker — never in local venv:
```bash
docker compose exec runtime bash
alembic revision --autogenerate -m "description"
alembic upgrade head
```

### ABI Sync

When newton-prover-avs contracts change:
```bash
python scripts/sync_abis.py
```

This updates `app/core/blockchain/abis/` — do not edit those files manually.

### Shared Database

This API shares the `newton_gateway` database with newton-prover-avs gateway. Alembic manages dashboard tables only. Gateway tables are bootstrapped by `scripts/init_gateway_tables.sql` (auto-generated, do not edit).

## Agent Behavior Directives

### Mid-Task Replanning

If tests fail unexpectedly or you realize the data model needs to change:
1. STOP implementing
2. Explain what you discovered
3. Propose the model change and its migration impact
4. Get confirmation before continuing

### Verify Before Claiming

All output — code, docs, comments, CLI instructions, config examples — must be verified against the actual codebase before submission. Do NOT assume or guess:

- **Makefile targets**: Grep the Makefile to confirm a target exists and what parameters it accepts before documenting it
- **CLI flags and subcommands**: Read the CLI parser to confirm exact flag names and required subcommands
- **File paths and naming**: Use Glob/ls to confirm files exist where you say they do
- **Config fields and defaults**: Read the config definition and its defaults, not just example files
- **Prerequisites and dependencies**: Verify tools are actually used (grep for invocations) before listing as prerequisites

If you are uncertain about any claim, ASK the user rather than guessing. A missing fact is better than a false statement.

### Quality Bar

Treat every change as if it goes to production:
- No `# TODO` or `pass` in submitted code
- Type hints on all function signatures
- Pydantic validation on all API inputs
- Match existing patterns (e.g., service injection via `Depends`)
- Error responses must not leak internal details

### Autonomous Bug Fixing

When you encounter a test failure:
- **Fix autonomously** if: the fix is in code you just wrote, OR it's a clear import/type error
- **Ask first** if: the fix involves auth logic, token validation, blockchain interactions, or migration changes
- **Always explain** what failed and what you changed

### Lessons Learned

When you make a mistake that could recur, record it in `.claude/rules/lessons.md` with:
- What went wrong
- Why it went wrong
- The prevention rule going forward
