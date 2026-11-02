---
name: visuals
description: When a review finding gets an image, how to draw it as HTML and screenshot it, and how to post a review that carries images
---

# Visuals in a review

A visual helps a human reader when a finding is about order, position or size. Agents read
the markdown, not the image, so the image only illustrates: every fact stays in the text.

## When

Only for one of these, and at most two per review:

| Finding is about | Visual |
| --- | --- |
| Order of events: a race, a retry, a hand-off between services | Swimlane timeline with the bad window shaded |
| Measured numbers: growth, before/after, counts over time | Annotated bar or line chart |
| A PR-level scope or over-engineering argument | Asked vs built: size bars, and added machinery → the findings it causes |

Never for nitpicks, one-line bugs, conventions, or the scope and earlier-findings lists. If two
sentences already make the point, skip the visual.

## Rules

- **Additive only.** The visible text and the Details block hold every fact, number and
  location. The image adds nothing that isn't written down.
- **Alt text states the takeaway and the key numbers**, as one sentence an agent can use.
- **Only what you read in the code or measured.** A polished wrong picture convinces more
  than a wrong sentence. A timeline whose spacing is not measured says "order of events,
  not to scale".
- **The title is the takeaway as a sentence** ("Each doubling makes one parse about 8×
  slower"), not the chart type. The subtitle names the source: file and lines, the
  measurement, or the window.
- **Labels sit on the chart:** the shaded window, the multipliers, "the fix". A legend only
  when colours need one.
- **Light card:** white background, system font, 13-16px text, red for the problem, green for
  the fix.

## Make it

1. Write one self-contained HTML page per visual: inline SVG and CSS, no scripts, no external
   files, a card exactly 880px wide (the width of a comment on GitHub). Save it in the review
   directory as `pr-{number}-{shortSha}-v{n}.html`.
2. Screenshot it at 2× so it stays sharp at that width:

   ```bash
   "$CHROME" --headless=new --disable-gpu --hide-scrollbars --force-device-scale-factor=2 \
     --window-size=880,{height} --screenshot="$REVIEW_DIR/pr-{number}-{shortSha}-v{n}.png" \
     "file://$REVIEW_DIR/pr-{number}-{shortSha}-v{n}.html"
   ```

   `$CHROME` is `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome` on macOS, or
   `google-chrome` / `chromium` on Linux. Start `{height}` at about 400.
3. **Look at the PNG before using it.** Fix overlapping or clipped labels, tags that wrap,
   text too small to read, and trailing white space (lower `{height}`), then screenshot again.
4. Put it in the finding's Details block, whose summary says what it shows:

   ```markdown
   <details><summary>Timeline: how the poll drops the final event</summary>

   ![{takeaway and key numbers}](./pr-{number}-{shortSha}-v1.png)

   {the evidence text, unchanged}

   </details>
   ```

## Post it

`gh pr comment` can upload images with `--attach`; `gh pr review` and the reviews API can't. A
review with a visual is therefore posted in two parts, from the review directory so the
`./` paths resolve:

1. **The body as a PR comment.** `--attach` uploads each image and rewrites the matching
   `![…](./…png)` reference to the uploaded asset:

   ```bash
   cd "$REVIEW_DIR" && gh pr comment {number} --repo {owner}/{repo} --body-file {body file} \
     --attach './pr-{number}-{shortSha}-v1.png#{alt text}'
   ```

   `{body file}` is the review file without its title line, plus the footer, as in Step 6.
   The command prints the comment URL.
2. **The inline comments as a review**, through the reviews API as in Step 6, with a one-line
   body: `Review with visuals: {comment URL}`. Skip this when there are no inline comments.

List both parts under option **(a)** in Step 5, so the user approves them together.
