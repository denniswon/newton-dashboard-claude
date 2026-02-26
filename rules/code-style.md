# Code Style Guidelines

## Formatting

- Use `ruff` for both formatting and linting
- Maximum line length: 100 characters
- Run before committing: `ruff format . && ruff check --fix .`

## Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Functions/Methods | snake_case | `get_user_by_id` |
| Variables | snake_case | `user_factor` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRIES` |
| Classes | PascalCase | `ApiKeyService` |
| Enums | PascalCase | `FactorType` |
| Enum Members | SCREAMING_SNAKE_CASE or lowercase | `FactorType.EMAIL` |
| Modules/Files | snake_case | `token_manager.py` |
| Private Members | Leading underscore | `_create_session` |

### Descriptive Names

Use descriptive names that convey intent:

```python
# Good: Clear and descriptive
verified_factor = await factor_service.get_active_factor(user_id, factor_type)
is_token_blacklisted = await token_store.is_blacklisted(token_jti)

# Bad: Abbreviated or unclear
vf = await fs.get(uid, ft)
bl = await ts.check(jti)
```

## Comments

### General Rules

- No emojis anywhere in comments
- No exclamation marks unless semantically necessary
- Write in neutral, professional tone
- Be concise; avoid filler words

### What to Comment

- **Why** something is done, not **what** is being done
- Non-obvious business logic and domain rules
- Edge cases and handling rationale
- Performance considerations for non-obvious optimizations

### What NOT to Comment

- Obvious code behavior
- Prompt-specific context or AI generation notes
- Changelog-style entries
- Commented-out code (delete it)
- Restating the function signature

### Examples

```python
# Good: explains why
# Skip internal magic.link users from email validation
# as they bypass standard verification
if email.endswith("@magic.link"):
    return True

# Bad: restates the obvious
# Check if email ends with magic.link
if email.endswith("@magic.link"):
    return True
```

### TODO Comments

Include clear description of the task:

```python
# TODO: implement rate limiting per API key
# TODO(#123): migrate to Postmark for all transactional emails
```

## Documentation

### Technical Documentation Files

When writing technical documentation:

**Current State Assumption**

- Documentation describes the current implementation by default
- Do not mark features as "Implemented", "Done", or "Complete" - this is implied
- Avoid phrases like "fully implemented", "properly integrated", "correctly implemented"

**Future Work**

- Use **TODO** prefix for planned but unimplemented features
- Be explicit about what remains to be done

**Tables**

- Remove columns that contain redundant information
- Avoid status columns when status is uniform across all rows
- Keep tables focused on distinguishing information

## Error Messages

### Format

- State what failed
- Include relevant identifiers
- Be specific
- Do not include sensitive data (keys, passwords, tokens)

```python
# Good: Specific with context
raise ValidationError(f"API key {api_key_id} has expired")
raise NotFound(f"policy client not found for address {address}")

# Bad: Vague
raise ValidationError("invalid")
raise NotFound("not found")
```

## Imports

### Organization

Group imports in order (enforced by ruff isort):

1. Standard library
2. Third-party packages
3. Local application modules

```python
import datetime
from typing import Optional
from uuid import UUID

from fastapi import APIRouter
from fastapi import Depends
from sqlalchemy import select

from app.db.models.user import User
from app.services.user import UserService
```

### Avoid Glob Imports

```python
# Good: Explicit imports
from sqlalchemy import ForeignKey, Index, Text

# Avoid: Wildcard imports
from sqlalchemy import *
```

## File Organization

### Python Files

```python
# Order within a file:
# 1. Module docstring (if needed)
# 2. Imports (stdlib, third-party, local)
# 3. Constants
# 4. Type aliases
# 5. Class/function definitions
# 6. Main execution (if applicable)
```

### Route Files

```python
# Order within a route file:
# 1. Imports
# 2. Router instantiation
# 3. Pydantic request/response models
# 4. Route handlers (CRUD order: create, read, update, delete)
```

## Anti-patterns to Avoid

- Do not use bare `except:` clauses
- Do not use mutable default arguments
- Do not use `type: ignore` without justification
- Do not leave commented-out code
- Do not use print() for logging (use the logging module)
- Do not mix string formatting styles (prefer f-strings)
