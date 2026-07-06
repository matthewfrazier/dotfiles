- use tree to find out file structure before creating any test or new component
- don't use any multiline text separators in logs scripts summaries and reports, no ==== no ---- etc. it is just  noise and wastes tokens
- avoid inline styles in html, prefer css whenever possible for all layout, appearance, themes, sizing needs.

## Engineering
- Smallest diff that solves the stated problem. No speculative abstraction, options, or flexibility.
- Prefer well-tested existing dependencies (repo deps first) over writing your own; justify any new dependency in one line.
- Fix root causes. No symptom wrapping (broad catches, sleeps, retries, test special-casing) without a comment stating why.
- Write for testability; run pre-commit hooks/linters/tests before declaring done.
- Non-obvious decisions: one-line rationale in code or PR — what was rejected and why.
- Uncertain about intent, or two failed attempts on one approach: stop and ask.