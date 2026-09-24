---
name: trim-comments
description: "Audit the comments and JSDoc that a change added or modified, cut the ones that are obvious or redundant, and shorten the ones that are too long - while keeping the comments that genuinely earn their length. Use when the user asks, in any language, to trim, prune, clean up, reduce, or review comments/JSDoc/docblocks on a diff, branch, PR, or file; asks whether comments a change added are obvious, redundant, or too verbose; or says /trim-comments."
license: MIT
metadata:
  author: marsidev
  version: "2026.09.24"
---

# Trim Comments

Agent-written code tends to over-comment: every line narrated, JSDoc that restates the
signature, changelog prose about what the change did. This skill reviews **only the comments
the change touched**, removes the noise, shortens the bloat, and leaves the ones that carry
real information.

## Usage

```
/trim-comments                  # comments touched by the current change (default)
/trim-comments --branch         # whole branch vs its base, not just the working tree
/trim-comments <path>           # limit to a file or directory
/trim-comments <PR#>            # comments touched by a GitHub PR
/trim-comments --all <path>     # every comment in scope, not just changed ones
/trim-comments --dry-run        # report the verdicts, apply nothing
```

## Step 1 - Resolve scope

Default scope is the change under discussion, not the whole repo.

1. If the user named a path, branch, or PR, use that.
2. Otherwise: if the working tree has changes (`git status --porcelain`), scope is
   staged + unstaged. If it is clean, scope is the branch: `git diff $(git merge-base HEAD
   <default-branch>)...HEAD`.
3. `--all` widens to every comment in the given files. Never widen on your own - unreviewed
   comments outside the diff are someone else's decision.

Skip generated/vendored files (`dist/`, `build/`, `*.gen.*`, lockfiles, `node_modules/`) and
files whose comments are content rather than code (changelogs, ADRs, migration notes).

## Step 2 - Collect candidates

Fast filter for comment lines the diff added:

```sh
git diff -U0 $RANGE | rg '^\+' | rg '^\+\s*(//|/\*|\*|#|<!--)'
```

Then read each hit **in its file with surrounding context**. A comment is judged against the
code it sits on, never against the diff line alone.

## Step 3 - Judge each comment

Three verdicts: **CUT**, **SHORTEN**, **KEEP**.

### CUT - the comment adds nothing the code does not already say

| Pattern | Example |
| --- | --- |
| Restates the next line | `// increment the counter` above `counter++` |
| Names the obvious block | `// loop over users` above a `for (const user of users)` |
| JSDoc that echoes the signature | `@param userId The user id` on `userId: UserId` |
| Section banners | `// ----- Helpers -----` |
| Changelog / agent voice | `// now returns null instead of throwing`, `// as requested, added retry` |
| Narrates the test | `// arrange`, `// act`, `// assert` on already-obvious blocks |
| Type restatement in TS | `@returns {Promise<User>}` when the return type is annotated |
| Commented-out code | dead blocks left "just in case" - git has it |
| Empty JSDoc shell | `/** Formats the date. */` on `formatDate()` |

### SHORTEN - the point is real, the prose is not

- Multi-paragraph preamble where one sentence does the work.
- JSDoc with a `@description` that repeats the summary line.
- A long story about a bug where the issue link plus one line suffices.
- Numbered walkthroughs of code that reads fine on its own - keep only the non-obvious step.

Target: one line for inline comments, summary line plus the tags that carry information for
JSDoc. Preserve the author's wording where possible; you are cutting, not rewriting.

### KEEP - the comment earns its length

- **Why, not what**: rationale, trade-offs, rejected alternatives.
- Non-obvious invariants, ordering requirements, concurrency or lifecycle constraints.
- Workarounds with a link to the upstream issue, spec, RFC, or vendor doc.
- Contract docs on exported/public API: what callers must know that types cannot express -
  units, ranges, throwing behavior, side effects, idempotency.
- Footgun warnings ("do not reorder these two calls - the second reads state the first sets").
- Regulatory/business rules that cannot be inferred from code.
- `TODO`/`FIXME` that names a condition or owner.
- Anything long *because the underlying thing is genuinely subtle*. Length alone is not a defect.

### Never touch

- Tool pragmas: `@ts-expect-error`, `@ts-ignore`, `eslint-disable*`, `biome-ignore`,
  `prettier-ignore`, `c8 ignore`, `istanbul ignore`, `noqa`, `#!` shebangs.
- Tags with runtime or tooling meaning: `@deprecated`, `@internal`, `@public`, `@module`,
  `@packageDocumentation`, `@type`/`@satisfies` in JS files under `checkJs`, JSDoc that feeds
  a docs generator or an MCP/tool schema.
- License and copyright headers.
- Comments inside test fixtures asserting on literal text.
- Comments in files the change did not touch (unless `--all` was given).

When genuinely unsure whether a comment carries information you cannot re-derive from the
code: **keep it**. A surviving mediocre comment costs less than a deleted insight.

## Step 4 - Apply

Skip this step under `--dry-run`.

1. Snapshot the files you will edit so the verification in Step 5 is exact:
   ```sh
   SNAP=$(mktemp -d) && for f in $FILES; do mkdir -p "$SNAP/$(dirname "$f")" && cp "$f" "$SNAP/$f"; done
   ```
2. Edit comments only. No renames, no logic changes, no import churn, no reformatting of
   untouched code.
3. Remove the blank line a deleted comment leaves behind only when it creates a double blank.
4. If cutting a whole JSDoc block would leave an exported symbol undocumented, shorten it to a
   one-line summary instead of removing it.

## Step 5 - Verify

1. Prove you only touched comments - every changed line here must be a comment or blank line:
   ```sh
   for f in $FILES; do diff -u "$SNAP/$f" "$f" | rg '^[+-]' | rg -v '^[+-]{3}'; done
   ```
   Any code line in that output is a bug in your edit. Revert it.
2. Run the repo's formatter/linter on the touched files if one exists (`pnpm lint`,
   `biome check`, etc.). Comment removal changes line wrapping.
3. If you touched JSDoc in a JS file with type-checking, or any `@type`-bearing block, run the
   typecheck.

## Step 6 - Report

Compact, grouped by file. Show what you cut, and - just as important - what you kept, so the
user can override a judgment call.

```
src/queue/tick.ts
  CUT      L42   "// increment the attempt counter"
  SHORTEN  L88   6-line rationale -> 1 line (kept the issue link)
  KEEP     L120  explains why the lock is released before the await

scripts/lib/client.mjs
  CUT      L15   JSDoc @param block restating the signature

12 comments reviewed - 5 cut, 2 shortened, 5 kept. Only comment lines changed.
```

If nothing deserves cutting, say exactly that in one line. Do not invent trims to look busy.

## Hard rules

- Comments only. If a comment is wrong because the *code* is wrong, report it - do not fix the
  code in this pass.
- Never delete a comment you do not understand.
- Keep the language the file already uses; do not translate comments.
- Match the surrounding comment density. A file that is deliberately heavily documented is a
  style choice, not slop.
