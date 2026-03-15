# fix-prs

Rebase feature branches onto up-to-date main, resolve merge conflicts, fix CI failures, and force-push.

**Input**: `$ARGUMENTS` — optional GitHub PR URL (e.g., `https://github.com/org/repo/pull/123`). If empty, process all local branches with open PRs.

---

## Mode Selection

- **If `$ARGUMENTS` contains a PR URL**: Process that single PR's branch.
- **If `$ARGUMENTS` is empty**: Find all local branches with open PRs against this repo, detect stacked PR chains, and process them bottom-up.

---

## Phase 0: Setup

1. Save the current branch name so we can return to it at the end.
2. Pull main up to date:
   ```bash
   git fetch origin
   git checkout main
   git pull origin main
   ```

---

## Phase 1: Discover Branches to Process

### Single PR mode (URL provided)

1. Extract owner, repo, and PR number from the URL.
2. Run `gh pr view <number> --json headRefName,baseRefName,state` to get the branch name, base branch, and state.
3. If state is `MERGED` or `CLOSED`, delete the local branch (`git branch -D <branch>`) and exit with a message.
4. Otherwise, the work list is just this one branch.

### All-PRs mode (no URL)

1. List all local branches: `git branch --format='%(refname:short)'` (exclude `main`/`master`).
2. For each local branch, check if it has an open PR: `gh pr list --head <branch> --state open --json number,baseRefName,headRefName`.
3. **Cleanup**: If a local branch has a PR that is `MERGED` or `CLOSED` (check via `gh pr list --head <branch> --state merged --json number` and `--state closed`), delete the local branch with `git branch -D <branch>` and print a message. Skip further processing.
4. Collect all branches with open PRs into a work list with their base branches.

### Detect Stacked PR Order

Stacked PRs are identified by their GitHub base branch pointing to another feature branch (not main/master).

1. Build a dependency graph: for each branch in the work list, record its `baseRefName`.
2. Topologically sort: branches whose base is `main` come first, then branches whose base is a branch already in the sorted list, etc.
3. Process in this topological order (bottom of stack first).

**Example**: If branch C is based on B, and B is based on A, and A is based on main, process order is: A, B, C.

---

## Phase 2: Process Each Branch (in order)

For each branch in the sorted work list:

### Step 2.1: Rebase onto updated base

```bash
git checkout <branch>
git rebase <base_branch>
```

- `<base_branch>` is `main` for the bottom of the stack, or the already-rebased parent branch for stacked PRs.
- If rebase encounters conflicts, resolve them:

### Step 2.2: Resolve Merge Conflicts

For each conflicted file:

1. Read both versions (ours = feature branch, theirs = base branch).
2. **Default to feature branch changes**, BUT:
   - If the base branch REMOVED or refactored code that the feature branch still references, adopt the base branch's removal/refactoring. Do not re-introduce deprecated or deleted code.
   - If the base branch added new code that doesn't conflict with the feature branch's intent, keep both.
   - If both sides modified the same lines with different logic, prefer the feature branch's version but verify it still makes sense in the context of the base branch's surrounding changes.
3. After resolving each file: `git add <file>`
4. Continue: `git rebase --continue`
5. Repeat until rebase completes.

### Step 2.3: Verify No Regressions

Run formatting and lint checks to catch issues introduced during conflict resolution. The commands depend on the project's tech stack — detect automatically:

| Indicator | Stack | Format Command | Lint Command |
|-----------|-------|----------------|--------------|
| `Cargo.toml` + `Makefile` with `fmt`/`clippy` targets | Rust/Foundry | `make fmt` | `make clippy` |
| `Makefile` with `fmt`/`lint` targets (no Cargo.toml) | Python (ruff/black) | `make fmt` | `make lint` |
| `biome.json` + `package.json` | TypeScript (Biome) | `npm run format` | `npm run lint` |
| `package.json` with `lint` script (no biome) | TypeScript (ESLint) | `npm run lint -- --fix` | `npm run lint` |
| `foundry.toml` only (no Cargo.toml, no package.json) | Solidity-only | `forge fmt` | `forge build` |
| None of the above | Unknown | Skip verification | Skip verification |

**Detection logic** (run at project root):
1. Check for `Cargo.toml` → Rust project. Use `make fmt && make clippy` if Makefile has those targets, otherwise `cargo fmt && cargo clippy --workspace -- -D warnings`.
2. Else check for `biome.json` → Biome project. Use `npm run format && npm run lint`.
3. Else check for `package.json` with `lint` script → Node/TS project. Use `npm run lint -- --fix && npm run lint`.
4. Else check for `Makefile` with `fmt`/`lint` targets → Python project. Use `make fmt && make lint`.
5. Else check for `foundry.toml` → Solidity-only project. Use `forge fmt && forge build`.
6. Else skip verification with a warning.

- If lint reports errors, fix them.
- If format changed files, those changes need to be amended into the last commit (or added as a fixup).
- Re-run until both pass cleanly.

### Step 2.4: Force Push

```bash
git push origin <branch> --force-with-lease
```

### Step 2.5: Update GitHub Base Branch (for stacked PRs)

After rebasing, if the PR's base branch on GitHub should now point to `main` (because the parent PR was merged or the stack was flattened), update it:

```bash
gh pr edit <pr_number> --base main
```

For mid-stack branches whose parent is still open, update the base to the parent branch name (it should already be correct, but verify):

```bash
gh pr edit <pr_number> --base <parent_branch>
```

---

## Phase 3: Check and Fix CI

For each branch that was pushed (process in the same order):

### Step 3.1: Check CI Status

```bash
gh pr checks <pr_number> --watch --fail-fast
```

If all checks pass, move to the next branch.

### Step 3.2: If CI Fails

1. Run `gh pr checks <pr_number>` to identify which checks failed.
2. For each failed check, fetch the logs:
   ```bash
   gh run view <run_id> --log-failed
   ```
3. Diagnose the failure:
   - **Code issue** (test failure, lint error, type error, build failure): Fix the code, commit the fix, and push.
   - **Workflow issue** (outdated CI config, missing step, wrong tool version): Update the relevant file in `.github/workflows/`, commit, and push.
   - **Pre-existing failure** unrelated to the rebase: Fix it anyway — the goal is all checks green.
4. After fixing, push with a regular `git push origin <branch>` (not force push unless amending).
5. This is a single round — if further failures appear after the fix, the user will re-invoke `/fix-prs`.

---

## Phase 4: Cleanup

1. Return to the original branch: `git checkout <saved_branch>`
2. Print a summary:
   - Branches processed (with PR numbers)
   - Branches deleted (merged/closed PRs)
   - Conflict resolution details (which files had conflicts)
   - CI fix details (what was fixed)
   - Any branches that still need attention

---

## Rules

- **Never use `git push --force`** — always `--force-with-lease`.
- **Never skip hooks** (`--no-verify`) — if a hook fails, fix the underlying issue.
- **Never modify git config** (user.name, user.email, etc.).
- **Never include AI attribution** in any commits (no Co-Authored-By Claude, no "Generated by AI").
- **Do not amend commits that were already pushed** unless the rebase itself requires it.
- If `gh` CLI is not authenticated, stop and inform the user.
- If a rebase results in an empty branch (all changes already in main), inform the user and skip.
- When fixing CI, make minimal targeted fixes — do not refactor surrounding code.
- Commit messages for CI fixes should follow conventional format: `fix: <description>` or `ci: <description>`.
