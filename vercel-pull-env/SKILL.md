---
name: vercel-pull-env
description: >
  Link the current repo to an existing Vercel project and pull environment
  variables into .env.local. Searches for a matching project by name, links to
  it, and downloads env vars. Use when the user says "pull env", "vercel env",
  "link vercel", or wants to set up Vercel environment variables locally.
argument-hint: "<optional project name>"
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Vercel Pull Env Skill

Link to an existing Vercel project and pull environment variables.

## Process

### 1. Determine Project Name

- If the user provides a project name as an argument, use that.
- Otherwise, infer the project name from the current directory name (the repo folder name).

### 2. Link to the Existing Vercel Project

Run:

```bash
npx vercel link --yes --project <project-name>
```

- This will search for an existing project and link to it, creating a `.vercel` directory.
- **Do NOT create a new project.** If the project is not found, report the error to the user and stop.

### 3. Pull Environment Variables

Run:

```bash
npx vercel env pull
```

- This downloads environment variables into `.env.local`.

### 4. Verify

Count the number of environment variables pulled:

```bash
grep -c "=" .env.local
```

- Confirm that at least 2 variables were pulled.
- If fewer than 2 variables exist, warn the user that the env file looks sparse.

### 5. Report

Tell the user:
- Which project was linked (name and scope)
- How many environment variables were pulled
- That `.env.local` was created/updated

## Examples

User: `/vercel-pull-env`
-> Infer project name from directory, link, pull env vars, report.

User: `/vercel-pull-env my-cool-app`
-> Link to "my-cool-app", pull env vars, report.
