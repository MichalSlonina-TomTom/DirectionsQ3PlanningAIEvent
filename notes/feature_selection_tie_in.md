# Feature Selection — criteria and the lightning-talk tie-in (internal notes)

Internal organizer notes for the Day 1 "pick a feature" block (10:30) and the design/labs flow.
Not for the audience-facing Confluence page — this explains *why* the selection criteria are
shaped the way they are, so the committee and facilitators can guide teams.

## Selection criteria (a candidate feature should clear all five)

1. **Worth doing — real business value.** Straight from the backlog; something the team actually
   wants shipped. No toy problems — the point is to test the workflow on work that matters.
2. **Researchable — there's something to understand first.** Forces digging into the codebase,
   domain, or existing data before writing code. If you already know exactly how to build it,
   there's nothing to learn.
3. **Parallelizable — splits into ~5–7 independent tasks.** Clear seams (modules, endpoints,
   tests, docs, migrations) so the whole team works concurrently, not one driver + six spectators.
4. **Exercises the techniques — uses what the talks just showed.** Gives a concrete reason to apply
   CLAUDE.md/hooks, spec-driven dev, AI review, workflows, MCP, etc.
5. **Safe and right-sized — scoped to the time, off the critical path.** Meaningfully advanceable in
   ~4h total (2.5h Day 1 + 1.5h Day 2) and not on the prod-critical path. Nothing has to ship, so
   teams are free to let Claude run, try the bold approach, and learn from what breaks.

## The tie-in

The criteria aren't arbitrary. Each one exists so that a feature clearing all five will, *by
construction*, force teams to use the techniques they watched in the morning lightning talks. The
morning then reads as one arc:

> watch the techniques (talks) → pick a feature that demands them (criteria) →
> research it (design block) → split it (work breakdown) → build it (labs).

| Criterion | Lightning-talk topics it sets up | Why |
|-----------|----------------------------------|-----|
| 1. Business value | D1, D2, D3 | Choosing *what's worth building* is human judgment — the "where humans still lead" goal. The team decides; the tool executes. |
| 2. Researchable | B1 (mine Slack/JIRA/Confluence), B2 (architecture docs), A4 (prompt patterns for large codebases) | The 10:50 design block *is* the research phase — these are exactly the techniques teams lean on to understand the problem before coding. |
| 3. Parallelizable | C3 (multi-agent workflows), C4 (worktrees), A3 (workflows) | Independent tasks are what let teams fan out agents / run worktrees in the afternoon. A non-parallel feature wastes both the team and these tools. |
| 4. Exercises techniques | B3 (spec-driven dev), B4 (AI code review), A1 (CLAUDE.md/hooks), B5 (CI/CD) | The build itself — turn the design into a spec, let Claude implement, review the output, wire up gates. |
| 5. Safe & right-sized | A5 (sandbox mode & permissions) | Off the critical path means teams can run permissively and let Claude move fast without fear — A5 is how they set that up safely. |

Goal letters refer to the lightning-talk goal groups on the event page (A = lower the entry barrier,
B = make our work more efficient, C = level up tooling, D = where humans still lead).
