# Optimize Lessons Learned

Analyze and optimize `.claude/rules/lessons.md` for size, deduplication, and decision-usefulness.

## Target

Under 40,000 characters. Current file should be measured first with `wc -c`.

## Core Principle

`lessons.md` is a **failure mode encyclopedia** — it describes how things *break*, not how they work. Architecture docs describe the system design; lessons.md captures the non-obvious ways agents can produce broken code. Every removal decision must pass the **agent safety test**: "If an agent encountered this scenario without this lesson, would they produce broken code or waste significant investigation time?" If yes, the entry stays — even if the topic appears in another file.

## Process

1. **Measure** the current file size (`wc -c` and `wc -l`)
2. **Read** the full file in chunks
3. **For each entry, classify** as one of:
   - **KEEP (sacred)**: Bug-fix-origin entries with silent failure modes, error selectors, or diagnostic patterns (symptom → cause → fix). These exist because an agent WILL make this mistake without them.
   - **CONDENSE**: Entry has the right content but is verbose — trim narrative, shrink code blocks, remove "Found in" provenance paragraphs while keeping the diagnostic triad.
   - **MERGE**: Same root cause described in multiple sections — combine into one entry.
   - **REMOVE**: Truly redundant (the *failure mode and diagnostic pattern* — not just the topic — is already captured elsewhere), or describes standard language/framework behavior any competent developer knows.
4. **Apply the "covered elsewhere" rule carefully**:
   - A topic appearing in architecture.md or testing.md is NOT grounds for removal from lessons.md
   - Only remove if the OTHER file captures the same **failure mode**, **symptom**, and **diagnostic steps** — not just the architectural description
   - Example: "Watcher path tests need block_time=1" as a config fact in testing.md does NOT replace the failure explanation (`referenceBlockNumber < block.number` fails as `N < N`) in lessons.md
   - Example: "Auto-generated files: do not edit" IS safe to remove — CLAUDE.md already says `make generate-bindings` and the failure mode is obvious (compilation errors)
5. **Identify** remaining optimization opportunities:
   - **Verbose code examples**: Strip code blocks to 4 lines max; reference files instead of inlining full examples
   - **Version migration narratives**: If the codebase already uses the new version, strip the migration story — keep only "current correct usage" as a one-liner
   - **Historical-only entries**: PRD provenance, one-time migration notes — remove unless there's a non-obvious gotcha
   - **Overlapping sections**: Merge related sections
6. **Rewrite** the file with these formatting rules:
   - Each entry: `### Title` + explanation + `**Prevention**: one-liner`
   - Standard entries: 1-3 sentences
   - High-severity entries (silent failures, security invariants): up to 5 sentences to preserve the diagnostic triad (symptom → cause → fix)
   - No code blocks longer than 4 lines
   - Use tables for listing multiple related items
   - Keep the intro paragraph and `---` section separators
   - Group entries by domain
7. **Verify** the result: `wc -c` and `wc -l` — must be under 40,000 chars

## What to NEVER remove (sacred entries)

These categories stay regardless of size pressure — condense them if needed, but never delete:

- **Silent failure modes**: Where the system produces wrong results without errors (BLS verification passes with wrong data, attestation mapping empty because evaluateResult is falsy, nonce races under concurrent load)
- **Error selectors and diagnostic patterns**: Hex selectors (`0xb9a620da`, `0x716dcc39`, `0xee762560`) with their symptom descriptions — these are the lookup key when debugging on-chain reverts
- **Security invariants that break silently**: Dedup key manipulation, hash encoding mismatches across storage paths, cross-chain slashing without proof binding
- **Concurrency and ordering bugs**: Nonce races, HashMap non-determinism vs on-chain array ordering, multiple AvsWriter instances sharing a signer
- **Block reference invariants**: aggregator_block must match taskCreatedBlock, chain-specific offsets, watcher path vs RPC path differences — these are the #1 source of BLS verification failures
- **Entries with NEWT-xxx ticket references**: These represent real bugs that took hours to investigate; the diagnostic context is irreplaceable

## What to preserve (high-value gotchas)

- Entries where the failure mode is silent (signatures fail silently, wrong data paths cause silent denials)
- Security-critical invariants (CEI ordering, key isolation, attestation validation)
- Encoding/format mismatches (JSON arrays vs hex, camelCase vs snake_case, flat vs nested)
- Deployment pipeline gotchas (wrong policyId, stale CIDs, scoped secrets)
- Concurrency bugs (race conditions, runtime binding)
- Error selectors and diagnostic reference tables
- Alloy-specific gotchas (receipt doesn't propagate reverts, B256 double-prefix, type incompatibility across contracts)
- Contract constructor changes breaking Etherscan verification silently
- Migration tracking inconsistencies (sqlx checksums, init_gateway_tables.sql artifacts)

## What to remove or condense

- **Safe to remove**: Entries where the failure mode is obvious and well-documented elsewhere (e.g., "auto-generated files: do not edit", "multichain is the default mode")
- **Safe to remove**: Standard language/framework patterns anyone would know (e.g., "zsh cp prompts for overwrite", "git submodule commit workflow")
- **Condense, don't remove**: "Found in: NEWT-xxx" paragraphs — strip the narrative but keep the ticket number as a reference tag
- **Condense, don't remove**: Verbose fix descriptions — keep the one-liner fix command, drop the multi-paragraph explanation
- Version migration stories where the codebase is already on the new version
- Full curl/cast command examples (reference scripts instead)
- Historical provenance (where files came from, why things were renamed)

## Output

Report: before/after size, number of entries merged, number removed, number condensed, and flag any entries where you were unsure (let the user decide).
