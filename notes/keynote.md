# Keynote — Talking Points (internal)

Internal-only. Not published to Confluence (see AGENTS.md).

## Topics

### Optimizing AI cost
- Cost is a first-class engineering concern now — token spend is a real budget line, not free.
- Right-size the model: default to the cheapest model that clears the bar; reserve the top tier for hard reasoning.
- Lean on prompt caching — repeated context (CLAUDE.md, large files, system prompts) should hit cache, not be re-billed every turn.
- Scope context tightly: read the part of the file you need, not the whole tree. More tokens in ≠ better answers.
- Fan out deliberately — multi-agent and worktrees multiply spend; use them when the task is worth it, not by default.
- Measure it: track tokens/cost per task so the team can see what a given workflow actually costs and tune it.
- Frame for the room: getting more efficient with AI *and* with AI's own cost is what makes the investment defensible.
