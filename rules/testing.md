# Testing Guidelines

## Testing Strategy

### Test Pyramid

```
     /\
    /  \     Integration Tests
   /----\    (API endpoints, auth flows)
  /      \
 /--------\  Unit Tests
/          \ (Services, utilities, models)
```

## Test Infrastructure

Tests run inside Docker containers with dedicated test database and Redis:

| Service | Port | Purpose |
|---------|------|---------|
| test_db | 5434 | PostgreSQL test database |
| test_redis | 6381 | Redis test instance |

### Running Tests

```bash
# Run all tests in Docker
make test

# Run tests with coverage report
make coverage

# Run specific test file (inside container)
docker compose exec runtime pytest tests/test_file.py -v
```

## Test Structure

### Location

```
conftest.py                  # Root-level shared fixtures (redis, client, cleanup)
tests/
├── fixtures/                # Test data and credential fixtures
│   ├── client_with_login.py
│   └── passkey_credential.py
├── integration/             # Integration tests (real DB/Redis)
│   ├── api/v1/routes/       # Route-level integration tests
│   └── services/            # Service-level integration tests
└── unit/                    # Unit tests (mocked dependencies)
    ├── api/v1/routes/       # Route handler unit tests
    ├── dependencies/        # Auth dependency tests
    ├── services/            # Service unit tests
    └── utils/               # Utility function tests
```

### Naming Convention

```python
def test_<action>_<condition>_<expected_result>() -> None:
    """Docstring describing the test."""
    ...

# Examples
def test_create_api_key_with_valid_data_returns_key() -> None:
def test_verify_otp_with_expired_flow_returns_400() -> None:
def test_refresh_token_with_blacklisted_token_returns_401() -> None:
```

### Test Structure (Arrange-Act-Assert)

```python
async def test_create_api_key_returns_key(client):
    """Test that creating an API key returns the key details."""
    # Arrange
    await client.login()
    payload = {"name": "test-key", "address": "0x" + "ab" * 20}

    # Act
    response = await client.post("/v1/api_key", json=payload)

    # Assert
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "test-key"
    assert data["api_key"].startswith("gw_")
```

## Fixtures

### Core Fixtures (conftest.py at project root)

```python
@pytest.fixture
def redis(mocker):
    """Mocked async Redis client via create_autospec."""
    ...

@pytest.fixture
def client():
    """Test HTTP client wrapping ClientWithLogin."""
    ...

@pytest.fixture(scope="function", autouse=True)
async def clean_up():
    """Clean up database after each test using Base.metadata.sorted_tables."""
    yield
    # Reverse table order for FK-safe cleanup, then flushall Redis
    # Reset RedisClient singletons so the next test creates a fresh
    # connection on its own event loop (prevents loop-mismatch errors)
    ...
```

### Fixture Patterns

- `autouse=True` is used for cleanup, email mocking, reCAPTCHA mocking, and email validation
- Prefer function-scoped fixtures for test isolation
- Mock external services (Redis, email, reCAPTCHA) at the fixture level using `mocker` (pytest-mock)
- Use the test client's `login()` helper for authenticated requests

## Async Tests

All test functions that interact with the database or Redis should be async:

```python
import pytest

@pytest.mark.asyncio
async def test_get_user_returns_profile(client):
    await client.login()
    response = await client.get("/v1/user")
    assert response.status_code == 200
```

## Mocking External Services

### Redis

```python
@pytest.fixture
def redis(mocker):
    client = mocker.create_autospec(Redis, instance=True)
    client.set = mocker.AsyncMock(return_value=None)
    client.get = mocker.AsyncMock(return_value=None)
    client.exists = mocker.AsyncMock(return_value=0)
    mocker.patch.object(RedisClient, "get_instance", return_value=client)
    return client
```

### Email (SendGrid)

```python
@pytest.fixture(autouse=True)
def send_email(mocker):
    return mocker.patch(
        "app.core.clients.transport.email.sendgrid.SendGridTransportClient.send_email",
        AsyncMock(return_value=None),
    )
```

### reCAPTCHA

```python
@pytest.fixture(autouse=True)
def mock_recaptcha_verify(mocker):
    return mocker.patch(
        "app.core.clients.recaptcha.RecaptchaClient.verify",
        return_value=None,
    )
```

## Database Tests

- Tests use a separate PostgreSQL instance (port 5434)
- Alembic migrations run automatically before tests
- Each test gets a clean database state via the `clean_up` fixture
- Use `async_session` for direct database operations in tests

## What to Test

### Must Test

- Authentication flows (challenge, verify, token refresh, logout)
- Authorization (access control, API key permissions)
- CRUD operations for all resources
- Input validation edge cases
- Error responses and status codes

### Should Test

- Token expiration and blacklisting
- Factor verification state machine transitions
- Ethereum signature verification
- Rate limiting / throttling logic

## Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: pre-commit-update        # Auto-update hook versions
  - repo: pre-commit-hooks
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: detect-private-key
      - id: requirements-txt-fixer
  - repo: codespell                 # Spell checking
  - repo: ruff-pre-commit
    hooks:
      - id: ruff (--fix)
      - id: ruff-format
```

## Anti-patterns to Avoid

- Do not test implementation details (internal method calls)
- Do not use real external services in tests
- Do not share mutable state between tests
- Do not skip database cleanup between tests
- Do not hardcode test data that depends on auto-generated IDs
