# verify-output

Verify that all recent output (code, docs, comments, CLI examples, config references) contains no false statements or incorrect instructions.

## Process

### Step 1: Identify Scope

Determine what was recently written or modified. Check:
- `git diff` for uncommitted changes
- `git diff $(git merge-base HEAD main)..HEAD` for branch changes
- Any files created or modified in the current conversation

### Step 2: Extract Verifiable Claims

For each modified file, extract every factual claim that can be verified against the codebase:

- **Makefile targets**: Does the target exist? What parameters does it accept?
- **CLI commands and flags**: Read the clap/argparse structs to confirm exact flag names, required subcommands, and valid values
- **File paths**: Use Glob/ls to confirm files exist where stated
- **Config fields and defaults**: Read the config struct AND its Default impl, not just the TOML file
- **Prerequisites and dependencies**: Grep for actual invocations (e.g., don't list Node.js if no `package.json` or `npm`/`node` usage exists)
- **Environment variables**: Verify the env var name matches what the code reads (check `std::env::var`, `config::Environment::with_prefix`, etc.)
- **Function signatures and behavior**: Read the actual function, not just the doc comment
- **Cross-references**: Verify linked files exist, linked sections exist in target docs

### Step 3: Verify Each Claim

For each extracted claim, run the appropriate verification:

```
Makefile target    → grep '^target_name:' Makefile
CLI flag           → Read the CLI parser struct (clap derive, Subcommand enum)
File existence     → Glob or ls
Config field       → Read the struct definition + Default impl
Env var name       → Grep for the exact string in source code
Function behavior  → Read the function body
```

### Step 4: Fix or Ask

- **False claim with clear fix**: Fix it immediately
- **Uncertain claim**: ASK the user for clarification rather than guessing
- **Claim that cannot be verified from source**: Flag it explicitly

### Step 5: Report

Provide a summary:
- Claims verified (count)
- False statements found and fixed
- Uncertain items flagged for user review
- Remaining items that could not be verified from source alone

## Rules

- A missing fact is always better than a false statement
- Do not trust comments or docstrings as ground truth — verify against actual code behavior
- Do not assume a Makefile target accepts a parameter just because another target does
- Do not assume CLI subcommands or flags exist — read the parser
- Check both the happy path AND edge cases (e.g., does a "default" actually exist in the Default impl?)
- When verifying env vars, check the exact string including prefix and separator convention
