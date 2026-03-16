---
name: commit
description: >
  Scan staged/unstaged git changes for sensitive data, oversized files, and
  data files that shouldn't be committed, then commit and push. Use when the
  user says "commit", "push", "ship it", or wants to finalize and push changes.
argument-hint: "<optional commit message hint>"
disable-model-invocation: true
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
---

# Commit Skill

Safely commit and push all git changes after scanning for problems.

## Process

### 1. Read the Repo State

Run these commands to understand the full picture:

```bash
git status
git diff
git diff --cached
git diff --stat
git log --oneline -5
```

Identify every file that is new, modified, or staged.

### 1b. Build Check (Node Projects)

Check if a `package.json` exists in the project root. If it does:

1. Read `package.json` and inspect the `"scripts"` section.
2. Look for build-related scripts in this priority order: `build`, `typecheck`, `type-check`, `tsc`, `check`, `lint`. Run whichever ones exist.
3. If any script **fails**, analyze the error output and fix the code.
4. Re-run the failing script. **Repeat up to 10 attempts max.**
5. If still failing after 10 attempts, stop and report the remaining errors to the user using AskUserQuestion. Do NOT proceed to commit.

**What to fix:**
- TypeScript type errors
- Missing/unused imports
- Syntax errors
- Resolvable module issues

**What to ignore (do not attempt to fix):**
- Missing environment variables (SUPABASE_URL, API keys, NEXT_PUBLIC_*, etc.)
- Network/connection errors to external services
- Warnings that don't cause the build to fail

### 2–4. Run Safety Checks (IN PARALLEL)

Launch **three checks simultaneously** using the Agent tool. Collect all findings into a single report before proceeding.

#### Check 2: Sensitive Information Scan

Search every changed/added file for:
- API keys, secrets, tokens (patterns like `sk-`, `AKIA`, `ghp_`, `Bearer`, `password =`, etc.)
- Private keys (`-----BEGIN.*PRIVATE KEY-----`)
- `.env` files or files containing environment variable assignments with real values
- Connection strings, database URLs with credentials
- Any string that looks like a secret or credential

If found: record the file, line, and what was detected.

#### Check 3: Large File Detection

For every changed/untracked file, check its size:

```bash
git diff --name-only | xargs -I{} du -m "{}" 2>/dev/null | awk '$1 >= 300'
git ls-files --others --exclude-standard | xargs -I{} du -m "{}" 2>/dev/null | awk '$1 >= 300'
```

Flag any file **300 MB or larger**.

If found: record the file and its size.

#### Check 4: Data File Detection

Look for files that are likely local data artifacts and should not be committed:
- CSV, JSON, JSONL, Parquet, SQLite, or Excel files that look like generated data (KPI results, scrape outputs, analytics exports)
- Large log files
- Local config files that are machine-specific (e.g., generated `.cfg`, `.ini`, `.local` files)
- Cache directories, `__pycache__`, `.cache/`, `node_modules/`
- Database dumps, `.sql` files with data

Use your judgment — small config JSONs that are clearly part of the project (like `package.json`, `tsconfig.json`) are fine. The concern is data artifacts that differ per machine or per run.

If found: record the file and why it looks like a data artifact.

### 5. Report Findings

If **any** of checks 2–4 found issues:

1. Print a clear report to the user grouped by category:
   - **Sensitive Data** — file, line, what was found, suggested fix (redact the value, add to `.gitignore`, use env var instead)
   - **Oversized Files** — file, size, suggested fix (add to `.gitignore`)
   - **Data Artifacts** — file, why flagged, suggested fix (add to `.gitignore`, move out of repo)
2. **Stop and wait** for user instructions using AskUserQuestion. Do NOT commit.
3. After the user responds, apply their requested fixes, then re-run the checks on any remaining changed files before proceeding.

If checks 2–4 found **nothing**, proceed directly to step 6.

### 6. Commit and Push

1. Stage all changed/new files with `git add` using specific file names (not `git add -A`).
2. Write a clear, descriptive commit message summarizing what changed and why.
   - If `$ARGUMENTS` was provided, incorporate it into the message.
   - Match the style of recent commits from `git log`.
3. Commit using the machine's default git credentials — **no Co-Authored-By, no "Generated with Claude" lines**:

```bash
git commit -m "$(cat <<'EOF'
<descriptive message>
EOF
)"
```

4. Push to the current branch:

```bash
git push
```

5. If push fails because the remote is ahead, pull with rebase first:

```bash
git pull --rebase && git push
```

### 7. Report

Tell the user:
- What was committed (summary of changes)
- The commit message used
- The push result (success / failure)

## Examples

User: `/commit`
→ Scan for issues, commit all changes with auto-generated message, push.

User: `/commit add watchface preview feature`
→ Scan for issues, commit with message incorporating "add watchface preview feature", push.
