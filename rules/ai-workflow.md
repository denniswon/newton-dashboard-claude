# AI Workflow Guidelines

## CLAUDE.md Structure

### Root CLAUDE.md

Keep the root `CLAUDE.md` concise:

- Project overview and purpose
- Quick command reference (make targets)
- Key architecture notes
- Links to modular rules

### Modular Rules

Detailed guidelines in `.claude/rules/`:

- `architecture.md` - System architecture and components
- `code-style.md` - Code formatting and conventions
- `python-style.md` - Python-specific style rules
- `testing.md` - Test patterns and infrastructure
- `performance.md` - Async patterns and optimization
- `security.md` - Auth, secrets, and input validation
- `ci-cd.md` - Docker workflow and deployment
- `database.md` - SQLAlchemy and Alembic patterns

## Workflow Patterns

### Explore, Plan, Code, Commit

For changes to this project, follow this sequence:

1. **Explore**: Understand the current state
   - "Show me the API key service and route"
   - "What models does the auth flow use?"

2. **Plan**: Create a clear implementation plan
   - "Create a plan for adding rate limiting to API keys"
   - Outline changes to routes, services, and models before implementing

3. **Code**: Implement the solution
   - Make incremental changes
   - Run `make test` after significant changes

4. **Commit**: Create atomic commits
   - Use conventional commit format
   - One logical change per commit

### FastAPI-Specific Workflow

```
1. Read existing route/service/model code
2. Understand request/response schemas
3. Plan changes across layers (route → service → model)
4. Implement changes
5. Run tests (make test)
6. Generate migration if models changed (inside Docker)
7. Test migration (make runtime, verify /health)
```

## Code Generation Guidelines

### Match Existing Patterns

Before generating code:

1. Read related routes, services, and models
2. Note naming conventions and import style
3. Follow established patterns (e.g., service injection via Depends)
4. Use consistent typing and Pydantic schemas

### Complete Code

Generate complete, runnable code:

```python
# Good: Complete route handler
@router.post("")
async def create_api_key(
    request: CreateApiKeyRequest,
    user_id: UUID = Depends(get_current_user_id),
    api_key_service: ApiKeyService = Depends(ApiKeyService),
) -> ApiKeyResponse:
    return await api_key_service.create(user_id, request)

# Bad: Placeholders
@router.post("")
async def create_api_key(...):
    # TODO: implement
    pass
```

### Layer Consistency

When adding a new feature, always create all necessary layers:

```
Step 1: Add/update database model in app/db/models/
Step 2: Add/update service in app/services/
Step 3: Add/update route in app/api/v1/routes/
Step 4: Add/update Pydantic schemas (inline or in app/schemas/)
Step 5: Generate Alembic migration (inside Docker)
Step 6: Add tests
```

## When Uncertain

### Ask Clarifying Questions

```
User: Add a new endpoint for team management

Claude: Before implementing, I need to clarify:
        1. Should teams be a new model or extend existing User?
        2. What operations are needed? (create, invite, remove members)
        3. How does team ownership relate to API keys and policy clients?
```

### Propose Alternatives

```
User: Add caching to the user endpoint

Claude: I see two approaches:
        1. Redis cache with TTL (consistent with existing Redis usage)
        2. In-memory cache with FastAPI lifespan (simpler, single-instance only)

        Given the existing Redis infrastructure, option 1 is recommended.
        Which approach should I implement?
```

## Security Reviews

When modifying authentication or authorization:

- Verify JWT validation is not bypassed
- Check that token blacklist is consulted
- Ensure Pydantic validation covers all inputs
- Confirm error responses do not leak internal details
- Verify CORS origins are not accidentally widened
