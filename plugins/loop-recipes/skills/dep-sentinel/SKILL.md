---
name: dep-sentinel
description: "Dependency security monitor that auto-detects project ecosystems (Node.js, Python, Rust, Go, Ruby, Java, PHP), runs security audits and outdated checks, classifies findings by severity, and uses delta reporting to only alert on new issues. Designed for /loop — runs periodically to watch your dependencies. Use this skill when the user wants continuous vulnerability scanning, mentions CVE monitoring, asks about dependency security or outdated packages on a recurring basis, wants automated npm audit/pip-audit/cargo-audit, or mentions security review compliance. Not for one-time audits — use npm audit directly for that. Usage: /loop 1h /dep-sentinel"
user-invokable: true
---

# Dependency Sentinel

You are a dependency security agent running as a recurring `/loop` iteration. Your job: detect project ecosystems, run security audits and outdated checks, classify findings by severity, and only alert the user about new issues (delta reporting).

## State Management

State file: `~/.claude/loop-recipes/dep-sentinel-state.md`

### On Start — Read State

1. Read `~/.claude/loop-recipes/dep-sentinel-state.md`. If it does not exist, initialize:
   ```yaml
   ---
   status: idle
   last_scan: "1970-01-01T00:00:00Z"
   previous_findings: []
   previous_outdated: []
   ---
   # Dependency Sentinel Log
   ```

2. If `status: in-progress` with a `locked_by` field set and `locked_by` timestamp is less than 15 minutes old, a previous iteration is still running. Output "Previous iteration still running — skipping." and **stop**. If `locked_by` is older than 15 minutes, treat as stale (previous iteration likely crashed), clear it, and proceed.

3. Set `locked_by: <current_timestamp>` and `status: in-progress`.

4. Ensure `~/.claude/loop-recipes/` directory exists (`mkdir -p`).

5. Read `previous_findings` and `previous_outdated` for delta comparison.

### On End — Write State

After every iteration:
- Clear `locked_by`, set `status: idle`
- Update `last_scan` to current timestamp
- Store current findings as `previous_findings` and `previous_outdated` for next iteration
- Append iteration summary to log section

## Iteration Logic

### Step 1: Auto-Detect Ecosystems

Scan the current project for manifest files:

```bash
find . -maxdepth 3 -not -path "*/node_modules/*" \( \
  -name "package.json" -o -name "requirements.txt" -o -name "pyproject.toml" -o \
  -name "Pipfile" -o -name "setup.py" -o -name "Cargo.toml" -o -name "go.mod" -o \
  -name "Gemfile" -o -name "pom.xml" -o -name "build.gradle" -o \
  -name "build.gradle.kts" -o -name "composer.json" \
\) 2>/dev/null
```

Build a list of detected ecosystems with their manifest paths.

**Monorepo handling:** If a root `package.json` contains a `workspaces` field, run audit commands from the root — npm/yarn handle workspaces natively. For non-workspace monorepos (multiple independent `package.json` files without a root workspace config), run audit commands separately from each manifest's directory.
- `package.json` → Node.js/npm
- `requirements.txt` / `pyproject.toml` / `Pipfile` → Python
- `Cargo.toml` → Rust
- `go.mod` → Go
- `Gemfile` → Ruby
- `pom.xml` / `build.gradle` → Java/Kotlin
- `composer.json` → PHP

If no ecosystems detected: output "No dependency manifests found." and **stop**.

### Step 2: Run Security Audits

For each detected ecosystem, check if the audit tool is available and run it. If a tool is not installed, skip that ecosystem with a note.

#### Node.js
```bash
which npm >/dev/null 2>&1 && npm audit --json 2>/dev/null
```
If `npm` is not available: note "npm not installed — skipping Node.js audit" and skip.

#### Python
```bash
which pip-audit >/dev/null 2>&1 && pip-audit --format json 2>/dev/null
```
If `pip-audit` is not available, try:
```bash
which pip >/dev/null 2>&1 && pip check 2>/dev/null
```
If neither is available: note "pip-audit/pip not installed — skipping Python audit" and skip.

#### Rust
```bash
which cargo-audit >/dev/null 2>&1 && cargo audit --json 2>/dev/null
```
If `cargo-audit` is not available: note "cargo-audit not installed — skipping Rust audit. Install with: cargo install cargo-audit" and skip.

#### Go
```bash
which govulncheck >/dev/null 2>&1 && govulncheck ./... 2>/dev/null
```
If `govulncheck` is not available: note "govulncheck not installed — skipping Go audit. Install with: go install golang.org/x/vuln/cmd/govulncheck@latest" and skip.

#### Ruby
```bash
which bundle-audit >/dev/null 2>&1 && bundle-audit check 2>/dev/null
```
If `bundle-audit` is not available: note "bundle-audit not installed — skipping Ruby audit" and skip.

For other ecosystems without standard audit tools, note the limitation and skip.

Parse audit output to extract: package name, vulnerability ID/advisory, severity (critical/high/medium/low), description.

### Step 3: Check for Outdated Packages

For each detected ecosystem where the tool is available:

#### Node.js
```bash
npm outdated --json 2>/dev/null
```

#### Python
```bash
pip list --outdated --format json 2>/dev/null
```

#### Rust
```bash
cargo outdated --format json 2>/dev/null
```

If the outdated check tool is not available for an ecosystem, skip with a note.

Parse output to extract: package name, current version, latest version.

### Step 4: Classify Findings by Severity

| Severity | Criteria | Default Alert |
|----------|----------|---------------|
| Critical | Known exploited vulnerabilities, CVSS 9.0+ | Yes — always |
| High | CVSS 7.0-8.9, actively exploited or network-facing | Yes — always |
| Medium | CVSS 4.0-6.9 | No — log only |
| Low | CVSS < 4.0, informational | No — log only |

Outdated packages are classified as:
- **Warning** if major version behind
- **Info** if minor/patch version behind

### Step 5: Delta Reporting

Compare current findings against `previous_findings` from state. Match findings by the tuple `(ecosystem, package_name, advisory_id)`:

- **New vulnerabilities:** Found now but not in previous scan → alert
- **Resolved vulnerabilities:** In previous scan but gone now → log as resolved
- **Unchanged vulnerabilities:** Same as before → do not re-alert

Same logic for outdated packages against `previous_outdated`, matched by `(ecosystem, package_name)`.

### Step 6: Report

If there are new high+ severity findings:

```markdown
## Dependency Sentinel — <timestamp>

### New Vulnerabilities

#### Critical
- **<package>** (<ecosystem>): <advisory ID> — <description>

#### High
- **<package>** (<ecosystem>): <advisory ID> — <description>

### Newly Outdated (Major Version)
- **<package>**: <current> → <latest>

### Resolved Since Last Scan
- **<package>**: <advisory> — no longer vulnerable

### Skipped Ecosystems
- <ecosystem>: <reason>
```

If no new high+ findings: output "Dependencies healthy. No new issues since last scan."

## Stop Conditions

This skill is designed to run periodically (e.g., hourly). The user should stop the loop when:
- They're done with their session
- They want to investigate reported vulnerabilities manually
- All dependencies are clean and no changes expected
