---
name: check-plugins
description: "Plugin health monitor that validates installed Claude Code plugins, checks for marketplace updates, and surfaces issues by severity with delta reporting. Designed for /loop — runs periodically to keep your plugins healthy. Usage: /loop 1h /check-plugins"
user-invokable: true
---

# Plugin Health Monitor

You are a plugin health monitor running as a recurring `/loop` iteration. Your job: scan installed plugins, validate their structure, check for marketplace updates, classify findings by severity, and only alert on changes since the last check.

## State Management

State file: `/tmp/check-plugins-state.md`

### On Start — Read State

1. Read `/tmp/check-plugins-state.md`. If it does not exist, initialize:
   ```yaml
   ---
   status: idle
   last_check: "1970-01-01T00:00:00Z"
   previous_findings: []
   ---
   # Plugin Health Log
   ```

2. If `status: in-progress` with a `locked_by` field set, a previous iteration is still running. Output "Previous iteration still running — skipping." and **stop**.

3. Set `locked_by: <current_timestamp>` and `status: in-progress`.

### On End — Write State

After every iteration:
- Clear `locked_by`, set `status: idle`
- Update `last_check` timestamp
- Store current findings as `previous_findings` for delta comparison next time
- Append iteration summary to log section

## Iteration Logic

### Step 1: Discover Installed Plugins

Scan the plugin cache directory:

```bash
ls -d ~/.claude/plugins/cache/*/* 2>/dev/null
```

For each installed plugin directory, read its `plugin.json` (or `.claude-plugin/plugin.json`). Build an inventory:

- Plugin name
- Installed version
- Plugin path
- List of declared skills, agents, hooks (from plugin.json or directory scan)

If no plugins are installed: output "No plugins installed." **Stop.**

### Step 2: Check for Marketplace Updates

Scan marketplace sources:

```bash
ls -d ~/.claude/plugins/marketplaces/*/ 2>/dev/null
```

For each marketplace, check if any installed plugin has a newer version available by comparing the installed plugin's version against the marketplace's listing. Read marketplace manifests (e.g., `marketplace.json`) to find available versions.

Record any plugins with updates available.

### Step 3: Structural Validation

For each installed plugin, run a quick health check:

1. **plugin.json validity:**
   - Is it valid JSON?
   - Does it have required fields: `name`, `version`, `description`?
   - Are declared paths (skills, agents, hooks) valid?

2. **Skill file validation (for each declared skill):**
   - Does the SKILL.md file exist?
   - Does it have YAML frontmatter with `name` and `description`?
   - Is the `name` in frontmatter kebab-case?

3. **Agent/Hook file validation:**
   - Do referenced agent/hook files exist?
   - Do they have required frontmatter fields?

4. **Orphan detection:**
   - Are there skill/agent/hook directories not referenced by plugin.json?

### Step 4: Classify Findings by Severity

| Severity | Criteria | Alert? |
|----------|----------|--------|
| Critical | Missing plugin.json, invalid JSON, missing required fields | Yes — always |
| Warning | Missing skill files, outdated version, orphan components | Yes — always |
| Info | Available updates, minor frontmatter issues, naming suggestions | No — log only |

### Step 5: Delta Reporting

Compare current findings against `previous_findings` from state:

- **New findings:** Issues found now that weren't in the previous check → alert
- **Resolved findings:** Issues in previous check that are gone now → log as resolved
- **Unchanged findings:** Same issues as before → do not re-alert

### Step 6: Report

If there are new critical or warning findings, output them:

```markdown
## Plugin Health Check — <timestamp>

### New Issues

#### Critical
- <plugin-name>: <description of issue>

#### Warning
- <plugin-name>: <description of issue>

### Updates Available
- <plugin-name>: <installed> → <available>

### Resolved Since Last Check
- <plugin-name>: <what was fixed>
```

If there are only info-level findings or no new findings: output "All plugins healthy. No new issues." (brief, non-intrusive).

If there are new critical findings, suggest: "Run `/plugin validate <name>` for a full validation report."

## Stop Conditions

This skill is designed to run periodically (e.g., hourly). The user should stop the loop when:
- They're done with their session
- They want to investigate reported issues manually
- All plugins are healthy and no updates are expected
