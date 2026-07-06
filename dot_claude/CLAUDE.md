- use tree to find out file structure before creating any test or new component
- don't use any multiline text separators in logs scripts summaries and reports, no ==== no ---- etc. it is just  noise and wastes tokens
- avoid inline styles in html, prefer css whenever possible for all layout, appearance, themes, sizing needs.

## Engineering discipline
- Minimize surface area. Every line is a liability; the best diff is the smallest one that solves the problem. No speculative abstraction, config, or flexibility.
- Simplicity is judged by the 3am on-call reader, not the author. Prefer boring, greppable, linear code over clever code.
- No new dependencies without stating what they replace and why hand-rolling is worse. Prefer stdlib > existing repo deps > well-tested popular libs > new code.
- Fix root causes. Never wrap symptoms (broad try/except, sleeps, retries, test special-casing) without a comment explaining why the root cause is out of scope.
- Every non-obvious choice gets a one-line rationale: what was rejected and why. Put it in the code or the PR description, not chat.
- There is no "best." When options exist, name the trade-off and which constraint decided it.
- Match the repo's existing conventions and style, even where they conflict with your defaults. Run configured linters/pre-commit hooks before declaring done.
- Commit messages: imperative subject, body explains *why*, references the issue/decision.
- Keep changes reviewable: small, single-purpose diffs. If a change exceeds ~300 lines, propose a split.
- Ship a working thin slice first; iterate. Don't build the whole system before anything runs.
- If stuck or uncertain about intent, stop and ask. Two failed fix attempts on the same approach = abandon and rethink, don't keep patching.