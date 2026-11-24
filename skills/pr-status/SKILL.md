---
name: pr-status
description: "Check a GitHub PR (or PR stack) for new updates - merged/closed state, reviews, informal approvals without a formal GitHub Approve, unresolved comments, timeline events, CI failures, and whether the branch needs to pull/rebase from the default or base branch - then summarize code changes, no-code actions, and what's next as concisely as possible. Read-only. Use when the user says /pr-status or asks for PR updates, new comments/reviews, whether a PR was merged, whether a reviewer informally approved, whether to pull latest from main/master, or what's next after reviews or inactivity."
license: MIT
metadata:
  author: marsidev
  version: "2026.11.24"
---

# PR Status

Give the user a fast, ultra-short status check on a PR (or PR stack) after colleagues have reviewed it or after days of inactivity: what changed, what needs code vs no-code action, whether to sync with the base branch, and what to do next.

Read-only: never edit files, commit, push, pull, rebase, reply on GitHub, resolve threads, or merge unless the user asks in a follow-up.

## Usage

```
/pr-status                # current branch's PR (and its stack, if any)
/pr-status <PR# | URL>    # specific PR (and its stack, if any)
```

## Step 1 - Resolve the PR (or stack)

If native/MCP PR tools are available in the harness and expose the needed fields, use them; otherwise use `gh`.

1. Identify the target PR (omit `<PR>` to use the current branch):
   ```bash
   gh pr view [PR] --json number,url,title,state,isDraft,baseRefName,headRefName,headRefOid,mergeable,mergeStateStatus,reviewDecision,mergedAt,closedAt
   ```
   Extract `{owner}/{repo}` from `url`. Detect the repo default branch dynamically (`gh repo view {owner}/{repo} --json defaultBranchRef --jq .defaultBranchRef.name`) instead of assuming `main` or `master`.

2. **Detect stacks.** Check both directions so stacked PRs are inspected together, ordered bottom-to-top (closest to the default branch first):
   - **Downstream (base):** if `baseRefName != defaultBranch`, find the parent PR with `gh pr list --repo {owner}/{repo} --state all --head <baseRefName> --limit 5 --json number,state,title,baseRefName,headRefName,mergedAt` and repeat until reaching the default branch.
   - **Upstream (children):** check `gh pr list --repo {owner}/{repo} --state open --base <headRefName> --json number,state,title,baseRefName,headRefName` and repeat upward.

## Step 2 - Gather GitHub activity

For each PR in scope:

- **If `state == MERGED` or `CLOSED`:** note when it merged/closed and skip deep review inspection for that PR (still check downstream/upstream stack layers, since a merged base means the next layer must be retargeted/rebased).
- **If `state == OPEN`:** fetch reviews, review threads, conversation comments, checks, and timeline events:

```bash
gh pr view {number} --repo {owner}/{repo} \
  --json reviewRequests,reviews,comments,statusCheckRollup,labels,autoMergeRequest
```

Fetch inline review threads (with resolution state, outdated flag, and comment bodies):
```bash
gh api graphql -f owner='{owner}' -f repo='{repo}' -F number={number} -f query='
query($owner:String!, $repo:String!, $number:Int!) {
  repository(owner:$owner, name:$repo) {
    pullRequest(number:$number) {
      reviewThreads(first:100) {
        nodes {
          isResolved isOutdated path line
          comments(first:20) {
            nodes { author { login } body createdAt }
          }
        }
      }
    }
  }
}'
```

Fetch recent timeline events (dismissed reviews, force pushes, draft/ready toggles, base changes, review requests):
```bash
gh api "repos/{owner}/{repo}/issues/{number}/timeline" --paginate \
  --jq '[.[] | select(.event | IN("reviewed","review_requested","review_dismissed","ready_for_review","convert_to_draft","base_ref_changed","head_ref_force_pushed","merged","closed","reopened")) | {event, actor: .actor.login, created_at: (.created_at // .submitted_at)}]'
```

## Step 3 - Check git sync status (read-only)

Skip local git checks if the PR belongs to a different repo than the current working tree.

1. Run `git fetch origin` (or the remote matching `{owner}/{repo}`). Never run `pull`, `rebase`, or `merge`.
2. Compare against the PR's base branch (`origin/<baseRefName>`, or `origin/<defaultBranch>` if a stack parent already merged):
   ```bash
   git rev-list --left-right --count "origin/<baseRefName>...HEAD"
   ```
   Combine with GitHub's `mergeable` (`CONFLICTING` vs `MERGEABLE`) and `mergeStateStatus` (`BEHIND`, `DIRTY`, `BLOCKED`, `CLEAN`) to judge whether pulling/rebasing is required or optional:
   - **Required:** merge conflicts (`CONFLICTING` / `DIRTY`), branch protection requires up-to-date (`BEHIND`), or a stack parent merged.
   - **Optional / Up to date:** behind by N commits with no conflicts and not blocked, or 0 commits behind.
3. Also check if `origin/<headRefName>` has commits not in local `HEAD` (e.g. a suggestion applied on GitHub) or if local has uncommitted/unpushed changes (`git status --porcelain`).

## Step 4 - Analyze

1. **Formal vs informal approvals:**
   - List formal `APPROVED` reviews (note if dismissed or stale after later commits) and `CHANGES_REQUESTED` reviews.
   - Scan `COMMENTED` reviews and top-level comments for approval intent ("LGTM", "looks good", "approved", "ship it", "good to go", "+1", "se ve bien", etc.) from someone who has **no** active formal `APPROVED` review. Call this out explicitly so the user can ask them to click Approve.
2. **Comments and events:**
   - Focus on **unresolved** review threads, unanswered conversation comments, and meaningful events since the last push. Ignore resolved threads and noise unless relevant.
   - Distinguish **human** reviewers from **bots** (summarize bot threads briefly; prioritize humans).
   - Separate what is needed into:
     - **Code changes:** requested fixes, bugs, refactors, or questions that require editing code.
     - **No-code actions:** replying to a question, resolving already-addressed threads, pinging a reviewer for formal approval, marking ready for review, updating PR title/body/labels, re-running flaky CI, or waiting on a base PR.
3. **CI:** mention only failing or pending checks; otherwise just say `CI green`.

## Step 5 - Reply concisely

Keep the reply as short as possible. Drop any bullet that does not apply. Reply in the user's language.

### Single PR format

```markdown
**#<num>** <short title> - <OPEN | DRAFT | MERGED (<date>) | CLOSED> · <review state> · <CI state>

- **Updates:** <1-3 terse bullets of what was posted/changed; call out informal approvals explicitly>
- **Code changes:** <short list of required code edits, or omit if none>
- **No-code actions:** <short list: reply/resolve threads, ping <user> for formal approval, etc., or omit if none>
- **Sync:** <up to date with `<base>` | behind `<base>` by N commits (rebase needed: conflicts / required by branch protection | optional, no conflicts) | local has unpushed/uncommitted changes>

**Next:** <1-2 concrete next actions in priority order>
```

### Stack format

List layers bottom-to-top. Keep merged layers to one line and open layers to 2-4 tight bullets, ending with one combined **Next:** line for the stack:

```markdown
**Stack (bottom -> top)**

1. **#<base>** <title> - **MERGED** (<date>)
2. **#<mid>** <title> - OPEN · LGTM in comment by @alice (no formal approval) · CI green
   - **No-code:** ping @alice for formal Approve
   - **Sync:** base #<base> merged - retarget/rebase onto `<default>`
3. **#<top>** <title> - DRAFT · 1 unresolved thread (@bob: handle null case) · CI failing (`lint`)
   - **Code:** handle null case in `foo.ts` and fix lint

**Next:** 1) Rebase #<mid> onto `<default>` and ping @alice for formal approval. 2) Fix @bob's comment + lint on #<top>.
```

## Hard rules

- **Shortest accurate reply wins.** If the PR is already merged and not part of a stack, 1-2 lines is enough. If nothing new happened, say so plainly.
- **Never miss an informal approval.** Always check `COMMENTED` reviews and conversation comments for approval language when formal approval is missing.
- **Never hardcode `master`.** Always check against the repo's actual default branch or the PR's base branch in a stack.
- **Stay read-only.** Do not pull, rebase, edit code, or post on GitHub unless asked.
