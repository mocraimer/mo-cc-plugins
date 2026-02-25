# Finding Matching Guide

## Overview

Matching uses **binary region-based overlap**: if a reviewer's highlighted line range overlaps an AI finding's line range in the same file, it's a Full Match. No overlap means Human Edge (reviewer-only) or Miss (AI-only). There is no Partial Match tier.

## Decision Tree

```
For each REVIEWER finding:
  Does any unmatched AI finding in the SAME FILE have an
  overlapping line range?
  (reviewer.startLine <= ai.endLine AND reviewer.endLine >= ai.startLine)
  |
  +-- YES --> FULL MATCH (100% base severity)
  |           Prefer highest-severity AI finding if multiple overlap.
  |           Mark both as matched (1:1).
  |
  +-- NO  --> HUMAN EDGE (100% base x 1.5 bonus)

For each AI finding with NO reviewer match:
  --> MISS (0 points, no penalty)
```

## Line Range Overlap

Two findings overlap when they reference the **same file** AND their line ranges intersect:

```
reviewerStart <= aiEnd AND reviewerEnd >= aiStart
```

Examples:
- Reviewer [40-45] + AI [42-48] → overlap → Full Match
- Reviewer [10-15] + AI [30-35] → no overlap → separate
- Reviewer [20-20] + AI [20-25] → overlap (single line at boundary) → Full Match

## Highlight Size Cap

Reviewer highlights are capped at approximately **20 lines**. If a user selects more than 20 lines, the range is truncated to prevent overly broad matching.

## One-to-One Constraint

Each reviewer finding matches at most ONE AI finding, and vice versa. When multiple AI findings overlap a single reviewer highlight, prefer:
1. Higher severity (CRITICAL > HIGH > MEDIUM > LOW)
2. Greater overlap area (more shared lines)

## Matching Tiers

| Tier | Condition | Score |
|------|-----------|-------|
| Full Match | Same file, line ranges overlap | base × floor × mode |
| Human Edge | Reviewer finding, no AI overlap | base × floor × mode × 1.5 |
| Miss | AI finding, no reviewer match | 0 (no penalty) |

There is **no Partial Match** tier. Matching is binary: overlap or no overlap.

## Ambiguity Resolution

Default to the more generous interpretation. If a reviewer's highlight barely touches an AI finding's range (even a single line of overlap), it counts as a Full Match.

## Examples

### Full Match
- **Reviewer highlight**: auth.ts lines 42-45
- **AI finding**: auth.ts lines 43-48 (null check issue)
- **Result**: Overlap on lines 43-45 → FULL MATCH

### Human Edge
- **Reviewer highlight**: api.ts lines 100-105 (retry needs backoff)
- **AI findings**: none in api.ts lines 100-105
- **Result**: No overlap → HUMAN EDGE

### Miss
- **AI finding**: helpers.ts lines 1-1 (unused import)
- **Reviewer**: no highlight touching helpers.ts line 1
- **Result**: MISS (0 points, no penalty)

### Multiple Overlaps (1:1 rule)
- **Reviewer highlight**: routes.ts lines 20-30
- **AI finding #1**: routes.ts lines 22-25 (MEDIUM)
- **AI finding #2**: routes.ts lines 28-32 (HIGH)
- **Result**: Both overlap, but 1:1 constraint applies. Match with #2 (higher severity). #1 becomes a Miss.
