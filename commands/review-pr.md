---
description: Review current PR changes for security, correctness, and code quality
---

Please use the pr-reviewer agent to review my changes against the main branch.

Run `git diff main...HEAD` to get the full diff, then analyze all changed files. Prioritize security (auth bypass, SQL injection, API key scoping), async correctness (blocking calls, session leaks), and data integrity (migrations, ownership verification).

Provide feedback as plain prose with specific file:line references. No emoji headers, no bold-label patterns like "**Problem:**", no markdown headings in the output. Write like a senior engineer leaving terse, direct review comments.

Skip files that are auto-generated (Alembic auto-gen), formatted (handled by ruff), or have only trivial changes.
