---
name: follow-up-review
description: Re-review a PR you already reviewed - track the earlier findings, review only what changed since, and post a short follow-up instead of a second full review
---

# Follow-up review

Use when Step 2 finds an earlier review of yours at an older commit. It changes Steps 3-6 as
described below; everything else in SKILL.md still applies. `{prevSha}` is the full 40-character
SHA of your latest earlier review: its `commit_id`, or the `Commit:` line of a local review file.
`git fetch` rejects the 8-character SHA in the file name.

## Earlier findings

Pull the finding headings, location links and status tables out of the latest earlier review,
without loading its full text:

```bash
gh api "repos/{owner}/{repo}/pulls/{number}/reviews/{review_id}" --jq .body \
  | grep -E '^(#### [BSN][0-9]+\.|- [^ ]+ \*\*[BSN][0-9]+\.|\| [BSN][0-9]+ \||\[`)'
```

If that review was itself a follow-up, its `### Earlier findings` table carries everything
older. Otherwise read the earlier reviews too, newest first, back to the last full one. Reviews
written by hand, or before finding IDs existed, have no IDs; their inline comments are the
findings.

Then the discussion: your inline threads with every reply, the first line of other reviewers'
threads, and human comments since your last review (where the author answers "fixed B1;
declined N1 because ..."):

```bash
gh api "repos/{owner}/{repo}/pulls/{number}/comments" --paginate | jq -s -c --arg me "{login}" '
  add | (map(select(.user.login == $me and .in_reply_to_id == null) | .id)) as $mine | .[]
  | (.in_reply_to_id // .id) as $root
  | if ($mine | index($root)) then {thread: $root, id, user: .user.login, path, line, body: (.body | .[0:300])}
    elif .in_reply_to_id == null then {thread: $root, user: .user.login, path, line, first: (.body | split("\n")[0] | .[0:120])}
    else empty end'

gh pr view {number} --repo {owner}/{repo} --json comments --jq '.comments[]
  | select(.createdAt > "{submitted_at}" and (.author.login | test("bot$|\\[bot\\]$") | not))
  | {author: .author.login, body}'
```

Give each earlier finding one status:

| Status | When |
| --- | --- |
| Fixed | The problem is gone from the code at `headRefOid`. Check the code; a "fixed" reply or a resolved thread is not proof. |
| Open | Still in the code. |
| Declined | The author gave a reason. Accept it unless it is factually wrong; then keep it Open and say why in one line. |
| Obsolete | The code it pointed at no longer exists. |

In a stack, a finding can be fixed in another layer. This PR's head includes the lower layers,
so check the code there, and name the layer's PR when the fix landed elsewhere.

When fixes to earlier findings keep causing new findings in the same code ("fixed, but it has a
new side effect"), say so in the summary and run the scope check from SKILL.md. More rounds of
fixes are a sign that the design is too complex, and one more fix won't change that.

## The delta

Review what changed since `{prevSha}`, limited to the PR's own files:

```bash
git fetch origin {prevSha}
gh pr view {number} --repo {owner}/{repo} --json files --jq '.files[].path' \
  | tr '\n' '\0' | xargs -0 git diff {prevSha} {headRefOid} --
```

Fetching by SHA works even after a force-push. A rebase or a merge from the base branch puts
base changes in this delta, so a new finding must sit on lines that are also in the PR's own
diff (`gh pr diff`), or follow directly from them. Without that file filter, a restacked branch
can show hundreds of unrelated files.

## Changes to Steps 3-6

**Step 3.**
- Dispatch agents for the delta files only, and give them the delta hunks in place of the full
  PR hunks.
- Add to each agent prompt: `**Already raised** (do not repeat): {earlier findings on this
  file, with status}`. Drop any new finding that another reviewer's thread already covers.
- Repeat the scope check only if the earlier review has no Scope section or the delta adds new
  machinery; otherwise carry its Scope section over unchanged.
- New IDs continue after the highest earlier ID in each tier (earlier S1-S8, so the next is
  S9). An ID then means one finding for the life of the PR.

**Step 4.** Same file naming. In the content:
- Metadata gains `- **Follow-up to:** [{prevShortSha}]({earlier review URL})`.
- The heading becomes `## {n} new blocking, {n} new should-fix, {n} new nitpicks; {k} still open`.
- After the summary, before Scope, list every earlier finding by its short title. Don't repeat
  their explanations; the earlier review has them. Open and declined findings stay visible;
  fixed and obsolete ones are collapsed:

  ```markdown
  ### Earlier findings

  - ❌ **S4.** {title}: still open. {anything new, one line}
  - 💬 **S3.** {title}: declined, "{author's reason, one line}"

  <details><summary>Fixed or obsolete ({n})</summary>

  - ✅ **B1.** {title}
  - ✅ **S2.** {title}: fixed in #{layer PR}
  - ⚪ **N2.** {title}: obsolete, the code is gone

  </details>
  ```

- The tier sections hold new findings only.

**Step 5.** Print `Earlier: {f} fixed, {o} open, {d} declined, {x} obsolete` above the `Scope:`
line. The findings table lists new findings only.

**Step 6.**
- Post the follow-up file as the review body, with inline comments for new Blocking findings,
  as usual.
- For an earlier Blocking finding that is still Open, or a declined finding whose reason is
  wrong, reply in its existing thread instead of opening a new one:

  ```bash
  gh api "repos/{owner}/{repo}/pulls/{number}/comments/{thread}/replies" \
    -f body='Still open at `{shortSha}`: {one sentence}. See the follow-up review.'
  ```

- Don't reply to confirm fixes (the table does that), and don't resolve threads; leave that to
  people.
- List the replies under option **(a)** in Step 5, so the user approves them together with the
  review.
