# Rovo Prompts — Create the Confluence page

These are prompts to paste into **Rovo** to create and refine the event page in Confluence.
This is meant to be a **live, audience-facing page** (not the internal organizer notes), so it
deliberately leaves out keynote talking points, the slide-curation process, and other prep-only items.

Before running prompt 1, fill in the two placeholders: **[SPACE]** (e.g. "PU Directions" / space key)
and **[PARENT PAGE]** (the page it should live under, or "top level of the space").

---

## Prompt 1 — Create the page

> Create a new Confluence page in the **[SPACE]** space, under **[PARENT PAGE]**.
> Title: **AI Engineering Session — Amsterdam, June 16–17, 2026**.
> Make it a clean, scannable, audience-facing page for engineers. Use this exact content and structure:
>
> **Top banner (use an info/success panel so it stands out):**
> 🚀 Interested? Join us! → link the text **#directions-ai-session-june-16-ams** to
> https://tomtomslack.slack.com/archives/C0B8EPN0YB0
> Subtitle: "Keynote speakers · lightning talk presenters · AI champions · AI evangelists — all teams welcome!"
>
> **Intro line:** TomTom · Amsterdam · June 16–17, 2026 · 1.5 days · ~30 engineers + 6 AI champions + 6 AI evangelists.
>
> **Section "Team Structure":** 6 teams × ~7 people each — ~5 TomTom engineers, 1 AI champion (TomTom
> engineer experienced with Claude, embeds in the team all day), 1 AI evangelist (external engineer,
> brings an outside perspective, embeds all day).
>
> **Section "Day 1 — Tuesday, June 16, 2026":** render as a table with columns Time / Duration / Block:
> - 09:00 / 30 min / Keynote
> - 09:30 / 48 min / Lightning talks: 3 AI champions + 3 AI evangelists alternating (5 min + 3 min Q&A each)
> - 10:18 / 12 min / Break — teams form with their champion + evangelist
> - 10:30 / 20 min / Teams pick a feature to implement
> - 10:50 / 30 min / Use Claude to design the feature: architecture, edge cases, open questions
> - 11:20 / 30 min / Work breakdown: split into per-person tasks for the afternoon
> - 11:50 / 25 min / Teams present their plan to the room (3–4 min each)
> - 12:15 / 60 min / Lunch
> - 13:15 / 150 min / Hands-on labs — implement the feature (champion + evangelist embedded)
> - 15:45 / 15 min / Break
> - 16:00 / 60 min / Team presentations: results + Day 2 work breakdown per person
> - 17:00 / — / End of Day 1 — sign-up opens for Day 2 optional lightning talks
>
> **Subsection "Lightning Talks":** Add a note panel: "We'll accommodate every lightning talk speaker —
> if you want to present, you'll get a slot across Day 1 and Day 2. The running order is determined by our
> AI champions." Then a line: "Topics are grouped by goal. AI champions pick 6 for the Day 1 scheduled
> block and place the rest across Day 1 overflow and Day 2." Then four sub-headings, each with a
> two-column table (Topic / Owner). Set every Owner cell to a yellow/neutral **TBD** status lozenge.
>
> - **Goal 1 — Lower the entry barrier (get productive fast):**
>   - CLAUDE.md and Hooks in a Real Repo — setup that pays off across every session
>   - TomTom Marketplaces and Skills — what's available and how to use them
>   - Building Workflows for Daily Use — practical automation you can ship this week
>   - Prompt Patterns for Large Codebases — what works at TomTom's scale
> - **Goal 2 — Make our work more efficient (hands-on, real tasks):**
>   - Mining Slack, JIRA & Confluence with Claude — turn scattered data into answers
>   - Data Mining in Practice — generating architecture docs and PR verification
>   - Spec-Driven Development — from a written spec to working code with Claude
>   - AI-Assisted Code Review — integrating Claude into your PR workflow
>   - Claude in CI/CD — automated test generation, linting, and review gates
> - **Goal 3 — Level up your tooling (keep pace with the tools):**
>   - MCP Servers at TomTom — internal servers, how to connect and extend them
>   - How to Build and Evaluate Skills — authoring, testing, and measuring quality
>   - Multi-Agent Workflows in Production — when to fan out and how to control it
>   - Using Worktrees Efficiently — and where worktrees + sandboxing break down
>   - Using the Anthropic API to Build Internal Tools — from script to product
> - **Goal 4 — Where humans still lead (perspective & experience):**
>   - Overview of AI Projects at TomTom — what teams are building right now
>   - How We Use AI — usage patterns and insights Claude surfaces about our own engineering behaviour
>   - Lessons from the Field — what failed, what we fixed, what we'd do differently
>
> **Subsection "End of Day 1 — Team Presentations (16:00–17:00)":** Each team presents:
> 1) what they built that afternoon (show actual output), 2) one thing that worked and one that broke,
> 3) the Day 2 plan with specific tasks named per person. Note that the facilitator opens sign-up for
> Day 2 optional lightning talks at the close.
>
> **Section "Day 2 — Wednesday, June 17, 2026":** table with Time / Duration / Block:
> - 09:00 / 48 min / Optional lightning talks: sign-up based, anyone can present (5 min + 3 min Q&A, up to 6 slots)
> - 09:48 / 12 min / Break
> - 10:00 / 90 min / Execute Day 2 plans (champion + evangelist stay embedded)
> - 11:30 / 15 min / Break
> - 11:45 / 45 min / Final share-out: what shipped + takeaways + CLAUDE.md templates to take home
> - 12:30 / — / End
>
> Add a short list of Day 2 optional lightning talk ideas: "Here's what our team actually built yesterday",
> "I hit this problem and solved it this way", "Here's a prompt pattern that surprised me",
> "What I'm doing differently today based on yesterday".
>
> **Section "Roles":** table Role / Responsibility — Facilitator (runs keynote, timekeeping, plenary
> sessions); AI Champion ×6 (presents a lightning talk, embeds in a team, unblocks during labs, seeds Day 2
> sign-up); AI Evangelist ×6 (presents a lightning talk, embeds in a team, brings external perspective);
> Timekeeper (enforces the 3 min Q&A limit with a visible countdown).
>
> **Section "Attending & Presenting Remotely" (use an info panel):** Lightning sessions are streamed live
> online — you don't have to be in Amsterdam to attend. You can present remotely over video. All sessions
> are recorded and shared afterwards.
>
> **Section "Call for Participation":** Lead line: "We're looking for people to fill these roles. Engineers
> from any team are welcome — if any of these sound like you, let us know and we'll check the budget."
> Then four sub-headings:
> - **Keynote Speaker** — open the event with a vision-setting talk on AI in engineering at TomTom.
> - **Lightning Talk Presenter** — share a topic or claim one from the list above; 5-minute slot + 3 min
>   Q&A, in person or remote. Everyone who wants to present will be accommodated; AI champions set the order.
> - **AI Champion** — your team's designated AI Engineering Champion under the Directions AI Engineering
>   Championship program; learns fast, coaches colleagues, and drives hands-on adoption. At the event:
>   embed in a team for the day, help others get productive with Claude, unblock during labs, present a
>   lightning talk.
> - **AI Evangelist** — someone genuinely enthusiastic about AI with some practical experience in
>   AI-assisted engineering; brings an outside perspective and shares what they've learned. No deep
>   expertise required — just real curiosity and a few miles on the clock.
>
> Close with a call to action: "Interested in any of these? Let us know which role and we'll follow up on
> availability and budget." Keep the tone energetic and welcoming.

---

## Prompt 2 — Link the Championship program (run after that page exists)

> On the AI Engineering Session page, in the **AI Champion** description, link the phrase
> "Directions AI Engineering Championship program" to the Confluence page for that program.
> If that page doesn't exist yet, leave the phrase as bold text instead of a link.

---

## Prompt 3 — Optional polish

> Review the AI Engineering Session page for consistency and readability: make sure the two agenda tables
> are aligned, the four lightning-talk goal sections look uniform, and all Owner cells show a TBD status
> lozenge. Add a short table of contents at the top, just under the banner. Don't change any of the wording,
> dates, or the Slack link.

---

## Notes for the operator (not for Rovo)

- Confirm the **space** and **parent page** before prompt 1.
- This page is the public/live version. Keep the internal organizer items (keynote talking points,
  slide template & curation flow, the Alex sync, PU Directions efficiency initiatives for the labs) out
  of it — those live in your private notes.
- Source of truth for content: `ai_event_plan.md` in this repo. If you edit the plan later, re-derive
  the relevant prompt section rather than hand-editing the page, so the two stay in sync.
