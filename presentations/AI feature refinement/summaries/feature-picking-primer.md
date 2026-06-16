# Picking a good feature — primer

You've got a fixed amount of time (say a day or two) and you want to build a feature *with* Claude — letting the agent do real work while you steer. The single biggest factor in whether that goes well isn't how hard you work. It's **what you pick**. This primer is a quick checklist for choosing well.

The throughline: pick a feature where Claude can tell on its own whether it succeeded, the worst case is harmless, and the whole job fits in your timebox with room to spare.

## The checklist

Score a candidate against these five. The first one is the strongest signal — if it fails, stop and pick something else.

1. **Verifiable** — Can you state "done" as a pass/fail check *before* you start? A test suite, a build that exits clean, a linter, a script that diffs output against a fixture, or a screenshot compared to a design. This is what lets Claude close its own loop: do the work, run the check, read the result, iterate until it passes. Without a check, "looks done" is the only signal available — and then *you* become the verification loop, eyeballing every change. Anthropic's rule is blunt: "If you can't verify it, don't ship it."

2. **Right-sized** — Does the whole explore → plan → build → verify loop fit in one session, with buffer left over? Claude's context fills up as it works, and quality drops as it fills. A good rule of thumb: if you can't describe the core feature in one sentence, the scope is too broad. Reserve roughly half your timebox for the unexpected — a feature that fills every minute with zero slack is too big.

3. **Clear, with a precedent** — Can you describe it concretely: which files, what the behavior is, and an existing pattern in the codebase to imitate? "Follow the pattern in `HotDogWidget.php`" beats a paragraph of prose. A precedent matters because it makes "looks plausible" and "is actually correct" line up — the agent has something real to copy instead of inventing.

4. **Safely scoped** — If Claude gets something wrong, is the damage contained and reversible? Prefer work that lives inside one repo, touches no shared or production infrastructure, and sits off the critical path. Think "blast radius": a new module, an isolated bug fix, a small internal tool, a mechanical refactor. A mistake in a fresh directory costs nothing; a mistake in shared auth middleware costs a lot.

5. **Genuinely valuable** — Is this real work someone would actually want, not a toy? Categories where agents reliably pay off: boilerplate, test suites, documentation, and mechanical refactors or migrations. Keep the value high and the stakes low — avoid security-critical or architecturally load-bearing work where bad judgment is expensive.

## Good vs. bad example features

| Good pick | Why | Bad pick | Why |
|---|---|---|---|
| Write `validateEmail` — here are example cases (`user@example.com` → true, `invalid` → false), run the tests after | Verifiable up front, tiny, clear | "Make the app faster" | No definition of done, unbounded scope |
| "Add server-side pagination to the `/api/products` endpoint" | One concrete change with a finish line and an obvious test | "Add a calendar widget" (no detail) | Vague — no named files, no behavior spec, no precedent |
| Move test utilities / mechanical import migration across many files | No business logic touched, verified by the build; tedious for humans, easy to check | "Modernize the application" | Open-ended, load-bearing, no pass/fail signal |

## Anti-patterns to avoid

- **No way to verify it.** If you can't name a pass/fail check in a sentence, neither Claude nor your audience will know when it's done. (Exception: a deliberate exploratory spike, where seeing how Claude interprets a loose prompt *is* the point.)
- **Vague requirements.** "Make it better." "Add a widget." The fix is naming exact files, the concrete interaction, and a real in-repo example to follow. (But don't overcorrect into a giant spec the model can't hold all at once — specific is not the same as exhaustive.)
- **Touching production or shared infrastructure.** Deleting branches, handling real credentials, migrating a live database, editing shared core. An overeager agent can do real damage here. Keep it in a disposable, reversible sandbox.
- **Overscoping — three features in a trenchcoat.** One solid feature is worth more than three broken ones. Pick the single vertical slice that runs end-to-end, and demote everything else to a stretch goal you only earn after the core works.

## Sources

- [Claude Code best practices](https://code.claude.com/docs/en/best-practices) — the spine: verification, scoping, context limits, failure patterns
- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — when a task warrants an agent at all
- [Greenfield is easy, brownfield is where AI gets real](https://medium.com/@arturormk/greenfield-is-easy-brownfield-is-where-ai-software-development-gets-real-b2afad4b7f2d) — blast radius and contained, weakly-coupled work
- [A practical guide to delivering a working MVP at a hackathon](https://medium.com/viewstools/a-practical-guide-to-delivering-a-working-mvp-at-the-hackathon-2b21095c4545) — one solid feature over three broken ones; reserve buffer
- [How to scope a project for a hackathon](https://medium.com/hack-western-2015/how-to-scope-a-project-for-a-hackathon-e3b03fca4751) — the one-sentence scope test
