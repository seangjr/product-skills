---
name: maintain
description: Codebase health check — type errors, lint, git hygiene, deps, Linear, build, and auto-fix
disable-model-invocation: true
argument-hint: "[quick|full|--fix|--category=code|deps|git|linear|build]"
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Run a structured codebase health check. Surface issues early with clear severity indicators,
auto-fix what's fixable, and route to the most impactful next action. Quick mode runs in ~10s
for inner-loop use; full mode covers all 17 checks.
</objective>

<context>
Input: $ARGUMENTS — mode selector
  - (empty) or "quick" → fast checks only (code quality + git hygiene)
  - "full" → all 17 checks
  - "--fix" → quick checks + auto-fix all fixable issues without prompting
  - "--category=code|deps|git|linear|build" → single category only
Requires: git repo, package.json
Optional: tsc, eslint/biome, knip, linear CLI, test runner, .env.example
</context>

<process>

## 0. Parse Arguments & Validate Environment

**Parse mode from $ARGUMENTS:**
- IF empty or "quick" → MODE=quick
- IF "full" → MODE=full
- IF "--fix" → MODE=fix (runs quick checks, then auto-fixes everything fixable)
- IF "--category=X" → MODE=category, CATEGORY=X (one of: code, deps, git, linear, build)
- IF unrecognized → show usage and stop

**Validate preconditions:**
- `git rev-parse --is-inside-work-tree` — must be in a git repo
- Check for `package.json` — must exist
- IF either fails → explain what's missing and stop

**Read project name:**
- Extract `name` from `package.json`
- Store as PROJECT_NAME for the report header

**Detect available tooling (run all in parallel):**
- TSC: `bunx tsc --version 2>/dev/null` → HAS_TSC
- Linter: check for `biome.json` or `biome.jsonc` → HAS_BIOME; else check for `.eslintrc*` or `eslint.config.*` → HAS_ESLINT
- Knip: check if `knip` is in package.json dependencies → HAS_KNIP
- Linear: `which linear 2>/dev/null` → HAS_LINEAR
- Test runner: look for `vitest.config.*`, `jest.config.*`, `playwright.config.*` → HAS_TESTS, TEST_RUNNER
- Env example: check for `.env.example` → HAS_ENV_EXAMPLE
- Build script: check if `package.json` has a `build` script → HAS_BUILD

**Initialize tracking arrays:**
- RESULTS = [] (each entry: {category, check, status, message, fixable, fix_command})
- Status values: pass, fail, warn, skip

## 1. Run Checks

Run only the checks appropriate for the current mode. For each check, record the result.
Skip checks whose tooling is missing — record as status=skip with message "Not configured".

### Code Quality [quick, full, fix, category=code]

**Check 1: TypeScript type errors**
- IF HAS_TSC:
  - Run `bunx tsc --noEmit 2>&1`
  - IF exit 0 → pass: "No type errors"
  - ELSE → count errors from output, fail: "N type error(s)"
- ELSE → skip: "TypeScript not configured"

**Check 2: Lint violations**
- IF HAS_BIOME:
  - Run `bunx biome check . 2>&1` (or `bunx biome ci . 2>&1` for non-fix mode)
  - Parse error/warning counts from output
  - IF clean → pass: "No lint issues"
  - ELSE → fail/warn based on error count, mark fixable=true, fix_command="bunx biome check . --fix"
- ELIF HAS_ESLINT:
  - Run `bun lint 2>&1` (assumes package.json lint script exists)
  - Parse error/warning counts
  - IF clean → pass: "No lint issues"
  - ELSE → fail/warn, mark fixable=true, fix_command="bun lint --fix"
- ELSE → skip: "No linter configured"

**Check 3: Code markers**
- Grep the `src/` directory (or `app/`, `lib/`, `components/` — whatever exists) for:
  - TODO, FIXME, HACK, @ts-ignore, `as any`
- Count each type
- IF zero → pass: "No code markers found"
- IF any → warn: "N TODO, N FIXME, N HACK, N @ts-ignore, N `as any`"
  - Only list types that have count > 0

### Dependencies [full, category=deps]

**Check 4: Unused dependencies**
- IF HAS_KNIP:
  - Run `bunx knip --no-progress 2>&1`
  - IF clean → pass: "No unused dependencies"
  - ELSE → warn with count, fixable=true, fix_command="bunx knip --fix"
- ELSE → skip: "knip not installed"

**Check 5: Lockfile freshness**
- Check if `bun.lockb` or `bun.lock` exists
- IF exists → check age with `stat -f %m` (macOS) or `stat -c %Y` (Linux)
  - IF older than 30 days → warn: "Lockfile is N days old — consider running `bun install`"
  - ELSE → pass: "Lockfile is fresh (N days old)"
- ELSE → warn: "No lockfile found"

**Check 6: Outdated packages**
- Run `bunx npm-check-updates --target minor 2>&1 | tail -20`
- IF no updates → pass: "All packages up to date"
- ELSE → count outdated, warn: "N package(s) have minor updates available"

### Git Hygiene [quick, full, fix, category=git]

**Check 7: Working tree status**
- Run `git status --porcelain`
- IF clean → pass: "Clean working tree"
- ELSE → count changes, warn: "N uncommitted change(s)"

**Check 8: Stale/merged branches**
- Run `git branch --merged` (exclude current branch and main/master)
- Count merged branches
- IF zero → pass: "No stale branches"
- ELSE → warn: "N merged branch(es)", fixable=true, fix_command will be `git branch -d {branch}` for each

**Check 9: Branch-issue alignment**
- Extract Linear issue ID from current branch via `linear issue id 2>/dev/null`
- IF on main/master → skip: "On default branch"
- ELIF issue ID found → pass: "branch-name → ISSUE-ID"
- ELSE → warn: "Branch doesn't reference a Linear issue"

### Linear Health [full, category=linear]

- IF NOT HAS_LINEAR → skip entire section: "Linear CLI not available"

**Read project scope:**
- IF `.linear-project` exists → read it, set PROJECT_FLAG
- ELSE → continue unscoped

**Check 10: WIP overload**
- Run `linear issue list -s started --limit 10 $PROJECT_FLAG`
- Count in-progress issues
- IF <= 2 → pass: "N issue(s) in progress"
- IF 3 → warn: "3 issues in progress — consider finishing before starting new ones"
- IF >= 4 → fail: "N issues in progress — WIP overload"

**Check 11: Stale issues**
- From the in-progress list, for each issue check if there are commits referencing it in the last 48h
- Issues with no recent commits → warn per issue
- IF none stale → pass: "No stale in-progress issues"

**Check 12: Orphaned branches**
- Run `git branch --list` and extract issue IDs from branch names
- For each, check if the Linear issue is cancelled or done
- IF found → warn: "N orphaned branch(es) reference closed issues", fixable=true
- IF none → pass: "No orphaned branches"

**Check 13: Backlog health**
- Run `linear issue list -s unstarted --limit 20 $PROJECT_FLAG`
- Check for overdue items (due date in the past)
- IF any overdue → warn: "N overdue item(s) in backlog"
- ELSE → pass: "Backlog is healthy"

### Build Health [full, category=build]

**Check 14: Build check**
- IF HAS_BUILD:
  - Run `bun run build 2>&1`
  - IF exit 0 → pass: "Build succeeded"
  - ELSE → fail: "Build failed"
- ELSE → skip: "No build script"

**Check 15: Bundle size**
- Check for `.next/` or `dist/` directory
- IF exists → run `du -sh .next/` or `du -sh dist/`
  - Report size as info/pass: "Bundle size: X"
  - IF > 500MB → warn: "Bundle size is large (X)"
- ELSE → skip: "No build output found"

**Check 16: Test summary**
- IF HAS_TESTS:
  - Determine runner and run:
    - vitest: `bunx vitest run --reporter=verbose 2>&1`
    - jest: `bunx jest --verbose 2>&1`
    - playwright: `bunx playwright test 2>&1`
  - Parse pass/fail counts
  - IF all pass → pass: "N test(s) passed"
  - IF failures → fail: "N passed, N failed"
- ELSE → skip: "No test runner configured"

**Check 17: Env validation**
- IF HAS_ENV_EXAMPLE:
  - Read keys from `.env.example`
  - Read keys from `.env` (if exists)
  - Diff the keys
  - IF all keys present → pass: "All env vars defined"
  - IF missing keys → warn: "N env var(s) missing from .env: KEY1, KEY2"
- ELSE → skip: "No .env.example found"

## 2. Present Report

Calculate health score:
- Start at 100
- Each FAIL: subtract 10
- Each WARN: subtract 3
- Floor at 0

Print the report following the `PRODUCT ►` header bar style:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRODUCT ► MAINTAIN [{PROJECT_NAME}] ({MODE})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Code Quality
  {status_icon} Types: {message}
  {status_icon} Lint: {message} [fixable: N]
  {status_icon} Code markers: {message}

Git Hygiene
  {status_icon} Working tree: {message}
  {status_icon} Branches: {message} [fixable]
  {status_icon} Branch-issue: {message}

Dependencies                          ← only in full/category=deps
  ...

Linear Health                         ← only in full/category=linear
  ...

Build Health                          ← only in full/category=build
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 HEALTH: {score}/100  ✗ N fail  ⚠ N warn  ✓ N pass  ○ N skip
 {IF fixable_count > 0: "Fixable: N issue(s) — run /product:maintain --fix"}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Status icons:
- pass → ✓
- fail → ✗
- warn → ⚠
- skip → ○

Health thresholds:
- 80-100 → healthy
- 60-79 → needs attention
- <60 → unhealthy

## 3. Drill-Down (unless --fix mode)

IF any FAIL or WARN results exist AND mode is NOT fix:
- Ask: "Want details on any category? (code / deps / git / linear / build / all / skip)"
- IF user picks one → show the full command output for those checks
- IF skip → move on

## 4. Auto-Fix

**Collect fixable issues** from results where fixable=true.

IF MODE=fix:
- Run all fix commands automatically without prompting
- Show each fix as it runs

ELIF fixable issues exist:
- List all fixable issues
- Ask: "Fix all / pick which to fix / skip?"
  - Fix all → run all fix commands
  - Pick → present each fixable issue, ask yes/no
  - Skip → move on

**Fix commands:**
| Issue | Command |
|---|---|
| Lint violations (biome) | `bunx biome check . --fix` |
| Lint violations (eslint) | `bun lint --fix` |
| Unused deps | `bunx knip --fix` |
| Merged branches | `git branch -d {branch}` for each |
| Orphaned branches | `git branch -d {branch}` for each |

## 5. Re-Verify (after fixes)

IF any fixes were applied:
- Re-run ONLY the checks that had fixable issues
- Show before/after delta:
  ```
  Re-verified after fixes:
    Lint: ✗ 3 errors → ✓ No lint issues
    Branches: ⚠ 2 merged → ✓ No stale branches

  HEALTH: 72 → 88 (+16)
  ```
- Recalculate and display new health score

## 6. Route to Next Action

Based on the final state, suggest the most impactful next step:

- IF type errors → "Fix type errors first — they block everything downstream"
- IF test failures → "Fix failing tests before proceeding"
- IF lint errors (unfixed) → "Run `/product:maintain --fix` to clean up lint issues"
- IF stale Linear issues → "Run `/product:sync` to reconcile stale issues"
- IF everything healthy → "Codebase is healthy. Run `/product:start-task` to pick up work"
- IF just finished a task → "Ready for PR? Run `/product:finish-task`"

</process>

<success_criteria>
- [ ] All checks for the selected mode ran (or were skipped gracefully with ○)
- [ ] Report printed with correct severity indicators and health score
- [ ] Fixable issues identified and fix offered (or auto-applied with --fix)
- [ ] Re-verification ran after any fixes, showing before/after delta
- [ ] Clear next action suggested based on results
</success_criteria>
