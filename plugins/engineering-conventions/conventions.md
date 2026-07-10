# Oaklake Studio engineering conventions

Default working conventions for Oaklake Studio. Apply them unless a project's own
instructions (CLAUDE.md, direct requests) say otherwise.

These are opinionated. If you installed this plugin and want your own house rules,
edit `conventions.md`; whatever it holds is exactly what gets injected each session.

## Writing code
- Explicit over clever. Write for the next person who reads it, and let comments explain why, not what.
- Match the surrounding code. Follow the naming, idioms, and comment density of the file you're editing.
- Single responsibility. Each unit should have one reason to change.
- Reuse before adding. Look for an existing helper, pattern, or component before writing a new one.
- Fail loudly. Surface errors; never swallow them in empty catches or silent fallbacks.
- Leave it cleaner than you found it. Delete dead code you touch instead of commenting it out.

## Making changes
- Descope. Ship the smallest change that solves the real problem, keep the diff small and reviewable, and don't fold in unrequested work (propose extras instead).
- Fix the root cause, not the symptom. Reproduce a bug before you fix it, and check whether the same bug exists elsewhere.
- Prove it before you call it done. Cover the new behavior with tests, get every quality gate green across the board (lint, build, format, typecheck, tests, spellcheck), and show the evidence (output, a run, a screenshot). No "it works" without it.
- Don't fabricate. If you're not sure, verify or say so; never invent APIs, flags, paths, or facts.
- Do a security pass. Watch for injection, secret leakage, missing authz, and unsafe input handling.
- Be deliberate with dependencies. Justify adding one (a few lines often beat a package; check its maintenance and security), and flag stale or risky ones instead of upgrading silently.
- Update the docs the change affects.
- Confirm before destructive or irreversible actions. Get explicit sign-off before deletes, migrations, force-pushes, or prod changes. No announce-and-execute in one step.
- Don't run git operations (commit, push, branch, PR) unless explicitly asked. Do the work, stop at the git boundary, and hand the commands to a human. If asked to commit, commit as the human, with no AI co-author trailers ("Co-Authored-By", "Generated with").
- Automate quality and systematize. Prefer linters, formatters, pre-commit hooks, and CI over manual vigilance, and turn a practice worth repeating into a rule, script, or checklist instead of a one-off.

## Frontend
- Keep the theme consistent. Reuse design tokens; no ad-hoc colors, spacing, or sizes.
- Accessibility is not optional: semantic HTML, labels, visible focus states, adequate contrast, working keyboard paths.

## Communicating
- Overcommunicate. Say what you did and why, call out decisions and trade-offs, and summarize the changes when a task is done.
- Surface things early. Ask when requirements are ambiguous instead of guessing, and flag it as soon as scope or risk grows rather than quietly pushing through.
- Be direct. Concise and action-oriented, no hedging, and push back when you have a reason instead of just agreeing.
- Flag opportunities you spot in passing: better UX, performance, security, or architecture, even when they fall outside the task.
- Think about product impact, not just the literal ticket.

## Writing prose (READMEs, PRs, docs, messages)
- Don't write text that reads as AI-generated. No em dashes; use commas, colons, parentheses, or two sentences. Vary sentence length. Cut filler like "it's worth noting" and "furthermore". Avoid stock LLM words (delve, leverage, robust, seamless). Skip the neat closing summary.
