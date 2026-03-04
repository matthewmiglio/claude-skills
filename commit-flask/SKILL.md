---
name: commit-flask
description: >
  Build-check, fix, commit, and push a Flask app. Verifies the app can start
  by importing it, fixes obvious errors iteratively, then creates a git commit
  with a descriptive message and pushes. Works with any Flask project structure.
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

# Commit Flask Skill

Verify a Flask app can start, fix errors, then commit and push.

## Process

### 1. Detect the Flask App

Find how the app is created. Check in order:

1. Look for `create_app` factory pattern in `app/__init__.py`, `src/__init__.py`, or any `__init__.py`
2. Look for `app = Flask(__name__)` in `app.py`, `run.py`, `main.py`, `wsgi.py`
3. Check for a `.flaskenv` or `.env` file that sets `FLASK_APP`

Determine the correct import statement and config name (if factory pattern).

### 2. Load Environment and Run the App Check

If a `.env` file exists, source it first. Then run the app import check:

**Factory pattern** (e.g., `create_app` in `app/__init__.py`):
```bash
set -a && [ -f .env ] && source .env; set +a && python -c "from app import create_app; app = create_app()" 2>&1
```

**Direct app instance** (e.g., `app = Flask(__name__)` in `app.py`):
```bash
set -a && [ -f .env ] && source .env; set +a && python -c "from app import app" 2>&1
```

Adapt the import path to match the project structure you discovered in step 1. If the factory takes a config name, try `'development'` first, then `'default'`, then no argument.

### 3. Fix Errors (Loop)

If the import check fails, analyze the error and fix **obvious** issues:
- `SyntaxError` — fix the syntax in the indicated file and line
- `ImportError` / `ModuleNotFoundError` — fix broken imports, typos in import paths
- `NameError` — fix undefined variable references
- `Jinja2 TemplateSyntaxError` — fix template syntax issues
- Missing route function references — fix `url_for` calls pointing to renamed functions

**Ignore these errors** (do not attempt to fix):
- Database connection failures (can't reach MySQL/PostgreSQL/etc.)
- Missing environment variables that are expected in production only
- Email/SMTP connection errors
- Third-party API connection errors
- Warnings that don't prevent the app from loading

After fixing, re-run the import check. Repeat until it succeeds or only ignorable errors remain.

**Safety limit**: Stop after 10 attempts. If still failing, report the remaining errors to the user and ask how to proceed.

### 4. Commit and Push

Once the check passes (or only has ignorable connection-related failures):

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
- Whether the app check succeeded
- What errors were fixed (if any)
- The commit message used
- The push result

## Examples

User: `/commit-flask`
-> Detect app, run import check, fix errors, commit with auto-generated message, push.

User: `/commit-flask add SEO improvements`
-> Detect app, run import check, fix errors, commit with message incorporating "add SEO improvements", push.
