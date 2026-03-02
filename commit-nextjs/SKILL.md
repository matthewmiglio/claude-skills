---
name: commit-nextjs
description: >
  Build, fix, commit, and push a Next.js app. Runs npm run build, fixes obvious
  build errors iteratively (ignoring missing env/supabase key errors), then
  creates a git commit with a descriptive message and pushes. Use when the user
  says "commit", "ship it", "build and push", or asks to finalize their changes.
argument-hint: "<optional commit message hint>"
disable-model-invocation: true
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Edit
  - Grep
  - Glob
---

# Commit Skill

Build a Next.js app, fix build errors, then commit and push.

## Process

### 1. Build the App

Run the Next.js build:

```bash
npm run build
```

### 2. Fix Build Errors (Loop)

If the build fails, analyze the error output and fix **obvious** build errors:
- TypeScript type errors
- Missing imports / unused imports
- Syntax errors
- Missing modules that can be resolved by editing code

**Ignore these errors** (do not attempt to fix):
- Missing environment variables (SUPABASE_URL, SUPABASE_KEY, NEXT_PUBLIC_*, etc.)
- Connection/network errors to external services
- Warnings that don't cause the build to fail

After fixing, re-run `npm run build`. Repeat until the build succeeds or only ignorable errors remain.

**Safety limit**: Stop after 10 build attempts. If still failing, report the remaining errors to the user and ask how to proceed.

### 3. Commit and Push

Once the build passes (or only has ignorable env-related failures):

1. Run `git status` and `git diff` to understand all changes.
2. Run `git log --oneline -5` to match the repo's commit message style.
3. Stage all changed files with `git add` (use specific file names, not `git add -A`).
4. Write a descriptive commit message summarizing what changed and why.
5. Commit:
```bash
git commit -m "$(cat <<'EOF'
<descriptive message>

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```
6. Push to the current branch:
```bash
git push
```

### 4. Report

Tell the user:
- Whether the build succeeded
- What errors were fixed (if any)
- The commit message used
- The push result

## Examples

User: `/commit`
-> Build, fix errors, commit with auto-generated message, push.

User: `/commit cleanup chart components`
-> Build, fix errors, commit with message incorporating "cleanup chart components", push.
