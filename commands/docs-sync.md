# docs-sync

Analyze branch changes and ensure all documentation and code comments are accurate and up to date.

## Phase 1: Gather Context

1. Read `.claude/rules/` directory thoroughly — these are mandatory guidelines for this repository.
2. Run `git log --oneline $(git merge-base HEAD main)..HEAD` to list all commits on this branch.
3. Run `git diff $(git merge-base HEAD main)..HEAD` to understand all code changes.
4. Run `git diff $(git merge-base HEAD main)..HEAD --stat` for a file-level summary.
5. For each commit, run `git show <sha> --stat` to understand the intent and scope of individual changes.

## Phase 2: Audit ALL Documentation Files

Review ALL markdown files in the project. Exclude `.git/`, `node_modules/`, `cdk.out/`, `.venv/`, `.pytest_cache/`, and `.cursor/`.

1. Run `find . -name '*.md' -not -path '*/.git/*' -not -path '*/node_modules/*' -not -path '*/cdk.out/*' -not -path '*/.venv/*' -not -path '*/.pytest_cache/*' -not -path '*/.cursor/*'` to enumerate every markdown file.
2. From the branch diff (Phase 1), identify changed CDK constructs, renamed config fields, updated defaults, deleted stacks, new environment configs, and modified Make targets.
3. Search ALL markdown files for stale references using `grep` for old names, removed config fields, changed Make targets, deleted constants, and renamed classes.
4. Read and evaluate every match in context — example code blocks and Make command references in documentation are particularly prone to staleness since they do not produce build errors.

For each documentation file, evaluate:

- Does it accurately reflect the current models, routes, services, and configuration?
- Are Make command references correct (target names, arguments)?
- Are environment configurations current?
- Are code examples consistent with actual class signatures and field names?
- Are there references to removed, renamed, or deprecated modules?
- Is there new functionality that should be documented but is missing?
- Do deployment instructions match current workflow and Make targets?

### `docs/API_SPEC.md` — Special Requirements

This file is the primary reference for frontend engineers. Every endpoint section must be self-sufficient so a frontend developer can implement the integration without reading source code or asking backend engineers.

For each endpoint or endpoint group, ensure:

- **Prerequisites** are listed explicitly (e.g., "User must have a linked wallet address — see Step 2")
- **Step-by-step instructions** describe the exact sequence of API calls needed, including any prior setup calls
- **Request/response examples** use realistic field values and match the current schema (field names, types, nullable fields)
- **Error scenarios** list every status code the endpoint can return, with the cause and resolution
- **Cross-references** link to prerequisite steps in the Integration Guide (Step 1, Step 2, Step 3, etc.)
- **New fields** added to response schemas are documented with type and description
- **Behavioral changes** (e.g., a new 403 error when a wallet is missing) are reflected in the Errors section

## Phase 3: Audit ALL Code Comments

Review code comments across the entire project (excluding `.git/`, `node_modules/`, `cdk.out/`, `.venv/`). Changes to CDK constructs, config fields, and constants can make comments stale in files that were not directly modified.

1. From the branch diff, extract key terms: renamed config fields, changed defaults, deleted helpers, new environment variables, modified secret keys.
2. Search all `.py`, `.yml`, `.yaml`, `.sh` files for these terms in comments.
3. Review code files changed on the branch for comment accuracy.

Evaluate:

- Are existing comments still accurate after the changes?
- Are there complex CDK patterns or environment-specific logic that lack a "why" explanation?
- Remove stale or misleading comments.

**Comment principles:**

- Only comment where the "why" is non-obvious.
- Do not comment what the code literally does — the code shows that.
- Concise and clear. No filler phrases.
- No TODO comments unless they are actionable and tracked.

## Phase 4: Audit Test Assertions

CDK infrastructure tests are uniquely prone to staleness — resource counts, property values, and stack outputs drift as stacks evolve but tests do not produce build errors until run.

1. From the branch diff, identify changes to stacks, constructs, and configuration defaults.
2. Search all test files (`tests/unit/*.py`, `tests/integration/*.py`) for hardcoded values that may be stale: resource counts, property strings, namespace names, ARN formats, tag values.
3. Cross-reference test assertions against the actual stack code to verify correctness.
4. Run `make test` to confirm all tests pass after updates.

## Phase 5: Apply Updates

1. Update documentation files with accurate information.
2. Fix or remove stale code comments.
3. Update stale test assertions.
4. Add minimal necessary comments where "why" context is missing.
5. Ensure consistency in terminology across all docs and comments.
6. Run `ruff check . && ruff format --check .` and `make test` to verify no regressions.

## Rules

- Follow all instructions in `.claude/rules/` without exception.
- Do not add verbose or redundant comments. Less is more.
- Do not fabricate features or behaviors — only document what exists.
- Preserve the existing style and tone of documentation unless it conflicts with accuracy.
- If a doc file does not exist but should, create it only if the gap is significant.
- No emojis in documentation or comments.
- No AI attribution anywhere.
- **NEVER commit `.claude/` or `CLAUDE.md` files to git.** These are local-only configuration files managed by the user. Update them locally but do not stage, commit, or push them.

## Phase 6: Evolve Agent Instructions

Review and contribute to the `.claude/` folder to improve future agent sessions:

**Rules (`.claude/rules/`):**

- Are there patterns, conventions, or lessons learned from this branch that should become rules?
- Are existing rules still accurate, or do they reference outdated Make targets, config fields, or stack names?
- Add new rules for recurring decisions or non-obvious project conventions discovered during this work.

**Commands (`.claude/commands/`):**

- Are there repetitive workflows that could become reusable commands?
- Do existing commands need updates based on how the project has evolved?

**CLAUDE.md:**

- Update project context, quick commands table, deployment targets, or repository structure if they have changed.
- Keep it concise — this is orientation context, not exhaustive documentation.

**Principles:**

- Rules should be actionable and specific, not vague.
- Prefer fewer, high-value rules over comprehensive but noisy ones.
- Commands should encode workflows that are stable and repeatable.
- Do not add speculative or one-off guidance.

**IMPORTANT: All `.claude/` and `CLAUDE.md` changes are LOCAL ONLY. Do not commit them to git.**

## Output

After completing updates, provide a summary:

- Files modified (committed to git)
- Local-only files modified (`.claude/`, `CLAUDE.md` — not committed)
- Key documentation changes made
- Comments added, updated, or removed
- Test assertions fixed
- Any gaps or ambiguities that require human decision
