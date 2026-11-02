---
name: review-pr
description: "Review a GitHub pull request like a senior engineer — analyze the diff for bugs, security issues, convention violations, quality problems, and over-engineering (checking what was built against what the linked issue asked for), then write the full review to a gitignored markdown file, show a condensed summary in the terminal, and optionally post it as a structured PR review with inline comments. Use when the user asks for a review of, or an opinion on, a pull request (by number, URL, or the current branch's open PR), or says /review-pr."
license: MIT
metadata:
  author: marsidev
  version: "2026.11.02"
---

# PR Code Review

You are a senior code reviewer. Your job is to review a pull request diff, find real issues worth raising, and present them clearly: write the full review to a gitignored file, print a condensed summary plus a findings table to the terminal, then optionally post it as a GitHub PR review with inline comments.

## Workflow

### Step 1: Identify the PR

If the user provides a PR number or URL (e.g., `/review-pr 251`), use it directly; `gh pr` accepts both.

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
| `/review-pr 1000 in spanish` | Write the review in that language instead of the default English |
| `/review-pr 1000 only src/server` | Restrict the review to files under that path |
| `/review-pr 1000 no post` | Terminal output only; skip the GitHub review step |
| `/review-pr 1000 full` | Ignore earlier reviews and review the whole PR again |
| `/review-pr 1000 inline` | Post every finding inline, not only Blocking ones; for PRs read mostly by humans |
| `/review-pr 1000 deep` | Drop the fan-out cap (one agent per file or coupled group) and run agents on the session model; for a release or a risky refactor |

Valid model aliases are `sonnet`, `opus`, `haiku`, `fable`. Map loose phrasing onto them
("opus 5" -> `opus`). If a directive conflicts with a default in this file, the directive wins.
If it is ambiguous, ask before starting rather than guessing - a wrong guess costs a whole review.

### Step 2: Gather context

Fetch the PR metadata and diff:

```bash
# Start time for the footer (see Timing); keep the printed number
date +%s

# PR metadata (includes headRefOid for building file URLs)
gh pr view {number} --json url,title,body,baseRefName,headRefName,headRefOid,changedFiles,additions,deletions,files,closingIssuesReferences

# Full diff
gh pr diff {number}
```

Save `headRefOid` (the head commit SHA) and `{owner}/{repo}`, taken from the PR `url` (not from
the current directory, which can be a different repo when the user passed a URL).
Every finding in the review links to its lines at that commit:
`https://github.com/{owner}/{repo}/blob/{headRefOid}/{file_path}#L{start}-L{end}`

**Code at the PR head.** Full files are read from disk, so the files on disk must be the PR's
head. If `git rev-parse HEAD` prints `headRefOid`, read from the working tree. Otherwise don't
switch the user's branch; check the head out into a temporary worktree and read from there,
giving agents absolute paths into it (`{shortSha}` is the first 8 characters of `headRefOid`):

```bash
REVIEW_TREE="${TMPDIR:-/tmp}/review-pr-{number}-{shortSha}"
git fetch origin "pull/{number}/head" && git worktree add --detach "$REVIEW_TREE" {headRefOid} && echo "$REVIEW_TREE"
```

Use the remote that points at `{owner}/{repo}` in place of `origin` if they differ. If the
repo isn't cloned here at all, say so and ask before cloning it.

**Linked issue.** The issue says what was asked; the diff shows what was built. Take the issue
numbers from `closingIssuesReferences`; if that is empty, look for `#N` or issue URLs in the PR
body. Read each issue with its comments, because maintainers often narrow or redirect the scope
in the thread:

```bash
gh issue view {issue} --repo {owner}/{repo} --json title,body,comments --jq '{title, body, comments: [.comments[] | {author: .author.login, body}]}'
```

Write down the deliverables, explicit exclusions ("no X", "in a follow-up"), conditions
("measure first", "agree on scope before coding") and acceptance criteria. If the PR is one
layer of a stack, judge scope against the whole stack, since the stack answers the issue. If
there is no linked issue, use the PR description as the stated intent and say so in the review.

**Earlier reviews.** Unless the invocation says `full`, check whether you have reviewed this PR
before, by hand or with this skill:

```bash
ME=$(gh api user --jq .login) && echo "$ME"
gh api "repos/{owner}/{repo}/pulls/{number}/reviews" --paginate --jq ".[] | select(.user.login == \"$ME\") | {id, html_url, commit_id, submitted_at}"
# A review with visuals is posted as a PR comment; its commit comes from the header link
gh api "repos/{owner}/{repo}/issues/{number}/comments" --paginate --jq ".[] | select(.user.login == \"$ME\" and (.body | contains(\"Review assisted by [Claude Code]\"))) | {id, html_url, created_at, commit_id: ([.body | capture(\"/commits/(?<sha>[0-9a-f]{40})\")] | first.sha)}"
```

A review whose body is only `Review with visuals: {URL}` belongs to that comment; count them as
one. Also look for `pr-{number}-*.md` in the review directory (run the probe from Step 4); a
review kept only as a file was never posted. Take the most recent of all of these:

- **None:** review the whole PR as below.
- **At `headRefOid`:** nothing changed since. Say so, give the review's URL or path, and ask
  before reviewing again.
- **At an older commit:** do a follow-up review. Read
  [references/follow-up-review.md](references/follow-up-review.md) now; it changes Steps 3-6.

Reading full files (not just the diff hunks) is essential — a change that looks fine in isolation may be wrong when you see the surrounding code. That reading happens during analysis (Step 3): inline for small PRs, or delegated to parallel per-file agents for larger ones. Either way, focus on files with substantive changes; skip trivial renames or lockfile updates.

Split the full diff by file now (the `diff --git` sections). You'll hand each file's hunks to the agent that reviews it in Step 3.

**Project conventions**: If this is a fresh session without much codebase context, read the project's `CLAUDE.md` and any relevant `AGENTS.md` files for the areas touched by the PR. These contain the team's conventions (e.g., naming, error-handling patterns, type rules, banned APIs). Convention violations are real findings. If you already have this context from the current session, skip this step. Keep the relevant conventions handy — when you dispatch review agents in Step 3, you must paste them into each agent's prompt, since agents don't inherit your session context.

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
  a narrow, fully-specified task. Left unpinned, agents run on the session model (or a
  configured default subagent model), usually a larger one than this needs; unpinned
  review-pr subagents once accounted for ~14% of a week's plan usage. Escalate a single
  agent to the session model only when that file is genuinely subtle: concurrency, protocol
  state machines, or security-sensitive parsing.
- **Cap the fan-out**: at most ~8 agents per review, unless the invocation says otherwise. Beyond that you are paying for
  redundant re-reads of shared context more than for coverage. Group the tail of small
  files into one agent rather than giving each its own.
- Each agent is **self-contained** — it does NOT inherit your session context. Give it everything it needs: the PR title and intent, the file path(s) it owns, that file's diff hunks (from Step 2), the relevant project conventions you gathered, and the full rubric below. A vague prompt produces a vague review.

Use this prompt template per agent:

````markdown
You are a senior code reviewer. Review the changes to `{file_path}` in this PR and return findings.

**PR:** #{number} — {title}
**Intent:** {1-2 sentences on what the PR is trying to do}
**Asked for:** {the linked issue's deliverables and exclusions, or "no linked issue"}

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

{paste the full rubric: "What to look for", "What NOT to flag", "Severity tiers", "Category tags" (with the severity note under it), and "Each finding needs" (with the length rule under it) — verbatim from below}

**Return format** — for each finding, a block exactly like this:

---
**Title:** {short, specific}
**Severity:** {Blocking | Should-Fix | Nitpick}
**Category:** {one or more of the category tags}
**File:** `{file_path}` L{start}-{end}   (line numbers on the NEW/right side of the diff)
**Problem:** {what goes wrong and when, in 1-2 sentences}
**Fix:** {one concrete fix, with code only when the code is the fix}
**Evidence:** {optional: measurements, traces, prod numbers or the chain of events that proves it}
---

End with: `Files read: N` and a one-line `Highlight:` of anything notably well done (or "none").
````

**The rubric** (this is what you paste into each agent prompt, and what you apply directly for inline review):

**What to look for:**

- **Bugs**: Logic errors, off-by-one, null/undefined access, race conditions, incorrect types, missing error handling on boundaries
- **Security**: Injection vectors, auth bypasses, secret exposure, unsafe deserialization
- **Convention violations**: Team rules from CLAUDE.md/AGENTS.md — these are real issues, not style preferences
- **Reliability**: Resource leaks, missing cleanup, unhandled promise rejections, missing retry/timeout on external calls
- **Quality**: Dead code introduced, confusing naming, missing validation at system boundaries
- **Over-engineering**: More machinery than the job needs. Look for abstractions, wrappers or factories with one caller, and options, flags or generic parameters that nothing varies. Look for defensive handling of states that can't occur, a new helper or layer next to an existing one that already does the job, and persistence, retries or recovery for cases the user can simply redo. Name the simpler version, what it drops, and why dropping it is safe.

**What NOT to flag:**

- Style preferences not backed by team conventions
- "Consider using X instead of Y" without a concrete reason
- "This could be simpler" without showing the simpler version doing the same job
- Complexity that the issue explicitly asks for, or that a documented decision requires
- Hypothetical future problems ("what if someone later...")
- Things that were already wrong before this PR and aren't made worse by it
- Minor formatting differences

**Severity tiers:**

| Tier | Label | Meaning |
|------|-------|---------|
| Blocking | `Blocking` | Must fix before merge. Bugs, security holes, data loss risks, broken builds. |
| Should-Fix | `Should-Fix` | Strong recommendation. Reliability, quality, or convention issues that will cause problems. |
| Nitpick | `Nitpick` | Minor improvements. Nice to have, fine to ignore. |

**Category tags:** `Bug`, `Security`, `Convention`, `Reliability`, `Quality`, `Performance`, `Maintainability`, `Over-engineering`, `Scope`

Over-engineering severity: `Blocking` when the extra machinery changes behavior outside the
ask (e.g. a new hard failure on flows the issue didn't touch) or contradicts scope agreed in the
issue; `Should-Fix` when it is in scope but much heavier than needed; `Nitpick` for small local
cases such as a one-use helper. Use `Scope` for a deliverable that is missing or differs from
the ask, and for unrelated changes that should be their own PR.

Each finding needs:
- A short, specific title
- The file path and line number(s), or `PR-level` for a finding about the PR as a whole
- A clear explanation of what goes wrong and when: "This will throw at runtime when `config` is undefined because the null check is on the wrong branch" is useful; "Consider adding a null check" is not
- A concrete fix (what to do instead, with code if helpful)

Keep each finding short. State what goes wrong and when in 1-2 sentences, then give one
recommended fix, with code only when the code is the fix. Anything a reader needs only to
verify the finding goes under Evidence: measurements, traces, prod numbers, or the order of
events when the order is the bug (4 steps or fewer). Leave out alternative fixes, "at minimum"
fallbacks and side suggestions: raise a side issue as its own finding or drop it. A bigger PR
earns more findings, not longer ones.

**Aggregating agent results:**

When the agents return, you own the synthesis — don't just concatenate their outputs:

1. **Collect** every finding from every agent.
2. **Dedupe and group.** If the same pattern or the same root cause shows up in several findings or files (e.g. missing error handling on every new API call), merge it into one finding that lists all locations — don't repeat it per file. This is the cross-file judgment a single-file agent can't make.
3. **Re-rank.** Apply severity tiers consistently across the whole PR; an agent may over- or under-rate in isolation.
4. **Filter false positives.** Agents occasionally flag non-issues or miss surrounding context you have. Drop anything you're not confident is a real problem — one false positive undermines trust in the rest.
5. **Tally** the total `Files read` across agents (for the footer). The `Highlight` lines feed the verdict, at most a few words of it.

The rubric, severity tiers, and false-positive filter apply to the aggregated set exactly as they would to an inline review.

**Scope check (whole PR; you do this, not the agents):**

Over-engineering usually shows at the PR level: every file looks reasonable, yet the PR builds a
subsystem where a small extension would do. Per-file agents can't see that. Do this yourself after
the agents return, or as part of the inline review for a small PR.

1. **Map asked to built.** For each deliverable from the issue, note whether the diff delivers
   it, misses it, or does something different. Then list what the diff adds that no deliverable
   asked for.
2. **Find the lean path.** Search the codebase for an existing flow, helper or extension point
   that already does most of the job; don't assume there is none. If the PR builds alongside it
   instead of extending it, that is the main finding.
3. **Question each new piece of machinery**: a module, route, store, state machine,
   persistence layer, gate, config option or dependency.
   - Which deliverable needs it? If none, it is out of scope.
   - What concretely breaks with the simpler version? "More robust" is not an answer; name
     the failure.
   - Does it enforce something reversible and visible to the user? Then it is a quality concern
     to measure, not a boundary to enforce. Enforcement belongs at real privilege boundaries,
     not on actions the user can already take or undo by hand.
   - Does it widen the blast radius? Examples: it runs on every request instead of the ones
     the issue targets, or it turns a path that used to fail open into a hard failure for
     existing flows.
   - Does it add recovery or persistence for a case where retrying or undoing is acceptable?
4. **Check the issue's conditions.** If the issue or its thread said to measure first, keep it
   small, leave something out, or agree on scope before coding, verify the PR followed that.
   Building past an unmet "measure first" is a finding.
5. **Check shippability.** Unrelated fixes riding along belong in their own PRs. A change that
   only works once every layer lands is harder to review, revert and ship than slices that
   each work alone.
6. **Weigh size against the ask.** A large diff for a small ask is a reason to look harder,
   not a finding by itself. The finding is the specific machinery from point 3.
7. **Read the findings as a signal.** When the findings cluster as races, stuck states and
   recovery gaps in machinery the PR adds, that is evidence for point 3. Ask whether the
   machinery should exist before asking for more fixes to it.

When the PR is over-built, raise one `PR-level` `Over-engineering` finding that lists the
machinery it would remove, rather than one finding per file. Its Fix names the lean alternative
and its rough size in one sentence. Its Evidence lists what to remove and the lean version in
3-6 bullets: what it reuses, what it leaves out and why that is safe. Without a linked issue,
still apply points 2, 3 and 5 against the PR's stated intent.

**Premise check (whole PR; you do this, when it applies):**

A diff can be correct and still fix the wrong thing. This applies when the PR's effect depends on
how real traffic or real config behaves: routing, branching on data, model behaviour,
user-authored config. Ask whether the change would have helped one real case the PR names, such
as an incident, a request or a record. You don't need to re-run the case: check whether the
PR's own evidence shows it, or only tests built on a hand-made shape. Skip this for changes whose
effect is visible in the code alone.

Raise a `PR-level` `Scope` finding when the evidence doesn't reach a real case and one of these
holds:
- **A static analysis stands in for runtime behaviour:** a graph walk or reachability check
  that ignores the conditions the runtime applies.
- **Most incidents come from one source:** one template, tenant, config or tool. Fixing that
  source may beat changing the platform.
- **The success measure is a stand-in:** for example "the request reached handler X" instead
  of "the user got the outcome they asked for".

> Example: a PR blocked a tool for the flows a graph walk said could reach it, ignoring the
> conditions on the routing steps. Every file-level check passed. In prod, one of the flows it
> named routed elsewhere, and the incidents came from one config that already had a correct
> path. The PR was closed in favour of fixing that config.

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

````markdown
# PR Review - #{number}: {title}

**{n} blocking · {n} should-fix · {n} nitpicks** · [`{shortSha}`](https://github.com/{owner}/{repo}/pull/{number}/commits/{headRefOid}) · {YYYY-MM-DD HH:MM}

{Verdict in 1-2 sentences: whether it can merge and what must change first. Don't re-describe the PR.}

#### B1. {Title}

[`file.ts:42-55`](https://github.com/{owner}/{repo}/blob/{headRefOid}/path/to/file.ts#L42-L55)

{What goes wrong and when, in 1-2 sentences}

**Fix:** {one recommended fix}

<details><summary>Details</summary>

{The finding's Evidence}

</details>

#### S1. {Title}
...

<details><summary>Nitpicks ({n})</summary>

- **N1. {Title}** · [`file.ts:12`](https://github.com/{owner}/{repo}/blob/{headRefOid}/path/to/file.ts#L12)
  {One sentence, only if the title doesn't already say it.} **Fix:** {fix}

</details>

<details><summary>Scope: {n} of {m} delivered{, how many differ or are missing}{, what goes beyond the ask}</summary>

Issue #{issue}: {title} (or "No linked issue; compared against the PR description")

- ✅ {deliverable}
- 🟡 {deliverable}: {how it differs}
- ❌ {deliverable}: missing

Beyond the ask: {additions no deliverable needs, or "none"}

</details>
````

The post has to be short enough that a human reads it whole; agents read the collapsed parts too.

- **Visible:** the header line, the verdict, and every Blocking and Should-Fix finding as title,
  location link, 1-2 sentences and Fix. A `PR-level` finding has no location line.
- **Collapsed:** a finding's Evidence (as `Details`, or a more specific summary such as "Why CI
  doesn't catch it"; leave the block out when there is no evidence), the nitpicks, and the
  scope. Each summary carries its count or verdict, so a reader can tell whether to open it.
- **IDs:** number findings per tier (`B1`, `S1`, `N1`). The terminal table and GitHub use the
  same IDs, so whoever fixes the PR can answer "fixed B1, S2; declined N1 because ...". The
  IDs carry the tier, so there are no tier headings and no category tags in the post.
- **Left out:** highlights and a summary of what the PR does. Praise worth giving fits in the
  verdict in a few words.
- **Lists, not tables.** GitHub squeezes a wide table's short columns until IDs and words break
  mid-character. The terminal tables in Step 5 are fine.
- **`#N` is a link.** GitHub links every `#N` to that issue or PR, so write it only for one.
- **Visuals, rarely.** A finding about an order of events, measured numbers, or a PR-level scope
  argument may get one image in its Details block, never in place of text. Read
  [references/visuals.md](references/visuals.md) before drawing one.
- Omit what is empty. With no findings, keep the header line, write "No issues found" as the
  verdict, and keep the scope block.

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

Scope: {matches the issue | N deviations, M additions beyond the ask | no linked issue}

Full review: {absolute path written in Step 4}
```

If you created a review worktree in Step 2, remove it before asking:
`git worktree remove --force {path printed in Step 2}`.

Do not reproduce the explanations or suggestions here - they are in the file. If a finding
title needs context to be intelligible, fix the title, do not add a paragraph.

Then ask:

> **What next?**
> **(a)** Post it to GitHub as a PR review - the full review as the body, Blocking findings also inline (with visuals: the body as a PR comment, plus a review for the inline comments)
> **(b)** Leave it as the file above
>
> Reply `a` or `b`.

Wait for an explicit answer.

- **(b)**: reply with the absolute path on one line and stop. Nothing else.
- **(a)**: go to Step 6.

Never post without explicit confirmation. If the invocation already said `no post`, skip the
question entirely and behave as if the user chose (b).

### Step 6: Post to GitHub (only if user confirms)

A review with visuals is posted as [references/visuals.md](references/visuals.md) describes,
because the reviews API can't upload images; what follows still applies to its inline comments.

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

A `PR-level` finding, or one on lines outside the diff, gets no inline comment: GitHub rejects the whole review if
any inline comment falls outside the diff. With no qualifying findings, post the body alone.

**Using GitHub Suggested Changes:**

When an inline finding has a concrete, self-contained code fix (not a vague "consider doing X"), append GitHub's suggested change syntax to its comment. This renders as a committable diff that the author can accept with one click:

````markdown
```suggestion
// the corrected code that replaces the lines covered by start_line..line
if (!user) return null;
return user.name;
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

`line` (and `start_line` for multi-line comments) are line numbers in the file, not positions
in the diff, and every line must fall inside a diff hunk. Use `side: "RIGHT"` with new-file line
numbers for added or unchanged lines, and `side: "LEFT"` with old-file line numbers for deleted
lines. Read them off the hunk headers (`@@ -old,len +new,len @@`).

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

Step 2 prints the start time (`date +%s`). At the end of Step 5, after printing the condensed
result, compute the duration from it:

```bash
ELAPSED=$(( $(date +%s) - {start} )) && echo "$((ELAPSED / 60))m $((ELAPSED % 60))s"
```

Use this duration string in the footer. This is the actual measured time, not an estimate.

### Signature and stats

Every review posted to GitHub must end with a footer so readers know it was AI-assisted and can see the review effort. Append this to the **review body** (not the inline comments):

```markdown

<sub>Review assisted by [Claude Code](https://claude.ai/code) · {model} · {duration} · {files_reviewed} files read · {tokens} agent tokens</sub>
```

The footer holds only measured or counted values, in this order, with no `~`, no estimates and
no extra wording. When a value is missing, drop its field rather than guessing.

- **model**: The session's model ID, from your system prompt (e.g. `claude-opus-5-5`). If review agents ran, append them as `+ sonnet agents`, `+ opus agents` or `+ opus and sonnet agents`.
- **duration**: Measured from the start time printed in Step 2 (e.g. `4m 30s`). If that number is gone, drop the field.
- **files_reviewed**: The exact sum of the `Files read` counts the agents reported, plus any files you read inline; not the PR's changed-file count.
- **tokens**: The sum of the token totals the agents reported, rounded to thousands (e.g. `728k`). Leave it out if any agent's total is unknown.

## Important guidelines

- **Keep each agent's context small.** Give an agent only the files it owns. Its cost scales
  with what you paste into its prompt plus what it reads, and every extra file it reads is
  re-sent on each of its own turns.
- **Less code is a valid finding.** When a smaller change does the same job as well or better, say so: it isn't a style preference. Back it with the concrete simpler version, and check the issue first, because complexity the issue asked for isn't over-engineering.
- **Be precise, not prolific.** A review with 3 real findings is worth more than one with 15 nitpicks. If you're unsure whether something is an issue, it probably isn't worth raising.

## References

| Reference | Load when |
| --- | --- |
| [follow-up-review](references/follow-up-review.md) | Step 2 finds your earlier review of this PR at an older commit: track earlier findings, review only the delta, post a short follow-up |
| [visuals](references/visuals.md) | A finding about an order of events, measured numbers, or a PR-level scope argument would read better with an image: when, how to draw and check it, how to post it |
