# Khaeli Product Skills

Workflow skills for Claude Code that integrate Linear issue tracking with git-based development. Built for the Khaeli team's discuss → plan → execute → verify workflow.

## Skills

| Skill | Description |
|-------|-------------|
| `/init-project` | Scaffold a new project with Next.js, GSD, skills, Linear, and MCP config |
| `/sync` | Reconcile codebase state with Linear — detect gaps, update stale issues |
| `/daily-standup` | Morning standup — surface blockers, stale work, and today's focus |
| `/start-task` | Pick up the next Linear issue — validate context, create branch, plan approach |
| `/finish-task` | Wrap up current task — commit, update Linear, optionally create PR |

## Install

```sh
# All skills (recommended)
npx skills add seangjr/product-skills

# Single skill
npx skills add seangjr/product-skills --skill sync

# Global (shared across all projects)
npx skills add seangjr/product-skills -g
```

## Prerequisites

- [linear CLI](https://github.com/useshortcut/linear-cli) authenticated (`linear auth`)
- git
- [bun](https://bun.sh) (for project scaffolding)
- [GSD framework](https://github.com/gsd-build/get-shit-done) (optional, `/init-project` installs it)

## Linear Project Scoping

All skills support scoping issue queries to a specific Linear project via a `.linear-project` file at repo root:

```sh
echo "Khaeli — Client Portal" > .linear-project
```

When present, all `linear issue list` calls are filtered to that project. When absent, skills show issues from all projects (same as default Linear CLI behavior).

`/init-project` handles creating this file interactively.

## Workflow

```
/init-project  →  scaffold repo, link Linear project
/daily-standup →  morning review, set focus
/start-task    →  pick issue, create branch, plan
  ... work ...
/finish-task   →  commit, update Linear, PR, next task
/sync          →  reconcile Linear ↔ git state
```
