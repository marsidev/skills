---
name: reuse-check
description: "Check whether a change re-implements helpers the codebase already has (normalizers, parsers, number/date/string helpers, type guards, timeouts, truncation, lookups), verify each candidate is actually importable and behaves the same, then swap in the existing helper where it does the same job or more. Use when the user asks, in any language, to check for duplicated code, reinvented utilities, 'is there a shared helper for this', 'do we already have something like this', or says /reuse-check."
license: MIT
metadata:
  author: marsidev
  version: "2026.11.14"
---

# Reuse Check

Agents write a fresh helper before they look for an existing one. The result is a second
normalizer, a second type guard, a second timeout race: code that drifts from the original and
misses edge cases the original already handles. This skill inventories what a change adds,
finds existing equivalents the change can really import, compares their behavior, and reuses
them where that is safe.

## Usage

```
/reuse-check                 # the current change: branch diff plus uncommitted work (default)
/reuse-check <path>          # limit to a file or directory
/reuse-check <PR#>           # a GitHub PR's diff
/reuse-check --dry-run       # report only, change nothing
```

## Step 1 - Resolve scope

1. **Caller brief wins.** When invoked from `polish-changes` (or with an explicit `git diff <base-ref>` scope), use that scope directly and skip standalone test/lint runs at the end when told verification is deferred.
2. Otherwise, use the path or PR the user named, or take `git diff <default-branch>...HEAD` plus `git diff HEAD` for uncommitted work (including untracked new files).
3. Never widen the scope. A duplicate outside the change is a follow-up, not part of this job.
4. Never re-read files already in your current session context; batch any unseen file reads in parallel.

## Step 2 - Inventory what the change adds

From the diff, list every new function, constant, regex, type guard, class, and small inline
idiom that does a nameable job (for example a `Promise.race` with a timer, a
`slice(0, n) + "…"`, or `typeof x === "object" && x !== null`). For each one, record:
- file and symbol;
- one line on what it does, including its edge cases.

Skip tests, generated files, and code that was only moved. A symbol that is removed in one
file and added in another within the same diff is a move, not a new helper.

## Step 3 - Map import boundaries

For each file that gains a helper, work out what it may import before you search:
- its own package or module;
- the internal packages declared in its manifest (`package.json` dependencies, `go.mod`,
  `pyproject.toml`, `Cargo.toml` and so on);
- path aliases;
- the project's layering rules. Read the project's agent/contributor docs (AGENTS.md,
  CLAUDE.md, CONTRIBUTING) for them.

A shared library usually cannot import from an app or a service. A helper the file cannot
import is not a reuse, however close it looks.

## Step 4 - Search (delegate it)

Hand the inventory and the boundaries to a search subagent, so its raw output stays out of
your context. Tell it to:
- **Search in order:** shared packages and libraries, then the same package, then siblings.
- **Search by behavior, not only by name.** Useful terms: normalize, fold, slug, canonical,
  sanitize, parse, format, digits, isObject/isRecord/isPlainObject, withTimeout, deadline,
  race, truncate/bound/ellipsis, retry, debounce, and the domain nouns involved.
- **Report per item:**
  - verdict: `reuse` / `partial` / `nothing`;
  - the existing symbol as `file:line`;
  - whether it is importable from the new code's location;
  - every behavior difference that matters. Ask for concrete inputs where the outputs
    differ, not just "similar".

## Step 5 - Compare behavior, both ways

For every `reuse` or `partial` candidate, read the existing implementation and compare both side-by-side on concrete edge cases before you decide: empty input, unicode and diacritics, separators, prefixes, long input, and a malformed value. Only run a one-shot command check if subtle regex/parsing behavior isn't obvious from reading the code directly.
- **The existing helper handles more:** that is often a bug in the new code. Say so
  explicitly, and add a regression test for that input.
- **The new code handles something the existing helper refuses:** reuse the helper for its
  core and keep only the difference as a small local step, or keep the new code and say why.

## Step 6 - Apply (skip under `--dry-run`)

- Replace a helper only when the existing one does the same job or more. Keep any glue small
  and local.
- Do not add a new internal dependency just to reuse a trivial helper. If you do add one,
  update the manifest and any build or packaging config that has to copy or bundle that
  package.
- Add tests for every edge case whose behavior changed. Then run the affected tests, the
  typecheck and the linter (unless deferred by `polish-changes`).
- Change only what the swap needs, and do not commit.

## Step 7 - Report

One line per inventoried helper, then the follow-ups:

```
slugifyTitle        nothing   closest: apps/docs/utils/anchor.ts:12 (not importable; strips dots)
parseDurationMs     reused    packages/shared/time.ts:44 parseDuration; fixes "1.5s" fractional seconds
isPlainRecord       reused    packages/core/guards.ts:19 isPlainObject (identical)
fetchWithDeadline   nothing   same pattern in apps/worker/http/timeout.ts:28 (not importable)

Follow-ups (out of scope): apps/admin/lib/guards.ts duplicates packages/core isPlainObject.
```

If nothing is reusable, say so in one line. Do not force a swap to look productive.

## Hard rules

- Importable or it does not count.
- Compare behavior on concrete inputs; a matching name proves nothing.
- A local helper that is more correct for this job stays, with one line saying why.
- Never widen scope, and never commit or push unless asked.
