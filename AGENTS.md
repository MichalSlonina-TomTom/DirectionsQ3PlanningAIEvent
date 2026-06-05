# AGENTS.md

## Confluence sync (required on every change)

This repo is the source of truth for the **AI Engineering Mini Conference** event and must be kept in
sync with its Confluence page.

- **Confluence page:** AI Engineering Mini Conference — Amsterdam, June 16–17, 2026
- **Page ID:** `2045706280`
- **Space:** PU Directions (`DIRECTIONS`, space ID `233897990`)
- **cloudId:** `tomtom.atlassian.net`
- **URL:** https://tomtom.atlassian.net/wiki/spaces/DIRECTIONS/pages/2045706280
- **Maps to local file:** `ai_event_plan.md` (the public/audience-facing content)

### Workflow — follow this order on every change

1. **Pull Confluence first.** Read the current page (`getConfluencePage`, `contentFormat: html`)
   and diff it against `ai_event_plan.md` to catch edits people made directly on the page.
2. **Update local files.** Bring any Confluence-side changes back into `ai_event_plan.md` (and
   other repo files if relevant) so the repo reflects reality.
3. **Reconcile both directions.** If the repo also has changes the page lacks, apply them to the
   page (`updateConfluencePage`, `contentFormat: html`). After this step the page and the local
   files must match.
4. **Commit and push** the local changes — every time, no exceptions.

### Scope notes

- Only the **public/audience-facing** content syncs to Confluence. Keep internal-only material out
  of the page: keynote talking points, slide template & curation flow, the Alex sync, and the
  PU Directions efficiency initiatives for the labs. Those live in `notes/` and are not published.
- `notes/` files (e.g. `20260605_boris_notes.txt`, `rovo_confluence_prompt.md`) and
  `ai_championship_program.md` are repo-only unless explicitly asked to publish them.
- If a sync would change wording, dates, or the Slack link on the page, surface it before
  overwriting rather than applying silently.
