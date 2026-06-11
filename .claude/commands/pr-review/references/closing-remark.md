# Closing Remark — Reference

The published PR review comment is a **closing remark**, NOT a summary of what the PR does. The author and other reviewers already know what the PR does — they wrote it / read it. Restating the diff back at them is noise.

A closing remark is short, verdict-focused, and matches the tone of how a senior reviewer actually closes a review in practice. It is the LAST thing the author reads before deciding what to do next.

## Hard rules

1. **Never describe the PR's changes.** No "this PR adds X" / "the new test verifies Y" / "the fix correctly mirrors Z". That information is in the diff, the PR body, and the inline comments.
2. **No standalone "## Summary" heading.** The closing remark is the entire body of the published review — no section headers, no markdown frame, just prose (or a short list when listing concerns).
3. **Length: 1 to 6 sentences.** Add a short bullet list ONLY when listing 2-3 specific concerns (archetype 5 / 7 / 11). Otherwise prose.
4. **Match the verdict to the inline findings.** If you posted blocking findings, the closing remark must reflect that — don't write "LGTM, approved" while a blocker comment sits inline. If only nitpicks, "Approving" is fine.
5. **Plain-text only.** No emojis in the closing remark (emojis live in the inline praise comments). Markdown bullets are fine for the list-style archetypes.
6. **Match the team's professional register.** No personality, no sarcasm, no "great effort" if the work is mediocre. Sound like a senior reviewer who respects the author's time.

## Archetype picker

Pick the archetype that matches the actual state of your review. Adapt the wording — these are templates, not boilerplate.

### A — Pure approve (clean PR, only praise)

Use when: zero non-praise findings inline. Decision is `APPROVE`.

```
LGTM, great work.
```

```
Re-reviewed the latest changes — the earlier concerns have been addressed and the additional test coverage looks good. Approving.
```

### B — Approve with brief acknowledgement

Use when: only `nitpick:` or `suggestion (non-blocking):` comments inline. Decision is `COMMENT` (or `APPROVE` if nits are truly trivial).

```
Looks good overall. The implementation is clean, tests cover the main scenarios, and the naming/readability are solid. Approved.
```

```
Solid implementation overall. A few minor comments inline for maintainability/readability, but functionally this looks good. Approving.
```

```
Nice work on this PR. The flow is easy to follow and I especially liked the separation of concerns between the service and controller layers. Tested locally and everything worked as expected. Approving.
```

```
Overall this looks good to merge. I left a couple of non-blocking suggestions around naming consistency and small readability improvements, but nothing that should hold this PR back.
```

```
Nice work — this is a meaningful improvement over the previous implementation. The code is much easier to follow now and the added tests help confidence quite a bit.

Left a few minor suggestions inline, but happy to approve.
```

### C — Approve with maintainability/readability nits

Use when: only readability/maintainability suggestions, no correctness concerns. Decision is `COMMENT`.

```
Good progress here overall. I like the direction and the implementation is mostly consistent with the existing patterns in the codebase.

Left several comments focused mainly on long-term maintainability and readability rather than correctness. Address those and this should be in a strong state to merge.
```

### D — Conditional pre-merge ask (single small ask)

Use when: 1 concrete ask (feature flag, follow-up, breaking-change handling, etc.) that should land in this PR.

```
Great effort but let's make sure we have a feature gate to control the behaviour and easy rollout.
```

```
I would suggest handling the breaking changes according to the other comments.
```

```
I still have some reservations about the approach, but given the current requirements and timeline I'm okay with moving forward.

Please make sure we track the follow-up improvements discussed in the comments/thread.
```

### E — Concerns to address before merge (list-style)

Use when: 2-3 blocking concerns inline that should be addressed before merge. Decision stays `COMMENT` per the no-`REQUEST CHANGES` rule, but the prose makes the expectation clear.

```
Thanks for the PR. The overall direction makes sense, but I have a few concerns that should be addressed before merging:

- Error handling for the API call is incomplete
- Some edge cases around null/empty responses are not covered
- The current implementation may introduce duplicate processing in concurrent scenarios

Please take another pass and I'll re-review afterward.
```

```
I have a couple of blocking concerns before approving:

- The current query pattern may create performance issues at scale
- User input is being passed through without sufficient validation/sanitisation

These should be addressed before merge since they could introduce production risks.
```

### F — Approach concern / synchronous discussion needed

Use when: the design itself needs alignment before line-by-line review is productive. Decision is `COMMENT`.

```
I think we should revisit the approach before merging. Right now the business logic is tightly coupled with the controller layer, which may make this harder to maintain and test over time.

I'd recommend extracting the transformation logic into a separate service/helper so responsibilities remain clearer.

Once that structure is adjusted, the rest of the PR should be in good shape.
```

```
I think we should align on the implementation approach before moving further. Some of the decisions here could impact other parts of the system, and I'm not fully convinced the current direction is the best tradeoff.

Let's discuss the approach synchronously and then continue the review afterward.
```

### G — Test coverage gap

Use when: the implementation looks reasonable but coverage is the main concern.

```
Functionally this looks close, but I'd like to see additional test coverage before we merge this.

In particular:

- Failure/error scenarios
- Boundary conditions
- Regression coverage for the existing behavior being modified

Please add tests for those cases and I'll take another look.
```

### H — PR scope / size concern

Use when: the diff mixes unrelated concerns or is too large to review confidently.

```
Thanks for putting this together. The changes themselves generally make sense, but the PR is quite large which makes it difficult to review confidently.

For future PRs, it would help to split changes into smaller logical units (e.g. refactor vs feature vs cleanup) so reviews are easier and safer.

I've left a few comments inline, but I may need another pass once those are addressed.
```

```
The implementation itself looks reasonable, but this PR now includes multiple unrelated concerns (feature changes, refactoring, formatting updates).

Can we reduce the scope to only the intended feature/fix? That will make the review and future debugging much easier.
```

## Picking — decision matrix

| Inline findings | Decision | Likely archetype |
|---|---|---|
| Only praise (or nothing) | APPROVE | A |
| Only nitpicks / non-blocking suggestions | COMMENT | B |
| Maintainability/readability suggestions only | COMMENT | C |
| One concrete pre-merge ask (feature flag, follow-up) | COMMENT | D |
| 2-3 blocking findings (correctness/perf/security) | COMMENT | E |
| Architectural disagreement at the design level | COMMENT | F |
| Implementation OK, coverage gap is the main issue | COMMENT | G |
| Diff is too large / mixes unrelated concerns | COMMENT | H |
| Re-review after prior round of changes addressed | APPROVE | A (second variant) |

If two archetypes both fit, pick the one that matches the SEVERITY of the worst inline finding. A single blocking issue trumps three nitpicks for archetype selection.

## Anti-patterns — what NOT to ship

```
### Summary
This PR fixes the CheckRetry callback by adding a ctx.Err() check before
evaluating the network error. The new test verifies that mid-flight context
cancellation is propagated correctly...
```
**Why bad:** That's a summary of the diff. The author wrote the diff. They know.

```
LGTM 🎉🚀✨
```
**Why bad:** Empty enthusiasm. Either say what's good or just "LGTM, great work".

```
Approved. Great work team! Ship it! 🚀
```
**Why bad:** Cheerleading register, not a senior reviewer's voice.

```
Looks fine I guess. Couple of things inline.
```
**Why bad:** Lazy and dismissive. Either the PR is fine ("looks good, minor comments inline") or it isn't ("a few concerns before merge: …").
