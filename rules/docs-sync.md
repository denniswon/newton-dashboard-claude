# Docs Sync Rules

## Deduplication During Sync

When running `/docs-sync` or reviewing/updating files in the `.claude/` directory:

1. **Scan for duplicate content** across all files in `.claude/` and `.claude/rules/`. If two files describe the same concept, pattern, or rule, consolidate into one location and remove the duplicate.

2. **Within a single file**, check for repeated sections, paragraphs, or bullet points that say the same thing in different words. Keep the most precise version, remove the rest.

3. **Cross-file overlap**: `CLAUDE.md` (root and `.claude/CLAUDE.md`) should be a concise index and quick reference. Detailed rules belong in `.claude/rules/*.md` files. If `CLAUDE.md` duplicates detailed content from a rules file, keep the detail in the rules file and replace the `CLAUDE.md` section with a brief summary and reference.

4. **Root CLAUDE.md vs .claude/CLAUDE.md**: These two files should have identical content. If they diverge, treat `.claude/CLAUDE.md` as the source of truth (it's what gets synced via git).

## What Counts as a Duplicate

- Same rule stated in two different files (e.g., "never use .unwrap()" in both `rust.md` and `CLAUDE.md`)
- Same architectural description in both `architecture.md` and `CLAUDE.md` with equal detail
- Same table or command reference copy-pasted across files
- Same lesson in `lessons.md` and also restated as a rule in another file

## What is NOT a Duplicate

- A brief summary in `CLAUDE.md` that references a detailed rule in `rules/*.md` — this is the intended pattern
- The same concept applied to different contexts (e.g., "bounded collections" in `performance.md` for Rust vs `security.md` for DoS prevention)
- Cross-references between files that point to the same source of truth

## Sync Direction

The `.claude/` directory is its own git repository, synced bidirectionally with remote via `smart-sync-claude.sh`. The sync script handles:

- **Pull**: Remote changes are pulled with rebase. Conflicts reset to remote.
- **Push**: Local changes are committed only if content differs from remote (tree-level dedup). Identical content is never pushed.
- **Commit messages**: Descriptive, listing changed filenames (not just timestamps).
