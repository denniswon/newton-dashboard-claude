# lessons-org

Reorganize and clean up `.claude/rules/lessons.md` to keep it high-signal and maintainable.

## Goal

The lessons file should be a concise reference of key gotchas, invariants, and tricky patterns — not a chronological incident log. Optimize for what an engineer (or agent) needs to keep in mind while working on this codebase.

## Phase 1: Audit Current State

1. Read `.claude/rules/lessons.md` in full.
2. Read `.claude/rules/` — other rule files may already cover some lessons.
3. Count total entries and total lines.

## Phase 2: Identify Cleanup Targets

For each entry, classify as:

- **KEEP**: Active invariant, recurring gotcha, or tricky pattern that prevents future bugs.
- **CONSOLIDATE**: Multiple entries describing the same root issue — merge into one.
- **REMOVE**: One-time incident unlikely to recur, overly specific PR/commit details, agent behavior guidance (not code), content already covered in other rule files, or historical cleanup that's already done.
- **SHORTEN**: Good lesson but too verbose — trim to the actionable rule.

### Signals for removal

- References a specific PR number as the primary context (PR numbers are ephemeral)
- Describes an agent mistake, not a code pattern (e.g., "agent edited generated file")
- CI/tooling quirk that's already been fixed
- Design limitation documented elsewhere (architecture.md, security.md)
- Dead code that's already been removed

### Signals for keeping

- Describes a constraint that the compiler/linter cannot enforce
- Has caused bugs more than once
- Involves subtle cross-component interactions
- Would surprise a new contributor
- Relates to cryptographic correctness or consensus safety

## Phase 3: Reorganize

Group entries by theme, not chronologically. Suggested sections:

- **Critical Invariants** — constraints that cause silent failures when violated
- **Multichain Gotchas** — cross-chain patterns unique to this codebase
- **Code Patterns** — where things go, how to extend, common mistakes
- **Testing Gotchas** — non-obvious test setup requirements
- **Security & Operations** — production safety patterns
- **Contract Patterns** — Solidity-specific lessons
- **Crypto Gotchas** — cryptographic implementation pitfalls

## Phase 4: Write

Rewrite the file with these principles:

- **Lead with the rule**, not the incident. The "what happened" is context, not the point.
- **One entry per root cause.** If 5 entries describe the same `aggregator_block` bug, merge them.
- **No PR numbers in rules.** The rule should stand on its own without knowing which PR introduced it.
- **Actionable and specific.** "Be careful with blocks" is useless. "aggregator_block must equal task_created_block_u64 in single-chain mode" is useful.
- **Target: under 150 lines.** If over 200, you haven't cut enough.

## Phase 5: Verify

1. Confirm no critical lessons were lost — every KEEP/CONSOLIDATE entry has a corresponding rule in the output.
2. Confirm no duplication with other `.claude/rules/*.md` files.
3. Count final lines — should be meaningfully shorter than the input.

## Output

Provide a summary:

- Lines before → lines after
- Entries removed (with one-line reason each)
- Entries consolidated (which ones merged)
- Any new entries added based on patterns found during review

## Rules

- Do not add entries that belong in other rule files — move them there instead.
- Do not add verbose incident narratives. The lesson is the rule, not the story.
- No emojis. No AI attribution.
- Preserve the existing header format and markdown structure.
