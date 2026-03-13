---
name: commit-monorepo
description: >
  Build all Next.js workspaces in a monorepo in parallel, fix build errors
  iteratively, then commit and push the whole monorepo. Discovers Next.js
  subprojects by finding next.config.* files, spawns a subagent per workspace
  to build and fix, then commits and pushes once all pass.
argument-hint: "<optional commit message hint>"
user-invocable: true
allowed-tools:
  - Agent
  - Bash
  - Read
  - Edit
  - Grep
  - Glob
---

# Commit Monorepo Skill

Validate Next.js workspaces in a monorepo by building them in parallel, fix build errors, then commit and push ALL changes across the entire repo (including non-Next.js code like Python, config, etc.).

Only Next.js subprojects are build-gated. Everything else is committed as-is.

## Process

### 1. Discover Next.js Workspaces

Find all Next.js subprojects in the repo:

```bash
find . -name "next.config.*" -not -path "*/node_modules/*" -not -path "*/.next/*"
```

For each match, the workspace root is the directory containing the next.config file. If no Next.js workspaces are found, skip straight to the commit step.

### 2. Build All Workspaces in Parallel

Spawn one **Agent subagent** per workspace. Each subagent should:

1. `cd` into the workspace directory
2. Run `npm run build`
3. If the build fails, analyze the error output and fix **obvious** build errors:
   - TypeScript type errors
   - Missing imports / unused imports
   - Syntax errors
   - Missing modules that can be resolved by editing code
4. **Ignore these errors** (do not attempt to fix):
   - Missing environment variables (SUPABASE_URL, SUPABASE_KEY, NEXT_PUBLIC_*, etc.)
   - Connection/network errors to external services
   - Warnings that don't cause the build to fail
5. Re-run `npm run build`. Repeat until the build succeeds or only ignorable errors remain.
6. **Safety limit**: Stop after 10 build attempts per workspace. Report remaining errors.

Launch ALL workspace subagents in a single message (parallel). Each subagent prompt should include:
- The workspace path
- Instructions to run `npm run build`, fix errors, and loop until passing
- The ignore list above
- The 10-attempt safety limit

Wait for all subagents to complete before proceeding.

### 3. Check Results

If any workspace failed to build after 10 attempts, report the failures to the user and ask how to proceed. Do NOT commit if any workspace has real (non-ignorable) build failures.

### 4. Commit and Push

Once all Next.js workspaces pass (or there are none), commit **everything** in the repo that has changed — Next.js code, Python code, config files, training scripts, whatever.

1. Run `git status` and `git diff` to understand all changes.
2. Run `git log --oneline -5` to match the repo's commit message style.
3. Stage all changed files with `git add` (use specific file names, not `git add -A`).
4. Write a descriptive commit message summarizing what changed and why.
5. Commit:
```bash
git commit -m "$(cat <<'EOF'
<descriptive message>
EOF
)"
```
6. Push to the current branch:
```bash
git push
```

### 5. Report

Tell the user:
- Which Next.js workspaces were found and built (if any)
- Whether each build succeeded
- What errors were fixed (if any)
- The commit message used
- The push result

## Examples

User: `/commit-monorepo`
-> Discover workspaces, build all in parallel, fix errors, commit, push.

User: `/commit-monorepo update dashboard and API`
-> Discover workspaces, build all in parallel, fix errors, commit with message incorporating "update dashboard and API", push.
