# loop-recipes

11 autonomous recurring skills designed for `/loop` — self-healing debugger, social media content pipeline, proactive work dispatcher, manifest preference learner, plugin health monitor, daily briefing, focus guardian, dependency sentinel, changelog writer, ambient researcher, and workhorse (general-purpose autonomous task agent).

## Installation

```
/plugin marketplace add mocraimer/mo-cc-plugins
/plugin install loop-recipes@mo-cc-plugins
```

## Skills

### self-heal

Self-healing debugger that autonomously reproduces a bug, analyzes errors, and attempts fixes in isolated worktrees. Iterates until the test passes or 5 attempts are exhausted. Detects flaky tests.

```
/loop 2m /self-heal npm test -- --grep 'failing test'
```

**Expected behavior:** Each iteration runs the test, analyzes the failure, attempts a fix in a worktree, and reports progress. Stops when the fix is found, flakiness is detected, or 5 attempts are exhausted.

### content-pipeline

Monitors git activity and generates Reddit/LinkedIn-ready content drafts in a review queue (`~/.content-queue.md`). Never auto-posts. Skips iterations with no interesting activity.

```
/loop 30m /content-pipeline
```

**Expected behavior:** Each iteration scans recent commits, filters out mundane changes, drafts platform-specific content for interesting activity, and appends to the review queue. Runs indefinitely until stopped.

### dispatch-work

Triages open GitHub issues and suggests delegating suitable ones to isolated worktree agents. Requires user confirmation before dispatching. Checks for conflicts with current work.

```
/loop 10m /dispatch-work
```

**Expected behavior:** Each iteration fetches open issues, filters by quality and complexity, checks for conflicts with your current branch, and presents candidates for approval. Only dispatches after explicit confirmation.

### learn-preferences

Reads `/define` manifests and discovery logs, extracts recurring user preference patterns, and suggests CLAUDE.md updates after multi-session confirmation (2+ sessions). Never modifies CLAUDE.md without approval.

```
/loop 1h /learn-preferences
```

**Expected behavior:** Each iteration scans for recent manifests, extracts preference patterns, tracks candidates across sessions, and suggests CLAUDE.md additions for confirmed patterns. Limits suggestions to 2-3 per iteration.

### check-plugins

Validates installed Claude Code plugins, checks for marketplace updates, and surfaces issues by severity. Uses delta reporting — only alerts on new issues, not previously reported ones.

```
/loop 1h /check-plugins
```

**Expected behavior:** Each iteration scans installed plugins, validates structure, checks for updates, and reports new critical/warning issues. Info-level findings are logged silently. Resolves alert fatigue with delta comparison.

### daily-briefing

Generates a comprehensive development briefing from multiple data sources: git activity, GitHub PRs/issues, Gmail (optional), and environment health. Writes briefing to file. Supports user-configurable custom checks.

```
/loop 4h /daily-briefing
```

**Expected behavior:** Each iteration gathers data from all available sources (gracefully skipping unavailable ones), compiles a structured briefing, writes it to `~/.claude/loop-recipes/daily-briefings.md`, and outputs a one-line notification. Every 5th iteration asks about custom check configuration.

### focus-guardian

Detects scope drift by comparing recent file changes against a user-declared task. Nudges gently via AskUserQuestion with actionable options. Tracks dismissed drift events and never re-alerts for the same files.

```
/loop 5m /focus-guardian 'implement user auth for login page'
```

**Expected behavior:** Each iteration scans changed files, classifies them as on-scope, incidental, or off-scope relative to your declared task. Only alerts when off-scope files are detected. Respects dismissals.

### dep-sentinel

Auto-detects project ecosystems, runs security audits and outdated checks, classifies findings by severity, and uses delta reporting to only alert on new issues. Supports npm, pip, cargo, go, and more.

```
/loop 1h /dep-sentinel
```

**Expected behavior:** Each iteration scans for dependency manifests, runs available audit tools, classifies vulnerabilities by severity, and only reports new high+ findings. Gracefully skips ecosystems where audit tools aren't installed.

### changelog-writer

Generates human-readable changelog entries by analyzing actual code diffs (not just commit messages). Uses Keep-a-Changelog format. Accumulates entries across sessions without overwriting.

```
/loop 30m /changelog-writer
```

**Expected behavior:** Each iteration reads new commits, analyzes diffs, generates user-facing changelog entries grouped by category (Added, Changed, Fixed, Removed), and appends to `~/.claude/loop-recipes/changelog-draft.md`. Skips iterations with no new commits.

### ambient-researcher

Reads current development context (branch, recent changes, libraries in use), identifies relevant research topics, and suggests them via AskUserQuestion. Only researches topics you explicitly approve. Never re-suggests declined topics.

```
/loop 15m /ambient-researcher
```

**Expected behavior:** Each iteration reads your development context, identifies 2-3 useful research topics, and presents them for opt-in. When approved, uses web search to gather findings and writes structured research notes to `~/.claude/loop-recipes/research/`.

### workhorse

General-purpose autonomous task agent that takes arbitrary development tasks, plans them via `/define`, executes via `/do`, and manages a persistent task queue. Simulates OpenClaw-like autonomous agent behavior — recurring execution, persistent state, human approval gates, bounded retries, and graceful degradation.

```
/loop 10m /workhorse 'build user auth for login page'
```

**Expected behavior:** Each iteration advances the active task through its lifecycle: plan (autonomous `/define`), approve (user reviews manifest), execute (`/do` with worktree isolation), retry on failure (max 3), complete and advance queue. Multiple tasks can be queued. Use `/workhorse status` to view current state.

**State:** `~/.claude/loop-recipes/workhorse-state.md` (cross-session) + per-task directories under `~/.claude/loop-recipes/workhorse-tasks/`.

## Important: Session Scope

`CronCreate` and `/loop` jobs **only run while the Claude Code session is alive**. When you close the terminal, the session ends and all scheduled jobs stop. They do not persist across sessions.

This means:
- `/loop 30m /content-pipeline` will run every 30 minutes **only while that terminal stays open**
- Closing the terminal kills the loop — no background process survives
- Re-opening Claude Code requires re-creating the cron/loop

If you need a skill to run independently of your terminal session, set up a system-level cron job (`crontab -e`) that invokes the `claude` CLI directly.

## State Files

| Skill | State Location | Persistence |
|-------|---------------|-------------|
| self-heal | `/tmp/self-heal-state.md` | Session-scoped |
| content-pipeline | `~/.claude/loop-recipes/content-pipeline-state.md` | Cross-session |
| dispatch-work | `/tmp/dispatch-work-state.md` | Session-scoped |
| learn-preferences | `~/.claude/loop-recipes/preference-candidates.md` | Cross-session |
| check-plugins | `/tmp/check-plugins-state.md` | Session-scoped |
| daily-briefing | `~/.claude/loop-recipes/daily-briefing-state.md` | Cross-session |
| focus-guardian | `/tmp/focus-guardian-state.md` | Session-scoped |
| dep-sentinel | `/tmp/dep-sentinel-state.md` | Session-scoped |
| changelog-writer | `~/.claude/loop-recipes/changelog-writer-state.md` | Cross-session |
| ambient-researcher | `~/.claude/loop-recipes/ambient-researcher-state.md` | Cross-session |
| workhorse | `~/.claude/loop-recipes/workhorse-state.md` | Cross-session |

## License

MIT
