# Python Code Style Guidelines

## Formatting and Linting

### ruff Configuration

Use ruff for both formatting and linting:

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py313"

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false

[tool.ruff.lint]
select = [
    "E4", "E7", "E9",  # pycodestyle errors
    "F",                # pyflakes
    "I",                # isort
]
ignore = [
    "E712",  # comparison to None
]

[tool.ruff.lint.isort]
known-first-party = ["app"]
force-single-line = true
```

### Commands

```bash
# Format code
ruff format .

# Lint code
ruff check .

# Lint and auto-fix
ruff check --fix .
```

---

## Type Hints

### Required for All Public Functions

```python
from typing import Optional
from uuid import UUID

async def create_api_key(
    self,
    user_id: UUID,
    name: str,
    address: bytes,
    permissions: list[str] | None = None,
) -> ApiKey:
    """Create a new API key for the user."""
    ...
```

### Use Modern Type Syntax

```python
# Good: Python 3.10+ syntax
def get_users() -> list[User]:
    ...

def get_factor(user_id: UUID) -> UserFactor | None:
    ...

def get_permissions() -> dict[str, list[str]]:
    ...

# Avoid: Legacy typing module for built-in generics
from typing import List, Dict  # Only when needed for older patterns
```

### Type Aliases for Complex Types

```python
from typing import TypeAlias

PermissionSet: TypeAlias = list[str]
FactorMeta: TypeAlias = dict[str, str | int | bool]
```

---

## Docstrings

### Function Docstrings

Use Google-style docstrings:

```python
async def verify_factor(
    self,
    flow_id: UUID,
    challenge_response: str,
) -> UserFactor:
    """Verify a factor challenge response.

    Args:
        flow_id: The verification flow identifier
        challenge_response: The user's response to the challenge

    Returns:
        The verified user factor

    Raises:
        ValidationError: If the response is incorrect or flow expired
    """
    ...
```

### Class Docstrings

```python
class TokenManager:
    """Manages JWT token lifecycle.

    Handles creation, validation, rotation, and blacklisting
    of access, refresh, and factor tokens using RS256 signing.
    """
    ...
```

---

## Naming Conventions

### Variables and Functions

Use snake_case:

```python
user_factor = await get_factor(user_id)
api_key_service = ApiKeyService()

async def get_active_factors(user_id: UUID) -> list[UserFactor]:
    ...
```

### Classes

Use PascalCase:

```python
class UserFactorService:
    ...

class ApiKeyResponse(BaseModel):
    ...

class FactorType(StrEnum):
    EMAIL = "email"
    PASSKEY = "passkey"
    SIWE = "siwe"
```

### Constants

Use SCREAMING_SNAKE_CASE:

```python
MAX_VERIFICATION_ATTEMPTS = 3
FLOW_TTL_SECONDS = 900
TOKEN_EXPIRY_DAYS = 30
```

### Private Members

Prefix with underscore:

```python
class TokenManager:
    def __init__(self):
        self._private_key = load_private_key()

    def _sign_token(self, payload: dict) -> str:
        """Sign a token (internal method)."""
        ...
```

---

## Exception Handling

### Raise Specific Exceptions

```python
# Good: Specific exception with context
if not user:
    raise NotFound(f"user {user_id} not found")

if factor.attempts >= MAX_VERIFICATION_ATTEMPTS:
    raise ValidationError("maximum verification attempts exceeded")

# Bad: Generic exception
if not user:
    raise Exception("not found")
```

### Custom Exceptions

The project uses custom exceptions mapped to HTTP status codes via handlers in `app/middleware/exceptions.py`:

```python
class ValidationError(ValueError):    # -> 400
class AuthorizationError(ValueError):  # -> 403
class NotFound(Exception):             # -> 404
class BlockchainError(Exception):      # -> 502
```

Additionally, `RedisError` and `sqlalchemy.exc.OperationalError` map to 503, and unhandled `Exception` returns 500.

---

## Imports

### Import Order

```python
# Standard library
import datetime
from typing import Optional
from uuid import UUID

# Third-party
from fastapi import APIRouter
from fastapi import Depends
from sqlalchemy import select
from sqlalchemy.orm import Mapped

# Local application
from app.core.exceptions import NotFound
from app.db.models.user import User
from app.services.user import UserService
```

### Import Style

```python
# Good: Single-line imports (enforced by ruff isort force-single-line)
from sqlalchemy import ForeignKey
from sqlalchemy import Index
from sqlalchemy import Text

# Avoid: Multi-line grouped imports
from sqlalchemy import ForeignKey, Index, Text

# Avoid: Wildcard imports
from sqlalchemy import *
```

---

## Comments

### When to Comment

- Explain **why**, not **what**
- Document non-obvious business logic
- Note workarounds or temporary solutions

```python
# Good: Explains why
# Retry key generation on collision since NaCl random
# can theoretically produce duplicates
for _ in range(3):
    try:
        return await self._create_key(user_id)
    except IntegrityError:
        continue

# Bad: States the obvious
# Loop 3 times
for _ in range(3):
```

### What Not to Comment

- Obvious code behavior
- Function signatures (use docstrings instead)
- Commented-out code (delete it)

---

## Style Rules

### No Emojis or Exclamation Marks

```python
# Good
logger.info("user created successfully")
raise ValueError("invalid email format")

# Bad
logger.info("User created! 🎉")
raise ValueError("Invalid email!!!")
```

### f-strings Over format()

```python
# Good
key_name = f"gw_{token_hex}"
log_msg = f"user {user_id} logged in via {factor_type}"

# Avoid
key_name = "gw_{}".format(token_hex)
```

### Guard Clauses

```python
# Good: Early return
async def get_user(self, user_id: UUID) -> User:
    user = await self._fetch_user(user_id)
    if not user:
        raise NotFound(f"user {user_id} not found")
    if not user.is_active:
        raise ValidationError("user account is deactivated")
    return user

# Avoid: Nested conditions
async def get_user(self, user_id: UUID) -> User:
    user = await self._fetch_user(user_id)
    if user:
        if user.is_active:
            return user
        else:
            raise ValidationError("user account is deactivated")
    else:
        raise NotFound(f"user {user_id} not found")
```

---

## Anti-patterns to Avoid

- Do not use `type: ignore` without justification
- Do not skip type hints for public functions
- Do not use mutable default arguments
- Do not use bare `except:` clauses
- Do not use wildcard imports
- Do not mix string formatting styles
- Do not leave commented-out code
- Do not use print() for logging
