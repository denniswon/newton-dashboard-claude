# Security Guidelines

## Authentication

### JWT Token Security

- RS256 asymmetric signing (RSA private key signs, public key verifies)
- JWKS endpoint exposes public keys for token verification
- Tokens include `verified_factors` array for multi-factor context
- DPoP (Demonstration of Proof-of-Possession) support for token binding

### Token Blacklisting

Logout invalidates tokens via Redis blacklist:

```python
# On logout: blacklist the token's JTI
await redis.set(f"blacklist:{jti}", "1", ex=remaining_ttl)

# On each request: check blacklist before accepting token
is_blacklisted = await redis.get(f"blacklist:{jti}")
```

### Factor Verification Security

- Verification flows have max 3 attempts before lockout
- Flows expire after 15 minutes (Redis TTL)
- OTP throttling prevents brute force (Redis-based rate limiting)
- reCAPTCHA required for challenge creation

## API Key Management

### Key Generation

- API keys use NaCl random (32 bytes) with `gw_` prefix
- Format: `gw_{hex(random_bytes)}`
- Keys are stored as-is (not hashed) for bearer token lookup
- Collision retry on generation

### Key Security

- API keys authenticate via `X-API-Key` header
- Keys can be rotated (generates new value, invalidates old)
- Keys can be deactivated (soft delete via `is_active` flag)
- Permissions stored as JSONB array for granular access control

## Secrets Management

### Environment Variables

- All secrets loaded via `.env` file (never committed to git)
- Use pydantic-settings for typed secret loading with validation
- RSA keys stored as PEM strings in environment variables

### What Must Be in .env

- Database credentials (`DATABASE__PASSWORD`)
- Redis connection (`REDIS__HOST`, `REDIS__PORT`)
- RSA key pair (`RSA_PRIVATE_KEY`, `RSA_PUBLIC_KEY`)
- Email API keys (`SENDGRID__*`, `POSTMARK__*`)
- RPC URLs (`RPC__ETHEREUM_MAINNET_URL`, `RPC__SEPOLIA_URL`, optional: `RPC__BASE_URL`, `RPC__BASE_SEPOLIA_URL`, `RPC__ANVIL_URL`)
- WebAuthn config (`WEBAUTHN__RP_ID`, `WEBAUTHN__ORIGIN`)

### Never Commit

- `.env` files
- Private keys (RSA, Ethereum)
- API keys or tokens
- Database connection strings with credentials

## Input Validation

### Pydantic Models

All request bodies are validated via Pydantic models:

```python
class CreateApiKeyRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    address: str = Field(..., pattern=r"^0x[a-fA-F0-9]{40}$")
    permissions: list[str] = Field(default_factory=list)
```

### Ethereum Address Validation

- Addresses must be 20 bytes (validated via DB check constraint)
- EthereumAddress custom type handles hex string to bytes conversion
- Signature verification uses eth_account library with EIP-191 encoding

### Email Validation

- Normalize: lowercase, trim whitespace
- Validate via SendGrid email validation API
- Risk scoring for disposable/suspicious addresses
- Exempt patterns for internal test emails

## CORS Configuration

Hardcoded allowed origins (no wildcards in production):

- `dashboard.newt.foundation`
- `dashboard.stagef.newt.foundation`
- Regex: `^https://newton-dashboard-.*-magiclabs\.vercel\.app$`
- `localhost:3000` only in non-production

## SQL Injection Prevention

- SQLAlchemy parameterized queries exclusively
- Never use raw SQL string interpolation
- Use `select()`, `insert()`, `update()` query builders

## Error Response Security

### Do Not Leak Internal Details

```python
# Good: Generic error for external consumption
raise HTTPException(status_code=500, detail="Internal server error")

# Bad: Leaking implementation details
raise HTTPException(status_code=500, detail=f"Database error: {str(e)}")
```

### Distinguish Client vs Server Errors

- 400: Validation failures (safe to include field-level details)
- 401: Authentication required (no details about why)
- 403: Authorization denied (no details about what permissions are needed)
- 404: Resource not found (no details about whether it exists but is unauthorized)
- 502: Blockchain/RPC errors (`BlockchainError` — safe to include contract address context)
- 503: Service dependency unavailable (`RedisError`, `OperationalError`)
- 500: Internal error (never include stack traces in response)

## Container Security

### Dockerfile Best Practices

- Use specific Python base image version
- Install only production dependencies in runtime image
- Separate build and runtime stages
- Do not copy `.env` into image

## Security Checklist

Before deploying:

- [ ] No secrets hardcoded in code
- [ ] No `print()` statements in production code (use `logger.debug()` instead)
- [ ] `.env` is in `.gitignore`
- [ ] All request bodies validated via Pydantic
- [ ] SQL queries use parameterized statements
- [ ] CORS origins are explicit (no wildcards)
- [ ] JWT tokens verified on all authenticated routes
- [ ] Token blacklist checked on every authenticated request
- [ ] Error responses do not leak internal details
- [ ] Ethereum signatures verified before trusting wallet addresses
- [ ] API keys checked for `is_active` status on every request
