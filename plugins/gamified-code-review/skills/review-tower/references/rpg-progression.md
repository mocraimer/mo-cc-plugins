# RPG Progression Reference

## Tier Thresholds

| Tier | XP Threshold | Unlocks |
|------|-------------|---------|
| 1 | 0 | Base game, all mechanics available |
| 2 | 500 | Badge showcase in session summary |
| 3 | 2,000 | Lifetime stats breakdown |
| 4 | 5,000 | Historical session comparison |
| 5 | 15,000 | Custom challenge modifiers |
| 6 | 50,000 | Full analytics dashboard |

Unlocks are informational/analytical only - they never change scoring mechanics.

## Theme Tables

### Medieval (default)

| Tier | Title |
|------|-------|
| 1 | Squire |
| 2 | Knight |
| 3 | Paladin |
| 4 | Champion |
| 5 | Warden |
| 6 | Archmage |

### Cyberpunk

| Tier | Title |
|------|-------|
| 1 | Script Kiddie |
| 2 | Hacker |
| 3 | Netrunner |
| 4 | Architect |
| 5 | Ghost |
| 6 | AI Overlord |

### Academic

| Tier | Title |
|------|-------|
| 1 | Intern |
| 2 | Analyst |
| 3 | Researcher |
| 4 | Senior Researcher |
| 5 | Principal |
| 6 | Distinguished Fellow |

## Session Badges

| Badge | Criteria | Behavior Rewarded |
|-------|----------|-------------------|
| Bug Hunter | Found 3+ findings of HIGH or CRITICAL severity | Hunting serious defects |
| Sentinel | 50%+ of findings were security-related | Security-focused reviewing |
| Eagle Eye | Challenged ALL floors (no scouting) | Deep independent review |
| Truth Seeker | Made 2+ correct disputes (Insight bonuses earned) | Critical AI evaluation |
| Deep Diver | Reviewed ALL floors (full tower completion) | Thoroughness/completeness |
| Lone Wolf | Found 3+ Human Edge findings | Uniquely human insights |
| Perfectionist | Found every AI-identified finding on at least one floor | Comprehensive floor coverage |

Badges are session-scoped (earned fresh each session), multiple can be earned per session, and all are positive (no shame badges).

## Stats Schema

```json
{
  "version": 1,
  "reviewer_id": "<generated-uuid>",
  "title_theme": "medieval",
  "current_title": "Squire",
  "current_tier": 1,
  "total_xp": 0,
  "sessions": {
    "total": 0,
    "towers_completed": 0,
    "towers_partial": 0,
    "floors_reviewed": 0
  },
  "findings": {
    "total": 0,
    "by_severity": { "critical": 0, "high": 0, "medium": 0, "low": 0 },
    "human_edge": 0,
    "disputes_made": 0,
    "disputes_correct": 0
  },
  "modes": {
    "challenge_floors": 0,
    "scout_floors": 0
  },
  "badges": {
    "bug_hunter": 0,
    "sentinel": 0,
    "eagle_eye": 0,
    "truth_seeker": 0,
    "deep_diver": 0,
    "lone_wolf": 0,
    "perfectionist": 0
  },
  "history": []
}
```

### History Entry Schema

Each entry in `history` (cap at 100, oldest dropped):

```json
{
  "session_id": "<uuid>",
  "pr_ref": "<owner/repo>#<number>",
  "date": "YYYY-MM-DD",
  "xp_earned": 0,
  "floors_reviewed": 0,
  "floors_total": 0,
  "completion": "full|partial|abandoned",
  "badges": []
}
```

### Tier Calculation

To determine tier from XP, find the highest threshold the reviewer's total_xp meets:
- >= 50000 -> Tier 6
- >= 15000 -> Tier 5
- >= 5000 -> Tier 4
- >= 2000 -> Tier 3
- >= 500 -> Tier 2
- otherwise -> Tier 1

### Reachability Estimates

Typical session: ~200-700 XP (midpoint ~300 XP)
- Tier 2: ~2 sessions
- Tier 3: ~7 sessions
- Tier 4: ~17 sessions
- Tier 5: ~50 sessions
- Tier 6: ~160 sessions
