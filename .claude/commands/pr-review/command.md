---
name: /pr-review
argument-hint: '[PR-NUMBER or PR-URL]'
description: Deep PR review with Conventional Comments, 2-why hard gate, and Jira alignment
mode: single-agent
dependencies:
  - gh CLI (GitHub CLI)
  - git
  - Optional: acli (Azure DevOps CLI) or jira CLI for ticket integration
---

# PR Review — Deep Code Review with Conventional Comments

## Purpose
Conduct a deep, opinionated code review for a GitHub PR. Uses Conventional Comments format, hard-gates each comment behind a private 2-why interrogation, and writes drafts to `.idea/pr-reviews/PR-<num>-review-attempt<X>.md` before posting.

**Announce at start:** "I'm using the /pr-review command to review PR #<num>."

## When this skill activates

Three trigger surfaces (per the description above):

1. **Explicit review verbs** + PR identifier: "review PR #923", "audit this PR", "give me feedback on https://github.com/.../pull/923". → Skip disambiguation, go to Phase 1.
2. **Bare PR URL or #number** with no clear verb: user pastes `https://github.com/.../pull/923` or just `#923`. → Phase 0 disambiguation.
3. **Ambiguous verbs** near a PR identifier: "look at #923", "check this PR", "thoughts on https://...". → Phase 0 disambiguation.

Resume verbs: "post it", "post the review", "post review for #<num>". → Skip Phases 0–6, jump to Phase 7's `yes` branch (see `references/posting-flow.md`).

## Workflow — 8 phases

### Phase 0 — Disambiguation (only on triggers 2 and 3)

Use `AskUserQuestion` with these options:
- Full review (default — file-by-file, hard-gated 2-why, Conventional Comments)
- Short summary of changes (no opinion, just describe what changed — for the user, NOT posted to the PR)

If the user picked option 2 (summary only), execute Phases 1, 5 (cross-file only — for the summary), and skip Phase 4. Output a summary file in the same `.idea/pr-reviews/` location with name `PR-<num>-summary-attempt<X>.md`. Skip Phases 6–7. **The summary stays local — it is NEVER posted to the PR.**

If the user picked option 1, proceed to Phase 1.

**Important:** The published PR review never contains a summary of the diff. The author and other reviewers already know what the PR does. The published review uses a short closing remark (see `references/closing-remark.md`) — NOT a summary.

### Phase 1 — Context gathering

Run in parallel where independent:
```bash
gh pr view <num> --json title,body,baseRefName,headRefName,files,additions,deletions,author,state,labels,headRefOid
gh pr diff <num>
gh pr diff <num> --name-only
gh pr view <num> --comments
git remote -v   # verify current dir matches PR's owner/repo
```

Then sequential:
1. Extract Jira ticket from PR title/body. Regex: `SSX-\d+`, `AGNTX-\d+`, `CSX-\d+`. If found:
   ```bash
   acli jira workitem view <TICKET> --json
   ```
2. Detect repo standards per `references/repo-standards-detection.md`:
   - Find repo `CLAUDE.md` files
   - Read linter configs (`.golangci.yml`, `.eslintrc*`, `pyproject.toml`, etc.)
   - Sample existing files in the directories the PR touches

### Phase 2 — Jira scope-alignment check

This is a SCOPE check, not an AC checklist match.

- If ticket has explicit ACs (bullet list, checklist, "Given/When/Then"): match each to PR changes, build a coverage table for the draft.
- If ticket has no explicit ACs: narrative comparison ("ticket says X, PR does Y, do they align?").
- If the ticket is missing, empty, or its scope diverges from the PR: announce it in chat (personality voice) and mark Jira block as SKIPPED in the draft. **DO NOT BLOCK THE REVIEW.**

Example skip announcement: "Jira ticket's empty / diverges from this PR. Skipping that block, continuing with the actual code."

### Phase 3 — Existing comments

Use the output of `gh pr view --comments` from Phase 1.

For each existing comment:
- Build a dedup index: `(file, line, topic)` → `@author`
- Classify: `valid` (still applies) / `addressed` (fixed in latest commits) / `stale` (code refactored away)

**Dedup policy (HARD RULES):**

1. **Never re-raise a topic already on the PR.** If a `(file, line, topic)` is already in the dedup index, do NOT add a new top-level comment for it. The author already has the feedback.
2. **Never write `note: already raised by @<author>` inline in file findings.** That phrasing is meta-commentary that clutters the review without adding value.
3. **Never mention other reviewers' status in the Decision rationale or Summary.** Phrases like "two of Copilot's blockers are still standing", "X reviewer's comment unresolved", "pending feedback from @Y" are FORBIDDEN in the published draft. The PR author can read the existing thread; the decision belongs to YOUR new findings only.
4. **Reply only when you add substance.** If you have *additional* context for an existing valid comment — affected callers, root-cause clarification, a sharper fix recommendation, evidence the topic is broader than the original comment captured — queue a reply in the draft's `## Replies to existing comments` section. If you have nothing new, leave the thread alone.
5. **Praise stands on its own.** Don't write "this addresses @Copilot's concern about X". Praise the implementation on its merits, no attribution to who originally flagged it.

The classification still appears in the `## Existing comments` section of the draft for the user's situational awareness — but it is metadata, not feedback.

### Phase 4 — File-by-file review

Files traverse in `gh pr diff --name-only` order (PR's natural order, not re-ranked).

For each file:
1. Read the FULL file (use `Read` tool with no offset). The 2-why gate needs full context.
2. Walk the diff hunks for this file in order.
3. For each candidate finding on diff lines:
   - Run the 2-why hard gate per `references/why-chain-discipline.md`. The why-chain is PRIVATE — it does not appear in the published comment.
   - If gate passes: write a Conventional Comment per `references/conventional-comments.md`. Apply the priority order from `references/repo-standards-detection.md` to decide whether the rule even applies.
   - If gate fails to answer either why: downgrade to `question:`.
   - Check the dedup index. If the topic is already raised: SKIP it from file-by-file findings entirely. If you have substantive additional context, queue a reply in the `## Replies to existing comments` section instead.
4. Comments are scoped to diff lines only. Do NOT comment on unchanged lines outside the PR's scope.
5. Each finding stands on its own merit. Do NOT reference other reviewers ("addresses @X's concern", "as @Y noted", "still pending from @Z") in the body of any finding.

For Go files specifically, also consult `references/go-helloFresh-conventions.md` (Tier 5 fallback).

### Phase 5 — Cross-file / architectural sweep

After per-file review, do a second pass:

- **Interface vs implementation:** new interfaces with single consumers — is the interface needed?
- **Test coverage:** new public APIs without unit OR integration test coverage.
- **Scope creep:** does the diff exceed what the Jira ticket describes?
- **Pattern inconsistency:** business logic in handler in one file but in service in another?

These findings go in the "Cross-file findings" section of the draft, NOT inline.

### Phase 6 — Draft assembly

Write to `<repo-root>/.idea/pr-reviews/PR-<num>-review-attempt<X>.md`.

Determine `<X>` by scanning existing files:
```bash
ls .idea/pr-reviews/PR-<num>-review-attempt*.md 2>/dev/null | wc -l
```
The next attempt number is the count + 1.

**Before writing, verify `.idea/` is gitignored:**
```bash
git check-ignore .idea/
```
If exit code is 0: gitignored, proceed.
If exit code is 1 (not ignored): warn the user, offer two options:
- (a) Add `.idea/` to repo's `.gitignore`
- (b) Write to `/tmp/pr-reviews/PR-<num>-review-attempt<X>.md` instead

The Decision block is COMPUTED last but RENDERED first. **Only two values are allowed: `APPROVE` or `COMMENT`.** Never `REQUEST CHANGES`.

Decision rules:
- Any blocking finding (issue (blocking) or suggestion (blocking)) → **COMMENT** (the blocking severity is conveyed in the inline comment body, not by gating the merge)
- Only nitpicks/suggestions/questions → **COMMENT**
- Truly clean PR (only praise) → **APPROVE**

**Why no REQUEST CHANGES:** The author owns merge timing. Hard-blocking via the GitHub `CHANGES_REQUESTED` state forces an explicit dismissal/re-review cycle and creates social friction even when the underlying issue was a misread on the reviewer's part (which happens — see retraction-discipline rules below). Severity belongs in the comment body (`issue (blocking):` prefix), not in the review state machine. The author can read the prefixes and decide.

**Posting flow implication:** Phase 7 must call `gh pr review --comment` (never `--request-changes`) and `gh pr review --approve` only when the draft is pure praise.

**Decision rationale rules:**

- The one-line rationale must justify the decision based on YOUR new findings only.
- FORBIDDEN phrases in the rationale: "X of @Y's blockers still standing", "@Z's comment unresolved", "pending other reviewers", or any reference to other reviewers' state.
- If your blocking-severity findings happen to overlap with topics other reviewers raised, do NOT cite them — re-derive the rationale from the underlying technical issues.

**Verification before raising a contract-naming or design-intent blocker (HARD RULE):**

Before tagging any finding as `issue (blocking)` that questions a field name, type, size, or contract shape, you MUST cross-check ALL of these signals and report any conflict you find:

1. The OpenAPI / schema example value (often more authoritative than test fixtures).
2. The Go struct field name and tag.
3. The DB column name and width.
4. The Jira AC text (terminology used in the ticket).
5. Test fixture values across all test files for that field.

If signals 1–4 agree and only signal 5 disagrees, the finding is a **test fixture suggestion**, not a contract blocker. Do NOT lead with "the design is wrong" — lead with "the fixture contradicts the documented contract".

If signals 1–4 disagree among themselves, you have a real contract-level finding worth blocking on.

This rule exists because anchoring on a single test fixture and missing the OpenAPI example has led to a published blocker requiring on-PR self-correction. Avoid that.

**Conventional Comments apply to the draft, not just the post.**

Every finding written into the draft — under `## File-by-file findings`, `## Cross-file findings`, AND `## Replies to existing comments` — MUST follow the Conventional Comments format per `references/conventional-comments.md`:

- Required `prefix:` (`issue:`, `suggestion:`, `nitpick:`, `question:`, `todo:`, `thought:`, `chore:`, `note:`) — or for praise, emoji + prose with no `praise:` prefix.
- `(blocking)` or `(non-blocking)` decorator is **REQUIRED** on `issue:` and `suggestion:`. Bare `issue:` or `suggestion:` is a draft-authoring failure — fix it before saving.
- One finding per block. Two findings = two blocks, even on the same line.
- Actionable body — show the fix, link to a pattern in the codebase, or describe the change concretely.
- Concise — max ~4 lines per finding. Heavier explanations link out.
- No hard-wrapping mid-sentence. Markdown handles wrapping.
- Professional register — sarcasm and roasting stay in chat, never in the draft body.

The **closing remark** is the only block that does NOT use Conventional Comments — it follows `references/closing-remark.md` instead. Everything else in the draft is a Conventional Comment.

Use exactly this draft structure:

```markdown
# PR Review — #<num>: <title>

**Repo:** <owner/repo>
**Author:** @<author>
**Branch:** <head> → <base>
**Diff:** +<additions> / -<deletions> across <N> files
**Jira:** <TICKET-123> — <title>  (or "no ticket linked / scope-alignment skipped")
**Reviewed:** <YYYY-MM-DD HH:MM> (attempt <X>)
**HEAD at review time:** <headRefOid>

## Decision

**REQUEST CHANGES** | **APPROVE** | **COMMENT**

<one-line rationale — based on your findings only, no references to other reviewers>

## File-by-file findings

### `path/to/file1.go`

**L<line>:**
<prefix>[ (decorator)]: <body>

(every finding is a Conventional Comment — `issue:` and `suggestion:` REQUIRE `(blocking)` or `(non-blocking)`. NEVER reference other reviewers in the body. Repeat per finding per file.)

## Cross-file findings

**`<file-or-area>`:**
<prefix>[ (decorator)]: <body>

(every cross-file finding is also a Conventional Comment — same prefix/decorator rules. Use a `<file-or-area>` heading to anchor the finding even though it spans files. Repeat per finding.)

## Jira scope-alignment

(coverage table OR narrative OR "skipped — no ticket / divergent")

## Replies to existing comments  *(only if you have substantive additional context)*

**@<author> on `<file>:L<line>` (<topic>):**
<prefix>[ (decorator)]: <reply body — what you would post as a threaded reply, ONLY if it adds new info/evidence/clarification beyond the original comment>

(replies are Conventional Comments too. Most replies are `note:` (adding context), `question:` (asking for clarification), or `suggestion:` (sharper fix recommendation). The `@<author> on <file>:L<line>` heading anchors the thread; the prefix carries the contract.)

## Existing comments

- @<author> on `<file>:L<line>` (<topic>): <valid / addressed / stale>  — *internal classification for situational awareness; not posted*

## Closing remark *(this is what gets posted as the PR review body)*

<short closing remark per `references/closing-remark.md`. Pick the archetype that matches your inline findings. NEVER summarise the diff — the author wrote it, they know what it does. 1-6 sentences of plain prose, or a short bullet list when listing 2-3 specific concerns. No emojis here (emojis live in inline praise comments only).>
```

**Hard rule:** The published review body is a **closing remark**, not a summary. If you catch yourself writing "this PR adds X" / "the new test verifies Y" / "the fix correctly mirrors Z" in the closing remark, STOP — that belongs in inline comments or nowhere. See `references/closing-remark.md` for the archetype catalog.

### Phase 7 — Post handoff

Per `references/posting-flow.md`:

1. Tell the user where the draft is and the finding count.
2. Ask `Post to PR? (yes / edit first / skip)`.
3. Branch on the answer.

**Posting model — ONE atomic review per submission:**

When the user says `yes` / `post it`, the skill submits a SINGLE GitHub review (`POST /repos/<owner>/<repo>/pulls/<num>/reviews`) that bundles `body` (the closing remark) + `event` (`APPROVE` or `COMMENT`) + `comments[]` (every inline finding) in one API call. The author gets ONE notification with every comment grouped under the same review.

Decision branches:
- **APPROVE with zero inline findings** (pure praise, archetype A) — `gh pr review --approve --body-file <closing-remark>`. Atomic by definition.
- **COMMENT or APPROVE with inline findings** — single `POST /pulls/<num>/reviews` with `body + event + comments[]`. Never loop per-comment, never post the closing remark as a separate review.
- **Replies to existing comment threads** (from `## Replies to existing comments`) — posted separately AFTER the main review submits, since the Reviews API doesn't accept `in_reply_to`. Each reply lands on its original thread.

**Hard rules:**
- Never post without explicit `yes` or `post it`.
- Never split inline findings into individual `POST /pulls/<num>/comments` calls — that creates orphaned PR comments not bound to a review and spams the author with N notifications. Always bundle via the Reviews API.
- Never post the closing remark as a separate review when there are inline findings — it must be the `body` field of the same atomic review submission.
- Never `git add` / `git commit` the draft.
- Never `--no-verify` or skip hooks.
- Never overwrite an existing `attempt<X>.md`.

## Retraction discipline

If the user (or your own re-check) shows a posted finding was wrong:

1. **Reply to the original comment thread** with `gh api repos/<owner>/<repo>/pulls/comments/<comment-id>/replies` — never edit or delete the original. The retraction goes on the same thread so future readers see both sides.
2. **Lead with "Self-correction — please disregard..."** No hedging, no "well technically". The author needs to know which findings are dead.
3. If the retracted finding was the only thing driving the review state away from APPROVE/COMMENT (legacy: REQUEST CHANGES), submit a **follow-up `--comment` review** clarifying the current standing of your overall position. Do not dismiss/re-submit; just layer a corrective review on top.
4. After any retraction, audit the remaining findings for the same root cause (e.g., if you anchored on a single fixture once, did you anchor again elsewhere?).

Retractions are cheap. Dragging out a wrong blocker is expensive.

## Personality

- Chat-side communication: sarcastic-senior-dev voice (per user's global CLAUDE.md). Roast bad code mercilessly in CHAT, never in published PR comments.
- Published PR comments: professional, Conventional Comments format only. The author of the PR sees no sarcasm — saving it for chat keeps the team relationship intact.
- Skip-and-continue announcements (e.g., "Jira ticket's empty, skipping") use personality voice in chat.

## What this skill does NOT do

- Auto-post — every post is gated by explicit `yes` or `post it`.
- Replace the existing `pr-review` skill — that skill is untouched. The decision to retire it is deferred to post-validation.
- Comment on unchanged lines outside the PR's diff.
- Lint stylistic preferences the linter doesn't already enforce.

## Reference files (loaded on-demand)

- `references/conventional-comments.md` — vocabulary, decorators, body rules, examples.
- `references/why-chain-discipline.md` — the 2-why hard gate + 6 worked examples.
- `references/go-helloFresh-conventions.md` — Tier 5 fallback rules for Go and HelloFresh.
- `references/repo-standards-detection.md` — the 5-tier priority order for resolving rule conflicts.
- `references/posting-flow.md` — `gh` and `gh api` commands, failure handling, resume mechanism.
- `references/closing-remark.md` — closing-remark archetypes for the published review body (no diff summaries).

When you need any of these, read the file. Do not duplicate the content here.
