---
description: Triage a reported issue — determine if it's a frontend integration problem or a backend bug, then act accordingly
---

# triage-issue

Investigate a reported issue, determine its root cause, and take the appropriate action: either update documentation for frontend consumers or fix the backend bug.

**Input:** The user will describe the issue (error message, unexpected behavior, user report, etc.). Treat `$ARGUMENTS` as the issue description.

## Phase 1: Understand the Report

1. Parse the issue description for key signals:
   - Error messages, HTTP status codes, endpoint paths
   - Expected vs actual behavior
   - Whether the reporter is a frontend developer, SDK user, or internal tester
2. Ask 1-2 clarifying questions if the issue description is ambiguous (e.g., which endpoint, which environment, reproducible steps).

## Phase 2: Investigate

### Locate the relevant code path

1. Identify the endpoint(s) involved from the issue description.
2. Read the route handler in `app/api/v1/routes/`.
3. Read the service layer in `app/services/`.
4. If blockchain-related, read the contract wrapper in `app/core/blockchain/contracts/`.
5. Check for related middleware logic in `app/middleware/`.

### Reproduce the logic

1. Trace the request flow from route -> dependencies -> service -> database/Redis/blockchain.
2. Identify where the reported behavior originates.
3. Check edge cases: missing fields, null values, ordering assumptions, race conditions.

### Check existing documentation

1. Read `docs/API_SPEC.md` — find the section for the affected endpoint(s).
2. Compare documented behavior (request/response schemas, error codes, prerequisites) against actual code behavior.
3. Note any discrepancies between docs and code.

## Phase 3: Classify

Based on investigation, classify the issue into one of two categories:

### Category A: Frontend / Integration Issue

The API is behaving correctly, but the caller is misusing it. Common signals:

- Missing prerequisite step (e.g., wallet not linked before creating API key)
- Wrong request format or missing required fields
- Misunderstanding of error codes or response schema
- Stale client-side assumptions about field types or nullability
- CORS or header misconfiguration on the client side
- Documentation is missing, unclear, or incorrect about the behavior

### Category B: Backend Bug

The API itself has a defect. Common signals:

- Incorrect status code returned (e.g., 500 instead of 400)
- Response schema doesn't match what the code actually returns
- Business logic error (wrong validation, missing check, incorrect query)
- Unhandled edge case (null pointer, missing await, race condition)
- Regression from a recent change

## Phase 4: Act

### If Category A (Frontend / Integration Issue)

1. **Explain the finding clearly:**
   - State that the API is working as designed
   - Explain what the caller should do differently
   - Provide the correct request/response example

2. **Update `docs/API_SPEC.md`:**
   - Add or clarify the prerequisite steps for the affected endpoint
   - Add the error scenario to the endpoint's Errors section with cause and resolution
   - If the issue involves a multi-step flow, add or update a step-by-step example
   - If relevant, add the scenario to the "Common Error Scenarios" table in the Integration Guide
   - Ensure cross-references to prerequisite steps (Step 1, Step 2, etc.) are present

3. **Documentation update rules:**
   - Every error code the endpoint can return must be listed with cause and resolution
   - Request/response examples must match the actual code schema exactly
   - Prerequisites must link to the Integration Guide steps
   - No internal implementation details in the docs (table names, service class names, etc.)
   - Follow the existing style and tone of `docs/API_SPEC.md`

4. **Draft a response** the user can forward to the reporter, explaining:
   - What went wrong on their side
   - The correct usage with a concrete example
   - A link to the relevant API spec section (if applicable)

### If Category B (Backend Bug)

1. **Explain the finding:**
   - Identify the root cause with file path and line number
   - Explain why it happens and under what conditions

2. **Fix the bug:**
   - Follow the layer architecture: model -> service -> route -> schema
   - Match existing code patterns (service injection via `Depends`, Pydantic validation, etc.)
   - Do not leak internal details in error responses
   - Add type hints on all new/modified function signatures

3. **Verify the fix:**
   - Run `make test` to confirm no regressions
   - If the fix changes API behavior, update `docs/API_SPEC.md` to reflect the new behavior
   - If a new error code is introduced, document it

4. **Update documentation if needed:**
   - If the bug caused undocumented behavior that callers may have relied on, note the change
   - If the fix introduces a new error scenario, add it to the endpoint's Errors section

## Phase 5: Summary

Provide a structured summary:

```
## Triage Result

**Classification:** Frontend Issue / Backend Bug
**Affected Endpoint(s):** `METHOD /v1/path`
**Root Cause:** [1-2 sentence explanation]

### Action Taken
- [List of changes made]

### Files Modified
- [file paths]

### Response for Reporter
[Draft message to send to the person who reported the issue]
```

## Rules

- Follow all instructions in `.claude/rules/` without exception.
- Do not guess — read the actual code before classifying.
- Do not assume the reporter is wrong — verify the API behavior matches documentation first.
- If both categories apply (docs are wrong AND code has a bug), fix both.
- No emojis in documentation, code, or responses.
- No AI attribution anywhere.
- Keep doc updates minimal and targeted — do not rewrite sections unrelated to the issue.
- Error responses in docs must include cause AND resolution, not just the status code.
