---
name: finish-task
description: Wrap up current task — commit, update Linear, optionally create PR, route to next
argument-hint: "[optional: commit type like feat, fix, refactor]"
disable-model-invocation: true
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Cleanly close out the current task. Validate that work is complete, create an atomic commit
with a proper conventional commit message, update Linear, and optionally create a PR.
Detect incomplete work and surface it before closing.
</objective>

<context>
Input: $ARGUMENTS — optional commit type override (feat, fix, refactor, etc.)
Requires: git repo with changes, on a feature branch with a Linear issue ID
Optional: .planning/ directory
</context>

<process>

## 0. Validate Environment

Check preconditions:
- Inside a git repo
- On a feature branch (not main/master)
- `linear` CLI available

**Read project scope (silent):**
- IF `.linear-project` exists → `PROJECT=$(cat .linear-project)`, store `PROJECT_FLAG`
- IF missing → continue unscoped

Extract issue ID from branch name using `linear issue id`.
- IF no issue ID detected: ask "I can't find a Linear issue for this branch. What's the issue ID?"
- IF issue not found in Linear: warn and continue without Linear updates

## 1. Assess Completeness

Before closing, check if the work looks done:

**Check for uncommitted changes:**
- `git status --porcelain`
- IF dirty working tree: show what's changed and ask:
  "You have uncommitted changes. Include them in the final commit?"
  - Yes → stage them
  - No, they're WIP for something else → stash them first
  - Actually I'm not done → abort finish-task

**Check for incomplete markers:**
- Grep the diff for TODO, FIXME, HACK, XXX markers
- IF found: "Found N incomplete markers in your changes:"
  Show each one with file and line.
  Ask: "Are these intentional (tech debt to track) or should you fix them first?"
  - Intentional → continue, note them in Linear comment
  - Fix first → abort, user fixes, re-runs /finish-task

**Check test status (if test runner exists):**
- Look for test config (jest.config, vitest.config, playwright.config, etc.)
- IF test runner found: "Want to run tests before finishing?"
  - Yes → run tests. IF failures: show them and ask "Fix or finish anyway?"
  - No → skip
  - No test config found → skip silently

## 2. Review Changes

Show a clear summary of what changed:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRODUCT ► FINISH TASK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Issue:  ENG-123 — Add user authentication flow
Branch: eng-123-add-auth-flow

Changes:
  5 files changed, 142 insertions, 23 deletions

  M src/middleware.ts
  A src/app/(auth)/login/page.tsx
  A src/components/auth-form.tsx
  M src/lib/utils.ts
  A src/tests/auth.test.ts
```

Run `git diff --stat` and `git diff` (summarize, don't dump the full diff).

## 3. Create Commit

**Determine commit type:**
- IF argument provided (e.g., "fix") → use it
- ELSE infer from changes:
  - New files + new functionality → `feat`
  - Bug fix / correction → `fix`
  - Only test files → `test`
  - Only docs/comments → `docs`
  - Restructuring without behavior change → `refactor`
  - Config/tooling → `chore`
- If unsure, ask: "What type of change is this? feat / fix / refactor / other?"

**Build commit message:**
- Format: `{type}({issue-id}): {concise description}`
- Example: `feat(ENG-123): add magic link authentication flow`
- Description should summarize the what, not list files

**Stage and commit:**
- `git add` the relevant files (not blindly `git add -A` — exclude any unrelated changes)
- Create the commit
- Show the commit hash and message

## 4. Update Linear

- `linear issue update $ISSUE_ID -s "Done"`
- `linear issue comment add $ISSUE_ID -b "$SUMMARY"`
  - Summary should include: what was implemented, files changed, commit hash
  - If TODO markers were found in step 1, note them as follow-up items

Confirm: "Linear issue ENG-123 moved to Done with implementation summary"

## 5. Offer PR Creation

Ask: "Create a pull request?"
- Yes →
  - Determine base branch (main/master/develop)
  - Draft PR title from commit message
  - Draft PR body with: summary, Linear issue link, what changed, test instructions
  - Create PR with `gh pr create`
  - Show the PR URL
- No → skip

## 6. Route to Next Action

Check what's next:
- `linear issue list -s unstarted --limit 5 $PROJECT_FLAG` — any backlog?
- `linear issue list -s started --limit 5 $PROJECT_FLAG` — any other in-progress?

Based on state:
- IF other in-progress issues exist → "You still have N issues in progress. Resume one?"
- IF unstarted issues exist → "Next highest priority in {project-name or "your backlog"}: ENG-456 — Description. Start it? (/start-task ENG-456)"
- IF nothing → "Backlog is clear{" for " + project-name if scoped}. Nice work."

</process>

<success_criteria>
- [ ] All changes reviewed before committing
- [ ] Incomplete markers surfaced and user made a decision
- [ ] Atomic commit with proper conventional commit message
- [ ] Linear issue moved to Done with implementation summary
- [ ] PR created if user requested
- [ ] User knows what to work on next
</success_criteria>
