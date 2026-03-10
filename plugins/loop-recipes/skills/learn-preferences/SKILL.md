---
name: learn-preferences
description: "Manifest preference learner that reads /define manifests and discovery logs, extracts recurring user preference patterns, and suggests CLAUDE.md updates after multi-session confirmation. Designed for /loop — runs periodically to learn from your workflow. Usage: /loop 1h /learn-preferences"
user-invokable: true
---

# Manifest Preference Learner

You are a preference learning agent running as a recurring `/loop` iteration. Your job: read recent /define manifests and discovery logs, extract patterns in user preferences, track candidates across sessions, and suggest CLAUDE.md updates for confirmed patterns. You NEVER modify CLAUDE.md without explicit user approval.

## State Management

State file: `~/.claude/loop-recipes/preference-candidates.md` (persistent across sessions)

### On Start — Read State

1. Read `~/.claude/loop-recipes/preference-candidates.md`. If it does not exist, initialize:
   ```yaml
   ---
   status: idle
   last_scan: "1970-01-01T00:00:00Z"
   total_patterns_found: 0
   total_patterns_promoted: 0
   ---
   # Candidate Patterns

   ## Candidates (seen once — need confirmation)

   ## Confirmed (seen 2+ sessions — eligible for CLAUDE.md)

   ## Applied (already written to CLAUDE.md)
   ```

2. If `status: in-progress` with a `locked_by` field set:
   - If `locked_by` timestamp is less than 2 hours old: a previous iteration is still running. Output "Previous iteration still running — skipping." and **stop**.
   - If `locked_by` is older than 2 hours: treat as stale lock (previous iteration likely crashed), clear it, and proceed.

3. Set `locked_by: <current_timestamp>` and `status: in-progress`.

4. Ensure `~/.claude/loop-recipes/` directory exists (`mkdir -p`).

### On End — Write State

After every iteration:
- Clear `locked_by`, set `status: idle`
- Update `last_scan` to current timestamp
- Update pattern counts
- Preserve all existing candidates, confirmed, and applied entries

## Iteration Logic

### Step 1: Find Recent Manifests and Logs

Search for manifests and discovery logs. Note: these files live in `/tmp` and are ephemeral (lost on reboot).

```bash
find /tmp -name "manifest-*.md" -mtime -1 2>/dev/null
find /tmp -name "define-discovery-*.md" -mtime -1 2>/dev/null
```

If no files found: output "No recent manifests or discovery logs." **Stop.**

Read each file found.

### Step 2: Extract Preference Patterns

Analyze the manifests and discovery logs for patterns. Look for:

1. **Recurring user preferences:** Tools, frameworks, approaches the user consistently chooses
2. **Consistently rejected suggestions:** Things Claude suggested that the user pushed back on
3. **Corrections:** Where the user corrected Claude's assumptions or approach
4. **Implicit constraints:** Rules the user enforces without being asked (naming conventions, file organization, testing approaches)
5. **Process preferences:** How the user likes to work (iterative vs. big-bang, test-first vs. test-after, etc.)

For each pattern found, record:
- **Pattern:** One-sentence description of the preference
- **Evidence:** Specific quote or reference from the manifest/log
- **Source file:** Which manifest/log it came from
- **Session date:** When the session occurred

### Step 3: Check Against Existing Candidates

For each new pattern, compare against existing candidates in the state file:

- **Already a candidate?** → Increment its confirmation count. If now seen in 2+ distinct sessions, promote to "Confirmed."
- **Already confirmed or applied?** → Skip (already tracked).
- **New pattern?** → Add to "Candidates" section with confirmation count = 1.

**Important:** Two observations from the same session/manifest count as 1 confirmation. The threshold is 2+ *distinct sessions*.

### Step 4: Suggest CLAUDE.md Updates for Confirmed Patterns

If there are confirmed patterns not yet applied:

1. Determine the target CLAUDE.md: check for a project-level CLAUDE.md first (in the current repo root). If it exists, use it. If not, fall back to the global `~/.claude/CLAUDE.md`. Present the chosen target to the user when suggesting additions — they can override.
2. For each confirmed pattern, check:
   - Does a similar preference already exist in CLAUDE.md? → Skip (or suggest consolidation).
   - Does it conflict with an existing entry? → Flag the conflict, do not suggest.
   - Is it genuinely useful and specific enough to be actionable? → Include in suggestion.

3. Limit suggestions to **max 2-3 per iteration** to avoid overwhelming the user.

4. Present suggestions via AskUserQuestion:

```
Based on patterns observed across multiple /define sessions, I'd like to suggest these CLAUDE.md additions:

1. **<pattern>**
   Evidence: <quote from session 1>, <quote from session 2>
   Suggested entry: "<the CLAUDE.md line to add>"

2. ...

Which suggestions would you like to apply?
```

Options (2-4, depending on suggestion count):
- "Apply suggestion 1" (for each suggestion, up to 3)
- "Skip all — not now"

- If user selects a suggestion: use the Edit tool to merge that addition into CLAUDE.md (never overwrite with Write). Move the pattern to "Applied" in state.
- If user declines (selects "Skip all"): leave suggestions as confirmed for next iteration.

### Step 5: CLAUDE.md Growth Control

Before adding any entry, perform these checks:

1. **Deduplication:** Search CLAUDE.md for semantically similar entries. If found, suggest updating the existing entry instead of adding a new one.
2. **Consolidation:** If 2+ confirmed patterns are closely related, suggest a single consolidated entry.
3. **Conflict detection:** If a new pattern contradicts an existing CLAUDE.md entry, flag it and ask the user which should take precedence.
4. **Staleness check:** If the user has declined a pattern in a previous iteration, do not re-suggest it.

## Stop Conditions

This skill is designed to run periodically (e.g., hourly). The user should stop the loop when:
- They're done with their session
- They don't want preference learning running
- CLAUDE.md has been updated and they want to review the changes
