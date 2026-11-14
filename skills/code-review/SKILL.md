---
name: code-review
description: "Review local code changes (uncommitted/staged working tree, a commit, or a branch diff) for confirmed bugs, security and reliability flaws, documented repo convention violations, and spec drift — fast inline on small diffs, bounded parallel subagents on larger ones. Use when the user asks, in any language, to review local changes, a branch, or 'changes since X' before committing or opening a PR, or says /code-review. For reviewing an open GitHub PR by number/URL or posting PR comments, use review-pr instead."
license: MIT
metadata:
  author: marsidev
  version: "2026.11.14"
---

# Code Review (Local Changes)

Review a local git diff before it ships: verify correctness in full file context, check alignment with what was asked, and check compliance with the repository's documented rules. By default it runs fast (inline on small diffs, 1–2 bounded subagents on larger ones) and reports only verified findings with concrete triggers.

For an open GitHub PR (fetching PR threads, checking prior reviews, or posting inline GitHub comments), use `review-pr` instead.

## Usage

```
/code-review                  # current change: working tree, or branch vs base if clean
/code-review <ref>            # changes since <ref> (e.g. main, HEAD~2, a commit SHA)
/code-review <path>           # restrict to a file or directory
/code-review deep             # deeper pass for risky changes (per-group agents + git history)
/code-review --fix            # fix confirmed, low-risk bugs after validating them
```

## Step 1 - Resolve scope, spec, and repo rules

1. **Caller brief wins.** When invoked from `polish-changes` (or with an explicit brief), use its `git diff <base-ref>` scope, goal, settled decisions, and flags directly without prompting.
2. **Standalone scope:**
   - If the user gave a `<ref>`, verify `git rev-parse --verify <ref>` and use `git diff <ref>` (compares `<ref>` against the working tree so uncommitted edits are included; if clean, matches `<ref>..HEAD`).
   - Otherwise, if `git status --porcelain` is non-empty, use `git diff HEAD` plus untracked new source files.
   - If the working tree is clean, use `git diff $(git merge-base HEAD <default-branch>)` (falling back to `git diff HEAD~1` when on the default branch itself).
   - Skip generated/vendored files (`dist/`, `build/`, `*.gen.*`, lockfiles, `node_modules/`). If the diff is empty, say so and stop.
3. **Locate the spec or intent (never block if absent):**
   - Check in order: caller brief or current conversation → a plan/spec file in `.local/`, `docs/`, or `specs/` matching the branch or issue → an issue reference (`#123`) in the branch name or `git log` commit messages (fetch via `gh issue view <n> --comments` when available and fast) → commit messages.
   - If none exists, note `Spec: none found` and skip spec checks without stopping to ask.
4. **Collect documented repo standards:**
   - Read root `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `CODING_STANDARDS.md`, plus any scoped `AGENTS.md`/`CLAUDE.md` in parent directories of touched files.

## Step 2 - Right-size execution and read budget

Count **substantive source files** in the diff — exclude locale/i18n JSONs, snapshots, lockfiles, docs, and pure mechanical test fixtures (inspect those via diff hunks only, never full-file reads):

- **Small diff** (1–3 substantive source files and under ~300 changed source lines, unless `deep`): **review inline in one pass** — spawn no subagents. Never re-read files already in your current session context.
- **Medium diff** (4–14 substantive source files, unless `deep`): spawn **1 read-only subagent** via the current harness's delegation tool covering Bugs, Security, Spec, and Repo Rules in one pass (avoiding paying twice to read the same files across two subagents).
- **Large diff** (15+ substantive source files, unless `deep`): spawn **at most 2 parallel read-only subagents**:
  1. **Bugs, Security & Reliability agent** — Lens 1 + Step 4 validation rules.
  2. **Spec & Standards agent** — Lenses 2 & 3; use a faster/lighter model tier (e.g. `flash` / `mini` / `haiku`) when the harness supports model selection *(skip this agent if neither a spec nor repo rules exist)*.
- **`deep` mode** (when the user explicitly passes `deep` for a risky refactor, concurrency change, or release): group tightly-coupled files (a module + its tests + direct callers) into **up to 6 parallel read-only subagents**, and check `git log -S` / `git blame` on subtle invariants the diff alters.

**Strict 2-batch read budget (target ≤ 4 tool turns per reviewer):**
1. **Batch 1 (one parallel turn):** read all substantive touched source files in full (plus `AGENTS.md` / spec if not already embedded in the prompt).
2. **Batch 2 (one parallel turn, if needed):** read only the direct callers/callees where a signature, return value, error path, or shared state shape actually changed.
3. **Stop reading and analyze.** Never drip-feed one file read or shell `ls`/`git` command per turn across dozens of turns.

Subagents only read and report; they never edit files. Give every subagent a self-contained prompt (scope command or hunks, substantive file list, intent, collected repo rules, 2-batch read rule, and return format).

## Step 3 - Review lenses

### 1. Bugs, security, and reliability *(primary)*
Read **full touched files** (and direct callers/callees whenever a signature, return value, error path, or state shape changed) — never judge diff hunks in isolation:
- **Correctness:** logic errors, off-by-ones, null/undefined paths, wrong boundary conditions, broken caller assumptions, type casts (`as`) hiding runtime mismatches, state/cache staleness.
- **Concurrency & lifecycle:** race conditions, missing cleanup/abort on unmount or error, unhandled rejections, missing timeouts on external I/O, non-idempotent retries.
- **Security:** auth/permission bypasses, injection vectors, SSRF/path traversal, secrets or PII leaked into logs/responses, unsafe deserialization.

### 2. Spec and intent *(when a spec, issue, or brief exists)*
Compare what was asked against what was built, quoting the spec/issue/brief line:
- **Missing or partial:** deliverables or acceptance criteria left out.
- **Wrong behavior:** a requirement implemented contrary to the spec.
- **Scope creep:** unrequested behavior changes on paths outside the ask.

### 3. Standards and design
- **Documented repo rules (hard violations):** breaches of an explicit rule in `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, or `CODING_STANDARDS.md` whose path covers the file. Always cite `file` + quoted rule.
- **Structural smells *(standalone only; skip when told `simplify` already ran)*:** flag clear structural smells introduced by the diff (shotgun surgery across many files for one rule, leaky module boundary, repeated switch/map drift across files) only as **judgement calls** (`Should-Fix` or `Nitpick`), never hard violations. A documented repo pattern always overrides a smell heuristic.

## Step 4 - Validate every candidate (no false positives)

Before keeping any finding, verify it against the real code:

- **Trace the trigger statically:** name the concrete input, state, or call sequence that hits the bug, and confirm no guard, caller invariant, or framework guarantee already makes it impossible.
- **No speculative REPL or runtime experiments:** never run `node -e`, `python -c`, `jsc`, or scratch scripts to probe theoretical engine edge cases (such as year-9999 date parsing quirks across JS engines). Prove triggers from the code and realistic caller inputs; if a candidate requires contrived engine trivia or out-of-band inputs, drop it immediately.
- **Drop noise:** discard pre-existing issues untouched by the diff, issues a linter or `tsc` already catches, style preferences without a quoted repo rule, silenced lint pragmas, hypothetical future requirements, and trade-offs the user already approved.
- If you cannot prove a candidate bug from the code, drop it.

## Step 5 - Report or `--fix`

- **Default (standalone without `--fix`):** report findings only; do not edit files.
- **With `--fix` or when invoked by `polish-changes`:**
  - Fix confirmed, in-scope, low-risk bugs and unambiguous repo-rule breaches whose fix does not change intended product/UX behavior.
  - Add or update a focused regression test when fixing a real logic bug.
  - Leave spec mismatches, UX/behavior changes, and multi-file refactors unapplied and report them for the user's call.
  - Run touched typecheck, lint, and tests — unless `polish-changes` defers verification to its final step.

## Step 6 - Output format

Group verified findings by severity (`Blocking` → `Should-Fix` → `Nitpick`), keeping `[spec]` tags explicit so spec drift never hides behind clean code:

```markdown
Scope: `git diff HEAD` (5 files, +142 / -38) · Spec: #184

Blocking
1. **[bug]** `src/auth/session.ts:84` — `refreshSession` returns `null` on expired token, while caller `requireUser` (`src/api/middleware.ts:31`) dereferences `.userId` assuming it throws.
   Fix: throw `UnauthorizedError` in `refreshSession`, matching its existing JSDoc and other callers.

Should-Fix
2. **[spec]** `src/auth/session.ts:112` — Issue #184 asks to revoke sibling sessions on password reset ("invalidate all other active tokens"); current diff only revokes the current token.
   Fix: call `revokeAllForUser(userId, { except: currentTokenId })`.
3. **[convention]** `src/api/routes.ts:47` — Uses raw `console.error`; `AGENTS.md` ("Logging") requires the request-scoped `ctx.log.error`.

Summary: 1 blocking, 2 should-fix, 0 nitpicks (checked bugs, #184 spec, and AGENTS.md).
```

If nothing survives validation, output one line:
`No issues found (checked bugs, <spec or "no spec">, and <standards files or "repo conventions">).`

## Hard rules

- Read full files before flagging bugs; never guess from a 3-line hunk.
- Every bug needs a concrete trigger; every convention finding needs a quoted rule.
- Never post to GitHub or mutate git state (`add`, `commit`, `push`, `stash`).
- Never silently rewrite behavior or tests to make a speculative fix fit.
