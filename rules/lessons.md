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

### 4. Cloudflare Free plan does not cover multi-level subdomains

- **What happened**: Created proxied CNAME records for `dashboard.api.newton.xyz` and `dashboard.api.stagef.newton.xyz` in the `newton.xyz` Cloudflare zone (Free plan). SSL handshake failed because Universal SSL only covers `*.newton.xyz` (one level), not `*.api.newton.xyz`.
- **Why**: Cloudflare's free Universal SSL certificate covers the apex and one wildcard level. Multi-level subdomains like `a.b.domain.xyz` require Advanced Certificate Manager (paid) or a zone plan upgrade.
- **Rule**: For backend API subdomains with multi-level names (e.g., `dashboard.api.newton.xyz`), use DNS-only mode (grey cloud) and let the origin (ALB) handle SSL via ACM certificates. This avoids Cloudflare edge SSL limitations and is appropriate for API backends that don't need CDN/WAF.

### 5. Database migration ownership is split across repos

- **What happened**: Confusion about where `chain_id` column migrations would run for `policy_client_secret` and `encrypted_data_refs` tables.
- **Why**: The `newton_gateway` database is shared between dashboard-api and the gateway. Each repo owns different tables and uses different migration tools. The migration execution happens in different CI pipelines.
- **Rule**: Migration ownership and execution:
  - **Dashboard tables** (`user`, `user_factor`, `user_key`, `cli_session`, `project`): Alembic in `newton-dashboard-api`. Runs automatically on container startup via `run.sh` (`alembic upgrade head`).
  - **Gateway tables** (`api_keys`, `policy_client_secret`, `encrypted_data_refs`): sqlx in `newton-prover-avs`. Runs automatically during `newton-prover-avs-deploy` CDK deploy via `scripts/run-migration.sh` as a one-shot ECS Fargate task.
  - **Local dev**: Gateway tables bootstrapped by `scripts/init_gateway_tables.sql` (runs at Docker init, not incrementally — recreate volumes for schema changes: `make clean && make runtime`).
  - **Deploy ordering for cross-repo schema changes**: Gateway migration must land first (deploy newton-prover-avs), then dashboard-api code that references new columns can deploy. Otherwise dashboard queries will fail on missing columns.

### 6. Exception types in blockchain layer masked RPC failures as authorization errors

- **What happened**: `POST /v1/policy-client-owner` returned `403 "caller does not own this policy client"` on Base Sepolia even though the user owned the policy client. The frontend engineer suspected the backend was querying the wrong chain, but the code correctly passed `chain_id`. The real issue was that `RPC__BASE_SEPOLIA_URL` was not configured in staging or prod, and the resulting `AuthorizationError` from the provider was indistinguishable from a genuine ownership failure.
- **Why**: `BaseContract._call` and `provider.get_async_web3` raised `AuthorizationError` for all failure modes — missing RPC URL, network timeout, contract not found, and actual contract reverts. The route handler caught `AuthorizationError` as 403 and `BlockchainError` as 502, but infrastructure failures never reached the `BlockchainError` path. Additionally, the CDK stack only injected `RPC__ETHEREUM_MAINNET_URL` and `RPC__SEPOLIA_URL` into ECS — Base chain URLs were never added when Base chain support was introduced.
- **Rule**:
  - **Exception semantics matter**: `AuthorizationError` = business logic denial (contract revert, ownership mismatch). `BlockchainError` = infrastructure failure (missing RPC URL, network error, provider unreachable). Never conflate the two — callers rely on the distinction to choose 403 vs 502.
  - **New chain checklist**: When adding support for a new chain, update all three layers: (1) `app/core/config/secrets.py` (config field), (2) AWS Secrets Manager (both stagef and prod), (3) `deploy/.../newton_dashboard_api_stack.py` (ECS secret injection). Missing any one of these causes silent failures.
  - **Test with the actual chain**: If an endpoint accepts `chain_id`, verify it works for every supported chain in staging, not just the default (Sepolia).

### 7. Sync TestClient causes event loop conflicts with async Redis in integration tests

- **What happened**: Integration tests for `POST /v1/auth/token` used the sync `TestClient` (`ClientWithLogin`). Tests that called endpoints internally using Redis (refresh, blacklist check) failed with `RuntimeError: Event loop is closed` and `Task got Future attached to a different loop`.
- **Why**: `TestClient` runs requests in a background thread with its own event loop. When the endpoint handler calls async Redis (`RedisClient.get_instance()`), it tries to reuse a singleton Redis connection from a different loop. The `clean_up` fixture resets singletons between tests, but within a single test the loop mismatch persists.
- **Rule**: Integration tests that hit endpoints which internally use async Redis or other async singletons MUST use `httpx.AsyncClient` with `ASGITransport(app=app)` instead of the sync `TestClient`. This keeps everything on the same event loop. Pattern:
  ```python
  async with httpx.AsyncClient(
      transport=ASGITransport(app=app), base_url="http://test"
  ) as client:
      response = await client.post("/endpoint", ...)
  ```
  See: `tests/integration/api/v1/routes/authn/session/test_refresh.py` for the established pattern.

### 8. New auth flows should not require changes to existing route handlers

- **What happened**: When designing agent authentication, the initial instinct was to add API key validation to each protected endpoint. Instead, the API key → JWT exchange pattern was chosen: agents exchange their `gw_` key for a standard JWT once, then use that JWT everywhere.
- **Why**: All existing routes use `get_current_user_id` which extracts `sub` from a JWT. By converting the API key to a JWT upfront, zero route handlers needed modification. The only endpoint that checks `verified_factors` content is `POST /v1/authn/cli-token` (requires >= 1 factor), which agents don't need.
- **Rule**: When adding a new auth mechanism, prefer converting to the existing token format (JWT) at the boundary rather than threading a new auth type through every route. Check `verified_factors` usage across all routes — if only one or two endpoints inspect it, the new flow can set it to `[]` without breaking anything.

### 9. Design specs should be updated to match actual implementation before merging

- **What happened**: The design spec referenced `get_by_api_key` (already existed) instead of the new `get_active_api_key`, and `ttl_seconds` instead of `ttl: Optional[timedelta]`. These were caught during docs-sync but could have confused future readers.
- **Why**: Spec was written before implementation. Implementation made different (better) naming choices but the spec wasn't updated until the docs-sync pass.
- **Rule**: After implementing from a spec, diff the spec against what was actually built. Update method names, parameter names, and descriptions to match reality. The spec becomes a historical reference — it should describe what was built, not what was planned.
