---
marp: true
theme: default
paginate: true
title: Refining features with Claude
author: Michał Słonina
---

<!-- _paginate: false -->

# Refining features with Claude

### Taking a backlog feature from *"sounds useful"* to *shippable*

AI Engineering Mini Conference — Amsterdam · June 16–17, 2026
AI feature refinement track · Michał Słonina

<!--
Speaker notes: this is the track intro talk. Goal — give people a default loop they can
run on their own feature today, and the Claude techniques that fit each step.
-->

---

## The problem with refinement

- Backlog items are **vague**: a title, maybe a sentence.
- Turning that into a plan eats senior time — research, scoping, edge cases.
- It's the work that's *easy to skip* and expensive to get wrong.

> Claude is good at exactly this: reading context fast, surfacing unknowns, drafting a plan you can react to.

---

## The loop

**Pick → Research → Plan → Build → Verify → Share**

- Claude does the legwork; **you stay the decision-maker**.
- Each step has techniques that fit it — covered in the lightning talks.
- It's a *default rhythm*, not a rule. Adapt it to your feature.

<!-- Mirrors the "Suggested Execution Plan" in ai_event_plan.md. Keep these in sync. -->

---

## 1. Pick

Choose a feature that is:

- **Valuable** — worth shipping, not a toy.
- **Researchable** — Claude can read its way in.
- **Parallelizable** — work can split across the team.
- **Technique-rich** — exercises what we're here to learn.
- **Safely scoped** — off the critical path.

---

## 2. Research

- Point Claude at the codebase, the ticket, the docs — let it **map the terrain**.
- Ask for *unknowns and risks*, not just a summary.
- Make it cite files and lines so you can verify.

```
"Read this module and the linked ticket. What would this feature touch?
 List the unknowns and the riskiest assumptions, with file:line references."
```

---

## 3. Plan

- Have Claude draft an **execution plan**: steps, order, what's parallel.
- Capture it in a `CLAUDE.md` / plan file so the build phase shares context.
- React to it — *cut, reorder, sharpen*. The plan is a conversation.

---

## 4. Build

- Small, reviewable changes — keep Claude on a short leash.
- Lean on the techniques from the lightning talks (hooks, spec-driven, subagents…).
- Commit often; let the plan file track what's done.

---

## 5. Verify

- Claude writes tests and runs them — **but you read the failures.**
- Make it prove the feature works, not just that it compiles.
- Distinguish *"Claude says done"* from *"verified done"*.

---

## 6. Share

Round-the-room before we wrap. In a sentence or two:

- What shipped.
- One thing that **worked**, one that **broke**.
- Where Claude led vs. where **you** had to.
- Any `CLAUDE.md` templates worth taking home.

---

## What works / what to watch

| Works | Watch |
|---|---|
| Fast context-loading & research | Confident-but-wrong claims |
| Drafting plans to react to | Over-long, unreviewable changes |
| Grinding tests & edge cases | "Done" ≠ verified |
| Parallel exploration | Losing the thread of *why* |

---

<!-- _paginate: false -->

# Pick a feature. Run the loop.

**Refinement is where Claude earns its keep — if you stay in the driver's seat.**

Track rhythm & candidate features → see the event page.
