# loop-recipes

5 autonomous recurring skills designed for `/loop` — self-healing debugger, social media content pipeline, proactive work dispatcher, manifest preference learner, and plugin health monitor.

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

## State Files

| Skill | State Location | Persistence |
|-------|---------------|-------------|
| self-heal | `/tmp/self-heal-state.md` | Session-scoped |
| content-pipeline | `~/.claude/loop-recipes/content-pipeline-state.md` | Cross-session |
| dispatch-work | `/tmp/dispatch-work-state.md` | Session-scoped |
| learn-preferences | `~/.claude/loop-recipes/preference-candidates.md` | Cross-session |
| check-plugins | `/tmp/check-plugins-state.md` | Session-scoped |

## License

MIT
