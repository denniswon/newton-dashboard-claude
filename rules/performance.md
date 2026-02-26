# Performance Guidelines

## Latency Targets

| Path | Target | Notes |
|------|--------|-------|
| Health check | < 10ms | Minimal processing |
| Authenticated API | < 200ms | Token validation + DB query |
| Auth challenge | < 500ms | Includes email send or WebAuthn |
| Auth verify | < 300ms | Token generation + DB writes |

## Async Patterns

### Always Use Async I/O

All database, Redis, and HTTP operations must be async:

```python
# Good: Async database query
async def get_user(self, user_id: UUID) -> User:
    result = await self.session.execute(
        select(User).where(User.id == user_id)
    )
    return result.scalar_one_or_none()

# Bad: Blocking I/O in async context
def get_user(self, user_id: UUID) -> User:
    result = self.session.execute(...)  # Blocks the event loop
```

### Background Tasks

Use FastAPI's `BackgroundTasks` for non-critical operations:

```python
@router.post("/challenge")
async def create_challenge(
    background_tasks: BackgroundTasks,
):
    # Critical path: create flow
    flow = await create_verification_flow()

    # Non-critical: send email in background
    background_tasks.add_task(send_verification_email, flow)

    return {"flow_id": flow.id}
```

## Database Performance

### Connection Pooling

SQLAlchemy async engine uses connection pooling by default. Configure appropriately:

```python
engine = create_async_engine(
    database_url,
    pool_size=20,           # Max persistent connections
    max_overflow=40,        # Additional connections under load
)
```

### Query Optimization

```python
# Good: Select only needed columns
result = await session.execute(
    select(User.id, User.is_active).where(User.id == user_id)
)

# Good: Use indexes for frequent queries
# Models define indexes in __table_args__

# Bad: N+1 queries - loading related objects in a loop
for user in users:
    factors = await get_factors(user.id)  # N queries
```

### Batch Operations

```python
# Good: Bulk insert
session.add_all(new_records)
await session.commit()

# Good: Single query for multiple IDs
result = await session.execute(
    select(ApiKey).where(ApiKey.id.in_(key_ids))
)
```

## Redis Performance

### Connection Management

Use a shared async Redis client with connection pooling:

```python
redis = aioredis.from_url(
    redis_url,
    max_connections=20,
    decode_responses=True,
)
```

### TTL for All Ephemeral Data

Always set TTL on Redis keys to prevent unbounded memory growth:

```python
# Good: TTL on verification flows (15 minutes)
await redis.set(f"flow:{flow_id}", data, ex=900)

# Good: TTL on token blacklist entries
await redis.set(f"blacklist:{jti}", "1", ex=token_remaining_ttl)

# Bad: No TTL
await redis.set(f"flow:{flow_id}", data)
```

### Pipeline for Multiple Operations

```python
# Good: Pipeline for atomic multi-key operations
async with redis.pipeline() as pipe:
    pipe.set(f"flow:{flow_id}", data, ex=900)
    pipe.incr(f"attempts:{flow_id}")
    await pipe.execute()
```

## API Response Optimization

### Pagination

Use pagination for list endpoints to limit response size:

```python
@router.get("/api_key")
async def list_api_keys(
    limit: int = Query(default=20, le=100),
    offset: int = Query(default=0, ge=0),
):
    ...
```

### Selective Field Loading

Only load fields needed for the response. Avoid returning entire model objects when a subset suffices.

## Middleware Performance

### Exclude Health Checks from Logging

The logging middleware skips `/health` to reduce noise and overhead:

```python
if request.url.path == "/health":
    return await call_next(request)
```

## Anti-patterns

| Anti-pattern | Problem | Solution |
|--------------|---------|----------|
| Sync I/O in async | Blocks event loop | Use async drivers (asyncpg, aioredis) |
| N+1 queries | Latency multiplication | Use joins or batch queries |
| Unbounded queries | Memory exhaustion | Always paginate list endpoints |
| Missing Redis TTL | Memory leak | Set TTL on all ephemeral keys |
| Large response bodies | Network latency | Paginate and select needed fields |
| Blocking email send | Slow API response | Use BackgroundTasks |
