# Skill Evaluation: mo-cc-plugins

**Evaluated:** 2026-03-09
**Plugins:** gamified-code-review (1 skill), loop-recipes (11 skills)
**Total skills:** 12

---

## Executive Summary

**Total findings:** 65
- **P0 (broken/wrong):** 7
- **P1 (significant):** 41
- **P2 (polish):** 17

**Top 5 highest-impact findings:**

1. **P0** ambient-researcher: AskUserQuestion called with 5 options (tool limit is 4) - will fail at runtime
2. **P0** content-pipeline: Safety check runs AFTER content is written to queue, not before
3. **P0** marketplace.json and README: skill count says "10" but there are 11 loop-recipes skills
4. **P0** dispatch-work: AskUserQuestion instructions describe free-text input, incompatible with tool's structured options
5. **P0** review-tower: Edge cases section truncated - only 1 row in table, header says "Edge Cases" (plural)

**Cross-cutting note:** Lock timeout handling varies across skills. 6 have no stale-lock detection, others use 5/10/15 min thresholds. Low priority but worth standardizing.

**Triage results:**
- **Fixed (P0+P1):** 40
- **False positive:** 17
- **Deferred (P2):** 8

---

## Scope

**Covered:**
- All 12 SKILL.md files across both plugins
- review-tower template.html and 3 reference files
- All plugin.json files and marketplace.json
- README.md
- 6 evaluation dimensions: prompt quality, state management, error handling, edge cases, description accuracy, UX

**Not covered:**
- Runtime testing (no actual execution of skills)
- Claude Code platform internals (how skills are loaded/parsed)
- Comparison with other plugin ecosystems
- Performance benchmarking

---

## Methodology

Each skill is evaluated across 6 dimensions:

| Dimension | What "good" looks like |
|-----------|----------------------|
| **Prompt quality** | Instructions are unambiguous, constraints are explicit, the LLM can follow them without guessing. No conflicting guidance. |
| **State management** | State file structure is clear, lock handling prevents concurrent runs, data persists appropriately for the skill's purpose. |
| **Error handling** | Tool unavailability, malformed input, permission issues, and timeouts are handled. Failures degrade gracefully. |
| **Edge cases** | Unusual inputs, empty states, boundary conditions, and platform differences are covered. |
| **Description accuracy** | Frontmatter description matches actual behavior, triggers correctly, doesn't mislead. |
| **UX** | Output is appropriate (not too verbose, not too terse), user interaction follows tool constraints, stop conditions are clear. |

**Priority definitions:**
- **P0** - Broken or wrong: skill produces incorrect behavior, wrong metadata, or instructions that would reliably mislead the LLM
- **P1** - Significant improvement: meaningful gain in reliability, clarity, or user experience
- **P2** - Polish: nice-to-have, minor inconsistency, cosmetic

**Finding format:** Each finding is a ready-to-file GitHub issue with title, description, file reference, and suggested fix.

---

## Repo-Level Findings

#### F-R1: **[P0]** [FIXED] marketplace.json skill count and description are stale

> **Title:** fix: update marketplace.json to include workhorse skill and correct count
>
> **File:** `.claude-plugin/marketplace.json:16-18`
>
> The loop-recipes entry says "10 autonomous recurring skills" and lists them by name. The actual plugin has 11 skills (workhorse was added in v2.1.0). The version field also says "2.0.0" while `plugins/loop-recipes/.claude-plugin/plugin.json` says "2.1.0".
>
> **Fix:** Update description to "11 autonomous recurring skills...", add "and workhorse" to the skill list, bump version to "2.1.0".

#### F-R2: **[P0]** [FIXED] README mirrors stale marketplace.json data

> **Title:** fix: update README to reflect 11 loop-recipes skills
>
> **File:** `README.md:22`
>
> Same stale "10 autonomous recurring skills" text as marketplace.json. Missing workhorse from the description.
>
> **Fix:** Mirror the corrected marketplace.json description.

#### F-R3: **[P2]** [DEFERRED] gamified-code-review plugin.json missing `user-invokable` context

> **Title:** chore: add keywords for discoverability to gamified-code-review plugin.json
>
> **File:** `plugins/gamified-code-review/.claude-plugin/plugin.json`
>
> Keywords list is good but could include "browser" and "html" since it generates browser-based output. Minor discoverability improvement.
>
> **Fix:** Add "browser" and "html" to keywords array.

---

## Cross-Cutting Themes

### Theme 1: Inconsistent stale-lock detection

**Affected skills:** self-heal, content-pipeline, dispatch-work, learn-preferences, ambient-researcher, check-plugins (no stale-lock detection) vs daily-briefing (10 min), focus-guardian (5 min), dep-sentinel (15 min), changelog-writer (10 min), workhorse (dynamic).

Skills without stale-lock detection will permanently lock if a previous iteration crashes. The next iteration sees `locked_by` set and skips forever. The user would need to manually edit or delete the state file.

**Suggestion:** Add stale-lock detection to all skills. Use the skill's expected loop interval as the threshold (e.g., self-heal at 2min interval should use ~5min stale threshold). Or extract a shared "state management protocol" reference that all skills follow.

### Theme 2: Session-scoped state losing valuable data

**Affected skills:** self-heal, dispatch-work, check-plugins, dep-sentinel, focus-guardian (all use `/tmp/`).

dep-sentinel loses `previous_findings` on reboot, so delta reporting resets and the first post-reboot scan alerts on everything already known. check-plugins has the same issue. dispatch-work loses `dispatched_issues` and `skipped_issues`, so it re-suggests the same issues.

**Suggestion:** Move dep-sentinel, check-plugins, and dispatch-work to `~/.claude/loop-recipes/` for cross-session persistence. self-heal and focus-guardian are genuinely session-scoped (task-specific), so `/tmp/` is appropriate for them.

### Theme 3: AskUserQuestion option count violations

**Affected skills:** ambient-researcher (5 options), dispatch-work (free-text instruction), workhorse IDLE phase (free-text instruction).

AskUserQuestion is limited to 2-4 structured options. Skills that exceed this or describe free-text input will fail at runtime or produce unexpected behavior.

**Suggestion:** Restructure option sets to stay within the 2-4 limit. For ambient-researcher, use "Research selected topics" + "Skip all" (2 options) instead of listing each topic as an option. For dispatch-work and workhorse, provide structured options that cover the common cases.

### Theme 4: Description length variance

**Affected skills:** dep-sentinel (54 words), ambient-researcher (61 words), focus-guardian (52 words) vs self-heal (29 words), check-plugins (24 words).

Longer descriptions consume more context window tokens on every conversation where the skill is listed. Some descriptions include extensive trigger examples ("Use this skill when the user wants continuous vulnerability scanning, mentions CVE monitoring, asks about dependency security or outdated packages...") that could be shortened.

**Suggestion:** Cap descriptions at ~30-35 words. Move trigger examples to the SKILL.md body where they don't consume context on every conversation.

### Theme 5: Safety check ordering

**Affected skills:** content-pipeline.

content-pipeline runs the safety check (Step 8) after writing drafts to the queue (Step 7). Unsafe content gets persisted before being flagged.

**Suggestion:** Move safety checking to Step 6 (during generation) or between Steps 6 and 7 (before writing).

---

## Per-Skill Evaluations

---

### Skill: review-tower

**Plugin:** gamified-code-review v1.0.0
**Last evaluated:** 2026-03-09
**File:** `plugins/gamified-code-review/skills/review-tower/SKILL.md`

#### F-1: **[P0]** [FIXED] Edge cases table is truncated

> **Title:** fix: complete the edge cases table in review-tower SKILL.md
>
> **File:** `plugins/gamified-code-review/skills/review-tower/SKILL.md:165-170`
>
> Section 8 "Edge Cases" contains a table with only 1 row (1-file PR). The section heading and table structure imply there should be more entries. Missing cases: binary files in diff, deleted-only PRs, PRs with only renamed/moved files, PRs exceeding context window limits, PRs with > 10 floors before merging.
>
> **Fix:** Add rows for: binary files (skip from diff, note in floor), rename-only files (group with source directory), deletion-only PRs (still valid floor), PRs > 10k lines (summarize instead of including full diff), draft PRs (note status in tower header).

#### F-2: **[P1]** [FIXED] No routing logic between two distinct flows

> **Title:** feat: add explicit input routing between review generation and comment posting
>
> **File:** `plugins/gamified-code-review/skills/review-tower/SKILL.md:1-8`
>
> The skill handles two separate flows: (1) generate a review tower from a PR, (2) post exported findings to a PR from pasted JSON. There's no explicit routing at the top. The LLM must infer which flow based on whether `$ARGUMENTS` contains a PR reference or JSON. This works in practice but could fail if the user pastes malformed JSON or a JSON-like PR URL.
>
> **Fix:** Add a routing section after Input: "If $ARGUMENTS is valid JSON with a `comments` array, go to Section 7. Otherwise, treat as a PR reference and go to Section 2."

#### F-3: **[P1]** [FIXED] No guidance for large PRs exceeding context limits

> **Title:** feat: add context window handling for large PR diffs
>
> **File:** `plugins/gamified-code-review/skills/review-tower/SKILL.md:52-54`
>
> Section 4 says to analyze all floors independently but gives no guidance for PRs where the combined diff exceeds the LLM's effective working memory. A 5000-line diff across 20 files would degrade analysis quality.
>
> **Fix:** Add a threshold: "If total diff exceeds ~3000 lines, analyze floors sequentially rather than loading all at once. For individual files > 500 lines, summarize from --stat and focus analysis on the changed hunks."

#### F-4: **[P1]** [FALSE POSITIVE — CDN dependency is standard web practice] Template.html depends on CDN with no offline fallback

> **Title:** fix: add offline fallback or inline critical CSS/JS in review-tower template
>
> **File:** `plugins/gamified-code-review/skills/review-tower/template.html:8-10`
>
> template.html loads highlight.js and diff2html from CDN (cdnjs.cloudflare.com, cdn.jsdelivr.net). If the user is offline or behind a restrictive firewall, the review tower renders without syntax highlighting or diff formatting.
>
> **Fix:** Either inline the critical diff2html CSS/JS or add a note in the SKILL.md that the generated HTML requires internet access. Alternatively, check if the CDN resources are cached.

#### F-5: **[P1]** [FIXED] Missing `user-invokable: true` in frontmatter

> **Title:** fix: add user-invokable frontmatter to review-tower
>
> **File:** `plugins/gamified-code-review/skills/review-tower/SKILL.md:1-4`
>
> All loop-recipes skills have `user-invokable: true` but review-tower does not. This may affect discoverability depending on how the Claude Code skill loader treats this field.
>
> **Fix:** Add `user-invokable: true` to the YAML frontmatter.

#### F-6: **[P2]** [FALSE POSITIVE — LLM knows platform-specific open commands] No browser-open command specified

> **Title:** chore: specify cross-platform browser open command in review-tower
>
> **File:** `plugins/gamified-code-review/skills/review-tower/SKILL.md:113`
>
> "Write to /tmp/review-tower-<pr-number>.html and open in browser" gives no specific command. macOS uses `open`, Linux uses `xdg-open`, WSL has its own approach. The LLM will likely pick the right one, but an explicit instruction prevents guessing.
>
> **Fix:** Add: "Open with `open` (macOS) or `xdg-open` (Linux)."

#### F-7: **[P2]** [FALSE POSITIVE — finding says no immediate action] Template diff contrast relies on theme-specific overrides

> **Title:** chore: document theme-specific diff styling in template.html
>
> **File:** `plugins/gamified-code-review/skills/review-tower/template.html:422-481`
>
> Three separate blocks of diff2html CSS overrides for default, academic, and cyberpunk themes. Each has ~20 lines of `!important` overrides. This works but is fragile; any diff2html update could break the contrast. Well-handled for now (recent commit addressed this), noting for awareness.
>
> **Fix:** No immediate action. Consider extracting diff theme overrides into a CSS custom property layer if diff2html compatibility becomes an issue.

#### F-8: **[P2]** [FALSE POSITIVE — documentation awareness note, not actionable] Reference files lack cross-linking to template.html data contract

> **Title:** chore: add data contract note linking reference files to template.html expectations
>
> **File:** `plugins/gamified-code-review/skills/review-tower/references/*.md`
>
> The three reference files (scoring-tables.md, rpg-progression.md, finding-matching-guide.md) are well-structured with clear examples and consistent data. However, they don't document which values must match template.html's JavaScript constants. If someone updates a tier threshold in rpg-progression.md without updating the template, scoring breaks silently.
>
> **Fix:** Add a note to each reference file: "These values must stay in sync with template.html. When updating, check that the template's JS constants match."

---

### Skill: self-heal

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/self-heal/SKILL.md`

#### F-9: **[P1]** [FALSE POSITIVE — cd && cmd in single Bash call works correctly] `cd` in worktree conflicts with Claude Code shell behavior

> **Title:** fix: use absolute paths instead of cd for worktree test execution
>
> **File:** `plugins/loop-recipes/skills/self-heal/SKILL.md:88-89`
>
> Step 3 instructs `cd /tmp/self-heal-worktree-<attempt> && <test_command>`. Claude Code's Bash tool resets working directory between calls and discourages `cd`. The LLM may either avoid `cd` (running tests in wrong directory) or use it and lose working directory context for subsequent commands.
>
> **Fix:** Rewrite as a single command or use explicit paths: "Run the test command from the worktree directory using a single Bash call: `cd /tmp/self-heal-worktree-<attempt> && <test_command> 2>&1`"

#### F-10: **[P1]** [FIXED] "Escalate" is undefined

> **Title:** fix: define escalation path when all fix strategies are exhausted mid-iteration
>
> **File:** `plugins/loop-recipes/skills/self-heal/SKILL.md:76`
>
> Step 2 says "try a different angle or escalate" but there's no escalation protocol. Escalate to whom? How? This leaves the LLM in a vague state.
>
> **Fix:** Replace with: "try a different angle. If no viable approaches remain for this iteration, log the analysis and stop. The next iteration will have a fresh perspective."

#### F-11: **[P1]** [FIXED] No stale-lock detection

> **Title:** fix: add stale-lock detection to self-heal
>
> **File:** `plugins/loop-recipes/skills/self-heal/SKILL.md:37`
>
> If a previous iteration crashes, `locked_by` stays set forever. The loop interval is 2 min, so a lock older than ~5 min is clearly stale.
>
> **Fix:** Add after step 2: "If `locked_by` is set and its timestamp is more than 5 minutes old, treat as stale. Clear it and proceed."

#### F-12: **[P1]** [FIXED] No handling for pre-existing worktree branches

> **Title:** fix: handle pre-existing self-heal branches from crashed previous runs
>
> **File:** `plugins/loop-recipes/skills/self-heal/SKILL.md:81-83`
>
> If a previous iteration created `self-heal-attempt-3` and crashed before cleanup, the next run trying the same attempt number will fail with "branch already exists". No recovery logic.
>
> **Fix:** Before creating worktree, check: "If branch `self-heal-attempt-<attempt>` already exists, remove the worktree and delete the branch first."

#### F-13: **[P2]** [FALSE POSITIVE — finding says session scope is appropriate] Session-scoped state loses debugging context across reboots

> **Title:** chore: document session-scoped state trade-off in self-heal
>
> **File:** `plugins/loop-recipes/skills/self-heal/SKILL.md:19`
>
> State in `/tmp/` is lost on reboot. If debugging spans across work sessions, all attempt history and analysis is gone. However, self-heal is designed for focused debugging bursts within a single session, so `/tmp/` is the appropriate scope.
>
> **Fix:** Add a comment in the State Management section: "State is intentionally session-scoped. Each debugging session starts fresh. If you need to preserve context across sessions, copy the state file before rebooting."

#### F-14: **[P2]** [FALSE POSITIVE — Bash tool has 2min default timeout] No timeout for test commands

> **Title:** chore: add timeout guidance for test command execution
>
> **File:** `plugins/loop-recipes/skills/self-heal/SKILL.md:59-61`
>
> Test commands that hang (e.g., waiting for a database connection) will block the iteration indefinitely. The Bash tool has a 2-minute default timeout, which may or may not be appropriate.
>
> **Fix:** Add: "Set a timeout appropriate for the test command. If no explicit timeout, rely on the Bash tool's default. If a test hangs consistently, log it as a finding in the report."

---

### Skill: content-pipeline

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/content-pipeline/SKILL.md`

#### F-15: **[P0]** [FIXED] Safety check runs after content is written

> **Title:** fix: move content safety check before writing to queue
>
> **File:** `plugins/loop-recipes/skills/content-pipeline/SKILL.md:205-215`
>
> Step 8 (Content Safety Check) runs after Step 7 (Write to Review Queue). Unsafe content with private references gets persisted to `~/.content-queue.md` before any safety verification. The safety check only adds a warning note but the content is already on disk.
>
> **Fix:** Move safety check to between Steps 6 and 7. If safety check fails, either skip the draft or add the warning BEFORE writing. Rename to Step 6.5 or restructure numbering.

#### F-16: **[P1]** [FIXED] `-newermt` flag not available on macOS

> **Title:** fix: use cross-platform file modification time check for session scanning
>
> **File:** `plugins/loop-recipes/skills/content-pipeline/SKILL.md:69`
>
> `find ~/.claude/projects -maxdepth 2 -name "*.jsonl" -newermt "<timestamp>"` uses `-newermt` which is a GNU findutils extension. macOS `find` does not support `-newermt`. On macOS, this command silently finds nothing.
>
> **Fix:** Use a cross-platform approach: `find ~/.claude/projects -maxdepth 2 -name "*.jsonl" -newer <reference_file>` with a touch-ed reference file, or use `stat` to filter by mtime.

#### F-17: **[P1]** [FIXED] No stale-lock detection

> **Title:** fix: add stale-lock detection to content-pipeline
>
> **File:** `plugins/loop-recipes/skills/content-pipeline/SKILL.md:32`
>
> Same issue as self-heal: crashed iteration leaves permanent lock.
>
> **Fix:** Add stale-lock check with ~60 min threshold (loop interval is 30 min).

#### F-18: **[P1]** [FIXED] No guidance on JSONL format for session extraction

> **Title:** feat: add Claude Code session JSONL format hints for content extraction
>
> **File:** `plugins/loop-recipes/skills/content-pipeline/SKILL.md:76`
>
> "Extract the human-readable conversation flow" from JSONL files without any guidance on the format. The LLM must discover field names, message types, and structure by reading the file. This works but wastes context and may produce inconsistent extraction.
>
> **Fix:** Add a brief format hint: "Each line is a JSON object. Look for objects with `type: 'human'` or `type: 'assistant'` and extract their `content` field. Skip objects with `type: 'tool_use'` or `type: 'tool_result'`."

#### F-19: **[P1]** [FIXED] Content queue location inconsistent

> **Title:** fix: move content queue from home root to loop-recipes directory
>
> **File:** `plugins/loop-recipes/skills/content-pipeline/SKILL.md:14`
>
> Content queue is at `~/.content-queue.md` (home directory root) while the state file is at `~/.claude/loop-recipes/content-pipeline-state.md`. Every other skill with output files uses `~/.claude/loop-recipes/`.
>
> **Fix:** Move to `~/.claude/loop-recipes/content-queue.md`.

#### F-20: **[P2]** [FALSE POSITIVE — LLM judgment is the deduplication mechanism] Deduplication logic is vague

> **Title:** chore: add concrete deduplication criteria for git+session overlap
>
> **File:** `plugins/loop-recipes/skills/content-pipeline/SKILL.md:121`
>
> Step 4 says to deduplicate when commits and a session "concern similar files or topics" with no matching criteria. The LLM will make reasonable guesses but deduplication quality will vary.
>
> **Fix:** Add concrete signals: "Match if the session file list overlaps >50% with the commit's changed files, or if the session's topic keywords match the commit messages."

---

### Skill: dispatch-work

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/dispatch-work/SKILL.md`

#### F-21: **[P0]** [FIXED] AskUserQuestion instructions incompatible with tool constraints

> **Title:** fix: restructure dispatch-work user interaction to use structured options
>
> **File:** `plugins/loop-recipes/skills/dispatch-work/SKILL.md:99-113`
>
> Step 6 tells the user "Reply with issue numbers (e.g., '42, 17') or 'skip' to pass." This is free-text input, but AskUserQuestion requires 2-4 structured options. The skill can't accept arbitrary issue number combinations through the tool.
>
> **Fix:** Restructure to structured options: present top 1-3 issues with options like "Dispatch #42", "Dispatch #17", "Dispatch all", "Skip all". If the user needs to pick a specific combination, they can use "Other" to type it.

#### F-22: **[P1]** [FIXED] No stale-lock detection

> **Title:** fix: add stale-lock detection to dispatch-work
>
> **File:** `plugins/loop-recipes/skills/dispatch-work/SKILL.md:29`
>
> Crashed iteration leaves permanent lock. Loop interval is 10 min.
>
> **Fix:** Add stale-lock check with ~20 min threshold.

#### F-23: **[P1]** [FIXED] Session-scoped state loses dispatched/skipped issue history

> **Title:** feat: move dispatch-work state to cross-session storage
>
> **File:** `plugins/loop-recipes/skills/dispatch-work/SKILL.md:14`
>
> State in `/tmp/` means `dispatched_issues` and `skipped_issues` are lost on reboot. The next session re-suggests the same issues the user already declined or dispatched.
>
> **Fix:** Move to `~/.claude/loop-recipes/dispatch-work-state.md`.

#### F-24: **[P1]** [FALSE POSITIVE — LLM estimation is the intended mechanism] "Estimate which files would be touched" is speculative

> **Title:** fix: clarify conflict detection approach for issue-file overlap
>
> **File:** `plugins/loop-recipes/skills/dispatch-work/SKILL.md:92-96`
>
> Step 5 asks the LLM to "estimate which files would be touched" for an open issue. Without reading the codebase in the context of each issue, this is unreliable guessing. The LLM might not have enough context to map issue descriptions to file paths.
>
> **Fix:** Acknowledge the limitation: "Estimate files from issue description, labels, and file paths mentioned in the issue body. If the issue doesn't reference specific files, skip the conflict check and note 'conflict status: unknown' in the suggestion."

#### F-25: **[P2]** [DEFERRED] No pagination for issues beyond first 20

> **Title:** chore: add note about issue pagination in dispatch-work
>
> **File:** `plugins/loop-recipes/skills/dispatch-work/SKILL.md:58`
>
> `gh issue list --limit 20` only fetches 20 issues. If all 20 are in dispatched/skipped, no candidates remain but there may be more issues beyond page 1.
>
> **Fix:** Add: "If all fetched issues are already processed, try `--limit 50` to find new ones. If still none, report 'No new actionable issues.'"

---

### Skill: learn-preferences

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/learn-preferences/SKILL.md`

#### F-26: **[P1]** [FIXED] Source artifacts are session-scoped but pattern learning is cross-session

> **Title:** fix: address /tmp manifest ephemerality in learn-preferences
>
> **File:** `plugins/loop-recipes/skills/learn-preferences/SKILL.md:55-57`
>
> Step 1 searches `/tmp` for manifests and discovery logs. These files are session-scoped and may be cleaned between sessions. The skill's cross-session pattern learning depends on being able to read manifests from /tmp, but if the user reboots between sessions, the previous session's manifests are gone. Patterns that appeared in session 1 can never be confirmed by session 2.
>
> **Fix:** Either (a) copy manifests to a persistent location when first discovered, or (b) extract and persist pattern evidence immediately rather than re-reading source files. Or (c) note this limitation and accept that patterns must be confirmed within the same /tmp lifecycle.

#### F-27: **[P1]** [FIXED] No stale-lock detection

> **Title:** fix: add stale-lock detection to learn-preferences
>
> **File:** `plugins/loop-recipes/skills/learn-preferences/SKILL.md:34`
>
> Loop interval is 1h. A crashed iteration locks permanently.
>
> **Fix:** Add stale-lock check with ~2h threshold.

#### F-28: **[P1]** [FIXED] "Which CLAUDE.md" is ambiguous

> **Title:** fix: specify which CLAUDE.md to read and modify in learn-preferences
>
> **File:** `plugins/loop-recipes/skills/learn-preferences/SKILL.md:93`
>
> Step 4 says "Read the existing CLAUDE.md file(s) in the project" but the skill runs from any directory. It could find a project-level CLAUDE.md, a global `~/.claude/CLAUDE.md`, or none. The skill doesn't specify which one to modify.
>
> **Fix:** Specify: "Read the project-level CLAUDE.md in the current working directory. If none exists, check for `~/.claude/CLAUDE.md` (global). Present found CLAUDE.md path(s) to the user and let them choose which to modify."

#### F-29: **[P1]** [FALSE POSITIVE — LLM pattern extraction is the design] Pattern extraction criteria too vague

> **Title:** feat: add concrete pattern recognition heuristics to learn-preferences
>
> **File:** `plugins/loop-recipes/skills/learn-preferences/SKILL.md:65-77`
>
> Step 2 lists 5 categories of patterns but no concrete heuristics. "Recurring user preferences" and "Implicit constraints" are entirely LLM judgment. One run might identify 0 patterns, another might identify 20, depending on the LLM's interpretation.
>
> **Fix:** Add concrete signals: "Look for: (1) AskUserQuestion responses where user chose non-recommended option (indicates preference), (2) User corrections starting with 'No' or 'Actually' (indicates wrong assumption), (3) Process Guidance items that appear in 2+ manifests (indicates workflow preference), (4) Global Invariants that repeat across manifests (indicates quality standard)."

#### F-30: **[P2]** [DEFERRED] Cross-project manifest confusion

> **Title:** chore: add project context tracking to pattern extraction
>
> **File:** `plugins/loop-recipes/skills/learn-preferences/SKILL.md:55-57`
>
> All /define manifests in /tmp follow the same naming pattern regardless of project. If the user runs /define in project A and project B, patterns extracted from project A's manifests could incorrectly influence project B's CLAUDE.md.
>
> **Fix:** When extracting patterns, note the working directory context from the manifest (if available) and only suggest CLAUDE.md updates relevant to the current project context.

---

### Skill: check-plugins

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/check-plugins/SKILL.md`

#### F-31: **[P1]** [FIXED] Session-scoped state defeats delta reporting

> **Title:** feat: move check-plugins state to cross-session storage
>
> **File:** `plugins/loop-recipes/skills/check-plugins/SKILL.md:12`
>
> State in `/tmp/` means `previous_findings` is lost on reboot. First iteration after reboot alerts on ALL existing issues (no delta). This defeats the core value proposition of delta reporting.
>
> **Fix:** Move to `~/.claude/loop-recipes/check-plugins-state.md`.

#### F-32: **[P1]** [FIXED] Orphan detection logic assumes per-skill declarations

> **Title:** fix: clarify orphan detection logic for directory-based skill discovery
>
> **File:** `plugins/loop-recipes/skills/check-plugins/SKILL.md:88-89`
>
> Step 3.4 checks for "skill/agent/hook directories not referenced by plugin.json." But plugin.json typically declares `"skills": "./skills"` (a directory), not individual skills. The orphan detection would need to compare the skill subdirectories against what Claude Code actually loads, which is discoverable only by scanning the directory.
>
> **Fix:** Rewrite: "Check for subdirectories under the skills path that don't contain a SKILL.md file. These are likely orphaned or misconfigured."

#### F-33: **[P1]** [FIXED] No stale-lock detection

> **Title:** fix: add stale-lock detection to check-plugins
>
> **File:** `plugins/loop-recipes/skills/check-plugins/SKILL.md:27`
>
> Loop interval is 1h. Crashed iteration locks permanently.
>
> **Fix:** Add stale-lock check with ~2h threshold.

#### F-34: **[P1]** [FIXED] Marketplace path assumptions may be wrong

> **Title:** fix: handle dynamic marketplace directory structure in check-plugins
>
> **File:** `plugins/loop-recipes/skills/check-plugins/SKILL.md:63`
>
> Step 2 assumes `~/.claude/plugins/marketplaces/*/` exists and contains marketplace manifests. This path is not documented by Claude Code and may change across versions. If the path doesn't exist, the skill silently finds nothing.
>
> **Fix:** Add: "If no marketplace directories found, note 'Marketplace sources not found; update checking unavailable' and skip Step 2. Don't treat this as an error."

#### F-35: **[P2]** [DEFERRED] `/plugin validate <name>` may not be a real command

> **Title:** chore: verify plugin validate command exists before suggesting it
>
> **File:** `plugins/loop-recipes/skills/check-plugins/SKILL.md:131`
>
> The report suggests "Run `/plugin validate <name>`" but this command may not exist in all Claude Code versions. Suggesting non-existent commands erodes user trust.
>
> **Fix:** Either verify the command exists or change to a generic suggestion: "Investigate the issues manually or reinstall the plugin."

---

### Skill: daily-briefing

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/daily-briefing/SKILL.md`

#### F-36: **[P1]** [FIXED] Gmail MCP tool name may be wrong

> **Title:** fix: use dynamic tool discovery instead of hardcoded gmail tool name
>
> **File:** `plugins/loop-recipes/skills/daily-briefing/SKILL.md:88`
>
> "Check availability by attempting `gmail_get_profile`" hardcodes a specific MCP tool name. If the Gmail MCP server uses different tool names (e.g., `mcp__gmail__get_profile`), this check fails and Gmail is always skipped.
>
> **Fix:** Replace with: "Check if any Gmail-related MCP tools are available by searching for tools with 'gmail' in their name. If found, use them. If not, skip with a note."

#### F-37: **[P1]** [FIXED] Briefing file grows unboundedly

> **Title:** fix: add rotation/size cap to daily-briefings.md
>
> **File:** `plugins/loop-recipes/skills/daily-briefing/SKILL.md:150`
>
> Every iteration prepends a new briefing to `~/.claude/loop-recipes/daily-briefings.md`. Over weeks of use, this file grows very large. No rotation or cleanup.
>
> **Fix:** Add: "If the briefing file exceeds 50 briefings, remove the oldest entries beyond 50."

#### F-38: **[P1]** [FIXED] Custom check commands not validated for safety

> **Title:** fix: add read-only validation guidance for custom check commands
>
> **File:** `plugins/loop-recipes/skills/daily-briefing/SKILL.md:118-120`
>
> The skill says custom checks are "read-only - informational output only" but there's no validation. A user could add `rm -rf /` as a custom check and the skill would execute it.
>
> **Fix:** Add: "Before adding a custom check, verify the command doesn't modify files (no rm, mv, sed -i, write operations). If the command appears destructive, warn the user and ask for confirmation."

#### F-39: **[P2]** [FALSE POSITIVE — command already has 2>/dev/null] `lsof` may require elevated privileges

> **Title:** chore: add fallback for lsof permission issues in daily-briefing
>
> **File:** `plugins/loop-recipes/skills/daily-briefing/SKILL.md:108`
>
> `lsof -i -P -n` may produce permission errors or incomplete output on some systems. The skill already says "skip any that fail" which covers this, but could be more explicit.
>
> **Fix:** Add `2>/dev/null` to the lsof command (already present on other commands but missing here).

#### F-40: **[P2]** [DEFERRED] Git time window is hardcoded to 24 hours regardless of loop interval

> **Title:** chore: make daily-briefing git lookback window configurable
>
> **File:** `plugins/loop-recipes/skills/daily-briefing/SKILL.md:53`
>
> `git log --since="24 hours ago"` is hardcoded. If the loop interval is 4 hours, the skill re-summarizes the same 24-hour window each time, producing duplicate content. If the user runs it less frequently, commits older than 24h are missed.
>
> **Fix:** Use `last_briefing` timestamp from state as the lookback start: `git log --since="<last_briefing>"`. Fall back to 24h only on first run.

---

### Skill: focus-guardian

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/focus-guardian/SKILL.md`

#### F-41: **[P1]** [FALSE POSITIVE — LLM understands semantic similarity] Task change detection is too literal

> **Title:** fix: use semantic similarity for task change detection in focus-guardian
>
> **File:** `plugins/loop-recipes/skills/focus-guardian/SKILL.md:39`
>
> "If the $ARGUMENTS task description differs from the stored task, clear alerted_files and dismissed_files (new task = fresh scope)." This is a string comparison. Minor rewording ("implement auth" vs "add authentication") resets all state even though the task is the same.
>
> **Fix:** Add: "Compare task descriptions semantically, not literally. If the new description appears to describe the same task with different wording, preserve existing state. If clearly a different task, reset."

#### F-42: **[P1]** [FALSE POSITIVE — LLM judgment IS the feature] Relevance assessment is pure LLM judgment with no anchoring

> **Title:** feat: add concrete relevance heuristics to focus-guardian
>
> **File:** `plugins/loop-recipes/skills/focus-guardian/SKILL.md:66-77`
>
> Step 2 lists generic considerations (file path, file type, cross-cutting files) but provides no concrete rules. The LLM's classification will be inconsistent across iterations. A file might be "on-scope" in one iteration and "off-scope" in the next.
>
> **Fix:** Add concrete heuristics: "Start by identifying the task's primary directories from on-scope files in previous iterations. New files in those directories are likely on-scope. Files in completely unrelated directories are likely off-scope. When uncertain, classify as incidental (don't alert)."

#### F-43: **[P1]** [FALSE POSITIVE — finding says no code change needed] Session-scoped state is appropriate but has a nuance

> **Title:** chore: document why focus-guardian uses session-scoped state
>
> **File:** `plugins/loop-recipes/skills/focus-guardian/SKILL.md:19`
>
> Session-scoped (/tmp/) is correct for focus-guardian since each task declaration is session-specific. But if the user restarts Claude Code mid-task without changing the task description, alerted/dismissed files are lost and the same files get re-flagged.
>
> **Fix:** Note this is acceptable behavior. Re-flagging after restart is a reasonable trade-off for session isolation. No code change needed, but add a brief comment in the state section.

#### F-44: **[P2]** [DEFERRED] No handling for repos with no commits

> **Title:** chore: handle zero-commit repos in focus-guardian
>
> **File:** `plugins/loop-recipes/skills/focus-guardian/SKILL.md:55-59`
>
> `git diff --name-only HEAD` fails if the repo has no commits. The error would be caught by `2>/dev/null` but the file list would be incomplete (only untracked files).
>
> **Fix:** Add: "If `git diff HEAD` fails (no commits), rely on `git ls-files --others` only."

---

### Skill: dep-sentinel

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/dep-sentinel/SKILL.md`

#### F-45: **[P1]** [FIXED] Session-scoped state is the worst fit in the collection

> **Title:** fix: move dep-sentinel state to cross-session storage
>
> **File:** `plugins/loop-recipes/skills/dep-sentinel/SKILL.md:12`
>
> dep-sentinel's entire value is delta reporting: only alerting on NEW vulnerabilities. With state in `/tmp/`, every reboot resets `previous_findings` and the first scan alerts on ALL known vulnerabilities. Users learn to ignore the first post-reboot scan, which defeats the purpose.
>
> **Fix:** Move to `~/.claude/loop-recipes/dep-sentinel-state.md`. This is the highest-value state persistence change in the collection.

#### F-46: **[P1]** [FIXED] 8 separate find commands could be consolidated

> **Title:** fix: consolidate ecosystem detection into fewer commands
>
> **File:** `plugins/loop-recipes/skills/dep-sentinel/SKILL.md:49-57`
>
> Step 1 runs 8 separate `find` commands. The LLM might run these sequentially (8 tool calls) instead of in parallel. Each find is fast, but the cumulative overhead adds up.
>
> **Fix:** Consolidate: `find . -maxdepth 3 \( -name "package.json" -o -name "requirements.txt" -o -name "pyproject.toml" -o -name "Cargo.toml" -o -name "go.mod" -o -name "Gemfile" -o -name "pom.xml" -o -name "build.gradle" -o -name "composer.json" \) -not -path "*/node_modules/*" 2>/dev/null`

#### F-47: **[P1]** [FIXED] "Most audit tools handle workspaces natively" is misleading

> **Title:** fix: clarify monorepo audit behavior in dep-sentinel
>
> **File:** `plugins/loop-recipes/skills/dep-sentinel/SKILL.md:59`
>
> "Most audit tools handle workspaces natively" is true for npm workspaces but not for unrelated package.json files in a monorepo without workspace config. Running `npm audit` from project root in a non-workspace monorepo would audit the root only.
>
> **Fix:** Replace with: "If the project uses npm workspaces (check root package.json for `workspaces` field), run audit from root. Otherwise, run audit in each package.json directory separately."

#### F-48: **[P2]** [FALSE POSITIVE — Bash tool default timeout handles this] No timeout for audit commands

> **Title:** chore: note potential timeout for slow audit tools
>
> **File:** `plugins/loop-recipes/skills/dep-sentinel/SKILL.md:74-107`
>
> `govulncheck ./...` can take several minutes on large Go projects. `npm audit` with network issues can hang. No timeout guidance.
>
> **Fix:** Add: "If an audit command takes more than 2 minutes, it may have hung. The Bash tool's default timeout should handle this, but note the timeout in the log."

---

### Skill: changelog-writer

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/changelog-writer/SKILL.md`

#### F-49: **[P1]** [FIXED] `processed_commits` array grows unboundedly

> **Title:** fix: add rotation cap to processed_commits in changelog-writer state
>
> **File:** `plugins/loop-recipes/skills/changelog-writer/SKILL.md:25`
>
> `processed_commits` accumulates every commit hash ever processed. After months of use, this array becomes very large, bloating the YAML frontmatter. The state file could become slow to parse and consume significant tokens when read.
>
> **Fix:** Cap at last 500 commits: "When adding new commits to `processed_commits`, if the array exceeds 500 entries, remove the oldest entries beyond 500."

#### F-50: **[P1]** [FIXED] No handling for squashed/rebased commit history

> **Title:** fix: handle force-pushed branches in changelog-writer
>
> **File:** `plugins/loop-recipes/skills/changelog-writer/SKILL.md:58-63`
>
> The range-based approach (`last_checkpoint_commit..HEAD`) fails silently when the checkpoint commit no longer exists (after force push or rebase). The skill falls back to time-based, which is correct, but may re-process commits that were already in `processed_commits` under different hashes.
>
> **Fix:** Add: "After a rebase, processed_commits hashes from the old history won't match the new history. For the time-based fallback, rely on content deduplication (don't create changelog entries for changes that are semantically identical to existing entries) rather than commit hash matching."

#### F-51: **[P1]** [FALSE POSITIVE — LLM grouping IS the design] "Group related commits" criteria is vague

> **Title:** feat: add commit grouping heuristics to changelog-writer
>
> **File:** `plugins/loop-recipes/skills/changelog-writer/SKILL.md:100`
>
> "Group related commits into a single entry if they're part of the same feature" provides no concrete criteria for what "same feature" means.
>
> **Fix:** Add: "Group commits that: (1) have the same conventional commit prefix and scope (e.g., `feat(auth):...`), (2) touch the same set of files within a short time window, or (3) reference the same issue number."

#### F-52: **[P2]** [FALSE POSITIVE — working as designed] changelog-draft.md could conflict with existing CHANGELOG.md

> **Title:** chore: document relationship between changelog-draft.md and project CHANGELOG.md
>
> **File:** `plugins/loop-recipes/skills/changelog-writer/SKILL.md:104-118`
>
> The skill writes to `~/.claude/loop-recipes/changelog-draft.md` which is separate from any project-level CHANGELOG.md. The stop conditions mention "copy entries to the official CHANGELOG.md" but don't explain the workflow for doing so.
>
> **Fix:** Add a brief note: "This draft is a staging area. To finalize: review the draft, edit as needed, then copy entries to your project's CHANGELOG.md."

---

### Skill: ambient-researcher

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/ambient-researcher/SKILL.md`

#### F-53: **[P0]** [FIXED] AskUserQuestion called with 5 options (tool limit is 4)

> **Title:** fix: reduce ambient-researcher topic selection to 4 options max
>
> **File:** `plugins/loop-recipes/skills/ambient-researcher/SKILL.md:106-111`
>
> Step 3 presents 5 options: "Research topic 1", "Research topic 2", "Research topic 3", "Research all", "Skip all". AskUserQuestion supports 2-4 options. This will cause a runtime error or the tool will reject the call.
>
> **Fix:** Reduce to 4: "Research all topics (Recommended)", "Research topic 1 only", "Research topic 2 only", "Skip all". The user can use "Other" for specific combinations.

#### F-54: **[P1]** [FIXED] Declining a topic is permanent and too aggressive

> **Title:** fix: make topic decline session-scoped, not permanent
>
> **File:** `plugins/loop-recipes/skills/ambient-researcher/SKILL.md:115-117`
>
> Step 4: "Add unselected topics to `declined_topics`." If the user selects topic 1 and skips 2 and 3, topics 2 and 3 are permanently declined and never re-suggested. The user just didn't pick them now. They might want them later.
>
> **Fix:** Only add to `declined_topics` when the user explicitly chooses "Skip all" or individually declines. Unselected topics should be re-suggested in future iterations (perhaps with reduced frequency).

#### F-55: **[P1]** [FIXED] No stale-lock detection

> **Title:** fix: add stale-lock detection to ambient-researcher
>
> **File:** `plugins/loop-recipes/skills/ambient-researcher/SKILL.md:32`
>
> Loop interval is 15 min. Crashed iteration locks permanently.
>
> **Fix:** Add stale-lock check with ~30 min threshold.

#### F-56: **[P1]** [FIXED] Uses `cat` via Bash instead of Read tool

> **Title:** fix: use Read tool instead of cat for dependency file reading
>
> **File:** `plugins/loop-recipes/skills/ambient-researcher/SKILL.md:68-71`
>
> Step 1 uses `cat package.json 2>/dev/null | head -30`. Claude Code explicitly prefers the Read tool over cat for reading files. This instruction fights the platform conventions.
>
> **Fix:** Replace with: "Read package.json (first 30 lines), requirements.txt (first 20 lines), or Cargo.toml (first 20 lines) if they exist."

#### F-57: **[P2]** [DEFERRED] Description is very long (61 words)

> **Title:** chore: shorten ambient-researcher description to save context tokens
>
> **File:** `plugins/loop-recipes/skills/ambient-researcher/SKILL.md:3`
>
> The description includes extensive trigger examples that consume context on every conversation where the skill is listed. At 61 words, it's one of the longest in the collection.
>
> **Fix:** Shorten to ~30 words: "Context-aware research assistant that suggests relevant topics based on your current development context. Only researches what you opt into. Designed for /loop. Usage: /loop 15m /ambient-researcher"

---

### Skill: workhorse

**Plugin:** loop-recipes v2.1.0
**Last evaluated:** 2026-03-09
**File:** `plugins/loop-recipes/skills/workhorse/SKILL.md`

#### F-58: **[P1]** [FIXED] PLANNING phase bypasses /define's interview process

> **Title:** fix: improve workhorse planning quality by not bypassing /define interview
>
> **File:** `plugins/loop-recipes/skills/workhorse/SKILL.md:98-106`
>
> PLANNING invokes /define with `"no questions. <description>"`. This tells /define to skip the interview, producing shallow manifests without discovered requirements, edge cases, or pre-mortem scenarios. The manifest quality will be significantly lower than an interactive /define session.
>
> **Fix:** Instead of "no questions", use "Generate a manifest for: <description>. Use your best judgment for all decisions. Select the recommended option for every choice. Log all decisions." This lets /define run its full analysis autonomously without needing user input.

#### F-59: **[P0]** [FIXED] IDLE phase describes free-text input for AskUserQuestion

> **Title:** fix: use structured options for workhorse IDLE phase task input
>
> **File:** `plugins/loop-recipes/skills/workhorse/SKILL.md:87-91`
>
> IDLE says "Options: free-text task description, or 'Go idle'" but AskUserQuestion requires structured options (2-4), not free-text. The LLM can't present a free-text input field through this tool.
>
> **Fix:** Restructure: Present options like "I have a task to add" (then use a follow-up question to get specifics), "Go idle", "Show status". Or skip asking entirely and just report "Queue empty. To add work: `/loop 10m /workhorse 'task description'`"

#### F-60: **[P1]** [FIXED] Phase machine complexity may confuse LLM across iterations

> **Title:** feat: add phase machine decision tree to workhorse for LLM clarity
>
> **File:** `plugins/loop-recipes/skills/workhorse/SKILL.md:79-221`
>
> The skill has 6 phases (IDLE, PLANNING, APPROVAL, EXECUTING, RETRYING, COMPLETE) with transitions between them. Across iterations, the LLM reads a long SKILL.md and must locate the right phase section, understand the transition rules, and execute correctly. This is the most complex skill in the collection and has the highest risk of LLM confusion.
>
> **Fix:** Add a decision tree at the top of the Phase Machine section: "Read `active_task.phase` from state. Go directly to that section. If no active task, go to IDLE." And simplify: consider whether COMPLETE needs to be its own phase or could be a subroutine of EXECUTING and RETRYING.

#### F-61: **[P1]** [FIXED] Hardcoded /define output format is fragile

> **Title:** fix: make manifest path extraction more resilient in workhorse
>
> **File:** `plugins/loop-recipes/skills/workhorse/SKILL.md:108`
>
> "Extract that path" from /define's output matching `Manifest complete: /tmp/manifest-<timestamp>.md`. If /define's output format changes (different wording, different path), the extraction fails and the iteration is wasted.
>
> **Fix:** Add fallback: "If the exact pattern isn't found, search the output for any file path matching `/tmp/manifest-*.md`. If still not found, check /tmp for the most recently created manifest file."

#### F-62: **[P2]** [DEFERRED] Description mentions "OpenClaw-like" which is unclear

> **Title:** chore: remove or clarify "OpenClaw-like" reference in workhorse description
>
> **File:** `plugins/loop-recipes/skills/workhorse/SKILL.md:3`
>
> "Simulates OpenClaw-like autonomous agent behavior" references an obscure project. This adds confusion without value for triggering or understanding.
>
> **Fix:** Remove "Simulates OpenClaw-like autonomous agent behavior within /loop." Simplify to: "General-purpose autonomous task agent that plans, gets approval, and executes development tasks via a persistent queue."

---

## Appendix: Evaluation Methodology

### Dimension Details

**Prompt quality** evaluates whether the SKILL.md instructions are unambiguous and complete. Key signals: conflicting instructions, undefined terms, vague guidance that leaves the LLM guessing, instructions that conflict with Claude Code platform conventions (e.g., using `cat` instead of Read tool).

**State management** evaluates the state file lifecycle: initialization, locking, persistence scope, cleanup, and whether the chosen scope (/tmp/ vs ~/.claude/) is appropriate for the skill's purpose.

**Error handling** evaluates graceful degradation: what happens when tools are missing, commands fail, files don't exist, or permissions are denied. Skills that just crash on first unexpected input score poorly.

**Edge cases** evaluates boundary conditions: empty inputs, platform differences (macOS vs Linux), large inputs, concurrent execution, and unusual but valid usage patterns.

**Description accuracy** evaluates the YAML frontmatter: does the description match behavior, are trigger terms accurate, is the description the right length (not wasting context tokens).

**UX** evaluates the human interaction: is output appropriate (not dumping walls of text), do AskUserQuestion calls respect the tool's constraints (2-4 options), are stop conditions clear, does the skill respect the user's attention.

### Tools That Informed This Evaluation

- Direct reading of all SKILL.md files, template.html, reference files, plugin.json files, marketplace.json, README.md
- Claude Code platform knowledge (Bash tool behavior, AskUserQuestion constraints, Read/Write/Edit tool preferences)
- Cross-referencing between skills for consistency patterns

### Suggested Re-evaluation Cadence

Re-evaluate after:
- Any version bump to either plugin
- Adding or removing skills
- Changes to Claude Code's tool constraints or skill loader behavior
- Quarterly at minimum to catch drift
