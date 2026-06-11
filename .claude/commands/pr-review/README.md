# /pr-review Command

Deep, opinionated PR review using Conventional Comments and 2-why hard gate discipline.

## Quick Start

```bash
/pr-review <PR-NUMBER>
# or
/pr-review <PR-URL>
```

## Features

- **8-Phase Workflow**: Context gathering → Jira alignment → Deduplication → File-by-file review → Cross-file sweep → Draft assembly → Post handoff → Retraction discipline
- **Conventional Comments**: Structured feedback with `issue:`, `suggestion:`, `nitpick:`, `question:`, `praise`, and more
- **2-Why Hard Gate**: Every finding must answer "why does this matter?" and "why now?" before inclusion
- **Jira Integration**: Automatic scope alignment with Jira tickets (if linked)
- **Draft-First Approach**: Reviews written to `.idea/pr-reviews/PR-<num>-review-attempt<X>.md` before posting
- **Deduplication**: Avoids re-raising topics already covered by other reviewers
- **One Atomic Review**: All findings posted in a single GitHub review (no notification spam)

## Usage Examples

### Basic Review
```bash
/pr-review 923
```

### Review with PR URL
```bash
/pr-review https://github.com/owner/repo/pull/923
```

### Resume and Post (after edits)
When the draft is ready, say:
```
post it
# or
post review for #923
```

## Review Output

Reviews are saved to:
```
.idea/pr-reviews/
├── PR-923-review-attempt1.md
├── PR-923-review-attempt2.md  (if re-run after edits)
└── ...
```

## Draft Structure

Each review draft contains:

1. **Decision** (`APPROVE` or `COMMENT` only — never `REQUEST CHANGES`)
2. **File-by-file findings** (inline comments on diff lines)
3. **Cross-file findings** (architectural/pattern issues)
4. **Jira scope-alignment** (coverage table or narrative)
5. **Replies to existing comments** (if adding substantive context)
6. **Existing comments classification** (situational awareness)
7. **Closing remark** (what gets posted as the review body — NOT a diff summary)

## Conventional Comments Format

Every finding uses this format:

```
<prefix>[ (decorator)]: <body>

Examples:
- issue (blocking): Race condition in payment processing
- suggestion (non-blocking): Extract this to a helper function
- nitpick: Typo in comment — "recieve" should be "receive"
- question: Why was the timeout increased from 30s to 60s?
- 🎉 Love the defensive nil check here!
```

## The 2-Why Hard Gate

Before any finding is included, it must pass:

1. **Why does this matter?** — Technical consequence (performance, correctness, maintainability)
2. **Why now?** — Urgency (blocking vs. future cleanup)

If either "why" can't be answered, the finding is downgraded to `question:`.

## Reference Files

The command uses these reference documents (automatically loaded on-demand):

- `references/conventional-comments.md` — Vocabulary, decorators, body rules
- `references/why-chain-discipline.md` — The 2-why hard gate + examples
- `references/go-helloFresh-conventions.md` — Tier 5 fallback rules
- `references/repo-standards-detection.md` — 5-tier priority order for rule conflicts
- `references/posting-flow.md` — GitHub API commands, failure handling
- `references/closing-remark.md` — Closing remark archetypes (no diff summaries)

## Dependencies

**Required:**
- `gh` CLI (GitHub CLI) — for fetching PR data and posting reviews
- `git` — for repository operations

**Optional:**
- `acli` (Azure DevOps CLI) or `jira` CLI — for Jira ticket integration

Check dependencies:
```bash
/setup-qa-assistant
```

## Configuration

The command respects `.idea/` gitignore status. If `.idea/` is not gitignored, you'll be offered two options:
1. Add `.idea/` to `.gitignore`
2. Write drafts to `/tmp/pr-reviews/` instead

## Retraction Discipline

If a posted finding is wrong:

1. **Reply to the thread** (never edit/delete the original)
2. **Lead with "Self-correction — please disregard..."**
3. **Audit remaining findings** for the same root cause
4. **Submit a follow-up review** if needed

Retractions are cheap. Dragging out wrong blockers is expensive.

## Integration with Other Commands

This command is standalone but can reference:

- `/verify-ac` — AC verification status
- `/post-to-jira` — Post review results to Jira
- `/generate-e2e-tests` — Test coverage suggestions

## Personality

- **Chat-side**: Uses sarcastic-senior-dev voice (per user's global CLAUDE.md)
- **Published PR comments**: Professional, Conventional Comments format only

Roast bad code mercilessly in chat, never in published comments.

## What This Command Does NOT Do

- ❌ Auto-post without explicit `yes` or `post it`
- ❌ Comment on unchanged lines outside the PR's diff
- ❌ Lint stylistic preferences the linter doesn't enforce
- ❌ Use `REQUEST CHANGES` review state (only `APPROVE` or `COMMENT`)

## Examples

### Full Review Flow

```bash
# 1. Start review
/pr-review 923

# 2. Review draft generated
#    → .idea/pr-reviews/PR-923-review-attempt1.md

# 3. Edit draft if needed (optional)

# 4. Post review
post it

# 5. Review posted atomically to GitHub
#    → All findings in one notification
```

### Summary Only (for local context)

When you paste a PR link without explicit review verbs, you'll be asked:

```
What would you like?
1. Full review (file-by-file, 2-why gated)
2. Short summary of changes (local only, not posted)
```

Choose option 2 for a quick summary without posting to the PR.

## Troubleshooting

### "No verification report found"
Run `/setup-qa-assistant` to install dependencies.

### "JIRA CLI not authenticated"
```bash
jira init
# or
/setup-qa-assistant
```

### ".idea/ not gitignored"
Add to `.gitignore`:
```bash
echo ".idea/" >> .gitignore
```

### "gh CLI not found"
```bash
brew install gh
gh auth login
```

## See Also

- `/verify-ac` — Verify acceptance criteria
- `/post-to-jira` — Post results to Jira
- `/code-review` — Built-in code review (different from this command)
