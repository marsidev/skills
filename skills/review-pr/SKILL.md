---
name: review-pr
description: "Review a GitHub pull request like a senior engineer — analyze the diff for bugs, security issues, convention violations, and quality problems, then write the full review to a gitignored markdown file, show a condensed summary in the terminal, and optionally post it as a structured PR review with inline comments. Use when the user asks for a review of, or an opinion on, a pull request (by number, URL, or the current branch's open PR), or says /review-pr."
license: MIT
metadata:
  author: marsidev
  version: "2026.09.24"
---

# PR Code Review

You are a senior code reviewer. Your job is to review a pull request diff, find real issues worth raising, and present them clearly: write the full review to a gitignored file, print a condensed summary plus a findings table to the terminal, then optionally post it as a GitHub PR review with inline comments.

## Workflow

### Step 1: Identify the PR

If the user provides a PR number (e.g., `/review-pr 251`), use it directly.

If no number is given, detect the current branch and find its open PR:

```bash
gh pr view --json number,title,url --jq '{number, title, url}'
```

If a PR is found, confirm with the user: "Found PR #X: 'title'. Review this one?" If no PR exists for the current branch, ask the user for a PR number.

**Arguments.** Everything after the command name is free-form text, not a flag parser. The PR
number is the only thing this skill requires; anything else you write is a directive for this
run and overrides the defaults in this file. Recognize at least:

| Invocation | Effect |
| --- | --- |
| `/review-pr 1000` | Review PR #1000 with the defaults below |
| `/review-pr 1000 use opus 5` | Dispatch review agents on `model: "opus"` instead of the default `sonnet` |
| `/review-pr 1000 use fable` | Same, with `model: "fable"` |
| `/review-pr 1000 in english` | Write the review in English |
| `/review-pr 1000 only services/conversational-ai` | Restrict the review to files under that path |
| `/review-pr 1000 no post` | Terminal output only; skip the GitHub review step |
| `/review-pr 1000 inline` | Post every finding inline, not only Blocking ones; for PRs read mostly by humans |
| `/review-pr 1000 deep` | Raise the fan-out cap and use the session model; for a release or a risky refactor |

Valid model aliases are `sonnet`, `opus`, `haiku`, `fable`. Map loose phrasing onto them
("opus 5" -> `opus`). If a directive conflicts with a default in this file, the directive wins.
If it is ambiguous, ask before starting rather than guessing - a wrong guess costs a whole review.

### Step 2: Gather context

Fetch the PR metadata and diff:

```bash
# PR metadata (includes headRefOid for building file URLs)
gh pr view {number} --json title,body,baseRefName,headRefName,headRefOid,changedFiles,additions,deletions,files

# Full diff
gh pr diff {number}
```

Save `headRefOid` (the head commit SHA) and `{owner}/{repo}` (`gh repo view --json nameWithOwner --jq .nameWithOwner`).
Every finding in the review links to its lines at that commit:
`https://github.com/{owner}/{repo}/blob/{headRefOid}/{file_path}#L{start}-L{end}`

Reading full files (not just the diff hunks) is essential — a change that looks fine in isolation may be wrong when you see the surrounding code. That reading happens during analysis (Step 3): inline for small PRs, or delegated to parallel per-file agents for larger ones. Either way, focus on files with substantive changes; skip trivial renames or lockfile updates.

Split the full diff by file now (the `diff --git` sections). You'll hand each file's hunks to the agent that reviews it in Step 3.

**Project conventions**: If this is a fresh session without much codebase context, read the project's `CLAUDE.md` and any relevant `AGENTS.md` files for the areas touched by the PR. These contain the team's conventions (e.g., `satisfies` over `as`, no enums, Result types, Dockerfile rules). Convention violations are real findings. If you already have this context from the current session, skip this step. Keep the relevant conventions handy — when you dispatch review agents in Step 3, you must paste them into each agent's prompt, since agents don't inherit your session context.

### Step 3: Analyze the diff with parallel agents

Each changed file is an independent review domain — reviewing one file doesn't depend on the findings from another. Exploit that: dispatch focused agents that review files concurrently. Each agent reads one file's full context and reasons hard about a narrow scope, so the review is both faster (parallel wall-clock) and sharper (less context to juggle per reviewer) than reading every file sequentially yourself.

**Choose a strategy by PR size:**

- **Small PR** (1–2 substantive files, or one tightly-coupled change): review inline yourself. Read each full file with the Read tool, then apply the rubric below directly. Dispatch overhead isn't worth it.
- **Larger PR** (3+ substantive files, or several independent areas): dispatch parallel review agents — **one agent per file, or per group of tightly-coupled files** (e.g. a module and its test, a function and its only caller — files that must be read together to judge correctness). Don't split a coupled change across agents, and don't lump unrelated files into one agent.

**Dispatching review agents:**

- Send all agent calls **in a single message** so they run concurrently. One agent per independent file/group.
- Use a **general-purpose** agent (subagent type that can Read and reason over full files — not a read-only excerpt searcher). The agents only read and report; they never edit.
- **Pin the model**: unless the invocation overrides it (see Arguments in Step 1), pass
  `model: "sonnet"` on every Agent call. A review agent reads one
  diff plus one file and reports findings against a rubric you already wrote out for it -
  a narrow, fully-specified task. Left unpinned, agents inherit the session model (Opus or
  Fable); measured against plan limits, review-pr subagents have accounted for ~14% of a
  week's usage that way. Escalate a single agent to the session model only when that file is
  genuinely subtle: concurrency, protocol state machines, or security-sensitive parsing.
- **Cap the fan-out**: at most ~8 agents per review, unless the invocation says otherwise. Beyond that you are paying for
  redundant re-reads of shared context more than for coverage. Group the tail of small
  files into one agent rather than giving each its own.
- Each agent is **self-contained** — it does NOT inherit your session context. Give it everything it needs: the PR title and intent, the file path(s) it owns, that file's diff hunks (from Step 2), the relevant project conventions you gathered, and the full rubric below. A vague prompt produces a vague review.

Use this prompt template per agent:

````markdown
You are a senior code reviewer. Review the changes to `{file_path}` in this PR and return findings.

**PR:** #{number} — {title}
**Intent:** {1-2 sentences on what the PR is trying to do}

**Diff for this file:**
```diff
{the diff --git hunks for this file only}
```

**Project conventions that apply** (violations are real findings):
{paste the relevant rules from CLAUDE.md / AGENTS.md, or "none documented"}

**Your task:**
1. Read the FULL file at `{file_path}` (not just the hunks) to understand surrounding context. Read tightly-related files if needed to judge correctness.
2. Review only what this PR changed. Apply the rubric below.
3. Return your findings in the exact return format below. If you find nothing, return "No findings." plus a one-line note of anything done well.

{paste the full rubric: "What to look for", "What NOT to flag", "Severity tiers", "Category tags", and "Each finding needs" — verbatim from below}

**Return format** — for each finding, a block exactly like this:

---
**Title:** {short, specific}
**Severity:** {Blocking | Should-Fix | Nitpick}
**Category:** {one or more of the category tags}
**File:** `{file_path}` L{start}-{end}   (line numbers on the NEW/right side of the diff)
**Problem:** {what's wrong and why it matters}
**Fix:** {concrete fix, with a code snippet if helpful}
---

End with: `Files read: N` and a one-line `Highlight:` of anything notably well done (or "none").
````

**The rubric** (this is what you paste into each agent prompt, and what you apply directly for inline review):

**What to look for:**

- **Bugs**: Logic errors, off-by-one, null/undefined access, race conditions, incorrect types, missing error handling on boundaries
- **Security**: Injection vectors, auth bypasses, secret exposure, unsafe deserialization
- **Convention violations**: Team rules from CLAUDE.md/AGENTS.md — these are real issues, not style preferences
- **Reliability**: Resource leaks, missing cleanup, unhandled promise rejections, missing retry/timeout on external calls
- **Quality**: Dead code introduced, confusing naming, unnecessary complexity, missing validation at system boundaries

**What NOT to flag:**

- Style preferences not backed by team conventions
- "Consider using X instead of Y" without a concrete reason
- Hypothetical future problems ("what if someone later...")
- Things that were already wrong before this PR and aren't made worse by it
- Minor formatting differences

**Severity tiers:**

| Tier | Label | Meaning |
|------|-------|---------|
| Blocking | `Blocking` | Must fix before merge. Bugs, security holes, data loss risks, broken builds. |
| Should-Fix | `Should-Fix` | Strong recommendation. Reliability, quality, or convention issues that will cause problems. |
| Nitpick | `Nitpick` | Minor improvements. Nice to have, fine to ignore. |

**Category tags:** `Bug`, `Security`, `Convention`, `Reliability`, `Quality`, `Performance`, `Maintainability`

Each finding needs:
- A short, specific title
- The file path and line number(s)
- A clear explanation of what goes wrong and when: "This will throw at runtime when `config` is undefined because the null check is on the wrong branch" is useful; "Consider adding a null check" is not
- A concrete suggestion (what to do instead, with code if helpful)

**Aggregating agent results:**

When the agents return, you own the synthesis — don't just concatenate their outputs:

1. **Collect** every finding from every agent.
2. **Dedupe and group.** If the same pattern shows up across multiple files (e.g. missing error handling on every new API call), merge it into one finding that lists all locations — don't repeat it per file. This is the cross-file judgment a single-file agent can't make.
3. **Re-rank.** Apply severity tiers consistently across the whole PR; an agent may over- or under-rate in isolation.
4. **Filter false positives.** Agents occasionally flag non-issues or miss surrounding context you have. Drop anything you're not confident is a real problem — one false positive undermines trust in the rest.
5. **Tally** the total `Files read` across agents (for the footer) and collect the `Highlight` lines.

The rubric, severity tiers, and "no false positives" bar apply to the aggregated set exactly as they would to an inline review.

### Step 4: Write the review to a file

Assemble the full review as markdown and write it to disk **before printing anything**. The
full text must never go to the terminal: it is long, and everything you print is re-sent on
every later turn of this session. The file is the artifact; the terminal gets a summary.

**Path.** Never write the review where git would report it as an untracked file, and never
assume a particular directory exists. Probe for a writable, git-ignored location in one call:

```bash
REVIEW_DIR=""
if ROOT=$(git rev-parse --show-toplevel 2>/dev/null); then
  for d in .local .tmp .scratch tmp .cache; do
    if git -C "$ROOT" check-ignore -q "$d" 2>/dev/null; then REVIEW_DIR="$ROOT/$d/reviews"; break; fi
  done
fi
[ -n "$REVIEW_DIR" ] || REVIEW_DIR="${TMPDIR:-/tmp}/claude-pr-reviews"
mkdir -p "$REVIEW_DIR" && echo "$REVIEW_DIR" && date +%Y%m%d-%H%M%S
```

This prefers a git-ignored directory inside the repo (so the review sits next to the code and
survives), and falls back to the system temp directory when the repo ignores none of those
names or when you are not in a git repo at all. Use whatever the command prints; do not
hardcode `.local`. If the fallback is used, say so when you report the path, since temp
directories are cleared periodically.

Name the file:

```
pr-{number}-{shortSha}-{YYYYMMDD-HHMMSS}.md
```

`{shortSha}` is the first 8 characters of the `headRefOid` you saved in Step 2, and the
timestamp is the one the command above printed. The SHA ties the review to the exact commit
reviewed, so a re-review after new commits lands in its own file; the timestamp separates
repeated reviews of the same commit. Never overwrite an existing review file.

**File contents.** The complete review, in this format. Step 6 posts this same text as the
GitHub review body, so it has to stand on its own:

```markdown
# PR Review - #{number}: {title}

- **Commit:** `{headRefOid}`
- **Branch:** `{headRefName}` -> `{baseRefName}`
- **Reviewed:** {YYYY-MM-DD HH:MM}
- **Files reviewed:** {n}

## {n} blocking, {n} should-fix, {n} nitpicks

{1-3 sentence summary of the PR and overall assessment}

### Blocking

#### B1. {Title} `Bug` `Reliability`

[`path/to/file.ts:42-55`](https://github.com/{owner}/{repo}/blob/{headRefOid}/path/to/file.ts#L42-L55)

{What goes wrong and when}

**Fix:** {What to do instead, with a code block if helpful}

### Should-Fix

#### S1. {Title} `Convention` `Quality`
...

<details><summary>Nitpicks ({n})</summary>

#### N1. {Title} `Quality`
...

</details>

### Highlights

- {2-4 bullets on what was done well}
```

- Number findings per tier (`B1`, `S1`, `N1`). The terminal table and GitHub use the same IDs,
  so whoever fixes the PR can answer with "fixed B1, S2; declined N1 because ...".
- Nitpicks always go inside `<details>`, so a long tail doesn't bury the findings that matter.
- Omit any tier with no findings. If there are none at all, still write the file, with
  `## No issues found` as the heading and no tier sections.
- Keep Highlights to 2-4 bullets; skip it if nothing stands out.

### Step 5: Show the condensed result and ask what to do

Print **only** this to the terminal. One line per finding, title only, no explanations:

```
## PR Review - #{number}: {title}

{1-3 sentence summary and overall assessment}

| Severity | Count |
|----------|-------|
| Blocking | N |
| Should-Fix | N |
| Nitpick | N |

| # | Severity | Finding | Location |
|---|----------|---------|----------|
| B1 | Blocking | {title} | `path/to/file.ts:42` |
| S1 | Should-Fix | {title} | `path/to/other.ts:88` |
| N1 | Nitpick | {title} | `path/to/third.ts:12` |

Full review: {absolute path written in Step 4}
```

Do not reproduce the explanations or suggestions here - they are in the file. If a finding
title needs context to be intelligible, fix the title, do not add a paragraph.

Then ask:

> **What next?**
> **(a)** Post it to GitHub as a PR review - the full review as the body, Blocking findings also inline
> **(b)** Leave it as the file above
>
> Reply `a` or `b`.

Wait for an explicit answer.

- **(b)**: reply with the absolute path on one line and stop. Nothing else.
- **(a)**: go to Step 6.

Never post without explicit confirmation. If the invocation already said `no post`, skip the
question entirely and behave as if the user chose (b).

### Step 6: Post to GitHub (only if user confirms)

Post one PR review whose body is the complete review. Most readers of these reviews are
agents, and `gh pr view --comments` shows review bodies but not inline comments, so nothing
may live only in an inline comment. Inline comments are pointers for humans reading the
"Files changed" tab.

**Review body:** the Step 4 file without its `# PR Review` title line, followed by the footer
from "Signature and stats" below.

**Inline comments:** one per Blocking finding, or one per finding if the invocation said
`inline`. The body already carries the explanation, so keep the comment short:

```markdown
**B1. {Title}** - {one sentence on what goes wrong}. Details in the review body.
```

A finding on lines outside the diff gets no inline comment: GitHub rejects the whole review if
any inline comment falls outside the diff. With no qualifying findings, post the body alone.

**Using GitHub Suggested Changes:**

When an inline finding has a concrete, self-contained code fix (not a vague "consider doing X"), append GitHub's suggested change syntax to its comment. This renders as a committable diff that the author can accept with one click:

````markdown
```suggestion
const MODEL_CARDS = [
  // ... the corrected code that should replace the lines covered by start_line..line
] satisfies { id: IntelligenceLevel; name: string }[];
```
````

The `suggestion` block replaces the lines covered by the comment's `start_line` to `line` range. So the comment must span exactly the lines being replaced — set `start_line` and `line` to cover the original code, and put the replacement inside the `suggestion` block.

Rules for suggested changes:
- Only use when the fix is unambiguous and complete — the author should be able to click "Commit suggestion" without editing
- The suggestion must be valid code that compiles/runs correctly in context
- For multi-line replacements, the `start_line`/`line` range must cover all lines being replaced
- If the fix is too complex or touches multiple locations, use a regular code block with explanation instead
- Not every finding needs a suggested change — only use when it genuinely saves the author time

**Determining the correct line numbers for the GitHub API:**

The GitHub PR review API requires line numbers that correspond to the **diff**, not the original file. Use `line` (the ending line in the diff-side of the file) and optionally `start_line` for multi-line comments. Both refer to the **new file** line numbers (the right side of the diff). Set `side: "RIGHT"` for comments on added/modified lines.

To get the correct line numbers:
1. Parse the diff hunks for the file
2. Find the line(s) your comment refers to in the `+` side of the diff
3. Use those line numbers

**Posting the review:** write the payload to a temp file and post it with `--input`. Build the
JSON programmatically (e.g. with `jq`) so the markdown bodies are escaped correctly; omit
`comments` when there are no inline comments. This is its shape:

```bash
PAYLOAD=$(mktemp)
cat > "$PAYLOAD" <<'JSON'
{
  "event": "COMMENT",
  "body": "{review body}",
  "comments": [
    { "path": "src/file.ts", "line": 42, "side": "RIGHT", "body": "{inline comment body}" },
    { "path": "src/other.ts", "start_line": 10, "line": 15, "side": "RIGHT", "body": "{multi-line comment body}" }
  ]
}
JSON
gh api repos/{owner}/{repo}/pulls/{number}/reviews --input "$PAYLOAD" && rm "$PAYLOAD"
```

After posting, confirm to the user with the review URL.

### Timing

Measure the wall-clock time of the review so it can be reported in the footer.

**At the very start of Step 2** (before fetching the diff), record the start time:

```bash
date +%s > /tmp/review-start-time
```

**At the end of Step 5** (after printing the condensed result), record the end time and compute the duration:

```bash
START=$(cat /tmp/review-start-time) && END=$(date +%s) && ELAPSED=$((END - START)) && MINS=$((ELAPSED / 60)) && SECS=$((ELAPSED % 60)) && echo "${MINS}m ${SECS}s" && rm /tmp/review-start-time
```

Use this duration string in the footer. This is the actual measured time, not an estimate.

### Signature and stats

Every review posted to GitHub must end with a footer so readers know it was AI-assisted and can see the review effort. Append this to the **review body** (not the inline comments):

```markdown

---
*Review assisted by [Claude Code](https://claude.ai/code) | {model} | {duration} | {files_reviewed} files reviewed | {tokens} tokens*
```

- **duration**: The measured wall-clock time from the timing step above (e.g., "4m 30s", "6m 12s"). Always include this — it's measured, not estimated.
- **files_reviewed**: Count of files actually read — sum the `Files read` counts the review agents reported (plus any you read inline), not just the total changed files from PR metadata.
- **tokens**: Total tokens used during the review. If you know the exact count (e.g., from subagent metadata), include it. If not available, omit the tokens field entirely rather than guessing — the footer should only contain facts.
- **model**: The session's model ID, from your system prompt (e.g. `claude-opus-5-5`). If review agents ran on a different model, name both (e.g. `claude-opus-5-5 + sonnet agents`) - the footer should only contain facts.

## Important guidelines

- **Keep each agent's context small.** Give an agent only the files it owns. Its cost scales
  with what you paste into its prompt plus what it reads, and every extra file it reads is
  re-sent on each of its own turns.
- **Be precise, not prolific.** A review with 3 real findings is worth more than one with 15 nitpicks. If you're unsure whether something is an issue, it probably isn't worth raising.
