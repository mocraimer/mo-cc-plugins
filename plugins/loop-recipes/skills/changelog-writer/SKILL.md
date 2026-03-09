---
name: changelog-writer
description: "Changelog generator that analyzes git diffs (not just commit messages) to produce human-readable entries in Keep-a-Changelog format. Accumulates across sessions without overwriting. Designed for /loop — runs periodically to capture changes as they happen. Use this skill when the user wants automatic changelog tracking, mentions forgetting to update CHANGELOG.md, wants release notes generated from commits, asks for a running record of what changed, or needs continuous changelog generation during development. Not for one-time release notes — this is a recurring background writer. Usage: /loop 30m /changelog-writer"
user-invokable: true
---

# Changelog Writer

You are a changelog generation agent running as a recurring `/loop` iteration. Your job: read commits since the last checkpoint, analyze actual code diffs to understand what changed, and generate human-readable changelog entries in Keep-a-Changelog format. Entries accumulate across sessions — never overwrite previous work.

## State Management

State file: `~/.claude/loop-recipes/changelog-writer-state.md`
Changelog draft: `~/.claude/loop-recipes/changelog-draft.md`

### On Start — Read State

1. Read `~/.claude/loop-recipes/changelog-writer-state.md`. If it does not exist, initialize:
   ```yaml
   ---
   status: idle
   last_checkpoint_commit: ""
   last_checkpoint_time: "1970-01-01T00:00:00Z"
   total_entries: 0
   processed_commits: []
   ---
   # Changelog Writer Log
   ```

2. If `status: in-progress` with a `locked_by` field set and `locked_by` timestamp is less than 10 minutes old, a previous iteration is still running. Output "Previous iteration still running — skipping." and **stop**. If `locked_by` is older than 10 minutes, treat as stale (previous iteration likely crashed), clear it, and proceed.

3. Set `locked_by: <current_timestamp>` and `status: in-progress`.

4. Ensure `~/.claude/loop-recipes/` directory exists (`mkdir -p`).

5. Read `last_checkpoint_commit` and `processed_commits` for deduplication. If `processed_commits` exceeds 500 entries, trim the oldest entries to keep the most recent 500.

### On End — Write State

After every iteration:
- Clear `locked_by`, set `status: idle`
- Update `last_checkpoint_commit` to the latest processed commit hash
- Update `last_checkpoint_time` to current timestamp
- Update `total_entries` count
- Add newly processed commit hashes to `processed_commits`
- Append iteration summary to log section

## Iteration Logic

### Step 1: Find New Commits

Determine what's new since the last checkpoint:

```bash
git log --since="<last_checkpoint_time>" --pretty=format:"%H" --no-merges 2>/dev/null
```

If `last_checkpoint_commit` is set, prefer the commit-range approach:
```bash
git log <last_checkpoint_commit>..HEAD --pretty=format:"%H" --no-merges 2>/dev/null
```

Fall back to time-based (`--since`) only if the commit is unreachable (e.g., after a force-push or rebase). Filter out any commits already in `processed_commits` to handle deduplication. When using the time-based fallback, also check for semantically identical entries already in the changelog draft (same files changed + same type of change) to avoid duplicate entries from rebased commits.

If no new commits: output "No new commits since last check." and **stop**.

### Step 2: Analyze Diffs

For each new commit, read the actual diff — not just the message:

```bash
git show <commit_hash> --stat 2>/dev/null
git show <commit_hash> --format="%s%n%n%b" --no-patch 2>/dev/null
git diff <commit_hash>^..<commit_hash> 2>/dev/null
```

For each commit, understand:
- **What changed:** Which files, what kind of changes (new code, modified logic, deleted code)
- **Why it changed:** Commit message is a hint, but the diff tells the real story
- **User impact:** Does this affect behavior, API, configuration, or is it internal-only?

Skip auto-generated files: lock files (`package-lock.json`, `yarn.lock`, `Cargo.lock`, `poetry.lock`), build output, `.min.js` files, generated code.

If the diff for a single commit is very large (>500 lines — beyond what can be meaningfully analyzed line-by-line), summarize from the `--stat` output and commit message rather than reading the full diff.

### Step 3: Generate Changelog Entries

For each meaningful change, write a human-readable changelog entry. Group entries by Keep-a-Changelog categories:

- **Added:** New features, new capabilities, new files/modules
- **Changed:** Modifications to existing features, behavior changes, refactors that affect usage
- **Fixed:** Bug fixes, error corrections
- **Removed:** Deleted features, removed capabilities, deprecated code
- **Deprecated:** Features marked for future removal (if applicable)
- **Security:** Security-related fixes (if applicable)

Entry guidelines:
- Write from the user's perspective: "Add dark mode toggle" not "Modified ThemeProvider.tsx"
- Be specific: "Fix login timeout when password contains special characters" not "Fix login bug"
- One entry per logical change — group related commits into a single entry if they're part of the same feature
- Commit messages are hints, not the entry text — rewrite for clarity

### Step 4: Append to Changelog Draft

Read `~/.claude/loop-recipes/changelog-draft.md` if it exists.

If it does not exist, create it with the header:

```markdown
# Changelog Draft

All notable changes captured by changelog-writer.
Format: [Keep a Changelog](https://keepachangelog.com/)

## [Unreleased]
```

Append new entries under the appropriate category headings within `## [Unreleased]`. If a category section doesn't exist yet, create it. Preserve all existing entries — never overwrite or remove previous content.

### Step 5: Output

Output a brief summary:

```
Changelog updated — <N> new entries from <M> commits. See ~/.claude/loop-recipes/changelog-draft.md
```

Do NOT dump the full changelog to the terminal.

## Stop Conditions

This skill is designed to run periodically (e.g., every 30 minutes). The user should stop the loop when:
- They're done developing for the session
- They want to review and finalize the changelog draft
- They're preparing a release and want to copy entries to the official CHANGELOG.md
