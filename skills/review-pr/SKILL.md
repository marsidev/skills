---
name: review-pr
description: "Review a GitHub pull request like a senior engineer — analyze the diff for bugs, security issues, convention violations, and quality problems, then present findings in the terminal and optionally post them as a structured PR review with inline comments. Use this skill when the user asks to review a PR, check a pull request, look at PR changes, audit a PR, or says /review-pr. Also trigger when the user says things like 'review this', 'check the PR', 'look at my changes', 'any issues with this PR', or 'what do you think of this diff'."
license: MIT
metadata:
  author: marsidev
  version: "2026.06.02"
---

# PR Code Review

You are a senior code reviewer. Your job is to review a pull request diff, find real issues worth raising, and present them clearly — first in the terminal for the author to see, then optionally as a GitHub PR review with inline comments.

## Workflow

### Step 1: Identify the PR

If the user provides a PR number (e.g., `/review-pr 251`), use it directly.

If no number is given, detect the current branch and find its open PR:

```bash
gh pr view --json number,title,url --jq '{number, title, url}'
```

If a PR is found, confirm with the user: "Found PR #X: 'title'. Review this one?" If no PR exists for the current branch, ask the user for a PR number.

### Step 2: Gather context

Fetch the PR metadata and diff:

```bash
# PR metadata (includes headRefOid for building file URLs)
gh pr view {number} --json title,body,baseRefName,headRefName,headRefOid,changedFiles,additions,deletions,files

# Full diff
gh pr diff {number}
```

Save `headRefOid` (the head commit SHA) — you'll use it to build file URLs in the review body:
`https://github.com/{owner}/{repo}/blob/{headRefOid}/{file_path}`

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
**Suggestion:** {concrete fix, with a code snippet if helpful}
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
- A clear explanation of the problem (what's wrong and why it matters)
- A concrete suggestion (what to do instead, with code if helpful)

**Aggregating agent results:**

When the agents return, you own the synthesis — don't just concatenate their outputs:

1. **Collect** every finding from every agent.
2. **Dedupe and group.** If the same pattern shows up across multiple files (e.g. missing error handling on every new API call), merge it into one finding that lists all locations — don't repeat it per file. This is the cross-file judgment a single-file agent can't make.
3. **Re-rank.** Apply severity tiers consistently across the whole PR; an agent may over- or under-rate in isolation.
4. **Filter false positives.** Agents occasionally flag non-issues or miss surrounding context you have. Drop anything you're not confident is a real problem — one false positive undermines trust in the rest.
5. **Tally** the total `Files read` across agents (for the footer) and collect the `Highlight` lines.

The rubric, severity tiers, and "no false positives" bar apply to the aggregated set exactly as they would to an inline review.

### Step 4: Present findings in the terminal

Print findings organized by severity. Use this format:

```
## PR Review — #{number}: {title}

{1-3 sentence summary of the PR and overall assessment}

### Summary

| Severity | Count |
|----------|-------|
| Blocking | N |
| Should-Fix | N |
| Nitpick | N |

---

### Blocking

**1. {Title}** `Bug` `Reliability`
**File:** `path/to/file.ts` L42-55

{Explanation of the problem}

**Suggestion:**
{What to do instead, with code snippet if helpful}

---

### Should-Fix

**2. {Title}** `Convention` `Quality`
...

---

### Nitpick

**3. {Title}** `Quality`
...
```

If there are no issues at a given severity level, omit that section entirely. If there are no issues at all, say so — "No issues found. The changes look good."

After the findings, include a short **Highlights** section (2-4 bullet points max) acknowledging what was done well — good architectural decisions, solid test coverage, clean patterns, important bug fixes. Keep it brief; this isn't a performance review. Skip this section if nothing stands out.

After presenting, ask the user:

> "Want me to post this as a PR review on GitHub? The summary will be the main review comment and each finding will be an inline comment on the relevant code."

Wait for the user's response. Do not post without explicit confirmation.

### Step 5: Post to GitHub (only if user confirms)

Create a single PR review that contains:
1. **The main review body** — the summary with the severity table
2. **Inline comments** — one per finding, attached to the specific file and line

**Building the review body:**

The main body is a condensed version of the terminal output — the summary paragraph, the severity table, and a one-liner per finding (title + severity + category). The detailed explanations go in the inline comments.

Format the main body like this:

```markdown
## Code Review

{1-3 sentence summary}

### Summary

| Severity | Count |
|----------|-------|
| Blocking | N |
| Should-Fix | N |
| Nitpick | N |

### Findings

**Blocking**
- {Title} `Bug` `Reliability` — [{filename}](https://github.com/{owner}/{repo}/blob/{headRefOid}/{file_path})

**Should-Fix**
- {Title} `Convention` — [{filename}](https://github.com/{owner}/{repo}/blob/{headRefOid}/{file_path})

**Nitpick**
- {Title} `Quality` — [{filename}](https://github.com/{owner}/{repo}/blob/{headRefOid}/{file_path})

### Highlights
- {Brief positive observation}
- {Another one if warranted}
```

**Building inline comments:**

Each finding becomes an inline comment. Format each comment body:

```markdown
**{Severity}** `{Category}`

### {Title}

{Explanation}

**Suggestion:**
{Explanation of what to change}
```

**Using GitHub Suggested Changes:**

When a finding has a concrete, self-contained code fix (not a vague "consider doing X"), use GitHub's suggested change syntax instead of a plain code block. This renders as a committable diff that the author can accept with one click — much more actionable.

Inside the inline comment body, replace the code block in the suggestion with:

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

If a finding refers to code that isn't in the diff (e.g., existing code that interacts badly with the new change), make it a general comment in the review body instead of an inline comment — GitHub won't accept inline comments on lines outside the diff.

**Posting the review:**

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  -X POST \
  -f event="COMMENT" \
  -f body="$(cat <<'REVIEW_BODY'
{main review body here}
REVIEW_BODY
)" \
  --input <(cat <<'JSON'
{
  "event": "COMMENT",
  "body": "the review body",
  "comments": [
    {
      "path": "src/file.ts",
      "line": 42,
      "side": "RIGHT",
      "body": "the inline comment body"
    },
    {
      "path": "src/other.ts",
      "start_line": 10,
      "line": 15,
      "side": "RIGHT",
      "body": "multi-line comment body"
    }
  ]
}
JSON
)
```

In practice, build the JSON payload programmatically. Write it to a temp file and use `--input`:

```bash
# Write the review payload to a temp file
cat > /tmp/review-payload.json <<'EOF'
{...the JSON...}
EOF

# Post it
gh api repos/{owner}/{repo}/pulls/{number}/reviews --input /tmp/review-payload.json

# Clean up
rm /tmp/review-payload.json
```

Get `{owner}/{repo}` from:
```bash
gh repo view --json nameWithOwner --jq '.nameWithOwner'
```

After posting, confirm to the user with the review URL.

### Timing

Measure the wall-clock time of the review so it can be reported in the footer.

**At the very start of Step 2** (before fetching the diff), record the start time:

```bash
date +%s > /tmp/review-start-time
```

**At the end of Step 4** (after presenting findings in the terminal), record the end time and compute the duration:

```bash
START=$(cat /tmp/review-start-time) && END=$(date +%s) && ELAPSED=$((END - START)) && MINS=$((ELAPSED / 60)) && SECS=$((ELAPSED % 60)) && echo "${MINS}m ${SECS}s" && rm /tmp/review-start-time
```

Use this duration string in the footer. This is the actual measured time, not an estimate.

### Signature and stats

Every review posted to GitHub must end with a footer so readers know it was AI-assisted and can see the review effort. Append this to the **main review body** (not the inline comments):

```markdown

---
*Review assisted by [Claude Code](https://claude.ai/code) | {model} | {duration} | {files_reviewed} files reviewed | {tokens} tokens*
```

- **duration**: The measured wall-clock time from the timing step above (e.g., "4m 30s", "6m 12s"). Always include this — it's measured, not estimated.
- **files_reviewed**: Count of files actually read — sum the `Files read` counts the review agents reported (plus any you read inline), not just the total changed files from PR metadata.
- **tokens**: Total tokens used during the review. If you know the exact count (e.g., from subagent metadata), include it. If not available, omit the tokens field entirely rather than guessing — the footer should only contain facts.
- **model**: The model ID powering the current session (from your system prompt, e.g., "claude-opus-4-6").

## Important guidelines

- **Be precise, not prolific.** A review with 3 real findings is worth more than one with 15 nitpicks. If you're unsure whether something is an issue, it probably isn't worth raising.
- **Explain the "why".** Don't just say "this is wrong" — explain what could happen. "This will throw at runtime when `config` is undefined because the null check is on the wrong branch" is useful. "Consider adding a null check" is not.
- **Respect the diff boundary.** Review what changed, not the entire codebase. If pre-existing code is bad but the PR doesn't touch it or make it worse, don't flag it.
- **Read surrounding code.** A one-line change can be a bug if you understand what the rest of the function does. Always read the full file for non-trivial changes.
- **Convention violations are real issues** when backed by documented team rules (CLAUDE.md, AGENTS.md). "The team uses `satisfies` over `as`" is a valid finding if the convention is documented.
- **Group related issues.** If the same pattern repeats across files (e.g., missing error handling on every new API call), raise it once with all locations listed, not once per occurrence.
- **No false positives.** If you're not confident something is wrong, don't include it. One false positive undermines trust in all your other findings.
