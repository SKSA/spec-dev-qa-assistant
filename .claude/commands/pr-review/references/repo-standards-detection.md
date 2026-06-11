# Repo Standards Detection — Reference

When reviewing a file, the rules that apply resolve in this priority order. **First match wins.** The skill consults this file in Phase 1 (context gathering) so the rules are loaded before per-file review starts.

## Priority order (first match wins)

| Tier | Source | What to look for |
|---|---|---|
| 1 | Repo's `CLAUDE.md` (root or directory hierarchy) | Explicit rules the repo author wrote for AI assistants |
| 2 | Linter configs | `.golangci.yml`, `.eslintrc.{json,yml}`, `pyproject.toml`, `tsconfig.json`, `Makefile` lint targets |
| 3 | Existing patterns in same package/directory | Test naming, error wrapping style, mock conventions |
| 4 | Language conventions | Effective Go, PEP 8, Airbnb JS, etc. |
| 5 | HelloFresh conventions | The fallback ruleset in `references/go-helloFresh-conventions.md` |

## Detection commands (run in Phase 1)

Execute these from the repo root being reviewed:

```bash
# Tier 1 — repo CLAUDE.md
find . -maxdepth 3 -name CLAUDE.md -not -path './node_modules/*' -not -path './vendor/*' 2>/dev/null

# Tier 2 — linter configs (Go, JS/TS, Python)
ls -la .golangci.yml .golangci.yaml .eslintrc .eslintrc.json .eslintrc.yml pyproject.toml tsconfig.json 2>/dev/null
grep -l 'lint' Makefile package.json 2>/dev/null

# Tier 3 — existing patterns (sample a few files in the same package as the file under review)
# When reviewing pkg/orders/service.go, sample pkg/orders/*.go to see what conventions are already in use
ls pkg/<domain>/*.go pkg/<domain>/*_test.go 2>/dev/null
```

If multiple `CLAUDE.md` files exist (e.g., one at repo root, one in a subpackage), the most-specific one wins for files under that subpackage.

## How to apply the priority

When you find a candidate finding (in Phase 4), and you're about to run the 2-why gate:

1. Identify which rule the finding would flag (e.g., "test names use `tt` not `tc`").
2. Walk the priority tiers from 1 to 5. The first tier that has an opinion on this rule is the one that applies.
3. If a higher tier disagrees with HelloFresh convention, the higher tier wins. SILENTLY drop the comment.

**Example:**
```
Candidate finding: "Test cases in loop use `tt` instead of `tc`"

Tier 1 (repo CLAUDE.md): repo says "use `tt` for test cases"
→ Tier 1 wins. DROP the comment. The repo's own rule supersedes HelloFresh convention.
```

## What if the repo is silent on something?

Walk down to Tier 5 (HelloFresh conventions). If even Tier 5 has nothing to say, the finding probably isn't a convention violation — it's a personal preference. Don't ship the comment.

## What if linter and CLAUDE.md disagree?

CLAUDE.md wins. The linter is Tier 2; CLAUDE.md is Tier 1.

## Special case — multi-language repos

For polyglot repos (Go + TypeScript + Python in one repo):

- Detect file's language by extension (`.go`, `.ts`, `.tsx`, `.py`, etc.)
- Apply Tier 4 conventions for THAT language only
- HelloFresh conventions (Tier 5) are Go-only as written; for non-Go files, Tier 5 has no opinion and the finding probably isn't worth shipping unless Tiers 1–3 explicitly agree

## Logging

When the skill detects which tier applied to a finding, log it briefly in the chat output (not the published comment) so the user can audit:

```
[pr-review-deep] L45 SQL injection — Tier 4 (Go security convention) applies. Shipping as issue (blocking).
[pr-review-deep] L62 test name uses `tt` — Tier 1 (repo CLAUDE.md says use `tt`) applies. Dropping comment.
```
