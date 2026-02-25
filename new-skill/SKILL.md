---
name: new-skill
description: >
  Scaffold a new Claude Code skill from a description.
  Use when creating a new skill, generating a SKILL.md,
  or adding automation to the project.
argument-hint: "<skill description>"
disable-model-invocation: false
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Glob
---

# New Skill

Scaffold a new Claude Code skill from a description.

## Reference
!`cat docs/skill-authoring.md`

## Existing Skills
!`ls .claude/skills/`

## Process

1. **Understand the request** — read `$ARGUMENTS` as a description of the desired skill.
2. **Ask clarifying questions** before generating:
   - What should trigger this skill? (slash command only, or auto-invocation too?)
   - What tools does it need? (MCP tools, Bash, Read/Write, etc.)
   - Should it be manual-only (`disable-model-invocation: true`)?
   - Should it run in a fork context or main conversation?
3. **Read existing skills** in `.claude/skills/` to match project conventions (frontmatter style, tone, structure).
4. **Generate the SKILL.md** with:
   - Complete YAML frontmatter (name, description, argument-hint, allowed-tools, etc.)
   - Clear process steps in the body
   - Examples where helpful
5. **Present the generated skill to the user for review** before writing.
6. **Write the file** to `.claude/skills/<name>/SKILL.md` after approval.

## Frontmatter Template

```yaml
---
name: <skill-name>
description: >
  <What it does>. <When to trigger it>.
argument-hint: "<hint>"
disable-model-invocation: false
user-invocable: true
allowed-tools:
  - <tool1>
  - <tool2>
---
```

## Guidelines

- Description field is the trigger — put all "when to use" info there
- Keep SKILL.md under 500 lines
- Prefer examples over explanations
- Pre-allow specific tools by exact name
- Use `disable-model-invocation: true` for destructive or side-effecting actions (push, deploy, delete)
- Use `context: fork` for isolated research tasks
