# Architecture Guidelines

## System Overview

Newton Dashboard API is a FastAPI-based authentication and management service for the Newton platform. It provides multi-factor authentication, API key management, and user management with Ethereum/Web3 integration.

## Component Architecture

```
Client (Dashboard / CLI)
    |
    v
FastAPI Application (app/main.py)
    |
    +-- Middleware Layer
    |     +-- CORS (app/middleware/cors.py)
    |     +-- Request Logging (app/middleware/logging.py)
    |     +-- Exception Handling (app/middleware/exceptions.py)
    |     +-- Request Context (app/middleware/request_context.py)
    |
    +-- API Routes (app/api/v1/routes/)
    |     +-- Authentication (authn/)
    |     |     +-- Factor Challenges (factor/email/, factor/passkey/, factor/siwe/)
    |     |     +-- Factor-by-ID Challenge (factor/_id/challenge.py)
    |     |     +-- Factor Verification (factor/verify.py)
    |     |     +-- Session Management (session/refresh.py, session/logout/)
    |     |     +-- Token Exchange (token.py)
    |     |     +-- CLI Token (cli_token.py)
    |     +-- User Factor Management (user_factor/)
    |     |     +-- Factor Challenges (email/, passkey/)
    |     |     +-- Factor-by-ID Challenge (_id/challenge.py)
    |     |     +-- Factor Overwrite (_id/overwrite/verify.py)
    |     |     +-- Factor Verification (verify.py)
    |     |     +-- Factor List (list.py)
    |     +-- User Management (user.py)
    |     +-- API Keys (api_key.py)
    |     +-- User Keys (user_key.py)
    |     +-- Policy Client Owner (policy_client_owner.py)
    |     +-- Policy Client Secrets (policy_client_secret.py)
    |     +-- KYC (kyc.py)
    |     +-- JWKS (certs/jwks.py)
    |
    +-- Dependencies (app/dependencies/auth.py)
    |     +-- JWT Token Verification
    |     +-- API Key Authentication
    |
    +-- Services (app/services/)
    |     +-- Token Management (tokens/)
    |     +-- Factor Verification Flows (authn/factor_verify_flow/)
    |     +-- Email Senders (authn/senders/)
    |     +-- Email Builders (builders/)
    |     +-- DPoP Validation (dpop/)
    |     +-- CRUD Services (user.py, api_key.py, project.py, etc.)
    |
    +-- Database Layer (app/db/)
    |     +-- SQLAlchemy Models (models/)
    |     +-- Async Session (session.py)
    |     +-- Custom Types (types.py)
    |
    +-- Blockchain Layer (app/core/blockchain/)
    |     +-- Chain Config (config.py)
    |     +-- AsyncWeb3 Provider Singletons (provider.py)
    |     +-- ABI + Deployment Registry (registry.py)
    |     +-- Base Contract Wrapper (base_contract.py)
    |     +-- Type-Safe Contracts (contracts/)
    |     +-- Synced ABIs (abi/)
    |     +-- Deployment Addresses (deployments/)
    |
    +-- Core Utilities (app/core/)
    |     +-- Structured Logging (logging.py)
    |     +-- Geolocation (geolocation.py)
    |     +-- User Agent Parsing (user_agent.py)
    |
    +-- Utilities (app/utils/)
    |     +-- JWT Bearer Auth (jwt_bearer.py)
    |     +-- API Key Generation (api_key.py)
    |     +-- User Key Generation (user_key.py)
    |     +-- Ethereum Signature Verification (signature.py)
    |     +-- On-Chain Ownership Verification (onchain_verification.py)
    |     +-- Request Context (context.py)
    |     +-- AAGUID Lookup (aaguid.py)
    |     +-- Constants (constants.py)
    |     +-- String Helpers (string.py)
    |     +-- Time/Email Helpers
    |
    +-- External Services
          +-- PostgreSQL (async via asyncpg)
          +-- Redis (token blacklist, flow state)
          +-- SendGrid/Postmark (email)
          +-- Ethereum RPC (multichain via AsyncWeb3)
```

## Core Components

### Authentication System

Multi-factor authentication with a pluggable factor verification framework:

- **FactorVerifyFlow** - Base class with metaclass registry pattern for automatic subclass registration
- **EmailOTPVerifyFlow** - One-time password via email
- **EmailLinkVerifyFlow** - Magic link verification
- **PasskeyVerifyFlow** - WebAuthn/FIDO2 biometric/security key
- **SIWEVerifyFlow** - Sign-in with Ethereum

Factor verification state is stored in Redis with TTL (15 minutes default).

### Token Management

- **TokenManager** - JWT creation/validation with RS256 signing
- **TokenStore** - Redis-based token blacklist for secure logout
- **DPoP** - Demonstration of Proof-of-Possession support
- JWKS endpoint for public key distribution

Token types: ACCESS_TOKEN, REFRESH_TOKEN, ID_TOKEN, FACTOR_TOKEN, FACTOR_VERIFICATION_TOKEN.

### Service Layer

Services encapsulate business logic and database operations:

- Services are instantiated per-request via FastAPI `Depends()`
- All database operations use async SQLAlchemy sessions
- Services handle validation, CRUD, and business rules

### Database Models

Both dashboard and gateway share a single `newton_gateway` database with split migration ownership.

**Dashboard-owned (Alembic):**

| Model | Purpose | Key Fields |
|-------|---------|------------|
| User | User accounts | id, is_active, public_address |
| UserFactor | Auth factors (email/passkey/SIWE) | user_id, type, value, meta |
| UserKey | User key pairs | user_id, public_key, secret_key |
| CLISession | CLI authentication sessions | session_id, user_id, factor_id |
| Project | Off-chain project metadata (names) | user_id, policy_client_address, chain_id, name |

**Gateway-owned (sqlx in newton-prover-avs):**

| Model | Purpose | Key Fields |
|-------|---------|------------|
| ApiKey | API bearer tokens | id, user_id, address, api_key, permissions |
| PolicyClientSecret | Policy client secrets (read/delete, writes via Gateway) | chain_id, policy_client_address, policy_data_address |

### Custom Column Types

- **EthereumAddress** - Stores 20-byte Ethereum addresses as BYTEA, converts to/from "0x..." hex strings
- **Secrets** - Stores secrets as BYTEA, converts to/from UTF-8 strings (encoding, not encryption)

## Design Principles

### Async-First

All I/O operations are async. Database queries use `async_session`, Redis operations use async client, HTTP calls use httpx/aiohttp.

### Stateless Application

Application state is externalized to PostgreSQL (persistent data) and Redis (ephemeral state like verification flows and token blacklist). This enables horizontal scaling.

### Dependency Injection

FastAPI's `Depends()` for service injection, database sessions, and authentication:

```python
async def get_api_keys(
    user_id: UUID = Depends(get_current_user_id),
    api_key_service: ApiKeyService = Depends(ApiKeyService),
):
```

### Input Validation at Boundaries

Pydantic models validate all incoming request data. Internal service-to-service calls trust validated data.

### Configuration Centralization

Environment-specific mappings (e.g., chain ID to RPC URL) belong in the Pydantic config models (`app/core/config/secrets.py`), not hardcoded in utility modules. Utility code should call config model methods rather than maintaining its own lookup tables.

### Idempotent Operations

UUIDs for primary keys, upserts where applicable, and unique constraints to prevent duplicates.
