# Changelog

## v1.1.0 (2026-03-16)

### Fixed

- Use `product:` namespace prefix in all cross-skill references (e.g., `/product:start-task` instead of `/start-task`) so commands resolve correctly when installed as a skill pack

### Skills affected

- `daily-standup` — updated routing to `/product:start-task`
- `finish-task` — updated routing to `/product:finish-task`, `/product:start-task`
- `init-project` — updated routing to `/product:sync`, success criteria
- `sync` — updated routing to `/product:init-project`, `/product:start-task`

## v1.0.0 (2026-03-15)

### Added

- `/maintain` skill — codebase health check with 17 checks across code quality, dependencies, git hygiene, Linear health, and build status. Supports quick, full (parallel agents), --fix, and --category modes
- Parallel agent execution in `/maintain` full mode for concurrent health checks
- `/init-project` skill — scaffold projects with Next.js, GSD, skills, Linear, and MCP config
- `/sync` skill — reconcile codebase state with Linear, detect gaps, resolve interactively
- `/daily-standup` skill — morning standup with blockers, stale work detection, and focus planning
- `/start-task` skill — pick up Linear issues, validate context, create branch, plan approach
- `/finish-task` skill — wrap up tasks with conventional commits, Linear updates, and PR creation
- `.linear-project` scoping support across all skills
- README with installation, prerequisites, and workflow documentation

### Fixed

- Correct linear CLI repo URL in README
- Correct GSD framework repo URL in README
