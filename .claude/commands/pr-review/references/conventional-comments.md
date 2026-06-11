# Conventional Comments — Vocabulary Reference

This is the comment vocabulary the `pr-review-deep` skill uses for every PR comment. It is based on https://conventionalcomments.org/, with project-specific decorator rules.

## Vocabulary (prefix table)

| Prefix | Use case | Decorator |
|---|---|---|
| Emoji + prose (✅ 👍 🚀 🎉 💯 🔥 🙌 👏 🧠 📈 ✨ — see Emoji Lexicon below) | Praise — genuinely good code worth highlighting. **Do NOT use the literal text `praise:`** — start with an emoji that fits and natural wording. **Skips the 2-why gate.** Use sparingly — overuse devalues it. | None |
| `nitpick:` | Cosmetic — naming, minor style, formatting a linter would catch. | Implicitly non-blocking |
| `suggestion:` | Architectural/design improvement. | `(blocking)` or `(non-blocking)` REQUIRED |
| `issue:` | Defect, security hole, correctness bug, broken contract. | `(blocking)` or `(non-blocking)` REQUIRED |
| `question:` | "Help me understand X." Used when the 2-why gate failed to answer either why. | Implicitly non-blocking |
| `todo:` | Small follow-up the author should make in this PR. | `(blocking)` if scope-relevant, else `(non-blocking)` |
| `thought:` | Out-loud musing, not actionable. Use rarely. | Always non-blocking |
| `chore:` | Mechanical task (regenerate mocks, update changelog, fix import order). | Optional decorator |
| `note:` | Reviewer information for the author (e.g., "this file is also touched by PR #XYZ"). | Optional decorator |

## Decorator rules

- Every `issue:` and `suggestion:` MUST end with `(blocking)` or `(non-blocking)`. Bare `issue:` or `suggestion:` is a plan failure.
- `nitpick:`, `question:`, `thought:`, `note:` are implicitly non-blocking — decorator optional and usually omitted.
- `(if-minor)` is allowed on `suggestion:` for "fix it if it's a small change, otherwise skip."
- The decorator is a single word in parentheses immediately after the prefix and before the colon? No — the prefix is `prefix (decorator):`. Example: `issue (blocking):`. Not `issue: (blocking)`.

## Body rules

1. **One issue per comment.** Never pack two findings into one block. If you find two things on the same line, write two comments.
2. **`file:line` reference required** in the draft markdown. Format: `**L<line>:**` or `**L<start>-<end>:**` as a heading line above the comment body. The `gh` CLI handles GitHub anchoring on post.
3. **Actionable.** The comment must show the fix, link to a pattern in the codebase, or describe the change concretely. "This could be better" without saying HOW is a plan failure.
4. **Concise.** Max ~4 lines per comment in the draft. If the explanation needs more space, link to a reference doc or a file in the codebase.
5. **No hard-wrapping** of the comment body. Markdown bullets and lists are fine. No `\n` mid-sentence at column 72 — that's a commit-message convention, not a comment convention.
6. **Professional register** in published comments. Sarcasm, jokes, and personality stay in the chat-side communication with the user. The author of the PR sees only the Conventional Comment.

## Examples — what to ship

### Praise — emoji + natural wording (NOT a `praise:` prefix)

The literal `praise:` prefix reads like a robot reading a label. Praise should sound human. Start with an emoji that fits the moment, then natural prose:

```
🎉 Great use of context-aware logging here — `logger.FromContext(ctx)` keeps the correlation IDs intact.
```

```
👏 The new middleware deliberately omits the offending key from the log line — good credential hygiene.
```

```
✨ Comprehensive coverage of the upstream-error mapping (4xx, 404, 5xx, 502, network error) — exactly the right abstraction to test against.
```

```
💯 Clean error wrapping with `fmt.Errorf("...: %w", err)` — keeps `errors.Is` working downstream.
```

```
🙌 Solid table-driven test pattern with `tc`/`got`/`subject`. Easy to extend.
```

Pick the emoji that matches the kind of praise — `🎉` for clean implementation choices, `👏` for thoughtful trade-offs, `✨` for thoroughness/coverage, `💯` for technically tight code, `🙌` for adherence to convention. No hard rule — the point is "this should feel like a person noticing good work, not a label."

### Other prefixes — emoji (optional) + literal `prefix:`

For non-praise comments, the literal `prefix:` form is REQUIRED. An emoji at the start is OPTIONAL but adds visual signal — see the Emoji Lexicon section below for the full mapping.

```
nitpick: Variable name `data` doesn't say much — consider `parsedRequest` or similar.
```

```
🌱 nitpick: Variable name `data` doesn't say much — consider `parsedRequest` or similar.
```

Both are valid. The emoji is decorative — the prefix is what carries the contract.

```
suggestion (non-blocking): This loop could be a single batch query.
For an N this small it's not worth blocking on — fine to leave as-is for now.
```

```
suggestion (blocking): Handler is doing business logic (lines 45-67).
Move to `service.go` per the hexagonal architecture rule. See `pkg/customers/http/handler.go` for the correct pattern.
```

```
issue (blocking): SQL string concatenation creates an injection vector.
Use parameterized query: `db.QueryContext(ctx, "SELECT ... WHERE id = $1", userID)`.
Repo convention is parameterized queries — see `pkg/orders/repo.go:67`.
```

```
issue (non-blocking): Error from `s.repo.Save` is ignored.
Wrap and return: `return fmt.Errorf("save user: %w", err)`. Non-blocking because the only caller already handles the broader error context, but worth fixing.
```

```
question: What's the reason for the goroutine here?
The downstream call to `processItem` looks synchronous to me — am I missing a non-obvious case?
```

```
todo (blocking): Update `CHANGELOG.md` with the new endpoint before merging.
```

```
chore: Regenerate mocks after changing the `paymentClient` interface — `make mocks` should do it.
```

```
note: This file is also touched by PR #1234 — there may be merge conflicts.
```

## Emoji Lexicon

Emojis serve as fast visual signals at the start of each comment. Two usage patterns:

1. **For praise:** emoji + natural prose, NO `praise:` prefix. The emoji IS the signal.
2. **For everything else:** emoji + Conventional Comments prefix + body. The emoji is OPTIONAL but speeds scanning. The prefix still carries the contract (e.g., `(blocking)` / `(non-blocking)` decorators still required on `issue:` / `suggestion:`).

### Praise / approval (no `praise:` prefix — emoji + prose only)

| Emoji | Meaning |
|---|---|
| ✅ | Approved, ready to merge |
| 👍 | Looks good / agree with change |
| 🚀 | Great improvement or performance win |
| 🎉 | Nice addition / celebration-worthy change |
| 💯 | Perfect implementation / no concerns |
| 🔥 | Important fix or impactful cleanup |
| 🙌 | Appreciation for thoughtful work |
| 👏 | Thoughtful trade-off / good judgment call |
| 🧠 | Clever solution or architecture note |
| 📈 | Improves metrics / performance / readability |
| ✨ | Elegant or polished implementation |

### Issues / blockers — pair with `issue (blocking|non-blocking):`

| Emoji | Meaning |
|---|---|
| ❌ | Blocking issue, do not merge yet |
| ⛔ | Merge blocked pending changes |
| 🚨 | Critical issue / security / system-level problem |
| 🚫 | Avoid this pattern / anti-pattern |
| 🐛 | Found a bug or regression risk |
| 🔒 | Security or privacy consideration |
| 📉 | Reduces quality / performance / readability |
| ⏱️ | Performance or latency concern |
| ⚠️ | Potential concern or edge case |

### Suggestions — pair with `suggestion (blocking|non-blocking):`

| Emoji | Meaning |
|---|---|
| 🛠️ | Needs technical fix before merge |
| 🔧 | Configuration / build / tooling adjustment |
| ♻️ | Refactor request without behavior change |
| 🧹 | Minor cleanup or refactor suggestion |
| 🗑️ | Dead code or unnecessary code removal |
| 🧼 | Style / formatting improvement |
| 🏗️ | Structural / architecture concern |
| 🎯 | Suggestion is precise and actionable |
| 🤝 | Collaborative suggestion or compromise |

### Nitpicks — pair with `nitpick:`

| Emoji | Meaning |
|---|---|
| 🌱 | Nitpick / optional suggestion |

### Questions / discussion — pair with `question:` / `note:` / `thought:`

| Emoji | Meaning |
|---|---|
| ❓ | Question about implementation / logic |
| 🔍 | Needs more investigation or clarification |
| 💬 | General discussion / comment |
| 👀 | Reviewing / taking a look |

### Action items — pair with `todo:` / `chore:`

| Emoji | Meaning |
|---|---|
| 📝 | Documentation or comment update needed |
| 📚 | Missing guides / examples / reference docs |
| 🧪 | Tests missing or should be added |
| 📸 | Screenshot / UI proof requested |
| 📦 | Dependency / package related change |
| 🧫 | Experimental approach / verify carefully |
| 🔄 | Needs rebase or conflict resolution |

### Combined-form examples

```
🚨 issue (blocking): SQL string concatenation creates an injection vector.
Use parameterized query: `db.QueryContext(ctx, "SELECT ... WHERE id = $1", userID)`.
```

```
🐛 issue (blocking): Off-by-one in the loop bound — `len(items)-1` excludes the last element when the intended invariant is "process all items".
```

```
🔒 issue (blocking): API key is logged in clear text on line 45 — strip it from the log line.
```

```
🛠️ suggestion (blocking): Handler is doing business logic (lines 45-67).
Move to `service.go` per hexagonal architecture.
```

```
♻️ suggestion (non-blocking): 3-layer passthrough adds no behavior — consider inlining two layers.
```

```
🏗️ suggestion (non-blocking): New `paymentGetter` interface has only one consumer — consider whether the interface is needed.
```

```
🌱 nitpick: Variable name `data` doesn't say much — consider `parsedRequest`.
```

```
❓ question: What's the reason for the goroutine here? Downstream calls look synchronous.
```

```
🧪 todo (blocking): Missing unit tests for `ProcessPayment` — add table-driven coverage.
```

```
📝 todo (non-blocking): Add a doc comment to `Service.GetActivities` explaining the proxy contract.
```

```
🔧 chore: Update `.golangci.yml` to enable the new `gosec` rule.
```

### Picking the right emoji

- If multiple emojis fit, pick the most specific. `🐛` is more specific than `❌` for a code-level bug. `🔒` is more specific than `🚨` for a security issue.
- For praise, pick the one that matches the *kind* of praise — `🚀` for perf wins, `🧠` for clever architecture, `🙌` for thoughtful work, `✨` for polish.
- One emoji per comment. Don't pile them up.
- Emojis are optional for non-praise. A bare `issue (blocking):` is still valid. The lexicon is there to speed scanning, not to add ceremony.

## Examples — what NOT to ship

**Bare prefix without decorator on issue/suggestion:**
```
issue: SQL injection on line 45.    ← MISSING (blocking)/(non-blocking) — plan failure
```

**Two findings in one comment:**
```
issue (blocking): SQL injection on line 45 AND the variable name is unclear.    ← Split into two comments
```

**Vague non-actionable:**
```
suggestion (non-blocking): This could be cleaner.    ← HOW? Show or link.
```

**Hard-wrapped body:**
```
issue (blocking): The query string concatenates user input into the SQL,
which creates an injection vector that can be exploited by sending a
crafted user ID parameter.    ← Drop the line breaks. Markdown handles wrapping.
```

**Sarcasm in published comment:**
```
issue (blocking): Whoever wrote this clearly never met a parameterized query.    ← Save it for chat. Author doesn't deserve mockery.
```

## Source

https://conventionalcomments.org/ — full spec.
