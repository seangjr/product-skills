---
name: sync
description: Sync project state with Linear — detect gaps, update stale issues, route to next action
disable-model-invocation: true
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Reconcile the codebase state with Linear. Detect mismatches between git history and issue statuses.
Surface gaps (stale issues, untracked work, missing branches) and ask the user how to resolve each one.
End by routing to the most impactful next action.
</objective>

<context>
Requires: git repo, `linear` CLI authenticated
Optional: .planning/ directory (GSD state), CLAUDE.md
</context>

<process>

## 0. Validate Environment

Check preconditions before doing anything:
- Confirm inside a git repo (`git rev-parse --is-inside-work-tree`)
- Confirm `linear` CLI is available (`which linear`)
- If either fails, explain what's missing and how to fix it, then stop

**Read project scope:**
- IF `.linear-project` exists → `PROJECT=$(cat .linear-project)`, note: "Scoped to Linear project: {name}"
- IF missing → warn: "No `.linear-project` found — showing issues from all projects. Run `/init-project` to link one."
- Store `PROJECT_FLAG` as `${PROJECT:+--project "$PROJECT"}` for use in all `linear` calls

## 1. Gather State

Collect data from both sources in parallel:

**From Linear:**
- `linear issue list -s started --limit 20 $PROJECT_FLAG` (in-progress issues)
- `linear issue list -s unstarted --limit 20 $PROJECT_FLAG` (backlog assigned to me)
- `linear issue list -s completed --limit 10 $PROJECT_FLAG` (recently completed)

**From Git:**
- `git log --oneline -30` (recent commits)
- `git branch --list` (local branches)
- Current branch name and any Linear issue ID it maps to

**From GSD (if exists):**
- Read `.planning/STATE.md` for current phase context
- Check if there's an active phase with incomplete work

## 2. Detect Gaps

Compare the two sources and classify each issue into exactly one bucket:

| Status | Definition |
|--------|-----------|
| Synced | Linear status matches git state (e.g., in-progress + commits on branch) |
| Stale in Linear | Issue is "In Progress" in Linear but no commits in 48h+ |
| Untracked work | Commits reference an issue ID but Linear still shows "Unstarted" |
| Done but open | Branch merged or work complete, but Linear issue not moved to Done |
| Orphan branch | Local branch references a Linear issue that doesn't exist or is cancelled |
| Missing branch | Linear issue is "In Progress" but no matching local branch exists |

## 3. Present Summary

Print a formatted sync report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRODUCT ► SYNC [{project-name or "All Projects"}]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Synced (N):
  ✓ ENG-123 — Feature description

Gaps Found (N):
  ⚠ ENG-456 — Stale: in-progress but no commits in 3 days
  ⚠ ENG-789 — Untracked: commits exist but Linear shows unstarted
  ⚠ ENG-012 — Done but open: branch merged, issue still in-progress

Backlog (N):
  ○ ENG-345 — Urgent — Unstarted
  ○ ENG-678 — High — Unstarted
```

## 4. Resolve Gaps (Interactive)

If gaps were found, walk through each one and ask what to do:

**For "Stale in Linear":**
Ask: "ENG-456 has been in-progress for 3 days with no commits. What happened?"
- Still working on it (keep as-is)
- Blocked — add a blocker comment to Linear
- Abandoned — move back to backlog
- Actually done — move to Done with summary

**For "Untracked work":**
Ask: "Found commits referencing ENG-789 but it's still Unstarted in Linear. Update it?"
- Yes, move to In Progress
- Yes, it's actually Done — move to Done
- No, those commits were reverted

**For "Done but open":**
Ask: "ENG-012 looks complete (branch merged). Close it in Linear?"
- Yes, move to Done with auto-generated summary
- No, there's still remaining work

**For "Orphan branch":**
Ask: "Branch `eng-999-old-feature` references a cancelled issue. Clean up?"
- Delete the branch
- Keep it

Skip the interactive loop if there are zero gaps.

## 5. Route to Next Action

Based on the final state after resolving gaps:

- IF there are unstarted high/urgent issues → suggest `/start-task`
- IF there's an active in-progress issue with no branch → suggest creating one
- IF GSD state exists with an incomplete phase → suggest `/gsd:progress`
- IF everything is synced and nothing urgent → "All clear. Pick up the next task when ready."

</process>

<success_criteria>
- [ ] Linear and git state compared with zero assumptions
- [ ] Every gap surfaced and presented to user
- [ ] User made an explicit decision on each gap
- [ ] Linear issues updated to reflect reality
- [ ] Clear next action suggested
</success_criteria>
