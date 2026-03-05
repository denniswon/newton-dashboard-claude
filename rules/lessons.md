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

### 2. Cross-repo dependency changes require impact analysis before coding

- **What happened**: newton-prover-avs gateway migrated from per-chain to multichain (single gateway serves N chains) and changed domain from `gateway-avs.*.newt.foundation` to `gateway.*.newton.xyz`. Dashboard-api had hardcoded gateway URLs in E2E scripts and docs that became stale.
- **Why**: Cross-repo changes (especially domain migrations and architectural shifts) can silently break scripts, docs, CORS origins, CI configs, and CDK stacks in dependent repos. The impact is spread across many files and categories.
- **Rule**: When a dependency repo makes infrastructure/domain/architectural changes:
  1. **Review all PRs** before coding — understand the full scope (domain, routing, API, DB schema).
  2. **Grep exhaustively** for old domain strings, URLs, and affected patterns across all file types (`.py`, `.md`, `.yml`, `.sh`, `.json`).
  3. **Categorize changes** into buckets: clearly-needed-now vs depends-on-decisions. Get confirmation before mixing buckets.
  4. **Check shared DB schema** — if the dependency repo ran new migrations, `init_gateway_tables.sql` may need regeneration.
  5. **Update E2E scripts and docs first** — hardcoded URLs break silently and block manual testing.

### 3. Gateway is now multichain — one endpoint per network, not per chain

- **What happened**: As of PR #411/#415/#416, the newton-prover-avs gateway serves all chains in a network from a single endpoint. URL scheme changed from per-chain (`gateway-avs.{env}.{chain}.newt.foundation`) to per-network (`gateway.{env}.{network}.newton.xyz`).
- **Why**: Multichain gateway routes by `chainId` in the RPC request payload. No separate gateway instances per chain.
- **Rule**: Gateway URL mapping is now:
  - Testnet (Sepolia + Base Sepolia): `gateway.{env?}.testnet.newton.xyz`
  - Mainnet (Ethereum + Base): `gateway.{env?}.newton.xyz`
  - Staging prefix: `stagef.` after `gateway.`
  - Dashboard-api does NOT proxy through gateway — it talks directly to chain RPCs for on-chain reads. Gateway multichain changes are transparent to dashboard's blockchain layer.
