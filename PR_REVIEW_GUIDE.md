# PR Review Guide for Newton Dashboard API

## Quick Start

Review current branch changes against main:

```bash
/review-pr
```

For reviewing a specific GitHub PR:

```bash
gh pr diff <PR_NUMBER> | less
```

## When to Review

**Always review before merge:**
- Database migrations (Alembic) — schema safety, rollback strategy, shared DB ownership
- API endpoint changes — `app/api/`
- Auth/SIWE logic — security-critical paths
- Async operations — proper await, connection pooling, session management
- Shared DB interactions — dashboard tables vs gateway tables boundary

## What Gets Reviewed

Blockers — data integrity (missing transaction boundaries, race conditions), async blocking (synchronous I/O in async functions), SQL injection, resource leaks (unclosed DB connections), missing input validation, auth bypass.

Concerns — error handling gaps, N+1 queries, missing indexes, inconsistent API response formats, missing structured logging, unnecessary code/logic duplication (refactor to keep logic in one place).

Minor notes — naming clarity, type hint completeness, Python idiom improvements.

## Key Patterns to Reference

| Pattern | Location |
|---------|----------|
| API routes | `app/api/` |
| DB models | `app/db/models/` |
| Auth/SIWE | `app/api/auth/` |
| Alembic migrations | `alembic/versions/` |
| Tests | `tests/` |

## Database Migration Review Checklist

When reviewing Alembic migrations:

- [ ] Can this run on production without downtime?
- [ ] Is there a rollback strategy (`downgrade` implemented)?
- [ ] Are new columns nullable or have defaults?
- [ ] Does it only touch dashboard-owned tables (not gateway tables)?
- [ ] Are indexes created concurrently if on large tables?

## Comment Style

Write like a senior engineer talking to a peer — direct, conversational, no filler.

### Voice and Framing

- **Use "we/us/let's" not "you"** — collaborative framing. Say "Let's do X", "if we update", "we should" instead of "you should", "if you update"
- **Direct imperatives for clear issues** — "DO NOT", "Shouldn't do this", "Let's do X" rather than hedging with "Consider" or "You might want to"
- **Prefix labels for non-blocking items:**
  - `Opinion:` — subjective feedback, architectural preferences, naming choices
  - `FYI:` — informational, good to know, not actionable now
  - `Suggestion:` — code improvement with example
  - `nit:` — minor/cosmetic issues
- **No praise sections** — don't include "What is working well" or "What looks good." The review body is for issues only.
- **No severity grouping in review body** — don't group items under "Should fix before merge" / "Nits" / "Nice to have" headers. Use a flat numbered list. Prefix labels (Opinion:, FYI:, nit:) signal weight inline.
- **Short, punchy comments preferred** — "This reads from disk on every request. Shouldn't do this." is better than a paragraph explaining why.
- **End with clear merge criteria** — "LGTM once above are addressed." or "LGTM once #1-2 are addressed. The rest are fine as follow-up."

### Inline Comments
- State the issue directly. No bold severity prefixes like "**Blocker:**" or "**Warning:**".
- Use "ditto:" when the same issue appears in another file — don't restate the full explanation.
- Use "nit:" for minor/cosmetic issues (typos, naming, unused params).
- Keep it short. One or two sentences for most comments. Only elaborate when the fix is non-obvious.
- Don't comment on things that are correctly done — no "this looks good" comments.
- Bold direct asks to the author: **Document this in the PR description.**
- Escalate clearly when blocking: "A blocker for this PR considering [reason]."
- Reference existing patterns: "We already do this for X in `app/path/to/file.py`"

### Review Summary (top-level)

Flat numbered list of remaining issues. No severity grouping. Use prefix labels (Opinion:, FYI:, Suggestion:) to signal weight. End with clear merge criteria.

```text
1. Missing rollback in migration — `downgrade()` is empty. We need this for safe prod deploys.

2. Suggestion: the auth check in `get_current_user` should validate token expiry. something like below can work.

3. Opinion: "status" enum — we should use a proper Python Enum instead of string literals.

4. FYI: the `SIWE_NONCE_TTL` default of 300s may be too short for slow mobile wallets.

LGTM once #1-2 are addressed. The rest are fine as follow-up.
```

### Anti-patterns (don't do these)
- No bold severity label prefixes ("**Deployment blocker:**", "**Critical:**")
- No positive-only comments ("This migration looks correct and safe")
- No over-formal language
- No markdown tables in the review body — use numbered lists
- No emoji
- No categorized essay-format review bodies with numbered items grouped by severity (e.g., "Critical Issues (Must Fix)" with items 1-5, "Warnings (Should Fix)" with items 6-10, "Suggestions" with items 11-14). This format belongs in inline comments, not the review body. The review body is a brief cover note, not a comprehensive report.
- No long code blocks in the review body. Code suggestions go in inline comments attached to the relevant line.
- No "What is working well" or praise sections — the review body is for issues only
- Using "you" — use "we/us/let's" for collaborative framing

### For follow-up reviews
Only mention what is NOT fixed from previous comments. Don't enumerate resolved items:

```text
From previous review comments, these seem to be missing
- X not addressed
- Y still has the issue

**New issues**
1. ...
```

## Combine With

1. **Run tests**: `make test`
2. **Database testing**: Apply migrations in Docker, verify rollback
3. **Linting**: `ruff check . && ruff format --check .`
4. **Human review**: Another team member for business logic and architecture
