---
name: review-tower
description: "Gamified code review that turns PRs into towers with floors. Choose Challenge mode (independent review, 2x multiplier) or Scout mode (AI-guided, 1x). Earn XP, RPG titles, and session badges. Triggers: /review-tower, gamify code review, review PR as a game, tower review"
user-invokable: true
---

# Review Tower

Generate a browser-based gamified code review for a pull request. You are the AI reviewer — analyze the code, package your findings as JSON, inject into an HTML template, and open the result in the browser. The template handles scoring, progression, and stats persistence at runtime; your job ends after opening the HTML.

**References** (in `./references/`):
- `scoring-tables.md` — severity criteria, multiplier values, XP formulas
- `rpg-progression.md` — tier thresholds, titles, badges, stats schema
- `finding-matching-guide.md` — how reviewer findings match AI findings

## 1. Input

Parse `$ARGUMENTS` to determine input type:
- **PR reference** (URL, number, or empty for current branch): proceed to step 2.
- **JSON comments array** (pasted export from Review Tower): skip to step 7 (Post Findings to PR).

If `gh` is not installed, tell the user: `brew install gh && gh auth login`.

## 2. Fetch PR Data

Retrieve the PR metadata and full unified diff needed to populate the JSON schema below. If no PR found, ask for a number/URL. If the diff is empty, inform the user and stop.

## 3. Construct Tower

### Group Files into Floors
- Same directory/module → same floor
- Test files (`.test.`, `.spec.`, `__tests__/`) → group with their implementation files
- Type/interface files → group with implementations
- Config files (package.json, .env*, CI configs, tsconfig, .eslintrc) → dedicated Easy floor
- Cap at 10 floors (template UI constraint); merge by parent directory if needed
- Single-file floors are fine; don't force-merge unrelated files

### Assign Difficulty

A "concern" is a distinct logical change area (e.g., "refactoring auth middleware" and "adding rate limiting" are 2 concerns).

Per floor, total lines changed across its files:
- **Easy**: < 50 lines AND single concern
- **Normal**: 50-200 lines, OR 2+ concerns
- **Boss**: > 200 lines, OR security-sensitive paths (auth, payments, data migrations, public API contracts, DB schemas)

If tower has 3+ floors, ensure at least 1 Boss. Promote the floor with the most lines changed among non-Boss floors.

### Tower Complexity
| Floors | Tier | Multiplier |
|--------|------|------------|
| 1-2 | Tiny | 0.5x |
| 3-4 | Standard | 1.0x |
| 5-7 | Large | 1.2x |
| 8+ | Massive | 1.5x |

## 4. Analyze All Floors

For each floor, independently generate structured findings. Analyze as if each floor is a standalone review — minimize anchoring to other floors. Clean floors can have 0 findings.

Each finding:
- **file**: file path
- **startLine** / **endLine**: relevant line range in the new file
- **concern**: what's wrong or risky
- **severity**: CRITICAL / HIGH / MEDIUM / LOW (see `references/scoring-tables.md`)
- **category**: Security / Performance / Correctness / Maintainability / Style

## 5. Build JSON

Read `~/.claude/review-tower/stats.json` if it exists. Construct a JSON object matching this schema:

```typescript
{
  pr: {
    title: string,       // PR title
    url: string,         // PR URL
    number: number       // PR number
  },
  tower: {
    complexity: {
      tier: string,      // "Tiny" | "Standard" | "Large" | "Massive"
      multiplier: number // 0.5 | 1.0 | 1.2 | 1.5
    },
    floors: Array<{
      id: number,              // 1-indexed from bottom
      name: string,            // descriptive name (e.g. "Auth Routes")
      difficulty: string,      // "Easy" | "Normal" | "Boss"
      files: Array<{
        path: string,
        additions: number,
        deletions: number
      }>,
      diffs: Array<{
        file: string,          // file path
        diff: string           // raw unified diff for this file
      }>,
      findings: Array<{
        id: number,            // unique within floor, 1-indexed
        file: string,
        startLine: number,
        endLine: number,
        concern: string,
        severity: string,      // "CRITICAL" | "HIGH" | "MEDIUM" | "LOW"
        category: string       // "Security" | "Performance" | "Correctness" | "Maintainability" | "Style"
      }>
    }>
  },
  existingStats: object | null  // contents of stats.json if it existed, else null
}
```

### Splitting the Diff

The unified diff is a single output. Split it by file boundary (`diff --git a/... b/...`) and assign each file's diff to the correct floor's `diffs` array.

## 6. Generate HTML

Inject the JSON into `./template.html` by replacing the `'__TOWER_DATA__'` token (including surrounding single quotes) with the JSON object literal.

**Critical escaping requirements:**
1. Escape any `</script>` sequences in diff strings as `<\/script>` to prevent premature script tag closure.
2. **Use a function replacement** when calling `String.replace` — diff data contains `$` characters (`$schema`, `$&`, `$'`, etc.) that JavaScript interprets as special replacement patterns. Use `template.replace("'__TOWER_DATA__'", () => jsonLiteral)` instead of `template.replace("'__TOWER_DATA__'", jsonLiteral)`.

**Verify before opening:** Confirm the output HTML has exactly 1 `</html>` tag and 3 `</script>` tags (2 CDN + 1 main). If counts differ, the injection corrupted the HTML.

Write to `/tmp/review-tower-<pr-number>.html` and open in browser.

## 7. Post Findings to PR

When the user pastes export JSON from the Review Tower, post findings as PR review comments.

### Export JSON Schema

```json
{
  "pr": {
    "title": "string",
    "url": "string",
    "number": 123
  },
  "comments": [
    {
      "file": "src/example.ts",
      "line": 42,
      "severity": "HIGH",
      "category": "Correctness",
      "concern": "Description of the issue",
      "reviewerNotes": "Optional reviewer notes or null",
      "source": "matched | human_edge | ai_unmatched | scout_acknowledged"
    }
  ]
}
```

### Workflow

1. Parse the JSON. Extract `{owner}` and `{repo}` from `pr.url`.
2. Present findings in a table for the user: file, line, severity, concern, source. Wait for user confirmation before posting. Do not auto-post.
3. Get the PR head commit SHA:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number} --jq '.head.sha'
   ```
4. For each confirmed comment, post a PR review comment:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/comments --method POST \
     --field commit_id="{commit_sha}" \
     --field path="{file}" \
     --field line={line} \
     --field side="RIGHT" \
     --field body="**[{severity}] {category}**: {concern}

   {reviewerNotes if present}

   *Source: {source} — Review Tower*"
   ```
5. Report how many comments were posted successfully.

## 8. Edge Cases

| Situation | Handling |
|-----------|----------|
| 1-file PR | Single Normal floor, Tiny tower |
| Binary files (images, compiled assets) | Skip from diff analysis; include in floor file list with note "binary file" |
| Rename-only changes | Single Easy floor; note the rename in findings only if the new name is misleading |
| Deletion-only PR | Floors from deleted file directories; findings focus on orphaned references |
| Large PR (>3000 lines changed) | Use a script (Node.js/Python) to parse the diff, split by file, group into floors, and build the JSON skeleton. Launch parallel review agents for different floor groups to stay within context limits. Merge agent findings into the skeleton before HTML generation. |
