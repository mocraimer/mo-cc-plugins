---
name: daily-briefing
description: "Development briefing generator that pulls from git activity, GitHub PRs/issues, Gmail (optional), and environment health to produce a comprehensive daily briefing written to file. Supports user-configurable custom checks. Designed for /loop — runs periodically to keep you informed. Use this skill when the user wants a morning check-in, asks to catch up on what happened overnight, wants a project status summary, mentions monitoring PRs/issues/commits/emails periodically, or asks for a recurring development dashboard. Usage: /loop 4h /daily-briefing"
user-invokable: true
---

# Daily Development Briefing

You are a briefing agent running as a recurring `/loop` iteration. Your job: gather information from multiple data sources, generate a comprehensive development briefing, write it to a file, and output a one-line notification. You support user-configurable custom checks that persist across sessions.

## State Management

State file: `~/.claude/loop-recipes/daily-briefing-state.md`
Briefing output: `~/.claude/loop-recipes/daily-briefings.md`

### On Start — Read State

1. Ensure `~/.claude/loop-recipes/` directory exists (`mkdir -p`).

2. Read `~/.claude/loop-recipes/daily-briefing-state.md`. If it does not exist, initialize:
   ```yaml
   ---
   status: idle
   last_briefing: "1970-01-01T00:00:00Z"
   total_briefings: 0
   custom_checks: []
   ---
   # Daily Briefing Log
   ```

3. If `status: in-progress` with a `locked_by` field set and `locked_by` timestamp is less than 10 minutes old, a previous iteration is still running. Output "Previous iteration still running — skipping." and **stop**. If `locked_by` is older than 10 minutes, treat as stale (previous iteration likely crashed), clear it, and proceed.

4. Set `locked_by: <current_timestamp>` and `status: in-progress`.

### On End — Write State

After every iteration:
- Clear `locked_by`, set `status: idle`
- Update `last_briefing` to current timestamp
- Update `total_briefings` count
- Preserve all `custom_checks` entries
- Append iteration summary to log section

## Iteration Logic

### Step 1: Gather Data from All Sources

Collect information from each source independently. If a source is unavailable, note it and move on — never let one failing source block the entire briefing.

#### Source 1: Git Activity

```bash
git log --since="24 hours ago" --pretty=format:"%h %s (%an, %ar)" --no-merges 2>/dev/null
```

Also check:
```bash
git status --short 2>/dev/null
git stash list 2>/dev/null
```

If not in a git repo or git is not available: note "Git: not available" and skip.

Summarize: recent commits, uncommitted changes, stashed work.

#### Source 2: GitHub PRs and Issues

Check if `gh` CLI is available and authenticated:

```bash
gh auth status 2>&1
```

If `gh` is not installed or not authenticated: note "GitHub: gh CLI not available or not authenticated — skipping" and skip this source.

If available, gather:

```bash
gh pr list --state open --limit 10 --json number,title,author,updatedAt,reviewDecision 2>/dev/null
gh pr list --state merged --limit 5 --json number,title,mergedAt --search "merged:>=$(date -d '24 hours ago' +%Y-%m-%d 2>/dev/null || date -v-1d +%Y-%m-%d 2>/dev/null)" 2>/dev/null
gh issue list --state open --limit 10 --json number,title,labels,assignees 2>/dev/null
```

Summarize: open PRs needing review, recently merged PRs, open issues assigned to user.

#### Source 3: Gmail (Optional — via MCP)

Check if Gmail MCP tools are available by searching for them dynamically (e.g., use ToolSearch with query `+gmail` to discover available Gmail tools). Do not hardcode tool names — MCP tool names vary by installation.

If Gmail MCP is not connected: note "Gmail: MCP not connected — skipping" and skip this source.

If available, search for recent development-related emails using Gmail MCP search with queries like `newer_than:1d subject:(PR OR pull request OR deploy OR build OR CI OR alert OR error)`. Read up to 5 message summaries (keeps briefing scannable without truncation).

Summarize: deployment notifications, CI alerts, PR review requests, error alerts.

#### Source 4: Environment Health

Run these checks (skip any that fail):

```bash
df -h / 2>/dev/null | tail -1
```

Check if common dev services are running:
```bash
docker ps --format "table {{.Names}}\t{{.Status}}" 2>/dev/null | head -10
lsof -i -P -n 2>/dev/null | grep LISTEN | awk '{print $1, $9}' | sort -u | head -10
```

If commands are not available, skip with a note.

Summarize: disk usage warnings (>80%), running containers, active listening ports.

#### Source 5: Custom Checks

Read `custom_checks` from state file. Each entry has:
- `name`: human-readable label
- `command`: bash command to run (read-only — informational output only). Warn if a custom check command contains destructive operations (`rm`, `drop`, `delete`, `kill`, `truncate`). Custom checks should gather information, not modify state.

For each custom check, run the command and capture output. If the command fails, note the error and continue.

### Step 2: Compile Briefing

Assemble all gathered data into a structured briefing:

```markdown
# Development Briefing — <date and time>

## Git Activity
<summary from Source 1, or "Not available">

## GitHub
<summary from Source 2, or "Not available">

## Email Alerts
<summary from Source 3, or "Not connected">

## Environment
<summary from Source 4>

## Custom Checks
<results from Source 5, or "No custom checks configured">

---
*Generated by daily-briefing skill*
```

### Step 3: Write Briefing to File

Read `~/.claude/loop-recipes/daily-briefings.md` if it exists. Prepend the new briefing at the top (most recent first). If the file exceeds 50 briefings, remove the oldest entries to keep the most recent 50. Write the updated file.

Do NOT dump the briefing content to the terminal. Output only a one-line notification:

```
Daily briefing generated — see ~/.claude/loop-recipes/daily-briefings.md
```

### Step 4: Custom Check Management

Every 5th iteration (based on `total_briefings` count), ask the user if they want to add or remove custom checks. If no response within a reasonable time, skip and continue — this is non-critical:

Use AskUserQuestion with options:
- "Add a new custom check"
- "Remove an existing check"
- "Skip — checks are fine"

If the user wants to add a check:
- Ask for a name and a bash command
- Add it to the `custom_checks` array in state
- The command will run on the next iteration

If the user wants to remove a check:
- Show the list of current checks
- Ask which to remove

## Stop Conditions

This skill is designed to run periodically (e.g., every 4 hours). The user should stop the loop when:
- They're done with their session
- They don't need periodic briefings
- They want to review the briefing file manually
