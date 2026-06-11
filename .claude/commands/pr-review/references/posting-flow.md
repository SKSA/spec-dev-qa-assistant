# Posting Flow — Reference

This reference covers Phase 7 of the workflow: how the skill posts the draft review to GitHub via `gh`. It is consulted on-demand when the user chooses `yes` or `post it` after editing.

## Three-way prompt (free-text in chat, NOT AskUserQuestion)

After the draft is written to `<repo-root>/.idea/pr-reviews/PR-<num>-review-attempt<X>.md`, the skill says (in personality voice):

```
Draft saved to .idea/pr-reviews/PR-<num>-review-attempt<X>.md.
Found <N> issues (<X> blocking, <Y> non-blocking), <Z> nitpicks, <W> questions.

Post to PR? (yes / edit first / skip)
```

The user replies in free text. The skill parses the reply:
- Starts with "yes", "y", "post", "go", "ship" → **`yes` branch**
- Starts with "edit", "wait", "hold" → **`edit first` branch**
- Starts with "skip", "no", "n", "cancel" → **`skip` branch**
- Anything else → ask once more, more directly

## `yes` branch — post to GitHub

### Step 1: Verify GitHub state

```bash
gh pr view <num> --json state,headRefOid
```

If `state` is `CLOSED` or `MERGED`, refuse to post:
```
PR #<num> is <state>. Refusing to post — would be noise on a settled PR.
Draft preserved at <path> for your records.
```

If `headRefOid` differs from what the draft was written against (commit hash recorded in draft header), warn:
```
PR has new commits since the draft was written. Re-run the review on the latest HEAD,
or post anyway? (rerun / post anyway)
```

### Step 2: Parse the draft

Extract:
- Decision: `APPROVE` / `COMMENT` (legacy `REQUEST_CHANGES` is coerced to `COMMENT` per Phase 6)
- Inline comments: list of `(file, line_start, line_end, body)` from the "File-by-file findings" section
- Cross-file block: from "Cross-file findings" (folded into the closing remark only if directly relevant; otherwise stays in the local draft for the user's reference)
- Closing remark: from "Closing remark" — this becomes the published review body
- Sections marked `<!-- skip -->` are excluded

**Do NOT extract or post any "Summary" section.** Older drafts may still have a `## Summary` heading from previous skill versions; treat it as draft-author scratch space and skip it. The published review body is the closing remark only.

### Step 3: Build the review payload

**Goal:** Submit ONE atomic review that groups every inline comment under the same review scope, with the closing remark as the review body. The author gets ONE notification, not N+1.

**Why atomic:** The previous posting flow used `POST /pulls/<num>/comments` per inline (creating standalone PR comments) and then `gh pr review --comment --body-file` for the closing remark. Result: scattered comments not bound to a review, plus a separate review with an empty inlines list. Authors got notification spam and the comments didn't visually cluster in the GitHub UI. The fix below uses `POST /pulls/<num>/reviews` (Reviews API) which accepts `body + event + comments[]` in one call.

**Extract from the draft:**
- Each finding under `## File-by-file findings` → one entry in `comments[]`.
- Each finding under `## Cross-file findings` → one entry in `comments[]`, anchored to the `<file-or-area>:L<line>` heading you wrote in the draft (pick the most representative line; if no specific line, use the file's line 1 with the cross-file scope explained in the body).
- Each `## Replies to existing comments` block → use `gh api repos/<owner>/<repo>/pulls/comments/<comment-id>/replies` separately (these need `in_reply_to` and don't fit the Reviews API). Post replies AFTER the main review submits successfully.
- Closing remark from `## Closing remark` → review body.
- Sections preceded by `<!-- skip -->` → exclude.

**Build a JSON payload file** at `/tmp/pr-review-deep-<num>-review.json`. Concrete shape:

```json
{
  "commit_id": "abc123def456...",
  "body": "Looks good overall. A few minor comments inline for maintainability/readability, but functionally this looks good. Approving.",
  "event": "COMMENT",
  "comments": [
    {
      "path": "pkg/orders/handler.go",
      "line": 67,
      "side": "RIGHT",
      "body": "issue (blocking): SQL string concatenation creates an injection vector.\nUse parameterized query: `db.QueryContext(ctx, \"SELECT ... WHERE id = $1\", userID)`."
    },
    {
      "path": "pkg/orders/service.go",
      "start_line": 45,
      "start_side": "RIGHT",
      "line": 52,
      "side": "RIGHT",
      "body": "suggestion (blocking): Handler is doing business logic.\nMove to `service.go` per hexagonal architecture."
    }
  ]
}
```

Field reference:
- `commit_id` — the `headRefOid` from Step 1.
- `body` — closing remark verbatim, plain prose or bullets, no `## Summary` heading, no diff summary.
- `event` — exactly one of `"APPROVE"` or `"COMMENT"` (`"REQUEST_CHANGES"` is forbidden — see below).
- `comments[]` — single-line entries use `path + line + side`. Multi-line entries add `start_line + start_side`. `side` is always `"RIGHT"`.

Rules for building the payload:
- `event` MUST be `APPROVE` or `COMMENT`. Coerce a legacy `REQUEST CHANGES` draft Decision to `COMMENT`. **`REQUEST_CHANGES` as an event value is FORBIDDEN** — see SKILL.md Phase 6 rationale.
- Single-line comments use `path + line + side: "RIGHT"`.
- Multi-line comments add `start_line + start_side: "RIGHT"` alongside `line + side`.
- For pure-praise comments (emoji + prose, no `praise:` prefix), the body is the emoji + prose verbatim.
- Use the `<commit_id>` recorded in the draft header (`HEAD at review time:`). If `headRefOid` from Step 1 differs, you've already prompted the user — use whichever they chose.

**Pre-submit sanity check on the closing-remark body field:**
- Does it start with "This PR …" / "The fix …" / "The new test …" / "This change …"? → REWRITE. That's a diff summary, not a closing remark.
- Does it contain `## Summary` or any other section heading? → Strip headings; closing remarks are body-only prose.
- Is it longer than 6 sentences (excluding bullets)? → Trim. Closing remarks are short.
- Does it contradict the inline findings? (e.g. "LGTM" while a blocker is queued) → Pick a different archetype per `references/closing-remark.md`.

### Step 4: Submit the review atomically

**Branch on the decision:**

#### 4a. APPROVE with zero inline findings (pure-praise case)

```bash
# Write closing remark verbatim (no heading) to a temp file:
# /tmp/pr-review-deep-<num>-closing.md

gh pr review <num> \
  --approve \
  --body-file /tmp/pr-review-deep-<num>-closing.md
```

One call. One notification. Done.

#### 4b. COMMENT or APPROVE with inline findings (everything else)

Submit the JSON payload from Step 3 in a single atomic call:

```bash
gh api repos/<owner>/<repo>/pulls/<num>/reviews \
  --method POST \
  --input /tmp/pr-review-deep-<num>-review.json
```

This creates ONE review of type `<event>` containing all `comments[]` grouped under it, with `body` as the review body. The author gets ONE notification.

**`event: REQUEST_CHANGES` is FORBIDDEN** even if the draft has `issue (blocking)` findings. Severity is communicated in the inline comment body via the `issue (blocking):` prefix, not by gating merge through the GitHub state machine. The author owns merge timing. See SKILL.md Phase 6 "Why no REQUEST CHANGES" for rationale.

If the draft's Decision section literally says "REQUEST CHANGES", treat it as a draft-authoring bug — coerce `event` to `"COMMENT"` and proceed.

#### 4c. Failure handling on the atomic POST

If the API returns 422 with a comment-level error (e.g., "line X is not part of the diff", "path Y not found in the PR"):

1. Parse the error response — GitHub identifies which `comments[]` entry failed.
2. Strip the offending entry from `comments[]` and retry the atomic POST once.
3. If a second attempt fails: post the review body alone (no inlines) via Step 4a's pattern with `--comment`, then report to the user which inlines were dropped and their content so they can decide whether to post manually:
   ```
   Submitted review with closing remark, but <K> inline comment(s) couldn't be attached
   (line moved or file renamed since draft was written):
     - <file>:L<line> — <first 80 chars of body>
     - ...
   Re-run the review on the latest HEAD if you want these re-attached.
   ```

If the API returns auth/network errors (5xx, 401, 403): refuse to retry blindly. Report the error and tell the user the draft is preserved — re-run after fixing the underlying issue.

#### 4d. Replies to existing comments (if any)

After Step 4b succeeds, post each entry from the draft's `## Replies to existing comments` section:

```bash
gh api repos/<owner>/<repo>/pulls/comments/<original-comment-id>/replies \
  --method POST \
  --field body="<reply body verbatim, in Conventional Comments format>"
```

Replies do NOT bundle into the main review (different API surface). Each reply is its own threaded comment on the original line. This is by design — replies belong on the original thread for continuity.

If a reply fails (original comment deleted, etc.): log it, skip, continue with the rest. Do not abort the whole post on a reply failure — the main review already landed.

### Step 5: Confirm

```
Posted 1 grouped review to PR #<num>:
- Decision: <APPROVE|COMMENT>
- <N> inline comments grouped under the review
- <K> dropped (line moved / file renamed) — listed above
- <R> replies posted to existing comment threads
Draft preserved at <path>.
```

If pure-approve (Step 4a):
```
Posted approval to PR #<num> with closing remark.
Draft preserved at <path>.
```

## `edit first` branch — pause for user edits

Skill says:
```
Draft is at <path>. Edit it, then say "post it" or "post the review" to resume.
```

Skill exits cleanly. NO files left open, NO automatic re-poll.

### Resume mechanism

When the user says "post it", "post the review", "post review for #<num>", or similar with PR context active, the skill is RE-TRIGGERED via the standard trigger description (the verb "post" near a PR identifier qualifies).

The re-triggered skill:
1. Detects the latest `attempt<X>.md` for the PR (highest X).
2. Skips Phases 0–6 (no re-review).
3. Jumps directly to the `yes` branch logic above (Step 1 onward).

If multiple `attempt<X>.md` files exist for the same PR, use the highest X (most recent draft). To use an older one, the user must say "post attempt 2" or similar.

### `<!-- skip -->` HTML comments

The user can mark sections of the draft to skip on post by inserting `<!-- skip -->` immediately before any heading (`###`, `**L<line>:**`, etc.). The parser recognizes this marker and excludes the section.

Example:
```markdown
<!-- skip -->
**L62:**
nitpick: Variable `data` is generic.
```

This nitpick won't be posted.

## `skip` branch — preserve and exit

Nothing posted. Draft remains at `<path>`. Skill says:
```
Skipped. Draft preserved at <path> for next time.
```

## Failure modes

| Failure | Behavior |
| --- | --- |
| `gh` not authenticated (`gh auth status` returns error) | Refuse to post. Say: `gh isn't authenticated. Run "gh auth login" and try again. Draft preserved at <path>.` |
| PR closed/merged | Refuse to post (Step 1). |
| Wrong repo (current dir's git remote ≠ PR's owner/repo) | Refuse to post. Say: `You're in <current-repo>, but the PR is on <pr-repo>. cd to the right repo and try again.` |
| One or more inline comments rejected by GitHub (line moved, file renamed) | Per Step 4c: strip offending entries, retry once. If still failing, post review body alone and report dropped inlines to the user. |
| Network error on the atomic POST | Don't retry blindly. Report the error, draft is preserved, user re-runs. The Reviews API is atomic — partial state isn't possible on a single failed call. |
| Reply to existing comment fails (Step 4d) | Log it, skip, continue. The main review already landed — don't abort the whole post on a reply failure. |
| Draft has 0 findings AND 0 closing remark | Don't post. Say: `Draft has nothing to post — file is empty or all sections marked <!-- skip -->. Draft preserved.` |

## Hard rules (must not)

- Never post without an explicit `yes` (or "post it" after edit).
- Never `git add` / `git commit` the draft file.
- Never use `--no-verify`, `--no-gpg-sign`, or skip hooks.
- Never delete or overwrite an existing `attempt<X>.md`.
- Never edit the draft programmatically before posting (parser is read-only).
