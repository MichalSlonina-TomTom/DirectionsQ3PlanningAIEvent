# Breaking a feature into parallel tasks — primer

A 3–5 minute read for someone who's never split a feature for parallel work — whether across teammates or across several Claude sessions running at once.

## Why split work at all

A whole feature dropped on one agent (or one person) tends to get half-built: Claude will try to "one-shot the app" and leave gaps, and a single context window fills up and degrades as it goes. Breaking the feature into named, independent tasks lets multiple workers run at the same time, each with clean focus and a clear "done." That's the upside.

The honest caveat up front: **most coding work parallelizes worse than you'd hope.** Anthropic's own engineers note "most coding tasks involve fewer truly parallelizable tasks than research" — research is read-heavy (safe to do in parallel), while building a feature is *write*-heavy, and writes collide. Treat parallelization as something you justify, not your default.

## How to find clean seams

A "seam" is a boundary where you can cut the feature so the pieces don't step on each other. The single most useful test comes from GitHub Spec Kit's `[P]` (parallel) marker. A task is parallelizable **only if both are true**:

1. it touches **different files** than the other concurrent tasks, and
2. it has **no dependency on a task that isn't finished yet.**

That's it — and it's mechanical, which is the point. If two tasks would edit the same file, they are *not* parallel, even if they feel logically separate.

So the reliable way to cut seams is **by module / directory ownership**, not by intent over shared files:

- Good split: Agent 1 owns `src/auth/*`, Agent 2 owns `src/payments/*`, Agent 3 owns the dashboard UI.
- Bad split: two agents both editing `src/utils.ts` "from different directions," or both touching the central `src/types/index.ts`. Separate branches won't save you — you get merge conflicts *and* contradictory design decisions.

**Avoid the most tempting wrong split: by lifecycle phase.** Handing a feature from a "plan" agent to a "code" agent to a "test" agent to a "review" agent is the *telephone game* — each handoff loses fidelity and forces re-explanation. Anthropic's rule: "decompose by context boundaries, not problem type." Those phases are *sequential*, not parallel.

One more move makes ~5–7 tasks possible: **freeze shared contracts first, as a sequential "phase 0."** Most features only split 2–4 ways naively, because everything depends on the data shapes. So one task defines the contract up front — API schema, DB migration, shared types/interfaces — and then the dependent tasks code against that frozen stub instead of waiting. Spending 10–15 minutes on a file-ownership table plus a frozen contract "saves hours of conflict resolution."

### A worked mini-example

Feature: **"Let users save and share a wishlist."** Naively it's one blob. Split by file ownership with a contract-first task:

| # | Task | Owns (files) | Parallel? |
|---|------|--------------|-----------|
| T1 | Define the contract: DB schema + API request/response types for wishlist & share-link | `migrations/*`, `src/types/wishlist.ts` | **No** — phase 0, must finish first |
| T2 | Wishlist data layer (CRUD against the schema) | `src/db/wishlist/*` | `[P]` after T1 |
| T3 | API routes (`GET/POST/DELETE /wishlist`) coded against the frozen types | `src/routes/wishlist.ts` | `[P]` after T1 |
| T4 | Share-link service (generate/resolve a public token) | `src/services/share/*` | `[P]` after T1 |
| T5 | Wishlist UI component (renders against the frozen API shape) | `src/components/Wishlist.tsx` | `[P]` after T1 |
| T6 | Share button + public view page | `src/components/Share.tsx`, `src/pages/shared.tsx` | `[P]` after T1 |
| T7 | Integration tests across the seam (run *while* T2–T6 build) | `tests/wishlist/*` | `[P]` after T1 |

T1 is the bottleneck you accept on purpose. Once it lands, T2–T7 each own a disjoint file set and run in parallel against the same agreed contract — nobody waits for anybody.

## How granular to go

There's no magic line count — it's a judgment call, bracketed by a few heuristics:

- **One feature per session/agent**, not the whole app. A well-sized task is one you could write an end-to-end test for.
- **Upper bound:** anything bigger than a ~2-day estimate, break into smaller tasks before parallelizing.
- **Lower bound:** if tasks get so small that coordinating and merging them costs more than the work, you've over-split. "If subtasks are too large, parallelism is wasted; too small, the overhead of coordination outweighs the benefit."
- **Match the count to the work.** Anthropic's effort ladder: simple work = 1 agent; comparisons = 2–4; genuinely complex work = more. Don't fan out to 7 agents if the feature honestly splits 3 ways.

## How to have Claude propose the split

You don't have to draw the task graph yourself — but you do have to *gate* it.

1. **Spec first, not a one-liner.** Ask Claude to interview you: *"I want to build [feature]. Interview me in detail using the AskUserQuestion tool — implementation, edge cases, concerns, tradeoffs — then write a SPEC.md."* A good spec "names the files and interfaces involved, states what is out of scope, and ends with an end-to-end verification step." Naming files is exactly what lets the tasks not collide.
2. **Let Claude draft the breakdown in plan mode** (Shift+Tab). It reads the repo, proposes a split, and makes *no edits* until you approve. This is your human gate — review and fix the seams before anything touches disk.
3. **Treat the proposed split as a draft, not gospel.** A plausible-looking split can quietly omit a task or mis-draw a seam. You approve ownership, contracts, and merge order; the agent doesn't.

## Pitfalls

- **Hidden coupling collapses parallelism back to serial.** In Anthropic's C-compiler project, parallel agents flew through many *independent* failing tests — but compiling the Linux kernel was "one giant task": "every agent would hit the same bug, fix that bug, and then overwrite each other's changes." Sixteen agents didn't help. If the pieces secretly share a root cause or shared state, splitting them is theater.
- **Worktrees isolate files, not logic.** Git worktrees stop two agents from editing the same file, but two branches can each pass their own tests and still be *incompatible* at integration (mismatched signatures, conflicting assumptions). File isolation buys safe concurrent edits; it does not buy correct integration. Lock the seams (interfaces, schemas, shared types) *before* you fan out.
- **Reintegration is a real, serial cost — budget for it.** Conflicts surface at *merge* time, not while agents work. Merge in dependency order (contracts → backend → frontend → tests), run the full test suite after each merge, and never merge agent output blindly. The "reduce" step that recombines parallel work is often the hardest part.
- **Over-splitting (and its cost).** Don't slice past natural module boundaries, and remember parallelism isn't free: running ten agents burns roughly ten times the tokens of one — no shared-context discount. The speed-up only pays off when tasks are genuinely independent.

**One-liner:** find seams by *file ownership* (different files, no unfinished dependency), freeze shared contracts as phase 0, keep each task feature-sized, let Claude draft the split but gate it yourself — and remember worktrees isolate files, never logic.

## Sources

- https://www.anthropic.com/engineering/multi-agent-research-system
- https://www.anthropic.com/engineering/building-c-compiler
- https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them
- https://code.claude.com/docs/en/best-practices
- https://code.claude.com/docs/en/worktrees
- https://raw.githubusercontent.com/github/spec-kit/main/templates/commands/tasks.md
- https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees
- https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents
- https://www.aakashx.com/blog/parallel-claude-code-agents/
- https://zachwills.net/how-to-use-claude-code-subagents-to-parallelize-development/
