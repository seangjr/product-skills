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
  - Task
  - AskUserQuestion
---

<objective>
Run a structured codebase health check. Surface issues early with clear severity indicators,
auto-fix what's fixable, and route to the most impactful next action. Quick mode runs checks
directly for speed; full mode spawns parallel agents per category for throughput.

This command acts as a thin orchestrator — it validates the environment, spawns agents for
heavy work, collects structured results, and presents the unified report. Agents get fresh
context windows and return short JSON summaries, keeping orchestrator context lean.
</objective>

<context>
Input: $ARGUMENTS — mode selector
  - (empty) or "quick" → fast checks only (code quality + git hygiene), run directly
  - "full" → all 17 checks, spawn parallel agents per category
  - "--fix" → quick checks + auto-fix all fixable issues without prompting
  - "--category=code|deps|git|linear|build" → single category via agent
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

**Detect available tooling (run these in parallel via Bash):**
- TSC: check for `tsconfig.json` → HAS_TSC
- Linter: check for `biome.json` or `biome.jsonc` → HAS_BIOME; else check for `.eslintrc*` or `eslint.config.*` → HAS_ESLINT
- Knip: check if `knip` is in package.json dependencies → HAS_KNIP
- Linear: `which linear 2>/dev/null` → HAS_LINEAR
- Test runner: look for `vitest.config.*`, `jest.config.*`, `playwright.config.*` → HAS_TESTS, TEST_RUNNER
- Env example: check for `.env.example` → HAS_ENV_EXAMPLE
- Build script: check if `package.json` has a `build` script → HAS_BUILD

Store detected tooling as TOOLING_CONTEXT (a short text block to pass to agents).

## 1. Run Checks

### Strategy by mode

**quick / fix mode** — Run checks directly (no agents). Only 6 checks across code + git,
fast enough to run inline without context overhead.

**full mode** — Spawn up to 5 parallel agents, one per category. Each agent runs its checks
and returns a structured JSON summary. This keeps the orchestrator lean and runs all 17
checks concurrently.

**category mode** — Spawn 1 agent for the requested category.

---

### Quick / Fix Mode (run directly, no agents)

Run these 6 checks inline using Bash/Grep:

#### Code Quality

**Check 1: TypeScript type errors**
- IF HAS_TSC → run `bunx tsc --noEmit 2>&1`, count errors
- ELSE → skip: "TypeScript not configured"

**Check 2: Lint violations**
- IF HAS_BIOME → run `bunx biome check . 2>&1`, parse counts, mark fixable
- ELIF HAS_ESLINT → run `bun lint 2>&1`, parse counts, mark fixable
- ELSE → skip: "No linter configured"

**Check 3: Code markers**
- Grep `src/`, `app/`, `lib/`, `components/` for TODO, FIXME, HACK, @ts-ignore, `as any`
- Count each type, warn if any

#### Git Hygiene

**Check 7: Working tree status**
- `git status --porcelain` → count changes

**Check 8: Stale/merged branches**
- `git branch --merged` excluding current/main/master → count, mark fixable

**Check 9: Branch-issue alignment**
- `linear issue id 2>/dev/null` → check current branch maps to an issue

Skip to **Step 2: Present Report** after collecting results.

---

### Full Mode (parallel agents)

Spawn up to 5 agents simultaneously using Task. Each agent is `general-purpose` with `model: "haiku"`.
Each agent receives the TOOLING_CONTEXT and returns a JSON array of check results.

**IMPORTANT:** Spawn all applicable agents in a single message (parallel tool calls).
Each agent returns ONLY a short JSON summary — never raw command output.

#### Agent prompt template

Each agent receives a prompt structured as:

```
You are a codebase health checker. Run the checks below and return ONLY a JSON array.
Do NOT return anything else — no explanation, no markdown, just the JSON.

Working directory: {cwd}
Tooling: {TOOLING_CONTEXT}

Each check result must be a JSON object:
{"check": "name", "status": "pass|fail|warn|skip", "message": "short description", "fixable": false, "fix_command": ""}

Checks to run:
{category-specific check instructions}
```

#### Agent 1: Code Quality
Spawn with `description: "Health: code quality"`, `model: "haiku"`.

Prompt includes checks 1-3 (TypeScript, lint, code markers) with the exact
commands and parsing logic from the check definitions below.

#### Agent 2: Dependencies
Spawn with `description: "Health: dependencies"`, `model: "haiku"`.

Prompt includes checks 4-6 (unused deps, lockfile freshness, outdated packages).

#### Agent 3: Git Hygiene
Spawn with `description: "Health: git hygiene"`, `model: "haiku"`.

Prompt includes checks 7-9 (working tree, merged branches, branch-issue alignment).

#### Agent 4: Linear Health
- IF NOT HAS_LINEAR → skip entire category (record all 4 checks as skip), do NOT spawn agent
- ELSE → spawn with `description: "Health: linear"`, `model: "haiku"`.

Prompt includes checks 10-13 (WIP overload, stale issues, orphaned branches, backlog health).
Include PROJECT_FLAG if `.linear-project` exists.

#### Agent 5: Build Health
Spawn with `description: "Health: build"`, `model: "haiku"`.

Prompt includes checks 14-17 (build, bundle size, tests, env validation).

**NOTE:** Build checks can be slow (build + tests). This agent may take longer than others.
That's fine — the orchestrator waits for all agents before presenting the report.

#### Category Mode

Spawn only the single agent matching CATEGORY. Same prompt structure as above.

---

### Check Definitions (included in agent prompts)

#### Code Quality

**Check 1: TypeScript type errors**
- IF tsconfig.json exists:
  - Run `bunx tsc --noEmit 2>&1`
  - IF exit 0 → pass: "No type errors"
  - ELSE → count "error TS" lines, fail: "N type error(s)"
- ELSE → skip: "TypeScript not configured"

**Check 2: Lint violations**
- IF biome.json/biome.jsonc exists:
  - Run `bunx biome check . 2>&1`
  - IF clean → pass: "No lint issues"
  - ELSE → fail/warn with count, fixable=true, fix_command="bunx biome check . --fix"
- ELIF eslint config exists:
  - Run `bun lint 2>&1`
  - IF clean → pass: "No lint issues"
  - ELSE → fail/warn, fixable=true, fix_command="bun lint --fix"
- ELSE → skip: "No linter configured"

**Check 3: Code markers**
- Grep src/app/lib/components for: TODO, FIXME, HACK, @ts-ignore, `as any`
- Count each type
- IF zero → pass: "No code markers found"
- IF any → warn with counts (only types with count > 0)

#### Dependencies

**Check 4: Unused dependencies**
- IF knip in package.json → run `bunx knip --no-progress 2>&1`
  - Clean → pass | else → warn, fixable=true, fix_command="bunx knip --fix"
- ELSE → skip: "knip not installed"

**Check 5: Lockfile freshness**
- Check bun.lockb/bun.lock age via `stat -f %m` (macOS)
- > 30 days → warn | fresh → pass | missing → warn

**Check 6: Outdated packages**
- Run `bunx npm-check-updates --target minor 2>&1 | tail -20`
- No updates → pass | else → warn with count

#### Git Hygiene

**Check 7: Working tree status**
- `git status --porcelain` → clean = pass, else warn with count

**Check 8: Stale/merged branches**
- `git branch --merged` excluding current/main/master
- Zero → pass | else → warn, fixable=true, fix_command="git branch -d {branch}" per branch

**Check 9: Branch-issue alignment**
- `linear issue id 2>/dev/null` on current branch
- On main/master → skip | ID found → pass | else → warn

#### Linear Health

**Check 10: WIP overload**
- `linear issue list -s started --limit 10 $PROJECT_FLAG` → count
- ≤2 → pass | 3 → warn | ≥4 → fail

**Check 11: Stale issues**
- For each in-progress issue, check commits referencing it in last 48h
- None stale → pass | else → warn per issue

**Check 12: Orphaned branches**
- Extract issue IDs from branch names, check if cancelled/done in Linear
- Found → warn, fixable=true | none → pass

**Check 13: Backlog health**
- `linear issue list -s unstarted --limit 20 $PROJECT_FLAG`
- Check for overdue items → warn if any | else → pass

#### Build Health

**Check 14: Build check**
- IF build script exists → `bun run build 2>&1`
  - Exit 0 → pass | else → fail
- ELSE → skip: "No build script"

**Check 15: Bundle size**
- Check .next/ or dist/ → `du -sh`
- > 500MB → warn | else → pass | missing → skip

**Check 16: Test summary**
- IF test config exists → run appropriate runner (vitest/jest/playwright)
  - All pass → pass | failures → fail with counts
- ELSE → skip: "No test runner configured"

**Check 17: Env validation**
- IF .env.example exists → diff keys against .env
  - All present → pass | missing → warn with key names
- ELSE → skip: "No .env.example found"

---

## 2. Present Report

**Collect results** — merge all agent JSON responses (or inline results for quick mode)
into a single RESULTS array.

**Calculate health score:**
- Start at 100
- Each FAIL: subtract 10
- Each WARN: subtract 3
- Floor at 0

**Print the report** following the `PRODUCT ►` header bar style:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PRODUCT ► MAINTAIN [{PROJECT_NAME}] ({MODE})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Code Quality
  {icon} Types: {message}
  {icon} Lint: {message} [fixable: N]
  {icon} Code markers: {message}

Git Hygiene
  {icon} Working tree: {message}
  {icon} Branches: {message} [fixable]
  {icon} Branch-issue: {message}

Dependencies                          ← only in full/category=deps
  {icon} Unused deps: {message}
  {icon} Lockfile: {message}
  {icon} Outdated: {message}

Linear Health                         ← only in full/category=linear
  {icon} WIP: {message}
  {icon} Stale issues: {message}
  {icon} Orphaned branches: {message}
  {icon} Backlog: {message}

Build Health                          ← only in full/category=build
  {icon} Build: {message}
  {icon} Bundle size: {message}
  {icon} Tests: {message}
  {icon} Env vars: {message}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 HEALTH: {score}/100  ✗ N fail  ⚠ N warn  ✓ N pass  ○ N skip
 {IF fixable_count > 0: "Fixable: N issue(s) — run /product:maintain --fix"}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Status icons: pass → ✓, fail → ✗, warn → ⚠, skip → ○

Health thresholds: 80-100 = healthy, 60-79 = needs attention, <60 = unhealthy

## 3. Drill-Down (unless --fix mode)

IF any FAIL or WARN results AND mode is NOT fix:
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
- Re-run ONLY the affected checks (directly, no agents — the fix scope is small)
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
- [ ] In full mode, agents spawned in parallel and returned structured JSON
- [ ] Orchestrator stayed lean — no raw command output in context, only JSON summaries
- [ ] Report printed with correct severity indicators and health score
- [ ] Fixable issues identified and fix offered (or auto-applied with --fix)
- [ ] Re-verification ran after any fixes, showing before/after delta
- [ ] Clear next action suggested based on results
</success_criteria>
