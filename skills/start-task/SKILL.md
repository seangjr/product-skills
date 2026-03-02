---
name: start-task
description: Pick up the next Linear issue — validate context, create branch, plan approach
argument-hint: "[optional: issue ID like ENG-123]"
disable-model-invocation: true
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Transition from idle to actively working on a Linear issue. Validate that the issue is ready
to start, create a branch, load all relevant context, and produce a brief plan. End with the
user confirming the approach before any code is written.
</objective>

<context>
Input: $ARGUMENTS — optional issue ID. If not provided, auto-select highest priority.
Requires: git repo (clean working tree), `linear` CLI authenticated
Optional: .planning/ directory, CLAUDE.md
</context>

<process>

## 0. Validate Environment

Check preconditions:
- Inside a git repo
- Working tree is clean (`git status --porcelain` should be empty)
  - IF dirty: show the status and ask "Commit, stash, or abort?"
- `linear` CLI available
- On main/default branch (warn if starting from a feature branch)

**Read project scope (silent):**
- IF `.linear-project` exists → `PROJECT=$(cat .linear-project)`, store `PROJECT_FLAG`
- IF missing → continue unscoped (don't interrupt task flow with setup prompts)

## 1. Select Issue

**If issue ID provided as argument:**
- Fetch issue details with `linear issue view $ISSUE_ID`
- Verify it's assigned to me and not already Done
- If it's already In Progress, ask: "This is already started. Resume it or pick a different one?"

**If no argument:**
- `linear issue list -s unstarted --limit 10 $PROJECT_FLAG` — fetch backlog
- IF zero unstarted issues:
  - Check for in-progress issues (`linear issue list -s started --limit 10 $PROJECT_FLAG`): "No unstarted issues. You have N in-progress. Resume one?"
  - If also zero and scoped: "Backlog is empty for {project-name}. Create a new issue in Linear or check another project."
  - If also zero and unscoped: "Backlog is empty. Create a new issue in Linear or check another team."
  - Stop here.
- Sort by priority (Urgent > High > Medium > Low > None)
- IF single top-priority issue → auto-select it
- IF multiple issues share the same top priority → ask user to pick:
  "Multiple issues at the same priority. Which one?"
  Show issue IDs, titles, and due dates if available.

## 2. Check Readiness

Before starting, validate the issue is ready:

**Check for blockers:**
- `linear issue view $ISSUE_ID` — look for blocking relations
- IF blocked by another issue: "This is blocked by ENG-XXX (title). Want to start that one instead, or start this anyway?"

**Check for missing context:**
- Does the issue have a description? If empty or very short (< 50 chars):
  Ask: "This issue has no description. Can you give me a one-liner on what needs to happen?"
  Capture the answer and update the Linear issue description.

**Check for GSD phase alignment:**
- IF .planning/ROADMAP.md exists, check if this issue maps to a planned phase
- IF it does: note the phase number and load its CONTEXT.md or PLAN.md
- IF it doesn't: note it as ad-hoc work (still fine to proceed)

## 3. Create Branch

- Extract a slug from the issue title (lowercase, hyphens, max 50 chars)
- Branch name format: `{team-prefix}-{number}-{slug}` (e.g., `eng-123-add-auth-flow`)
- `git checkout -b $BRANCH_NAME`
- Confirm: "Created branch: `$BRANCH_NAME`"

## 4. Move to In Progress

- `linear issue update $ISSUE_ID -s "In Progress"`
- Confirm: "Linear issue moved to In Progress"

## 5. Load Context & Plan

Gather all relevant context:
- Full issue description and comments from Linear
- Related GSD planning files (CONTEXT.md, RESEARCH.md, PLAN.md) if they exist
- CLAUDE.md for project conventions
- Quick scan of the codebase areas most likely affected (use issue description keywords)

Produce a brief implementation plan (3-7 bullet points):
```
Plan for ENG-123: Add user authentication flow
─────────────────────────────────────────────
1. Create auth middleware in src/middleware.ts
2. Add login page at app/(auth)/login/page.tsx
3. Implement session management with Convex Auth
4. Add protected route wrapper component
5. Write tests for auth flow
```

## 6. Confirm Approach

Ask: "Ready to start with this plan, or want to adjust anything?"
- Proceed → begin implementation
- Adjust → ask what to change, update plan, re-confirm
- Wrong task → go back to step 1

</process>

<success_criteria>
- [ ] Issue selected with clear rationale
- [ ] All blockers and missing context surfaced before starting
- [ ] Clean branch created from main/default branch
- [ ] Linear issue moved to In Progress
- [ ] Implementation plan presented and confirmed by user
- [ ] User explicitly approved before any code changes
</success_criteria>
