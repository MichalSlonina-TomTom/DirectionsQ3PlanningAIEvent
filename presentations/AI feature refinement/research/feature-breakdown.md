# Breaking a feature into parallel tasks

Splitting one feature so that several Claude agents (or people) can build pieces of it at the same time is appealing, but it is the exception you justify, not the default. The honest headline from the research: **most coding is less parallelizable than it looks.** Anthropic's engineering team states it plainly — "most coding tasks involve fewer truly parallelizable tasks than research, and LLM agents are not yet great at coordinating and delegating to other agents in real time" ([multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system)). Research parallelizes because it is read-heavy (independent exploration in separate context windows); a feature is write-heavy, and writes conflict. Everything below is about earning a parallel split when one genuinely exists — and recognizing when it doesn't.

---

## How teams decompose a feature

The reliable decomposition move is to **split along domain/module boundaries, not by intent over shared files.** Assign each task a cohesive system — one owns `src/auth/*`, another `src/routes/*`, another DB migrations under `migrations/*` — so the conflict surface is empty. The named anti-pattern is two tasks both editing `utils.ts` "from different directions," which produces merge conflicts *and* contradictory design decisions ([git worktrees for parallel AI agents](https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents), [team of AI agents](https://dev.to/battyterm/how-i-run-a-team-of-ai-coding-agents-in-parallel-p7c)).

Before launching anything, run a short **planning backbone** that is what actually lets you carve a feature apart:

1. **Build a file-ownership table** mapping each candidate task to the files it touches (Auth → `src/auth/*`, API routes → `src/routes/*`, DB migrations → `migrations/*`).
2. **Do a dependency check** — any task that needs another's output cannot run in parallel.
3. **Freeze the interfaces/contracts** (the data shapes each component exposes) up front in a shared brief.

MindStudio reports this takes 10–15 minutes and "saves hours of conflict resolution," and the file-ownership map is the single highest-leverage step — conflicts are prevented in planning, not at merge time. The rule is explicit: "If two tasks have overlapping file lists, they need to be sequential, not parallel" ([parallel agentic development](https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees), [git worktrees for parallel AI agents](https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents)).

GitHub's Spec Kit formalizes a compatible ordering as **phases**: Phase 1 Setup → Phase 2 Foundational (blocking prerequisites that MUST finish before any user story) → Phase 3+ one phase per user story in priority order → a final Polish/cross-cutting phase. Because user stories are designed to be largely independent, whole stories can be handed to separate agents *once the foundational phase is complete*. Decompose the same way: pull shared/blocking infra out first, then carve the rest into story-sized vertical slices ([Spec Kit tasks template](https://raw.githubusercontent.com/github/spec-kit/main/templates/commands/tasks.md), [DeepWiki: tasks command](https://deepwiki.com/github/spec-kit/5.3-tasks-command)).

A realistic expectation: most features yield only **2–4 genuinely independent pieces**, not 5–7. MindStudio's own worked example shows Frontend Forms depending on the Auth System, so it must sequence — unless you freeze the auth API contract first and code the dependent task against the stub instead of waiting ([parallel agentic development](https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees)). Scale the number of tasks to the feature's honest complexity; Anthropic's effort ladder is simple work = 1 agent, direct comparisons = 2–4, complex work = 10+ with clearly divided responsibilities — so don't fan out to 7 if the work only splits 3 ways ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)).

---

## Designing clean seams

A clean seam has two halves: a **data contract** and **file ownership**. You need both — agreeing the data shape doesn't help if two tasks still write the same file.

**Make the contract the deliverable before any task starts.** Write the seam as an explicit, version-controlled spec — an OpenAPI/schema definition or a typed interface — and get all sides to agree on it. Once the contract exists, the consumer and producer tasks build against it simultaneously instead of sequentially; "neither team waits for the other" ([API-first development](https://medium.com/codetodeploy/api-first-development-why-your-next-project-should-start-with-the-contract-not-the-code-5a5a25828b4b)). For Claude specifically, the equivalent artifact is the `AGENTS.md`/`CLAUDE.md` contract document — the single biggest factor in parallel-agent success — because each agent independently interprets the rules, and a vague contract makes them invent incompatible interfaces ([Claude Code agent teams](https://www.contextstudios.ai/blog/claude-code-agent-teams-a-builders-guide-to-parallel-ai-coding)).

**Stub or mock the seam so each side codes against a running fake, not a promise.** Generate the stub from the contract rather than hand-writing it: Prism, Stoplight, or WireMock spin up a mock server directly from an OpenAPI spec, and Spring Cloud Contract emits a stub JAR of contract-generated response mappings that WireMock loads to simulate the not-yet-built service ([API-first development](https://medium.com/codetodeploy/api-first-development-why-your-next-project-should-start-with-the-contract-not-the-code-5a5a25828b4b), [modernize WebSphere applications](https://ibagroupit.com/insights/modernize-websphere-applications/)).

**Give every task non-overlapping file/directory ownership as part of the seam.** Map directories to owners explicitly (`/src/api/` → API task, `/src/db/` → DB task, `/src/components/` → frontend task) and add a coordination rule: before touching a shared file, message the owner. The canonical collision is the central types/barrel file — "Two agents modifying `src/types/index.ts` simultaneously will create a conflict that's tedious to resolve." The fix: make shared interface/type files a sequenced phase-0 task that one agent defines first, then parallelize the implementations that consume it ([Claude Code agent teams](https://www.contextstudios.ai/blog/claude-code-agent-teams-a-builders-guide-to-parallel-ai-coding), [parallel agentic dev with worktrees](https://www.mindstudio.ai/blog/parallel-agentic-development-claude-code-worktrees), [parallel agentic development](https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees)).

**Isolate runtime state, not just code.** Separate stateful resources behind the seam: a per-branch database (a SQLite file per worktree, or a schema per feature), distinct ports (3001/3002/3003), Redis key prefixes or DB numbers, and separate credentials. A shared dev database produces interleaved state that makes neither feature testable alone and causes confusing bugs when migrations collide ([parallel agentic dev with worktrees](https://www.mindstudio.ai/blog/parallel-agentic-development-claude-code-worktrees)).

**Reconcile at merge, in dependency order, and make the contract executable.** A consumer-driven contract test verifies that the real implementation still satisfies the same spec the mock was generated from, so the seam is checked in CI rather than discovered broken at runtime; you can also have a dedicated "test" task write integration tests against the seam while the others build ([parallel agentic dev with worktrees](https://www.mindstudio.ai/blog/parallel-agentic-development-claude-code-worktrees), [modernize WebSphere applications](https://ibagroupit.com/insights/modernize-websphere-applications/), [Claude Code agent teams](https://www.contextstudios.ai/blog/claude-code-agent-teams-a-builders-guide-to-parallel-ai-coding)).

**Honest gap — contract-reality drift is the central failure mode of this whole approach.** Mocks generated from a contract only prove conformance *to the contract*, not to reality; integration issues surface only when you swap the mock for the real service. Contract-first also assumes the design phase captured needs correctly — it does not help when consumer feedback later reveals a flaw, at which point you pay to re-agree the contract across every parallel task at once. Treat the contract as a durable shared artifact, not a throwaway scaffold ([API-first development](https://medium.com/codetodeploy/api-first-development-why-your-next-project-should-start-with-the-contract-not-the-code-5a5a25828b4b), [modernize WebSphere applications](https://ibagroupit.com/insights/modernize-websphere-applications/)).

---

## Vertical vs horizontal slicing

The answer depends entirely on which kind of "parallel" you mean — and the two meanings give **opposite** answers.

**Delivery parallelism** — splitting a feature into thin end-to-end stories shipped incrementally over sprints — favors **vertical** slices. Value is delivered early, feature validation happens fast, and rework risk drops because bad cross-layer assumptions surface quickly. Horizontal layering defers all value until the whole stack is deployed and risks late, project-wide refactoring ([vertical vs horizontal slicing](https://www.datascience-pm.com/vertical-vs-horizontal-slicing-data-science-deliverables/)).

**Simultaneous-code parallelism** — multiple agents in isolated worktrees that can't coordinate mid-flight — flips the answer. Different vertical slices of the *same* feature tend to touch the same areas across layers (same DB tables, same modules), producing merge conflicts; one practitioner argues it is smoother to deliver vertical slices *sequentially*. Horizontal layers have non-overlapping file sets, so they *can* be parallelized — but only "if the interactions between the layers are very clear and well-understood" ([building new features: horizontal or vertical slices](https://gregpark.io/blog/building-new-features-horizontal-or-vertical-slices)).

> **Honest gap:** the "vertical rarely parallelizes" position is one practitioner's blog opinion, not a measured result. It holds specifically for concurrent code on shared files; it is not a claim about delivery sequencing.

This is the apparent tension with Spec Kit (which slices vertically by user story): it resolves the same way the data does — pull the shared/foundational layer out as a blocking phase first, then the remaining per-story slices touch disjoint files and parallelize.

The bridge that makes **either** split parallelizable for agents is a written contract locked *before* fan-out. Anthropic's best-practices guide recommends an explore → plan → implement loop and a self-contained spec that "name[s] the files and interfaces involved, state[s] what is out of scope, and end[s] with an end-to-end verification step." With the interface frozen, a horizontal split works — one agent builds the backend to the schema, another the frontend to the same schema — without them ever talking ([Claude Code best practices](https://code.claude.com/docs/en/best-practices)).

One caution: do **not** confuse competitive redundancy with decomposition. A popular pattern spins up N worktrees that each implement the *same* full spec so you can pick the best result. That exploits model non-determinism for design diversity; it is not breaking a feature into independent tasks and does nothing for seams or integration — useful as a contrast, not a decomposition technique ([parallel AI coding with git worktrees](https://docs.agentinterviews.com/blog/parallel-ai-coding-with-gitworktrees/)).

---

## Task sizing for agents

There is **no magic line or token count** — sizing is a judgment call anchored by a few concrete heuristics.

- **One feature per agent session.** Anthropic's long-running-agent harness has the coding agent target "only one feature at a time" to counter the model's tendency to "one-shot the app," which produced incomplete implementations and wasted context ([effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).
- **Under-decomposition is a documented failure mode.** The failing prompt "build a clone of claude.ai" succeeded only after an initializer agent first wrote a requirements file of 200+ discrete features such as "a user can open a new chat, type in a query, press enter, and see an AI response." The granularity bar: a feature you could write an end-to-end test for ([effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).
- **Use the context window as the sizing constraint.** A specialized subagent "can handle focused tasks with clean context windows" and returns "only a condensed, distilled summary of its work (often 1,000–2,000 tokens)." Heuristic: a task is well-sized if the agent can do it *and* report it back in ~1–2k tokens — if it needs far more to summarize, it was too big ([effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).
- **Over-decomposition has a real cost too.** "If the subtasks are too large, parallelism is wasted; if they're too small, the overhead of coordination outweighs the benefit." Real tasks are component-level ("Set up database schema," "Build authentication endpoints," "Write unit tests"), not line-by-line ([Claude Code agent teams + shared task list](https://www.mindstudio.ai/blog/claude-code-agent-teams-parallel-shared-task-list)).
- **A usable upper bound:** break anything bigger than a ~2-day estimate into smaller serial tickets *before* parallelizing. Together with the lower bound, this brackets the sweet spot: bigger than a trivial edit, smaller than a multi-day epic ([Claude Code subagents to parallelize development](https://zachwills.net/how-to-use-claude-code-subagents-to-parallelize-development/)).

What parallelizes well: independent streams with clear boundaries (backend / frontend / tests / docs), distinct modules, separate output files. What stays sequential: tightly coupled components needing real-time coordination, and chains where one agent's output is the next agent's critical input. End each task in a **mergeable state** — Anthropic's bar is "code that would be appropriate for merging to a main branch" — and mark complete only after end-to-end verification ([Claude Code subagents to parallelize development](https://zachwills.net/how-to-use-claude-code-subagents-to-parallelize-development/), [effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).

A precise, hand-applicable seam test comes from Spec Kit's `[P]` (parallelizable) marker, attached "ONLY if task is parallelizable (different files, no dependencies on incomplete tasks)." Both conditions are required, and tasks name the exact file path so the file-conflict check is mechanical — for example `- [ ] T005 [P] Implement authentication middleware in src/middleware/auth.py`, versus `- [ ] T014 [US1] Implement UserService in src/services/user_service.py` (no marker, because it depends on the model). Within a single story there is still a fixed sequential spine you cannot parallelize away: Tests → Models → Services → Endpoints → Integration ([Spec Kit tasks template](https://raw.githubusercontent.com/github/spec-kit/main/templates/commands/tasks.md), [Spec Kit on GitHub](https://github.com/github/spec-kit/blob/main/templates/commands/tasks.md), [DeepWiki: tasks command](https://deepwiki.com/github/spec-kit/5.3-tasks-command)).

---

## Using Claude to draft the breakdown

Anthropic's docs treat Claude-drafted decomposition as a real workflow — but gate it.

**Drive the breakdown from a Claude-authored spec, not a one-line prompt.** Give a minimal description, then ask Claude to interview you with the AskUserQuestion tool ("Ask about technical implementation, UI/UX, edge cases, concerns, and tradeoffs… dig into the hard parts I might not have considered"), write the result to `SPEC.md`, and start a **fresh** session to execute. The interview surfaces seams and out-of-scope boundaries before any split is committed. The most useful specs "name the files and interfaces involved, state what is out of scope, and end with an end-to-end verification step" — and "time spent making the spec precise pays off more than time spent watching the implementation" ([Claude Code best practices](https://code.claude.com/docs/en/best-practices)).

**Use plan mode as the human gate, before anything touches disk.** Enter plan mode (Shift+Tab or `claude --permission-mode plan`): Claude proposes a plan but makes no edits until you approve; press Ctrl+G to open it in your editor and correct the decomposition first. The documented loop is **Explore → Plan → Code → Commit**, explicitly to "separate research and planning from implementation to avoid solving the wrong problem." Skip it for one-sentence changes — it pays off exactly in the multi-file case that parallelizes ([Claude Code best practices](https://code.claude.com/docs/en/best-practices), [common workflows](https://code.claude.com/docs/en/common-workflows)).

**For batch-style fan-out** (many similar files, not one feature with distinct parts), have Claude generate the task list ("list all 2,000 Python files that need migrating"), then loop `claude -p` over each with `--allowedTools` scoping permissions — but "Test on a few files… Refine your prompt based on what goes wrong with the first 2–3 files, then run at scale." Treat the generated list as a draft to validate on a sample ([Claude Code best practices](https://code.claude.com/docs/en/best-practices)).

**Add an adversarial reviewer in a fresh context** before trusting the split or the build. A reviewer "running in a fresh subagent context sees only the diff and the criteria you give it, not the reasoning that produced the change" and is less biased toward code it just wrote — e.g. "review the rate limiter diff against PLAN.md… report gaps." Scope it to correctness/requirement gaps only, because "a reviewer prompted to find gaps will usually report some, even when the work is sound… Chasing every finding leads to over-engineering" ([Claude Code best practices](https://code.claude.com/docs/en/best-practices)).

**Honest gaps.** Multi-agent coordination is explicitly non-deterministic: "agents are reasoning models making judgment calls, coordination isn't perfectly predictable," debugging is "harder to trace than a sequential failure," and the orchestrator's own review is *post*-execution — so a bad split isn't caught until work is wasted. The pre-execution gate has to come from you (plan mode), not the orchestrator ([Claude Code agent teams + shared task list](https://www.mindstudio.ai/blog/claude-code-agent-teams-parallel-shared-task-list)). The headline reliability risk is the **trust-then-verify gap**: "Claude produces a plausible-looking implementation that doesn't handle edge cases" — and the same applies to the breakdown itself, which can look complete while quietly omitting a task or mis-drawing a seam. The fix: "If you can't verify it, don't ship it" — a spec that ends in an end-to-end check turns a mis-seamed task into a *failing test* rather than reviewer intuition ([Claude Code best practices](https://code.claude.com/docs/en/best-practices)).

---

## Mapping tasks to worktrees/branches and reintegrating

Git worktrees give each task its own checkout and branch so concurrent edits never collide on the filesystem. Claude Code has built-in support: `claude --worktree feature-auth` (or `-w`) creates an isolated worktree under `.claude/worktrees/<name>/` on branch `worktree-<name>`, branched from `origin/HEAD`; run it again with a different name in another terminal for a second session. You can branch from a PR with `claude --worktree "#1234"`, or set `worktree.baseRef: "head"` to carry unpushed work in ([worktrees](https://code.claude.com/docs/en/worktrees)).

To parallelize **subagents** rather than terminals, add `isolation: worktree` to a custom subagent's frontmatter (or just ask Claude to "use worktrees for your agents"); each gets a temporary worktree, auto-removed when it finishes without changes. Claude Code's broader parallelization surfaces — subagents, agent view, agent teams (experimental, disabled by default), and dynamic workflows / the `/batch` skill — are chosen by *who coordinates* and *whether workers edit the same files*; the docs are explicit: "Do the tasks touch the same files? Isolate the work with worktrees" ([worktrees](https://code.claude.com/docs/en/worktrees), [agents](https://code.claude.com/docs/en/agents)).

Two separate isolation mechanisms matter and you need both: **context isolation** (a subagent works in its own context window and returns only a summary, so logs don't flood the main conversation) and **file isolation** (worktrees so "edits in one session never touch files in another") ([worktrees](https://code.claude.com/docs/en/worktrees), [agents](https://code.claude.com/docs/en/agents)).

Setup details that bite: a worktree is a fresh checkout, so gitignored local config (`.env`, secrets) is **not** carried over — add a `.worktreeinclude` file (gitignore syntax) at the repo root to auto-copy them, re-run dependency install per worktree, and add `.claude/worktrees/` to `.gitignore`. For runtime isolation, give each worktree its own ports and database (e.g. derive `SERVICE_PORT = BASE_PORT + (WORKTREE_INDEX * 10) + SERVICE_OFFSET`, with per-worktree DB instances), since dev servers all default to 3000/5432/8080 ([worktrees](https://code.claude.com/docs/en/worktrees), [git worktrees for parallel AI coding agents](https://developer.upsun.com/posts/ai/git-worktrees-for-parallel-ai-coding-agents)). The manual equivalent for a talk demo: `git worktree add -b feature-a ../project-feature-a main`, `git worktree list`, then `git worktree remove ../project-feature-a` and `git worktree prune` after merging ([worktrees](https://code.claude.com/docs/en/worktrees), [git worktrees for parallel AI coding agents](https://developer.upsun.com/posts/ai/git-worktrees-for-parallel-ai-coding-agents)).

**Reintegrate through a staging branch, not straight to main.** `git checkout -b integration main`, then `git merge feat/auth`, `git merge refactor/api`, etc., run the **full** test suite there to catch cross-branch contradictions, and only then `git checkout main && git merge integration`. Merge in dependency order (foundational branch first), have dependent worktrees `git pull origin main` before continuing, and "don't merge agent output blindly" — review each diff for half-finished work and changes that contradict other branches. The integration protocol scales: PR-per-task for 2–5 tasks; an orchestrated ordered merge for 5+; DB migrations run sequentially only; never let agents touch shared lock files unless explicitly assigned ([git worktrees for parallel AI agents](https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents), [parallel agentic development](https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees), [team of AI agents](https://dev.to/battyterm/how-i-run-a-team-of-ai-coding-agents-in-parallel-p7c)).

Plan the final **verification/reduction** step explicitly. Synthesis — merging branches, cross-checking outputs, resolving conflicts — is reported as the hardest part of parallel work, and verifier agents tend toward "early victory bias," so instruct them to "Run the COMPLETE test suite before marking passed" ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system), [building multi-agent systems: when and how](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)).

Cleanup semantics matter when scripting: a worktree with no uncommitted changes is removed automatically on exit; one with changes prompts you to keep or remove; **non-interactive `claude -p --worktree` runs are NOT auto-cleaned** — you must `git worktree remove` (add `--force` for uncommitted changes) yourself ([worktrees](https://code.claude.com/docs/en/worktrees)).

---

## Failure modes

**Worktrees isolate files but NOT logical conflicts — the central failure mode.** Two agents on separate worktrees can each pass their own tests yet make incompatible assumptions (mismatched function signatures, shared schema, conflicting interfaces) so the merged result fails. "Green-in-isolation is a lie at the system level." File isolation buys safe concurrent edits, not correct integration. Mitigation: lock the seams (interfaces, schemas, shared types) *before* fanning out — make them a foundational task that completes first — and run an integrated validation pass with one designated validator ([worktrees](https://code.claude.com/docs/en/worktrees), [parallel Claude Code agents](https://www.aakashx.com/blog/parallel-claude-code-agents/)).

**Hidden coupling collapses parallelism back to serial.** In Anthropic's C-compiler build, task-based parallelism worked well while there were many independent failing tests (each agent claimed one via a lock file). But compiling the Linux kernel was "one giant task" — "Every agent would hit the same bug, fix that bug, and then overwrite each other's changes," and "having 16 agents running didn't help because each was stuck solving the same task" ([building a C compiler](https://www.anthropic.com/engineering/building-c-compiler)).

**Reintegration is a first-class cost.** In that same project, merge conflicts were frequent — each agent had to "pull from upstream, merge changes from other agents, push" — and "integration of new features frequently broke existing functionality," forcing continuous-integration enforcement. Budget explicitly for a serial merge+validation tail; the parallel speedup is partly eaten by reconciliation ([building a C compiler](https://www.anthropic.com/engineering/building-c-compiler)).

**Agents invent incompatible shared contracts.** "Do not let multiple agents invent shared contracts independently" — a types-agent and an API-agent running in parallel with no agreed interface shape is the classic integration blowup. Define API schemas, migrations, event names, and error conventions sequentially in the main session first, then protect those files from agent edits ([parallel Claude Code agents](https://www.aakashx.com/blog/parallel-claude-code-agents/)).

**Vague task descriptions cause duplicated work.** Anthropic's multi-agent system found a short brief like "research the semiconductor shortage" led "one subagent [to explore] the 2021 automotive chip crisis while 2 others duplicated work." The fix is detailed objectives, output formats/schemas, named tools, and explicit boundaries — in code, a sharp spec like "own `src/auth/`, implement these 3 functions against this interface, do not edit shared types" ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)).

**No formal dependency solver — only heuristics.** Spec Kit "doesn't explicitly define a formal dependency graph algorithm"; parallelizability is decided by phase ordering plus the single file-conflict rule. So `[P]` markers are only as correct as the file-path annotations and the judgment that produced them; a runtime/config dependency not reflected as a same-file relationship can be wrongly marked parallel. Treat generated markers as a draft to review ([DeepWiki: tasks command](https://deepwiki.com/github/spec-kit/5.3-tasks-command), [agents](https://code.claude.com/docs/en/agents)).

**Scope creep, eager building, and a brittle reduce step.** Agents are "eager to build adjacent features" and scope-creep past their slice unless acceptance criteria are tight; each should report assigned scope, files changed, commands run, test results, and unresolved risks before merge. Keep a checklist for **when NOT to parallelize**: task under ~5 minutes, unknown root cause, architecture redesign, work needing tight sequencing, work you can't split cleanly by directory/layer/test-surface, or no final validation gate — if three or more are true, defer parallelization ([parallel Claude Code agents](https://www.aakashx.com/blog/parallel-claude-code-agents/), [10 parallel agents, week 1](https://findskill.ai/blog/claude-code-10-parallel-agents-week-1/)).

**Tooling and resource friction.** Worktrees can balloon disk usage (Upsun cites Cursor-forum users reporting a ~2GB codebase consuming 9.82GB of worktrees in a 20-minute session — an anecdote, not a benchmark); concurrent installs over a shared symlinked `node_modules` can corrupt it; Claude Code's `/ide` command has been reported to fail to recognize a worktree ("No available IDEs detected"); and worktrees don't fix LLM context degradation in long sessions ([git worktrees for parallel AI coding agents](https://developer.upsun.com/posts/ai/git-worktrees-for-parallel-ai-coding-agents), [git worktrees for parallel AI agents](https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents)).

**Parallelism is not free — and the impressive numbers are misused.** "Running ten agents in parallel uses quota roughly ten times as fast as running one" — there is no shared-context discount, so cost scales roughly linearly with agent count (a community report, treat the multiplier as directional) ([10 parallel agents, week 1](https://findskill.ai/blog/claude-code-10-parallel-agents-week-1/), [parallel Claude Code agents](https://www.aakashx.com/blog/parallel-claude-code-agents/)). Reported token multipliers disagree and come from non-feature contexts: ~15× vs single chats in one Anthropic piece, 3–10× vs single-agent in another — both measured on research/agentic workloads, not feature builds ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system), [building multi-agent systems: when and how](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)). Crucially, Anthropic's headline result — a multi-agent system beating single-agent Opus 4 by **90.2%** — comes from their **research** system, and the *same article argues coding is less parallelizable than research*. Citing 90.2% to justify parallelizing a coding feature inverts the source's own argument; use it only to make the cost point, not a speed promise ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)).

**The decisive guardrail across all of this:** decompose by context boundaries, not problem type. Anthropic's rule is explicit — "Decompose by context boundaries, not problem type." Splitting one feature into plan → code → test → review handed between agents is the "telephone game" antipattern, where each handoff loses fidelity. Reserve parallel agents for separate components with well-defined interfaces; keep tightly-coupled shared-state work and sequential phases in one session ([building multi-agent systems: when and how](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)).

---

## Sources

### Anthropic official — Claude Code docs
- https://code.claude.com/docs/en/best-practices
- https://code.claude.com/docs/en/common-workflows
- https://code.claude.com/docs/en/worktrees
- https://code.claude.com/docs/en/agents

### Anthropic official — engineering & multi-agent
- https://www.anthropic.com/engineering/multi-agent-research-system
- https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them
- https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- https://www.anthropic.com/engineering/building-c-compiler

### GitHub Spec Kit — task decomposition & [P] markers
- https://raw.githubusercontent.com/github/spec-kit/main/templates/commands/tasks.md
- https://github.com/github/spec-kit/blob/main/templates/commands/tasks.md
- https://deepwiki.com/github/spec-kit/5.3-tasks-command

### Worktrees, file ownership & reintegration (practitioner)
- https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees
- https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents
- https://www.mindstudio.ai/blog/parallel-agentic-development-claude-code-worktrees
- https://developer.upsun.com/posts/ai/git-worktrees-for-parallel-ai-coding-agents
- https://dev.to/battyterm/how-i-run-a-team-of-ai-coding-agents-in-parallel-p7c
- https://docs.agentinterviews.com/blog/parallel-ai-coding-with-gitworktrees/

### Clean seams, contracts & agent teams
- https://medium.com/codetodeploy/api-first-development-why-your-next-project-should-start-with-the-contract-not-the-code-5a5a25828b4b
- https://ibagroupit.com/insights/modernize-websphere-applications/
- https://www.contextstudios.ai/blog/claude-code-agent-teams-a-builders-guide-to-parallel-ai-coding
- https://www.mindstudio.ai/blog/claude-code-agent-teams-parallel-shared-task-list

### Task sizing
- https://zachwills.net/how-to-use-claude-code-subagents-to-parallelize-development/

### Vertical vs horizontal slicing
- https://www.datascience-pm.com/vertical-vs-horizontal-slicing-data-science-deliverables/
- https://gregpark.io/blog/building-new-features-horizontal-or-vertical-slices

### Failure modes & cost (practitioner)
- https://www.aakashx.com/blog/parallel-claude-code-agents/
- https://findskill.ai/blog/claude-code-10-parallel-agents-week-1/

File written to: /Users/slonina/repo/DirectionsQ3PlanningAIEvent/presentations/AI feature refinement/research/breaking-a-feature-into-parallel-tasks.md
