---
name: simplify
description: "Simplify and tighten recently changed code across three angles — Reuse, Quality/Clarity, and Efficiency — while preserving exact behavior. Cuts unnecessary nesting, gold-plating, redundant state, nested ternaries, sequential awaits, and avoidable work without sacrificing readability for fewer lines. Use when the user asks, in any language, to simplify, tighten, declutter, or reduce complexity/over-engineering in recent changes, or says /simplify."
license: MIT
metadata:
  author: marsidev
  version: "2026.11.14"
---

# Simplify

Agent-written code tends to work on the first draft while carrying extra weight: deep nesting where a guard clause works, single-use wrappers, options nobody asked for, repeated passes over the same data, or sequential `await`s that do not depend on each other. This skill reviews a change across **Reuse**, **Quality**, and **Efficiency**, applies the simplifications that preserve exact behavior, and skips anything that trades clarity for brevity.

## Usage

```
/simplify                         # lean cleanup of the current change (default)
/simplify <path>                  # limit to a file or directory
/simplify focus on <area>         # weight or narrow to a concern (e.g. memory, control flow)
/simplify thorough                # allow deeper structural cleanups (loops, local helpers, props)
/simplify --dry-run               # report findings, edit nothing
```

## Step 1 - Resolve scope and read efficiently

1. **Caller brief wins.** When invoked from another skill (such as `polish-changes`) with an explicit `git diff <base-ref>` scope, decisions already made, or angle/agent directives, honor those directly.
2. **Standalone scope:**
   - Use the path, commit range, or PR the user named.
   - Otherwise, if the working tree has changes (`git status --porcelain`), review `git diff HEAD` plus untracked new source files.
   - If the working tree is clean, review `git diff $(git merge-base HEAD <default-branch>)` (falling back to `git diff HEAD~1` when on the default branch itself).
3. **Skip** generated or vendored files (`dist/`, `build/`, `*.gen.*`, lockfiles, `node_modules/`), locale/i18n JSONs, snapshots, and files outside the change.
4. **Zero re-reads + parallel batching:**
   - **Never re-read files already in your current session context** unless something external modified them since you last read or edited them.
   - For unseen files, read full files (up to the tool's per-call line cap — never slice a 500-line file into 3 sequential reads) together with `AGENTS.md` / `CLAUDE.md` / `CONTRIBUTING.md` in **one parallel tool batch**. Follow the repo's actual conventions — never impose external style preferences.

## Step 2 - Review angles (lean by default)

Cover active angles with a **lean bar** by default (and always when called from `polish-changes`): cut clear bloat inside touched functions without reshaping component contracts or static UI markup. Unlock structural cleanups only when the user passes `thorough`.

### 1. Reuse *(skip when told `reuse-check` already ran)*
- Existing helpers in the same file, module, or already-imported package that replace new inline code with identical behavior.
- Substantial duplicated logic (not 2-line expressions) introduced within the diff that belongs in one local helper.
- Do not extract cross-module abstractions for two call sites that merely look similar.

### 2. Quality and clarity
- **Flatten control flow:** replace deep `if`/`else` pyramids with guard clauses and early returns; replace nested ternaries with `if`/`else` or `switch`.
- **Remove gold-plating:** delete single-use pass-through wrappers, unused parameters/options/props/state, dead branches, speculative fallback paths for impossible states, and unnecessary intermediate variables.
- **Tighten types and names locally:** align variable names with domain terms already in the file; replace loose casts where existing types already express the constraint cleanly.
- **`thorough` mode only *(skip by default and inside `polish-changes`)*:**
  - Consolidating split state objects or changing internal component/helper signatures across files (in default lean mode, stay **file-local** unless deleting a completely unused prop/argument).
  - Converting repeated static UI markup (`<th>`, buttons, form rows) into config arrays + `.map()`/`v-for` loops (in default lean mode, leave 3–6 explicit, readable template blocks alone).
  - Extracting small 2–5 line formatting helpers or `EMPTY_*` constants within a file unless they encode a domain invariant that must not drift.

### 3. Efficiency
- **Redundant work:** repeated parsing, serialization, regex compilation, object cloning, or multi-pass iterations where a single clear pass or hoisted value suffices.
- **Missed concurrency:** sequential `await` calls on independent operations that can run safely via `Promise.all`.
- **Loop and query waste:** N+1 database/API/filesystem calls inside loops, linear `.find()`/`.includes()` lookups inside hot loops where a `Map`/`Set` fits cleanly.
- **Frontend churn:** derived values stored in state + effects instead of computed inline, or unstable inline objects/callbacks where they demonstrably trigger avoidable work.

## Step 3 - Balance and guardrails

Every edit must pass these checks:

- **Exact behavior preservation.** Outputs, side effects, ordering that callers rely on, error types/codes/messages, and public signatures stay identical. If an "improvement" changes edge-case behavior, leave it alone or report it as a question.
- **Clarity over line count.** Explicit, readable code beats dense one-liners, clever `.reduce()` chains, or compressed boolean algebra. Removing lines is a byproduct of removing complexity, not the goal.
- **Keep earned abstractions.** Do not inline a helper or collapse a boundary that isolates I/O, names a non-obvious step, or makes a unit testable.
- **Respect settled decisions.** Never reverse an approach or trade-off the user already chose.
- **Leave comment-only passes to `trim-comments`.** When running inside `polish-changes`, do not spend turns auditing comments (`trim-comments` runs too); standalone, remove only obvious narration attached to code you are already simplifying.
- **Protect tests.** Do not weaken or rewrite existing assertions to make a refactor fit. Simplifying verbose setup in *new* tests added by the diff is fine when coverage stays identical.

## Step 4 - Right-size execution

Count changed source lines in scope (excluding tests, docs, and locale files):

- **Under ~800 lines, or when asked to use one agent:** run all active angles in a single pass — directly in the current session when context is small, or via one read-only subagent (using the current harness's delegation tool) when context is already heavy.
- **~800 lines or more:** fan out one read-only subagent per active angle in parallel via the current harness's delegation tool, passing each the scope command, repo conventions, and brief. Aggregate and deduplicate their findings in the parent before editing, so two subagents never edit the same file concurrently.

## Step 5 - Apply and verify

Skip edits under `--dry-run`.

1. Apply only findings that clearly earn their keep. Keep edits surgical — do not reformat untouched lines or reorder unrelated imports.
2. **Verification:**
   - **Standalone:** run the project's typecheck, linter, and affected tests for the files you edited. If an edit breaks a check and the fix is not immediate, revert that edit.
   - **Called from `polish-changes` (or when told to defer checks):** skip running typecheck, tests, and lint so the caller runs them once at the end.

## Step 6 - Report

Compact, grouped by file, with angle tags and net line delta:

```
src/sync/worker.ts
  [quality]     L48   flattened 3-level nested if/else into early returns
  [efficiency]  L92   parallelized independent metadata + cursor reads with Promise.all
  [quality]     L134  replaced nested ternary on status badge with switch

src/sync/types.ts
  [quality]     L19   dropped unused `allowPartial` option and pass-through wrapper

Net: +14 / -39 lines across 2 files.
```

If the code is already clean and nothing clears the bar, say so in one line and touch nothing.

## Hard rules

- Never alter observable behavior, public contracts, or error semantics.
- Never apply external style dogma (`function` vs arrow, import extensions, return-type verbosity) unless the repo's own docs or linter require it.
- Never widen scope beyond the diff under review.
- Never commit, stage, or push unless the user explicitly asks.
