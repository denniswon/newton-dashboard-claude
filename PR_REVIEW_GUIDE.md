# PR Review Workflow with Claude Code

This guide explains how to use the manual PR review agent for the Newton Prover AVS codebase.

## Quick Start

### Option 1: Slash Command (Easiest)

```bash
# From anywhere in your Claude Code session, after making changes:
/review-pr
```

This will automatically:

1. Get the diff against `main` branch
2. Invoke the `pr-reviewer` agent
3. Analyze all changes for critical issues, warnings, and suggestions
4. Provide structured feedback

### Option 2: Direct Agent Invocation

```bash
# More explicit invocation with custom instructions:
Use the pr-reviewer agent to review my changes focusing on security in the gateway crate
```

### Option 3: Targeted Review

```bash
# Review specific files or areas:
Use the pr-reviewer agent to check the BLS signature changes in crates/keys/ for cryptographic correctness
```

## When to Use

**Always review before:**

- Creating a PR from your feature branch
- Requesting review from teammates
- Merging to main/develop

**Especially important for:**

- Security-sensitive changes (crypto, key management, contract interactions)
- Consensus logic changes (quorum calculations, aggregation)
- Cross-chain operations (bridge, multi-chain coordination)
- New external integrations (RPC endpoints, contract calls)

## What Gets Reviewed

The agent checks for issues at three severity levels:

Blockers — security (key exposure, missing validation, unsafe soundness), consensus (quorum bugs, Byzantine faults, race conditions), resource management (memory leaks, unbounded growth, panic paths), and cryptographic correctness (signature verification, point validation).

Concerns — error handling (missing context, unwrap/expect usage), async patterns (cancellation safety, proper spawning), performance (N+1 queries, excessive cloning, lock contention), missing tests for critical paths, and unnecessary code/logic duplication (refactor to keep logic in one place).

Minor notes — naming clarity, Rust idiom improvements, and observability gaps.

## Example Workflow

### Scenario: Adding a new operator endpoint

```bash
# 1. Make your changes
vim crates/rpc/src/operator_api.rs

# 2. Self-review before committing
/review-pr

# 3. Address any critical issues or warnings
# (Agent will provide specific file:line references)

# 4. Commit when clean
git add .
git commit -m "feat: add operator health check endpoint"

# 5. Create PR (optional: review again if you made fixes)
git push origin feature/operator-health-check
gh pr create --fill
```

### Scenario: Reviewing someone else's PR

```bash
# 1. Check out their branch
gh pr checkout 123

# 2. Review their changes
/review-pr

# 3. Leave feedback on the PR with specific issues found
gh pr comment 123 --body "Found security issue in operator_api.rs:45 - see Claude review"
```

## Understanding the Output

The agent provides feedback grouped by severity — blockers first, then concerns, then minor notes. Output is plain prose with file:line references, not formatted with emoji headers or bold labels.

Example output:

```
`crates/keys/src/bls.rs:42` — deserializing untrusted BLS keys without subgroup validation.
Invalid points could cause signature verification to panic. Add `pubkey.validate()`
after deserialization to check G1 subgroup membership.

`crates/operator/src/task_processor.rs:89` — error doesn't explain which task failed.
Add `.context("failed to process task {task_id}")`.

`crates/rpc/src/server.rs:120` — same validation logic repeated in 3 endpoints.
We already have a pattern for this in `crates/gateway/src/rpc/types/mod.rs` — extract
a shared `validate_operator_auth()` helper.
```

## Advanced Usage

### Review Specific Commits
```bash
# Review last 3 commits
Use the pr-reviewer agent to review the changes in the last 3 commits (git diff HEAD~3...HEAD)
````

### Review Against Different Base

```bash
# Review against develop branch instead of main
Use the pr-reviewer agent to review my changes against the develop branch
```

### Focus on Specific Concerns

```bash
# Security-only review
Use the pr-reviewer agent to review my changes focusing ONLY on security and cryptographic correctness

# Performance review
Use the pr-reviewer agent to review my changes focusing on performance and resource usage
```

### Pre-commit Hook Integration

Add to `.git/hooks/pre-commit`:

```bash
#!/bin/bash

# Only run if staged changes exist
if ! git diff --cached --quiet; then
    echo "🔍 Running Claude PR review on staged changes..."

    # Use Claude Code CLI to run review
    # (This requires Claude Code CLI to be installed)
    # Uncomment when ready to automate:
    # claude-code "/review-pr" || exit 1
fi
```

## Customizing the Agent

The agent configuration is in `.claude/agents/pr-reviewer.md`. You can:

**Adjust review strictness:**

- Add new checklist items
- Change critical → warning priority
- Add domain-specific checks

**Modify tools available:**

```yaml
tools:
  - Read
  - Grep
  - Glob
  - Bash # Remove if you want to restrict shell access
```

**Change model:**

```yaml
model: sonnet # or "opus" for more thorough reviews (slower/costlier)
```

## Tips & Best Practices

**Do:**

- ✅ Run review before creating PR (catch issues early)
- ✅ Address all critical issues (they're blockers for good reason)
- ✅ Consider warnings seriously (they prevent tech debt)
- ✅ Use focused reviews for specific concerns (faster feedback)

**Don't:**

- ❌ Ignore critical security issues (especially crypto/validation)
- ❌ Over-rely on agent (it's a tool, not a replacement for human review)
- ❌ Dismiss warnings without understanding (they often catch real bugs)
- ❌ Review trivial changes (formatting, generated code)

**Agent Limitations:**

- Won't catch high-level architecture problems
- Can't test if code actually works (run tests!)
- May miss context-specific security issues
- Doesn't understand full system design

Always combine agent reviews with:

- Running tests (`cargo test --workspace`)
- Manual testing of changed functionality
- Human review for architectural decisions

## Troubleshooting

**"Agent not found"**

```bash
# Check agent exists:
ls .claude/agents/pr-reviewer.md

# Reload Claude Code session if you just created it
```

**"No changes to review"**

```bash
# Make sure you have commits on your branch:
git log main..HEAD

# Or specify commits explicitly:
git diff HEAD~2...HEAD
```

**Review takes too long**

```bash
# Review specific files only:
Use the pr-reviewer agent to review only the changes in crates/gateway/

# Or use focused review:
Use the pr-reviewer agent for a quick security-only review
```

**Getting too many suggestions**

```bash
# Ask for critical/warnings only:
Use the pr-reviewer agent but skip the suggestions section
```

## Integration with Team Workflow

**For individual developers:**

1. Self-review with `/review-pr` before pushing
2. Fix critical issues and warnings
3. Create PR with clean code

**For teams:**

1. Author self-reviews before requesting review
2. Reviewer runs `/review-pr` independently
3. Both discuss findings in PR comments
4. Merge only when both agree critical issues are resolved

**For CI/CD (future enhancement):**

- GitHub Action could run agent on PR open
- Post review as PR comment automatically
- Block merge if critical issues found

## Cost Considerations

**Token usage per review:**

- Small PR (< 500 lines): ~5-10k tokens (~$0.01-0.02)
- Medium PR (500-2000 lines): ~20-50k tokens (~$0.05-0.12)
- Large PR (> 2000 lines): Consider reviewing in chunks

**Optimization tips:**

- Review smaller PRs (better code review practice anyway)
- Focus reviews on specific concerns when possible
- Skip trivial changes (formatting, generated code)
- Use focused reviews: "security only" uses fewer tokens than full review

## Comment Style Guidelines

When posting inline PR review comments, follow these style conventions:

### General Principles

- Be **direct and concise** - state the issue immediately without preamble
- No bold category prefixes like "**Critical:**" or "**Warning:**"
- No formal headers like "Suggested fix:" - just provide the fix
- No markdown headings (`##`, `###`) in the review body — use `**bold**` sparingly
- Reference existing patterns in the codebase when applicable
- Use conversational directives: "Let's not do this way" or "something like below can work"
- Use "ditto:" for repeated issues in different files — don't restate the full explanation
- Use "nit:" for minor/cosmetic issues (typos, naming, unused params)
- For follow-up reviews: reference previous comments casually ("still not addressed from previous review"), not formally ("Previous review comment #11 — still not addressed")
- Keep the review body focused on what's wrong, not what's been fixed
- Don't comment on things that are correctly done — no "this looks good" or "correct pattern" comments
- Bold direct asks to the author: **Document this in the PR description.**
- Escalate clearly when blocking: "A blocker for this PR considering [reason]."

### Voice and Framing

- **Use "we/us" not "you"** — collaborative framing. Say "Let's do X", "if we update the PDF", "we should make it more specific" instead of "you should", "if you update"
- **Direct imperatives for clear issues** — "DO NOT", "Shouldn't do this", "Let's do X" rather than hedging with "Consider" or "You might want to"
- **Prefix labels for non-blocking items:**
  - `Opinion:` — subjective feedback, architectural preferences, naming choices
  - `FYI:` — informational, good to know, not actionable now
  - `Suggestion:` — code improvement with example
  - `nit:` — minor/cosmetic (already documented above)
- **No praise sections** — don't include "What is working well" or "What looks good." The review body is for issues only.
- **No severity grouping in review body** — don't group items under "Should fix before merge" / "Nits" / "Nice to have" headers. Use a flat numbered list. Prefix labels (Opinion:, FYI:, nit:) signal weight inline.
- **Short, punchy comments preferred** — "This hardcodes a macOS Chrome path. Shouldn't do this." is better than a paragraph explaining why hardcoding is bad. The team is senior engineers — don't over-explain obvious fixes or well-known patterns.
- **No formulaic openers or closers** — Do NOT always start with a compliment-then-summary pattern ("Solid X — wiring Y into Z is necessary for...") or always end with "LGTM once X are addressed." Both openers and closers must vary naturally. Opener examples: jump straight into the cross-cutting issue, or just "a few things inline." Closer examples: "should be good after the inline stuff", "just the one thing in tx.rs", or just end with the last point — no closer needed. Repetitive phrasing at either end looks robotic.

### Review Body

Keep the overall review body minimal and conversational. No markdown headers (`##`, `###`) — use `**bold**` for light section separation if needed.

**No duplication with inline comments.** The review body and inline comments are both visible on the PR. Every finding that has a specific line goes as an inline comment ONLY — do not repeat it, summarize it, or rephrase it in the review body. The review body contains only cross-cutting items that have no specific line. Do NOT use the review body to enumerate or recap what the inline comments say — the author can read them directly.

**For first-time reviews:**

```text
made some suggestions inline.
```

Or, if there are cross-cutting items without a specific line:

```text
4. FYI: `handler/mod.rs:2057-2058` — when both nonce calls fail, only one error surfaces. Minor, but worth logging both.

10. `tx_worker.rs` — `queue_depth()` has no test coverage. Worth a unit test.

Rest is inline.
```

**For follow-up reviews (re-reviews):**

Flat numbered list of remaining issues. No severity grouping. Use prefix labels (Opinion:, FYI:, Suggestion:) to signal weight. End with clear merge criteria.

```text
1. API route: Loops API response still not checked — the `fetch` to Loops fires and the response is silently discarded. If Loops returns a non-2xx, there is no way to know. Add a response check inside the inner try block.

2. Whitepaper page description could be stronger — we should make it more specific about what the prospect will learn...

3. Opinion: "Neobanks & Neo Finance" use case — Not really a use case. The whitepaper covers: Stablecoins, RWAs, Institutional DeFi...

4. FYI: **Cache-Control for PDF** — `s-maxage=2592000` (30 days) means if we update the PDF, CDN will serve the stale version for up to 30 days. Let's lower to `s-maxage=86400` (1 day).

Deferred items (contrastive differentiators, remaining visuals, P2s) are fine as follow-up.
```

If previous round had unaddressed items, mention them briefly before new issues:

```text
From previous review comments, these two seem to be missing
- `aws-config`/`aws-sdk-kms` not feature-gated → should actually do the `TODO`
- `data_source` pub breaks encapsulation → this is still `pub`

**New issues**

1. EIP712 domain separator is computed incorrectly...
2. SP1 circuit is broken...
```

Avoid:
- Markdown headings (`## Fresh Review of PR #375`)
- Full checkbox lists of all 12 previous comments with resolution status
- Formal "Summary" sections with wrap-up paragraphs
- Structured tables of findings by severity category
- Categorized essay-format review bodies with numbered items grouped by severity (e.g., "Critical Issues (Must Fix)" with items 1-5, "Warnings (Should Fix)" with items 6-10, "Suggestions" with items 11-14). This format belongs in inline comments, not the review body. The review body is a brief cover note, not a comprehensive report.
- Long code blocks in the review body. Code suggestions go in inline comments attached to the relevant line.
- "What is working well" or praise sections — the review body is for issues only
- Using "you" — use "we/us/let's" for collaborative framing
- Formulaic/template-sounding openers or closers — never use the same opening pattern (e.g., "Solid X improvement — wiring Y...") or closing pattern (e.g., "LGTM once X are addressed") repeatedly. Vary naturally or skip entirely.
- Over-explaining straightforward issues — the team is senior engineers. "missing chain_id" is enough; don't add a paragraph about why consistency matters.
- Summarizing inline comments in the review body — the author sees both. The body is for cross-cutting items only.

### Inline Comment Format

Good (direct, references codebase):

```text
The `tasks` HashMap grows unboundedly as new tasks are tracked via `track_task()` but never cleaned up. Per `performance.md`, all collections MUST have size limits to prevent DoS.

1. Add bounded capacity with LRU eviction (consider using the `lru` crate)
2. Remove tasks after evaluation in `evaluate_response()`
```

```text
Let's not do this way. Impl `From` trait. We already do this for CreateTaskRequest, etc. in `crates/gateway/src/rpc/types/mod.rs`
```

```text
Same issue as `listener.rs` - the spawned task exits silently on WebSocket failure with no reconnection logic.
```

Avoid (too formal, verbose prefixes):

```text
**Critical: Unbounded HashMap growth**

The `tasks` HashMap grows unboundedly...

**Suggested fix:**
1. Add bounded capacity...
```

### Code Suggestions

Only include code examples when the fix is non-obvious or when showing an existing codebase pattern. Most comments should be prose-only — don't pad every comment with a code block.

When you do include code, introduce it casually:

```text
something like below can work.
```

```rust
tokio::spawn(async move {
    let mut retry_count = 0;
    loop {
        // ...
    }
});
```

Not:

```text
**Suggested fix:** Implement reconnection with exponential backoff per `performance.md:163-194`.
```

### Cross-References

Always point to existing patterns in the codebase using **GitHub permalinks** (with commit SHA and line range), not raw file paths:

- "We already do this for X in https://github.com/newt-foundation/newton-prover-avs/blob/abc123/crates/path/to/file.rs#L42-L50"
- "Per the pattern in https://github.com/newt-foundation/newton-prover-avs/blob/abc123/crates/gateway/src/lib.rs#L168"
- "Same issue as in https://github.com/newt-foundation/newton-prover-avs/blob/abc123/crates/chain-watcher/src/lib.rs#L73"

Never use raw text like `file.rs:42` — always link to the actual source on GitHub so reviewers can click through.

### Brief Follow-up Comments

For related issues, keep it minimal:

```text
Same issue as `listener.rs` - should use `try_into()` with proper error handling.
```

### Actionable Suggestions

Be direct about what should change:

```text
please improve code inline comments or remove unnecessary comment lines.
```

```text
Using the default nonce manager **with retries** should be okay for now
```

### When Reviewing Observability

Check all changed files against `.claude/rules/observability.md`:

- **Log levels**: Is the level appropriate per the Log Level Decision Matrix? INFO in a hot path is a red flag. Missing ERROR on a failure path is a gap.
- **Zero-logging zones**: No logging (not even TRACE) in BLS signing, ECDSA attestation inner loop, Merkle hash computation, ABI encoding hot paths, or Moka L1 cache hits.
- **chain_id in structured fields**: Every log statement touching chain-specific data must include `chain_id`. Every new Prometheus metric that is chain-specific must have a `chain_id` label.
- **Metric cardinality**: No `task_id`, `address`, or `error_message` as Prometheus labels. Only bounded label sets.
- **Privacy data in logs**: Never log HPKE ciphertext, decrypted privacy data, DKG key shares, identity PII, or raw signatures. Log operation results (count, status) not content.
- **Timer placement**: Histogram timers should wrap the full operation, not individual iterations inside a loop.
- **Metric naming**: Must follow `newton_avs_<component>_<metric>_<unit>` convention.
- **Overhead budget**: BLS hot path = zero overhead (counters only). RPC handler < 1%. Background workers can have full instrumentation.

Flag as a concern (not blocker) unless the violation is in a hot path or leaks sensitive data.

### When reviewing Solidity Review

Focus on not just correctness but also "security" and efficiency (ethereum gas, code efficiency) of the solidity contract code are EXTREMELY important. If something is unclear in the changes brought to the pull request, we should ask as a comment inline in the pull request.

Make github PR comments inline to the remote pr using MY github `@denniswon` configured in `~/.gitconfig`. PR comments should be made using "my style of commenting".

## Further Reading

- [CLAUDE.md](../CLAUDE.md) - Full review standards and examples
- [Sub-agents docs](https://docs.claude.com/en/docs/claude-code/sub-agents.md) - Understanding Claude Code agents
- [Newton AVS Architecture](../README.md) - Understanding system design

## Feedback & Iteration

This workflow is designed to evolve. If you find:

- False positives (agent flags non-issues)
- Missing checks (real issues not caught)
- Unclear feedback (suggestions not actionable)

Update `.claude/agents/pr-reviewer.md` and `CLAUDE.md` to improve the system!
