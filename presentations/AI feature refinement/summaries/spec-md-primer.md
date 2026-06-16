# SPEC.md — a practical 3-minute primer

For someone who's never used one. Speaker's framing, now grounded in Anthropic's official Claude Code
docs and practitioner write-ups. Cross-refs: [`key-findings.md`](key-findings.md),
[`candidate-techniques.md`](candidate-techniques.md), [`collaboration-patterns.md`](collaboration-patterns.md).

## What it is

A single Markdown file, living in your repo, that describes *what you're about to build and how
you'll know it works* — written **before** you let Claude write code. The agreed brief. **[strongly
supported]** Anthropic's [best-practices doc](https://code.claude.com/docs/en/best-practices) has
Claude write it to `SPEC.md` and says "the most useful specs are self-contained: they name the files
and interfaces involved, state what is out of scope, and end with an end-to-end verification step
that proves the feature works."

One caveat on "keep it short": the docs prize *self-contained* and *precise*, not *brief* — "time
spent making the spec precise pays off more than time spent watching the implementation." So aim for
just-enough, not artificially small. A thorough spec for a hard feature can be long.

**One thing to keep straight:** a SPEC.md is a **per-feature** artifact (one change's brief). That's
different from a project-wide `CLAUDE.md`, which holds standing repo rules Claude reads every session
([addyosmani](https://addyosmani.com/blog/good-spec/)). Two complementary documents, not one.

## Why it exists

Left to fill gaps on its own, Claude can "solve the wrong problem"
([docs](https://code.claude.com/docs/en/best-practices)). **[strongly supported]** The spec
front-loads scope, edge-case, and file decisions while they're cheap to change (a sentence) instead
of expensive (rewriting code) — "the more precise your instructions, the fewer corrections you'll
need." A second, mechanical reason: Claude's context window fills up fast and "performance degrades
as it fills... Claude may start forgetting earlier instructions or making more mistakes." A written
spec lets a *fresh* session execute with clean context, and lets you, a teammate, or a future Claude
pick the work up later without re-explaining everything. The plan stops living in a disposable chat
and becomes a durable artifact.

## What goes in it

The docs name three must-haves; the other three come from the surrounding interview/plan workflow. **[supported]**

1. **Goal** — one or two sentences. What and why.
2. **Scope / Out of scope** — the out-of-scope list is the most underrated part. It **guards
   against** (doesn't hard-stop) Claude "helpfully" adding features you never asked for. Real
   example: asked to *streamline* docs, Claude unprompted added a Discord group, a support email, and
   "best practices" ([kleiber.me](https://kleiber.me/blog/2025/10/12/claude-code-five-observations/)).
   Specs *shape* behavior, they don't *enforce* it — so pair the list with review, linters, or tests.
3. **The files and interfaces involved** — name them. Anchors Claude to reality instead of inventing structure.
4. **Approach** — the plan in steps, and what's parallelizable if a team is splitting the work.
5. **Edge cases / unknowns / risks** — what might break, what you're unsure of.
6. **Verification** — *how you'll prove it's done.* An end-to-end check, and the highest-leverage line
   in the file. **[supported]** "Claude stops when the work looks done. Without a check it can run,
   'looks done' is the only signal available, and you become the verification loop." Give it a
   pass/fail check and "the loop closes on its own." Write criteria as runnable conditions, not
   adjectives: `npm test passes` beats "well-structured code"; "page load under 2s" beats "performs
   well"; "create/edit/delete; priority Low/Medium/High" beats "users manage tasks easily"
   ([chatprd](https://www.chatprd.ai/learn/PRD-for-Claude-Code),
   [augmentcode](https://www.augmentcode.com/guides/claude-code-spec-driven-development)).

## A tiny example

```markdown
# SPEC: Rate-limit the login endpoint

## Goal
Block brute-force logins: max 5 attempts per IP per minute on POST /login.

## Out of scope
- Account lockout, CAPTCHA, IP allowlists. Just rate limiting.

## Files
- middleware/rateLimit.ts (new)
- routes/auth.ts (wire middleware into /login)

## Approach
1. Add a sliding-window limiter (reuse existing Redis client in lib/redis.ts).
2. Return 429 with Retry-After when exceeded.

## Edge cases
- Shared IPs (office NAT) — acceptable for now, note in PR.
- Redis down -> fail open (allow the request), log a warning.

## Verification
- Unit test: 6th request within 60s returns 429.
- `npm test` green; manual: curl 6x fast -> last one 429.
```

## How you actually create one (the good trick)

Don't write it from a blank page. Tell Claude:

> "I want to build [this feature]. Interview me in detail using the AskUserQuestion tool —
> implementation, UI/UX, edge cases, concerns, and tradeoffs. Keep interviewing until we've covered
> everything, then write a complete spec to SPEC.md."

That prompt is straight from the [docs](https://code.claude.com/docs/en/best-practices). **[strongly
supported]** Claude asks the questions you forgot to think about (often as multiple-choice cards via
the AskUserQuestion tool, which is why it feels like answering one prompt at a time
([SDK docs](https://code.claude.com/docs/en/agent-sdk/user-input))); you answer; it drafts the spec.

Then **edit it** — the cheap moment to catch a missing requirement. Your answers during the interview
*are* the judgment that makes this work: a rubber-stamped, AI-generated spec is worse than none,
because it bakes wrong assumptions into the whole downstream build
([buildthisnow](https://www.buildthisnow.com/blog/guide/mechanics/spec-driven-development)).

*Then* — key — start a **fresh session** and say "implement SPEC.md."  **[strongly supported]** "Once
the spec is complete, start a fresh session to execute it. The new session has clean context focused
entirely on implementation, and you have a written spec to reference"
([docs](https://code.claude.com/docs/en/best-practices)).

## Want it more structured? GitHub Spec Kit

If hand-rolling the workflow feels like a lot, [GitHub Spec Kit](https://github.com/github/spec-kit)
(MIT, GitHub-maintained, works with Claude Code) automates it. Bootstrap with `specify init
my-project`, then drive it with slash commands: `/speckit.constitution` (standing project rules) →
`/speckit.specify` (the what) → `/speckit.plan` (the how) → `/speckit.tasks` (ordered task list) →
`/speckit.implement`, with `/speckit.clarify` and `/speckit.analyze` to fill gaps and catch
contradictions ([Spec Kit README](https://github.com/github/spec-kit/blob/main/README.md),
[datacamp](https://www.datacamp.com/tutorial/spec-driven-development-with-claude-code)). It writes
plain Markdown per feature under `specs/<feature>/` (`spec.md`, `plan.md`, `tasks.md`) that you commit
like code. Note: current commands carry the `/speckit.` prefix — older tutorials show bare
`/specify`, `/plan`, `/tasks`. The on-disk file is just Markdown either way; Spec Kit adds the
process (constitution + clarify/analyze) around it.

## Pitfalls

- **Don't over-spec.** "If you could describe the diff in one sentence, skip the plan" — examples:
  fixing a typo, adding a log line, renaming a variable
  ([docs](https://code.claude.com/docs/en/best-practices)). **[strongly supported]** (The verbatim
  rule says *plan*; we extend it to *specs*, which practitioners do too
  ([alexop](https://alexop.dev/posts/spec-driven-development-claude-code-in-action/)).) Spec ceremony
  earns its keep on multi-file, unfamiliar, or uncertain work.
- **A spec is not a contract Claude can't deviate from** — review the result against it; Claude can
  quietly drift or do only part of the work.
- **Watch for spec drift.** A spec is a separate artifact that goes stale as the code changes, and "a
  stale spec misleads agents that don't know any better"
  ([augmentcode](https://www.augmentcode.com/blog/what-spec-driven-development-gets-wrong)). Update it
  as you go, or trust the code.
- **Keep it in the repo**, not in `~/.claude/`, so teammates and future sessions can see, review, and
  reuse it ([docs](https://code.claude.com/docs/en/best-practices) — "check it into git so your team
  can contribute"; [numustafa](https://numustafa.medium.com/claude-code-real-workflows-plan-mode-specs-f196fb0c3590)). **[supported]**

**One-liner:** SPEC.md is the shared brief you and Claude agree on before building — scope, the
files, and *how you'll prove it works* — so nobody (human or AI) has to guess.

---

*Note on grounding: every existing claim in this primer came back **supported** or **strongly
supported** against the sources — none were contradicted or returned no evidence. The corrections
folded in above are tightenings of overreach, not reversals: "stops Claude" → "guards against"
(specs shape, not enforce); "one question at a time" was dropped (the docs say "interview me in
detail" — the one-at-a-time feel comes from the AskUserQuestion card UI); and "the single most
important line" softened to "highest-leverage" (the docs list verification as one of three co-equal
spec elements).*

## Sources

- https://code.claude.com/docs/en/best-practices
- https://code.claude.com/docs/en/agent-sdk/user-input
- https://kleiber.me/blog/2025/10/12/claude-code-five-observations/
- https://github.com/github/spec-kit
- https://github.com/github/spec-kit/blob/main/README.md
- https://www.datacamp.com/tutorial/spec-driven-development-with-claude-code
- https://alexop.dev/posts/spec-driven-development-claude-code-in-action/
- https://numustafa.medium.com/claude-code-real-workflows-plan-mode-specs-f196fb0c3590
- https://addyosmani.com/blog/good-spec/
- https://www.buildthisnow.com/blog/guide/mechanics/spec-driven-development
- https://www.chatprd.ai/learn/PRD-for-Claude-Code
- https://www.augmentcode.com/guides/claude-code-spec-driven-development
- https://www.augmentcode.com/blog/what-spec-driven-development-gets-wrong
