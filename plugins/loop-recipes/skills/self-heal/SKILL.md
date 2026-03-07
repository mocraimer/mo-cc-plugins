---
name: self-heal
description: "Self-healing debugger that autonomously reproduces a bug, analyzes errors, and attempts fixes in isolated worktrees. Designed for /loop — runs iteratively until the test passes or 5 attempts are exhausted. Usage: /loop 2m /self-heal npm test -- --grep 'failing test'"
user-invokable: true
---

# Self-Healing Debugger

You are an autonomous debugging agent running as a recurring `/loop` iteration. Your job: reproduce a failing test, analyze the error, attempt a fix in an isolated worktree, and track progress across iterations.

## Input

`$ARGUMENTS` = the test command to run (e.g., `npm test -- --grep 'auth timeout'`, `pytest tests/test_auth.py::test_login`, `cargo test auth_flow`).

If `$ARGUMENTS` is empty, ask the user what test command to run and stop this iteration.

## State Management

State file: `/tmp/self-heal-state.md`

### On Start — Read State

1. Read `/tmp/self-heal-state.md`. If it does not exist, this is the first iteration — initialize state:
   ```yaml
   ---
   status: in-progress
   test_command: "<from $ARGUMENTS>"
   attempt: 0
   max_attempts: 5
   flaky_runs: 0
   flaky_passes: 0
   started: "<timestamp>"
   ---
   # Self-Heal Log
   ```

2. If the state file exists and `status: in-progress` with a `locked_by` field set, a previous iteration is still running. Output "Previous iteration still running — skipping." and **stop this iteration**.

3. If `status: completed` or `status: flaky` or `attempt >= max_attempts`, output the final report from state and tell the user to stop the loop. **Stop this iteration.**

4. Set `locked_by: <current_timestamp>` in frontmatter to claim the lock.

5. Read the current `attempt` count and all previous attempt logs.

### On End — Write State

After every iteration (success or failure), update `/tmp/self-heal-state.md`:
- Clear `locked_by`
- Update `attempt` count
- Append the attempt log entry
- Update `status` if terminal (completed/failed/flaky)

## Iteration Logic

### Step 1: Reproduce the Bug

Run the test command from `$ARGUMENTS` (or from state if continuing):

```bash
<test_command> 2>&1
```

- If the test **passes**: increment `flaky_passes`. If this is the first iteration and no fixes were applied, run it 2 more times to check for flakiness.
  - If it passes consistently (3/3): report "Test is passing — nothing to fix." Set `status: completed`. Stop.
  - If it's inconsistent: set `status: flaky`, write a flakiness report. Stop.
- If the test **fails**: proceed to Step 2.

### Step 2: Analyze the Error

Read the error output. Identify:
- The failing assertion or exception
- The relevant source file(s) and line numbers
- The root cause hypothesis

Check previous attempts in the state log — do NOT repeat a fix that already failed. If all obvious approaches have been tried, try a different angle or escalate.

### Step 3: Attempt a Fix in Worktree

Create an isolated worktree for this attempt:

```bash
git worktree add /tmp/self-heal-worktree-<attempt> -b self-heal-attempt-<attempt>
```

Apply your fix in the worktree. Then run the test command inside the worktree:

```bash
cd /tmp/self-heal-worktree-<attempt> && <test_command> 2>&1
```

### Step 4: Evaluate Result

- **Test passes in worktree**:
  - Run it 2 more times to confirm it's not flaky.
  - If stable: set `status: completed`. Log the successful fix. Output a summary of what was changed and why. Tell the user: "Fix found! Review the changes in branch `self-heal-attempt-<attempt>` and merge if approved. You can stop the loop now."
  - If inconsistent: note flakiness, continue.

- **Test still fails**:
  - Log: what was tried, the error output, why it didn't work.
  - Clean up the worktree: `git worktree remove /tmp/self-heal-worktree-<attempt> --force`
  - Delete the branch: `git branch -D self-heal-attempt-<attempt>`
  - Increment attempt count. If `attempt >= max_attempts`, generate final report (Step 5).

### Step 5: Final Report (After 5 Failed Attempts)

Set `status: exhausted`. Write a report to the state file and output it:

```markdown
## Self-Heal Report: EXHAUSTED

**Test:** <test_command>
**Attempts:** 5/5

### Errors Encountered
<list each unique error seen>

### Fixes Attempted
<for each attempt: what was tried, what happened>

### Analysis
<what was learned, patterns noticed>

### Recommended Next Steps
<suggestions for manual debugging — what to look at, what hasn't been tried>
```

Tell the user to stop the loop.

## Flakiness Detection

If during any iteration the test produces inconsistent results (passes sometimes, fails sometimes) without any code changes:

1. Run the test 5 times without modifications
2. Record pass/fail for each run
3. If results are mixed: set `status: flaky` and report:

```markdown
## Self-Heal Report: FLAKY TEST DETECTED

**Test:** <test_command>
**Results:** <X/5 passes, Y/5 failures>

### Failure Pattern
<describe when it fails vs passes — timing, ordering, etc.>

### Likely Causes
<race condition, test isolation issue, external dependency, etc.>

### Recommended Fix
<suggestions for stabilizing the test>
```

Tell the user to stop the loop.

## Stop Conditions

Tell the user to stop the loop (`Ctrl+C` or cancel the cron) when:
- A fix is found (`status: completed`)
- Test is flaky (`status: flaky`)
- All attempts exhausted (`status: exhausted`)
- Test was already passing (`status: completed`)
