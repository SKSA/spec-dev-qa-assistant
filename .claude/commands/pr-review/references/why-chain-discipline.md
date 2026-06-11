# The 2-Why Hard Gate — Discipline Reference

Before any non-praise comment ships, you (the reviewer) must internally answer two why-questions. The chain stays PRIVATE — it does not appear in the published comment. It gates whether the comment ships at all.

(Praise comments — those starting with an approval emoji like ✅ / 👍 / 🚀 / 🎉 / 💯 / 🔥 / 🙌 / 👏 / 🧠 / 📈 / ✨ — skip this gate; see the bottom of this doc.)

This is the skill's main differentiator. Without this, you're just another LLM nitpick-spammer.

## The gate (private reasoning, not published)

```
Candidate finding: "X looks wrong / suboptimal / risky"

Why #1: WHY is the code written this way?
  → Read constructors, callers, helper functions, types, interfaces
  → Read the Jira ticket if relevant
  → Check commit message and PR body for stated reasons
  → Read the test file if there is one — it often documents intent
  Answer required. If unanswered → DOWNGRADE to `question:`

Why #2: WHY does that reason exist? (root cause / constraint / decision)
  → Project convention? (check repo standards — CLAUDE.md, in-package patterns)
  → Framework constraint? (Go idiom, library limitation, language quirk)
  → Deliberate tradeoff? (perf vs readability, simplicity vs flexibility)
  → Validated/handled elsewhere? (constructor validates dep, caller validates input)
  Answer required. If unanswered → DOWNGRADE to `question:`

Both whys answered → comment ships at intended severity
Either why answers "actually correct given context" → DROP comment entirely
Either why is unanswerable → DOWNGRADE to `question:` (don't issue/suggest blindly)
```

## Outcomes

| Why #1 answered? | Why #2 answered? | Outcome |
|---|---|---|
| Yes, AND reveals the code is correct | — | DROP (false positive) |
| Yes | Yes, AND reveals it's a deliberate convention | DROP (legitimate decision) |
| Yes | Yes, AND reveals no good reason | SHIP at intended severity |
| Yes | No | DOWNGRADE to `question:` |
| No | — | DOWNGRADE to `question:` |

## Worked example #1 — DROP (false positive prevented)

```
File: pkg/orders/service.go
Candidate: issue (blocking): Service method ProcessOrder has no nil check on s.repo before calling s.repo.Save.

Why #1: Why no nil check?
  → Read constructor at line 23: NewService panics if repo is nil.
  → Constructor validates the dependency at construction time.
Answer: Constructor guarantees s.repo is non-nil.

Why #2: Why does the constructor panic instead of returning an error?
  → Project convention (HelloFresh existing pr-review skill, Section 2.3): "Constructor-level validation. Methods can safely use dependencies without nil checks."
Answer: Documented project convention.

→ DROP. The code is correct given the constructor contract. Commenting on this would be a false positive.
```

## Worked example #2 — SHIP (legitimate finding)

```
File: pkg/users/repo.go
Candidate: issue (blocking): SQL string concatenation on line 45.

Why #1: Why concatenation?
  → Read full file. Author wrote `query := "SELECT * FROM users WHERE id = '" + userID + "'"`. No helper function, no parameterized fallback.
Answer: Author chose this pattern; no architectural reason found.

Why #2: Why no parameterized query?
  → Repo uses parameterized queries elsewhere (grep `\$1` finds 12 matches in pkg/orders/repo.go and pkg/payments/repo.go).
  → Framework (database/sql) supports parameterized queries.
  → No comment in the code explaining the deviation.
Answer: No legitimate reason. This is a bug.

→ SHIP as `issue (blocking): SQL string concatenation creates an injection vector. Use parameterized query: db.QueryContext(ctx, "SELECT * FROM users WHERE id = $1", userID). Repo convention is parameterized — see pkg/orders/repo.go:67.`
```

## Worked example #3 — DOWNGRADE (Why #1 answerable, Why #2 not)

```
File: pkg/jobs/processor.go
Candidate: suggestion (non-blocking): Why goroutine here? Looks synchronous.

Why #1: Why goroutine?
  → Author wrote `go processItem(ctx, item)` inside a for-loop. No comment.
Answer: Author chose to spawn goroutines.

Why #2: Why a goroutine if downstream is synchronous?
  → Read processItem: it's synchronous (just a few function calls, no I/O).
  → No semaphore, no waitgroup, no error collection. Spawned-and-forgotten goroutines.
  → No code comment, no PR body explanation, no Jira ticket detail explaining concurrency.
Answer: CAN'T DETERMINE the reason. Could be a deliberate fire-and-forget, could be a bug, could be premature optimization.

→ DOWNGRADE. Ship as `question: What's the reason for the goroutine here? processItem looks synchronous to me — am I missing a non-obvious case? If it's fire-and-forget, a comment explaining the intent would help future readers.`
```

## Worked example #4 — DROP (Why #2 reveals it's handled elsewhere)

```
File: pkg/api/handler.go
Candidate: issue (blocking): Handler doesn't validate the email field is non-empty before passing to service.

Why #1: Why no validation?
  → Read full handler. The line in question is `s.service.CreateUser(ctx, req.Email, req.Name)`.
Answer: Handler trusts the request as parsed.

Why #2: Why does the handler trust the request?
  → Read the request type. `CreateUserRequest` has `Email string \`json:"email" validate:"required,email"\``.
  → Read the middleware in pkg/api/middleware.go. The `validateRequest` middleware runs the validator before reaching the handler.
Answer: Validation IS happening — at the middleware layer, before the handler executes.

→ DROP. Validation is at the boundary, just not in this file. Commenting would be a false positive AND would violate the user's "input validation philosophy" rule (Section 2.3 of existing pr-review skill).
```

## Worked example #5 — SHIP non-blocking (legitimate but minor)

```
File: pkg/payments/service.go
Candidate: suggestion (non-blocking): Loop allocates a string per iteration via fmt.Sprintf.

Why #1: Why fmt.Sprintf in the loop?
  → Author writes `result := fmt.Sprintf("Processed: %s", msg)` for each msg in messages.
Answer: Author chose Sprintf for readability.

Why #2: Why is the allocation worth flagging?
  → Read the call site: `messages` is bounded (max 100 per the Jira ticket). Not a hot path.
  → No benchmark, no perf SLO mentioned.
Answer: Allocation is real but performance impact is negligible at this scale.

→ SHIP as `suggestion (non-blocking): fmt.Sprintf allocates per iteration. For N <= 100 it doesn't matter, but if this ever scales, consider strings.Builder. Non-blocking — fine to leave.`
```

## Worked example #6 — DOWNGRADE (you're missing context)

```
File: pkg/contentful/parser.go
Candidate: issue (blocking): Returns nil silently when entry has no fields.

Why #1: Why return nil?
  → Read parser. Author writes `if entry.Fields == nil { return nil, nil }`.
  → No log line, no error.
Answer: Author chose silent skip.

Why #2: Why silent skip instead of error?
  → CAN'T DETERMINE. Could be intentional (some Contentful entries are templates with no fields), could be a bug.
  → No PR body explanation, no Jira ticket detail.
Answer: Insufficient context.

→ DOWNGRADE. Ship as `question: What's the intent when entry.Fields is nil? Returning (nil, nil) silently might mask malformed entries — was the silent skip deliberate (e.g., template entries), or should this return an error/log?`
```

## Exception — `issue (blocking)` may include a one-line context note

For `issue (blocking)` comments only, you MAY include a one-line context note in the published comment if it helps the author understand the root cause. This is the ONLY case where a fragment of the why-chain leaks into the public comment.

Example:
```
issue (blocking): SQL string concatenation creates an injection vector.
Use parameterized query: `db.QueryContext(ctx, "SELECT ... WHERE id = $1", userID)`.
Repo convention is parameterized queries — see `pkg/orders/repo.go:67`.    ← context note
```

The context note is a single short line. Not multiple paragraphs. Not the full why-chain.

## Praise (emoji-prefixed) skips the gate

Praise comments do not require the 2-why gate. The gate's purpose is to prevent false negatives (shipping a bad finding); praise has no false-negative risk worth gating.

**Format:** start with an approval emoji (✅ / 👍 / 🚀 / 🎉 / 💯 / 🔥 / 🙌 / 👏 / 🧠 / 📈 / ✨) followed by natural prose. **Do NOT use the literal text `praise:`** — it reads like a robot reading a label. The full Emoji Lexicon — including emojis for issues, suggestions, questions, and action items — is in `references/conventional-comments.md`.

That said: don't praise everything. Save it for genuinely good code. Inflated praise is worse than none.
