---
name: polish-changes
description: "Fast pre-commit / pre-PR second look over recent changes covering five lenses: reuse-check (swap reinvented helpers for existing ones), lean simplify, code-review (fix confirmed bugs), a gap check (docs, READMEs, skills, logs, monitoring and alerting left stale or missing), and trim-comments. Stages the current work as a baseline first (`git diff` shows only what polish changed), runs read-only discovery in one parallel subagent wave on larger diffs (or inline on small diffs), applies fixes and cleanup in a single edit pass without re-reading files already in context, and runs checks once at the end. Use when the user asks, in any language, to 'polish', 'clean up', 'tidy' or 'review and fix' what was just written, asks for reuse-check, simplify, code-review and trim-comments together, asks what docs, skills or monitoring a change still needs, or says /polish-changes."
license: MIT
metadata:
  author: marsidev
  version: "2026.11.14"
---

# Polish changes

A long agent turn leaves a working change that is usually slightly larger or rougher than it needs to be: a helper the codebase already has, nested control flow or dead branches, an overlooked bug, stale docs or skills, and obvious narration comments. This skill gives the change a fast second look before committing or opening a PR, applies high-signal fixes and cleanup, and keeps its edits separate from the original work in git's index so `git diff` shows only what polish changed.

The goal is a smaller, clearer, correct change in a few minutes — not a second refactor session. A polish that adds more code than it removes, reshapes working UI/component contracts just for compactness, or reopens decisions the user already made has gone wrong. The gap check is the one lens expected to add lines, and only to bring docs and skills in line with the code.

## Usage

```
/polish-changes              # polish the current change
/polish-changes deep         # run code-review in deep mode (for risky changes)
```

## 1. Set the baseline

The comparison relies on git's index: the **staged** changes are the original work, and the **unstaged** changes are what polish changed.

- **Nothing staged, working tree has changes:** check untracked files (`git status --short`) first and leave out anything that shouldn't be in the repo (secrets, `.env` files, build output, large binaries). Stage everything else as the baseline: `git add -A` when all untracked files belong, or `git add -u` plus the wanted new paths when leaving files out (say what you left out). `<base-ref>` is `HEAD`.
- **Everything already staged, nothing unstaged:** the user set the baseline. Use it. `<base-ref>` is `HEAD`.
- **Clean working tree:** the work is already committed, and `HEAD` is the baseline. Nothing to stage. `<base-ref>` is `$(git merge-base HEAD <default-branch>)` (or `HEAD~1` when on the default branch itself, or when the user asked to polish only the last commit).
- **Some staged and some unstaged:** the baseline is ambiguous. Ask before touching the index.

In every case, `git diff <base-ref>` compares `<base-ref>` against the working tree, so it covers both the baseline and any unstaged polish edits.

From here until the end, don't stage, commit, stash or reset anything. Doing so would mix polish's edits into the baseline.

## 2. Write the brief

Every subagent or pass gets the same short brief, so nothing has to rediscover the context or argue with it:

- **Scope:** the exact `git diff <base-ref>` command from step 1, the list of **substantive source files** (excluding locale/i18n JSONs, snapshots, lockfiles, and docs), and anything below `<base-ref>` that is out of scope (commits from another PR the branch sits on, for example).
- **Goal:** one or two lines on what the change is for.
- **Decisions already made:** what the user chose or approved in the conversation (an approach, a trade-off, something deliberately left out). Passes must not propose reversing these; at most they report a concrete risk in one line.
- **Size:** substantive source file count and changed source lines (excluding tests, docs and locale files).

## 3. Discover in parallel, edit once

`/polish-changes` usually runs at the end of a long implementation session where every extra parent tool round-trip re-sends a large conversation history. Avoid serial multi-pass churn and redundant file reads:

- **Never re-read files already in your session context** unless something external modified them.
- **Read each sibling skill's `SKILL.md` in one parallel batch** (`reuse-check`, `simplify`, `code-review`, `trim-comments`) if not already loaded in this session.

Choose execution path by substantive source file count:

### Fast path — small diff (1–3 substantive source files and under ~250 changed source lines, unless `deep`)

Run inline without spawning subagents (unless `reuse-check` or `gap check` needs a broad search across unfamiliar packages):
1. Batch any unseen file reads + targeted `grep` searches for reusable helpers and stale doc/skill/log references in **one parallel tool turn**.
2. Apply all five lenses (`reuse-check`, lean `simplify`, `code-review --fix`, `gap check`, `trim-comments`) in **one surgical edit pass**, following "What to apply" below.

### Standard path — larger diff (4+ substantive source files, or `deep`)

1. **Wave 1 — One parallel read-only discovery wave in fresh subagents:**
   Launch in the **same turn** via the current harness's delegation tool so wall-clock time is `max(A, B)` while the heavy parent context waits once:
   - **Subagent A (`code-review`):** pass the brief, `git diff <base-ref>` (or hunks), collected repo rules (`AGENTS.md` / `CLAUDE.md`), and the rules from `code-review` (`deep` only when requested; skip structural smells; enforce its 2-batch read budget and no-REPL rule). Returns verified bugs, repo-rule breaches, and spec/UX questions — edits nothing. *(In `deep` mode on large diffs, split into module-grouped `code-review` subagents in this same parallel wave.)*
   - **Subagent B (`reuse + gap scan`):** pass the brief, `git diff <base-ref>`, the inventory of newly added helpers/idioms plus changed/removed surface symbols (routes, flags, env vars, exports, error/log/metric names), and the search rules from `reuse-check` + "The gap check" below. Searches shared packages, docs, skills, logs, and alerts in batched parallel calls, compares candidate helper implementations in code on concrete edge cases, and returns verified `reuse` swaps + classified `stale`/`missing` gaps — edits nothing.
2. **Wave 2 — One combined parent edit pass (zero redundant re-reads):**
   Using the files already in your session context (reading only previously unseen files in one parallel batch), apply all confirmed changes per file in one pass:
   - verified `reuse-check` helper swaps;
   - confirmed `code-review` bug fixes (plus focused regression tests where needed);
   - **lean `simplify`** (default mode of `simplify`, skipping Reuse: flatten nested ternaries/pyramids, drop dead code/unused props, `Promise.all` on independent awaits — **no** static-UI-to-loop rewrites, micro-helper sprawl, or cross-file signature churn);
   - **`trim-comments`** (CUT/SHORTEN/KEEP rules on comments touched by the diff or your edits);
   - `stale` and `missing and required` doc/skill/log updates from the gap check.
3. Run `git diff` (unstaged polish delta only) once to sanity-check that every edit is surgical before Step 4.

Skip each individual skill's own test/typecheck/lint run — they all run once in Step 4.

### What to apply

Apply a finding only when it is correct, in scope, low-risk, and does at least one of these:

- fixes a confirmed bug whose trigger can happen in normal use;
- removes code or duplication without reshaping explicit UI markup into config loops;
- prevents two places from drifting apart (a shared formula or constant copied by hand, a rule written twice);
- makes the code read clearly better (not just differently).

Report instead of applying:

- **Changes to behaviour, UX or product semantics** (what a screen shows, what a gate allows, a new state or check). These go to "Needs your call", even when the pass is confident.
- **Rare edge cases**, where the trigger needs an out-of-band action, an unlikely race, a dev-only mode or several unlikely things at once. Apply the fix only if it is a line or two and adds no new concept; otherwise report it with its trigger.
- **Refactors beyond the diff** or structural reshapes (`thorough`-only `simplify` items such as cross-file prop rewrites or extracting shared modules for code the change only touched).

Be suspicious when a fix needs an existing test rewritten to pass. Never weaken a test to make an edit fit. If a lens finds nothing worth applying, record `none` and move on.

Record wall-clock time (per subagent in Wave 1 plus the Wave 2 edit pass, and subagent tokens when reported) for the report.

### The gap check

It asks two questions. What outside the code still describes, watches or depends on the old behaviour? And what new behaviour has nothing describing or watching it? On the fast path, check it inline; on the standard path, Subagent B (`reuse + gap scan`) runs the search in Wave 1 and returns the classified findings so the raw grep output stays out of your context.

1. **List what other files could name.** Include routes and their parameters, response shapes, CLI commands and flags, env vars and config keys, exported functions and types, error codes and messages, log messages and their fields, and metric and event names. Include everything the change removed or renamed.
2. **Read the project's rules on what changes together.** Look in AGENTS.md, CLAUDE.md, CONTRIBUTING and the PR template (for example "update the README and the knowledge base in the same change"). Note any docs check they name.
3. **Search for the exact strings from step 1, not the topic,** in:
   - **Docs:** READMEs, sub-area agent docs, knowledge bases, API descriptions (OpenAPI specs, route descriptions), docs sites, changelogs, runbooks.
   - **Skills and agent prompts:** skill files with their references and scripts, and any prompt that calls or describes the changed code.
   - **Logs:**
     - an error path the change creates or touches that is caught and swallowed without a log;
     - a log line whose message or fields changed while a query, skill or doc still reads the old ones.
   - **Monitoring and alerting:**
     - alert rules, dashboards, saved queries, health checks and on-call docs keyed on a message, metric, route or error code the change removed or renamed;
     - a new failure mode, limit or background job that nothing watches.
4. **Classify each finding:**
   - **Stale:** text that is now wrong. Fix it. This is the "prevents two places from drifting apart" rule, so it applies. Match the file's tone and change only what the code changed.
   - **Missing and required:** something the project's rules require that the change lacks, such as a README row for a new endpoint. Add it, kept short.
   - **Missing and optional:** a new alert, dashboard, metric or log the change arguably should have. Report it under "Needs your call", saying what it would watch and why. Exception: one log line on an error path the change made silent. Apply that one when it fits in a line and uses the logger already in the file.
   - **Fine:** leave it, and don't list it.

Don't hand-edit generated docs. Regenerate them with the project's command, or report them. Gaps that existed before the change are follow-ups: don't fix them, and report only the relevant ones (see the report).

## 4. Verify

Once the edit pass is done, run the project's usual checks once for what was touched (in parallel where independent): typecheck, tests, lint. If the gap check edited docs, also run the docs checks the project names, such as a knowledge-base validator, a docs build or a link checker. Scope them to the affected packages or files where the project allows that. If a shared package changed, rebuild it first when its consumers read its build output. If something fails, check `git diff` to spot which edit caused it and fix it; if the fix isn't obvious, revert that edit and report it. A failure that already exists on the baseline is not polish's fault: say so and show it.

## 5. Report

Keep it short. What polish applied is in `git diff`, so it gets a few words per lens in the table, not a list of explanations. The rest of the report is only what is still open for the user:

```markdown
Baseline: <staged by me | staged by you | HEAD>. `git diff` shows only what polish changed.

| Lens | Cost | Applied |
| --- | --- | --- |
| reuse-check | <subagent B or inline time[, tokens]> | … |
| simplify (lean) | <edit pass> | … |
| code-review [deep] | <subagent A or inline time[, tokens]> | … |
| gap check | <shared with reuse scan> | <stale docs/skills fixed, required docs added> |
| trim-comments | <edit pass> | … |

Total wall time: <time> · Net: <+added / -removed lines in `git diff`>
Checks: <command → result>

Needs your call:
1. **<Title>.** <What it is and what happens if it stays as it is, in one or two sentences.>
   Suggestion: <what you would do.>

Gaps that predate the change:
- **<Title>** (`file:line`). <What is wrong.> Suggestion: <what you would do.>

Left out: <N> findings not worth acting on. Ask to see them.
```

**Needs your call** holds only open decisions: ones you did not make, that the user has to. Leave out:

- anything you already decided, applied or not ("I left `formatPayload` in the handler file" is not open);
- anything whose honest suggestion would be "leave it as it is", unless leaving it has a cost the user should know about before shipping, such as a backfill or migration order needed before deploy.

**Gaps that predate the change** lists only gaps in or next to the code the change touches that mislead a reader or can break something: a wrong comment or doc about that code, an error path nothing logs or watches. Leave out dead code, style and trivia.

Every listed item ends with a **Suggestion**. Say what you would do and be concrete:

- If it means adding or changing code, say which code, in which file, roughly how big, and whether it belongs in this change or a follow-up: "Add a 5-line check in `parseWebhookBody` (`webhooks.ts`) in this change."
- If the item is a trade-off, give your pick and the alternative.
- If it needs someone to act outside the code, say who does what and when: "Run the `sessions` backfill script before enabling strict validation."

For example:

> 1. **Legacy sessions rejected on token refresh.** Sessions created before `schema_v2` lack `refresh_family_id` and will fail refresh validation once this ships.
>    Suggestion: run the one-time backfill script against `sessions` before deploying so active users stay signed in. The alternative is accepting a one-time re-login for sessions older than 7 days.

If nothing is open, write "Needs your call: nothing" and drop the empty sections. Count what you left out in the "Left out" line, so the user can ask for them. If the net line count went up, say what the added lines buy.

Useful commands for the user: `git diff` (polish's edits); when the baseline was staged, `git diff --cached` (the baseline) and `git diff HEAD` (both); when `HEAD` is the baseline, `git diff <base-ref>..HEAD` (the baseline) and `git diff <base-ref>` (both).

Don't commit or push unless the user asks.
