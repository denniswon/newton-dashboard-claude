# Optimize Lessons Learned

Analyze and optimize `.claude/rules/lessons.md` for size, deduplication, and decision-usefulness.

## Target

Under 40,000 characters. Current file should be measured first with `wc -c`.

## Process

1. **Measure** the current file size (`wc -c` and `wc -l`)
2. **Read** the full file in chunks
3. **Identify** these optimization opportunities:
   - **Duplicate entries**: Same root cause described in multiple sections — merge into one
   - **Verbose code examples**: Strip code blocks to 4 lines max; reference files instead of inlining full examples (curl commands, helper functions, cast call sequences)
   - **Version migration narratives**: If the codebase already uses the new version, strip the migration story — keep only "current correct usage" as a one-liner
   - **Entries covered elsewhere**: If the same info exists in other `.claude/rules/` files, architecture docs, or CLAUDE.md, remove from lessons.md
   - **Historical-only entries**: PRD provenance, one-time migration notes, obvious patterns (standard Rust/TS idioms) — remove unless there's a non-obvious gotcha
   - **Overlapping sections**: Merge related sections (e.g., separate "SDK" + "TypeScript" sections → one)
4. **Rewrite** the file with these formatting rules:
   - Each entry: `### Title` + 1-3 sentence explanation + `**Prevention**: one-liner`
   - No code blocks longer than 4 lines
   - Use tables for listing multiple related items (e.g., function comparison, dependency versions)
   - Keep the intro paragraph and `---` section separators
   - Group entries by domain (Solidity, Testing, SDK, CLI, Gateway, Platform API, etc.)
5. **Verify** the result: `wc -c` and `wc -l` — must be under 40,000 chars

## What to preserve (high-value gotchas)

- Entries where the failure mode is silent (signatures fail silently, wrong data paths cause silent denials)
- Security-critical invariants (CEI ordering, key isolation, attestation validation)
- Encoding/format mismatches (JSON arrays vs hex, camelCase vs snake_case, flat vs nested)
- Deployment pipeline gotchas (wrong policyId, stale CIDs, scoped secrets)
- Concurrency bugs (race conditions, runtime binding)
- Error selectors and diagnostic reference tables

## What to remove or condense

- Version migration stories where the codebase is already on the new version
- Full curl/cast command examples (reference scripts instead)
- Standard language/framework patterns anyone would know
- Historical provenance (where files came from, why things were renamed)
- Entries that just describe how existing code works (that's what reading the code is for)

## Output

Report: before/after size, number of entries merged, number removed, and any entries you were unsure about.
