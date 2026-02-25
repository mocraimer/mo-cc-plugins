# Scoring Tables Reference

## Severity Base Points

| Severity | Base Points | Criteria |
|----------|-------------|----------|
| CRITICAL | 100 | Security vulnerabilities, data loss risk, crashes in production paths, auth/authz bypasses |
| HIGH | 50 | Logic errors producing wrong results, race conditions, resource leaks, missing error handling on critical paths |
| MEDIUM | 20 | Performance issues, maintainability concerns, missing edge cases, inconsistent patterns |
| LOW | 5 | Style issues, naming suggestions, minor doc gaps, unused imports |

## Multipliers

| Type | Key | Value |
|------|-----|-------|
| Floor: Easy | floor_multiplier | 0.8x |
| Floor: Normal | floor_multiplier | 1.0x |
| Floor: Boss | floor_multiplier | 1.5x |
| Mode: Challenge | mode_multiplier | 2.0x |
| Mode: Scout | mode_multiplier | 1.0x |
| Tower: Tiny (1-2 floors) | tower_complexity | 0.5x |
| Tower: Standard (3-4 floors) | tower_complexity | 1.0x |
| Tower: Large (5-7 floors) | tower_complexity | 1.2x |
| Tower: Massive (8+ floors) | tower_complexity | 1.5x |
| Human Edge bonus | finding_bonus | 1.5x |
| Insight (correct dispute) | insight_bonus | 0.75x on base only |

Floor difficulty and tower complexity assignment rules are defined in SKILL.md.

## Formulas

### Per-Finding Score

**Full Match (Challenge or Scout acknowledged)**:
```
finding_score = base_severity_pts x floor_multiplier x mode_multiplier
```

**Human Edge** (Challenge mode only):
```
finding_score = base_severity_pts x floor_multiplier x mode_multiplier x 1.5
```

**Insight Bonus** (correct false-positive dispute):
```
insight_bonus = base_severity_pts x 0.75
```
Note: Insight bonus has NO floor or mode multipliers.

### Aggregation

**Floor Total**:
```
floor_score = sum(all finding_scores) + sum(all insight_bonuses)
```

**Session XP**:
```
session_xp = sum(all floor_scores) x tower_complexity_multiplier
```

Round all scores to nearest integer, minimum 1.

## Worked Examples

### Challenge Mode, Boss Floor, Full Match on CRITICAL
```
100 (CRITICAL) x 1.5 (Boss) x 2.0 (Challenge) = 300
```

### Challenge Mode, Boss Floor, Human Edge on MEDIUM
```
20 (MEDIUM) x 1.5 (Boss) x 2.0 (Challenge) x 1.5 (Human Edge) = 90
```

### Scout Mode, Easy Floor, Acknowledged MEDIUM
```
20 (MEDIUM) x 0.8 (Easy) x 1.0 (Scout) = 16
```

### Scout Mode, Insight Bonus on LOW
```
5 (LOW) x 0.75 = 3.75 -> 4 (rounded)
```

### Maximum Possible Single-Finding Score
```
100 (CRITICAL) x 1.5 (Boss) x 2.0 (Challenge) x 1.5 (Human Edge) = 450
```

## Key Rules

1. AI classifies severity, not the reviewer (prevents inflation)
2. No penalty for misses (AI findings reviewer didn't catch = 0 pts, no deduction)
3. No penalty for unmatched reviewer findings (score 0, not negative)
4. Speed is NEVER scored - no timers, no time bonuses
5. In Scout mode, acknowledge and dispute are mutually exclusive per finding
6. Severity disputes are feedback only - they do NOT earn Insight bonus
