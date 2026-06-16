# AGENTS.md — "Refining features with Claude" presentation

Agent instructions for building and maintaining this talk. Scope is **this directory only**;
the repo-wide rules in `../../AGENTS.md` (Confluence sync) do **not** apply here — this is
internal slide material and is not published to Confluence.

## What this is

A lightning/track talk for the **AI feature refinement track** at the AI Engineering Mini
Conference (Amsterdam, June 16–17, 2026), owned by Michał. Topic: how to take a real backlog
feature and refine it into shippable work *with Claude* — the loop, the techniques, what works
and what to watch for. See `../../pitches/guidance_planning.md` and the "Suggested Execution
Plan" in `../../ai_event_plan.md` for how the track runs; keep this talk consistent with that.

## Layout

- `presentation/slides.md` — the deck. Marp Markdown (`marp: true` frontmatter). One source of truth.
- `presentation/` — also holds any images/assets the deck references.
- `research/` — notes, sources, and supporting material that feed the slides. Not shown on screen.
- `Makefile` — builds the PDF (and other formats) from `presentation/slides.md`.

## Tooling

- Slides are **Marp** Markdown, rendered with `marp-cli` (installed via Homebrew: `marp`).
- `make` / `make pdf` → `presentation/slides.pdf`. `make html`, `make pptx`, `make watch` also exist.
- PDF export drives a headless Chromium; if it can't find a browser, set `CHROME_PATH` (see Makefile).

## Working rules for agents

1. **Edit `slides.md`, never the generated artifacts.** `*.pdf`, `*.html`, `*.pptx` are build output.
2. **Keep slides skimmable.** One idea per slide, short bullets, speaker detail goes in
   presenter notes (`<!-- comments -->`) or in `research/`, not in dense body text.
3. **Ground claims in `research/`.** Before adding a technique or a number to a slide, leave the
   supporting note/source under `research/` so the deck stays defensible.
4. **Match the track.** Techniques shown should map to the track's phases (Pick → Research →
   Plan → Build → Verify → Share). Don't invent a workflow that contradicts the event plan.
5. **Rebuild before declaring done.** Run `make` and confirm the PDF generates without errors.
6. **No secrets in slides or research** — follow the repo's secrets-outside-repo rule.
