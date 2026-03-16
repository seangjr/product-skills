---
name: init-project
description: Scaffold a new project with Next.js, GSD, skills, Linear integration, and MCP config
argument-hint: "[optional: project name or framework override]"
disable-model-invocation: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - AskUserQuestion
---

<objective>
Bootstrap a fully configured project from an empty or existing directory. Detect what already
exists (git, package.json, framework, GSD, skills) and only install what's missing. Ask the
user to configure MCP integrations. End with a clean initial commit and clear next step.
</objective>

<context>
Input: $ARGUMENTS — optional project name or framework override (e.g., "my-app", "vite", "remix")
Defaults: Next.js with TypeScript, Tailwind, App Router, src-dir, bun
Requires: bun installed, git installed
</context>

<process>

## 0. Detect Existing State

Before installing anything, audit what already exists:

```
Checks:
  [ ] git repo initialized?
  [ ] package.json exists?
  [ ] Framework detected? (next, vite, remix, nuxt, etc.)
  [ ] .gitignore exists?
  [ ] GSD installed? (.claude/commands/gsd/ or .planning/)
  [ ] Skills installed? (.claude/skills/)
  [ ] CLAUDE.md exists?
  [ ] .claude/settings.json exists?
  [ ] .claude/commands/product/ exists?
  [ ] .linear-project exists?
```

Print a brief status:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRODUCT ► INIT PROJECT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Detected:
  ✓ Git repo
  ✓ package.json (Next.js 15)
  ○ No GSD framework
  ○ No skills
  ○ No CLAUDE.md
  ○ No project settings
  ○ No .linear-project

Will install: GSD, skills, CLAUDE.md, settings, commands
```

IF everything is already set up: "Project is already fully initialized. Run `/gsd:new-project` or `/product:sync` to get started."

## 1. Git & Framework

**Git:**
- IF no git repo → `git init`
- IF git repo exists → skip

**Framework:**
- IF package.json exists → detect framework, skip scaffolding
- IF no package.json AND no argument → scaffold Next.js:
  `bunx create-next-app@latest . --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-bun`
- IF argument specifies a different framework → use that instead
  - Recognized: "vite", "remix", "nuxt", "astro", "svelte"
  - For unrecognized: ask "How should I scaffold this? Give me the init command."

**.gitignore:**
- IF .gitignore missing → create with standard entries:
  node_modules/, .env, .env.*, !.env.example, .DS_Store, *.log,
  dist/, build/, .next/, .nuxt/, .output/, .vercel/, coverage/, .planning/quick/
- IF .gitignore exists → check if it covers the essentials, append missing entries

## 2. MCP Integrations

Ask which MCP integrations to enable for this project:

"Which integrations do you need for this project?"
Options (multi-select):
- Linear (issue tracking) — recommended for all projects
- Pencil (design tool for .pen files)
- Slack (team messaging)
- Gmail (email)
- Google Calendar (scheduling)
- Discord Status (presence updates)
- None

Create or update `.claude/settings.json` with only the selected MCPs in `permissions.allow`:
```json
{
  "permissions": {
    "allow": ["mcp__claude_ai_Linear__*"]
  }
}
```

IF settings.json already exists → merge into it, don't overwrite other settings.

## 3. Link Linear Project

- IF `.linear-project` exists → skip (show linked project name)
- ELSE:
  - Read project name from `package.json` name field or directory name
  - Run `linear project list` and fuzzy-match against the project name
  - IF match found → confirm with user: "Link this repo to Linear project '{name}'?"
    - Yes → `echo "{name}" > .linear-project`
    - No → show all available projects, let user pick
  - IF no match → ask:
    "No matching Linear project found. What would you like to do?"
    - Select from existing projects (`linear project list`)
    - Create a new project (`linear project create "{name}"`)
    - Skip — don't link a Linear project
  - IF skipped → continue without `.linear-project`

## 4. Install GSD Framework

- IF GSD already installed (`.claude/commands/gsd/` exists) → skip
- ELSE → `bunx get-shit-done-cc@latest --claude --local`

## 5. Install Skills

Check each skill individually and only install missing ones:

```
Skills to check:
  web-design-guidelines  → ~/.claude/skills/web-design-guidelines or .claude/skills/
  agent-browser          → same pattern
  seo-audit              → same pattern
  copywriting            → same pattern
  gsap-react             → same pattern
  linear-cli             → same pattern
```

For each missing skill, run:
`bunx skills add $REPO --skill $NAME --local`

Skip any that are already installed (global or local).

## 6. Create CLAUDE.md

- IF CLAUDE.md exists → ask "CLAUDE.md already exists. Update it with workflow rules, or leave it?"
- IF no CLAUDE.md → create it:

Infer project name from: argument, directory name, or package.json name field.

Structure:
```markdown
# {project-name}

## Linear Integration
- Use linear-cli skill for all issue management
- Check Linear before starting tasks, update issues after completing work
- Reference issue IDs in commits: `feat(ENG-123): description`

## GSD Framework
- Follow workflow: discuss → plan → execute → verify
- Keep .planning/ updated
- Use atomic commits per task
- Run `/gsd:progress` to check state

## Skills Available
- linear-cli, web-design-guidelines, agent-browser
- seo-audit, copywriting, gsap-react

## Conventions
- Conventional commits: feat, fix, docs, chore, refactor, test
- Always reference Linear issue IDs
- Run `/product:sync` after milestones
```

## 7. Copy Slash Commands

- IF .claude/commands/product/ already has files → skip
- ELSE → copy from ~/.claude/commands/product/ into .claude/commands/product/

## 8. Initial Commit

- `git add -A`
- `git commit -m "chore: scaffold project with GSD + skills + linear-cli"`
- IF nothing to commit (all clean) → skip

## 9. Summary & Next Step

Print what was set up:
```
Setup complete:
  ✓ Next.js 15 with TypeScript + Tailwind
  ✓ GSD framework (v1.22.0)
  ✓ 6 skills installed
  ✓ Linear + Slack MCPs enabled
  ✓ CLAUDE.md with workflow rules
  ✓ Linked to Linear project: {project-name}
  ✓ /product: commands available
  ✓ Initial commit created

Next: run /gsd:new-project to define your roadmap
```

</process>

<success_criteria>
- [ ] Existing state detected — nothing overwritten or duplicated
- [ ] Framework scaffolded (or skipped if exists)
- [ ] MCP integrations configured per user choice
- [ ] GSD and skills installed (only what's missing)
- [ ] CLAUDE.md exists with workflow rules
- [ ] .linear-project created (or explicitly skipped)
- [ ] Slash commands available as /product:*
- [ ] Clean initial commit
- [ ] User knows the next step
</success_criteria>
