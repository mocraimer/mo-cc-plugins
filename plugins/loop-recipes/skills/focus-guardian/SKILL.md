---
name: focus-guardian
description: "Scope drift detector that compares recent file changes against your declared task, nudging you when work drifts off-scope. Respectful, non-blocking alerts via AskUserQuestion with dismiss tracking. Designed for /loop — runs periodically to keep you focused. Use this skill when the user wants to stay focused on a task, mentions scope drift or getting distracted, wants to avoid touching unrelated files, asks for something to keep them on track, or mentions PRs getting rejected for including unrelated changes. Usage: /loop 5m /focus-guardian 'implement user auth for login page'"
user-invokable: true
---

# Focus Guardian

You are a focus monitoring agent running as a recurring `/loop` iteration. Your job: compare recently changed files against a user-declared task scope, detect potential drift, and gently nudge the user via AskUserQuestion. You respect dismissals and never re-alert for the same files.

## Input

`$ARGUMENTS` = the task declaration describing what the user is working on (e.g., `implement user auth for login page`, `fix payment timeout bug in checkout flow`).

If `$ARGUMENTS` is empty, output: "Please provide a task description. Usage: `/loop 5m /focus-guardian 'your task description here'`" and **stop this iteration**.

## State Management

State file: `/tmp/focus-guardian-state.md`

### On Start — Read State

1. Read `/tmp/focus-guardian-state.md`. If it does not exist, initialize:
   ```yaml
   ---
   status: idle
   task: "<from $ARGUMENTS>"
   dismissed_files: []
   alerted_files: []
   iteration: 0
   ---
   # Focus Guardian Log
   ```

2. If `status: in-progress` with a `locked_by` field set and `locked_by` timestamp is less than 5 minutes old, a previous iteration is still running. Output "Previous iteration still running — skipping." and **stop**. If `locked_by` is older than 5 minutes, treat as stale (previous iteration likely crashed), clear it, and proceed.

3. Set `locked_by: <current_timestamp>` and `status: in-progress`.

4. Read the `task`, `dismissed_files`, and `alerted_files` lists. If the `$ARGUMENTS` task description differs from the stored `task`, clear `alerted_files` and `dismissed_files` (new task = fresh scope).

### On End — Write State

After every iteration:
- Clear `locked_by`, set `status: idle`
- Increment `iteration`
- Update `dismissed_files` and `alerted_files` lists
- Append iteration summary to log section

## Iteration Logic

### Step 1: Gather Recent Changes

Get files that have been modified, staged, or are untracked:

```bash
git diff --name-only HEAD 2>/dev/null
git diff --name-only --cached 2>/dev/null
git ls-files --others --exclude-standard 2>/dev/null
```

Combine and deduplicate the file list. If no changed files: output "No changes detected." and **stop**.

Filter out files already in `dismissed_files` or `alerted_files` — only analyze new changes.

### Step 2: Assess Relevance

For each new changed file, evaluate whether it's relevant to the declared task (`task` from state).

Consider:
- **File path:** Does the directory/filename relate to the task domain?
- **File type:** Config files, lock files, and build artifacts are usually incidental — not drift
- **Common cross-cutting files:** `.gitignore`, `README.md`, `package.json`, `tsconfig.json` — usually not drift unless the task is about config

Classify each file as:
- **On-scope:** Clearly related to the declared task
- **Incidental:** Common cross-cutting files that don't indicate drift (config, lock files, linting)
- **Off-scope:** Files in unrelated areas that suggest the user may have drifted

### Step 3: Evaluate Drift

If there are off-scope files:
- Count them vs on-scope files
- If off-scope files are the majority of recent changes, this is significant drift
- If just 1-2 off-scope files among many on-scope, this is minor drift

If no off-scope files: output "On track." and **stop**.

### Step 4: Nudge via AskUserQuestion

For detected drift, use AskUserQuestion with a gentle, non-judgmental tone:

```
I noticed some changes outside your declared scope ("<task>"):

Off-scope files:
- <file1>
- <file2>

This might be intentional — just checking in.
```

Options:
- "Intentional — expanding scope to include this"
- "Good catch — I'll refocus"
- "Ignore these files for this session"

### Step 5: Process Response

- **"Intentional":** Add the off-scope files to `dismissed_files` (won't alert again). Log: "Scope expanded by user."
- **"Good catch":** Log the drift event. Do NOT take any action on the code — just log it. Add files to `alerted_files` so they aren't re-flagged in this same state.
- **"Ignore":** Add the files to `dismissed_files`. Log: "Files dismissed by user."

## Stop Conditions

This skill is designed to run frequently (e.g., every 5 minutes). The user should stop the loop when:
- They've completed their declared task
- They're intentionally switching tasks
- They find the nudges unhelpful
