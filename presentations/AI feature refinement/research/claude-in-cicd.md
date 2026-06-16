# Claude in CI/CD

<!-- Research report for the AI feature-refinement talk. Companion to refining-features-with-claude.md (the Share phase). Every URL below appears in the research data; nothing borrowed from other docs. -->

## Overview: how teams run Claude in CI/CD

Two distinct integration paths exist, and teams choose between them based on how much control they need over execution, data, and billing ([github-actions](https://code.claude.com/docs/en/github-actions), [hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)):

1. **Managed / GitHub-hosted.** Install the official Claude GitHub App and use `anthropics/claude-code-action@v1`. Claude runs inside a GitHub-hosted runner, triggered by an `@claude` mention in an issue/PR comment, or by any GitHub event when you pass a `prompt`. The Action is a thin wrapper around the same headless execution.
2. **Self-hosted / headless.** Invoke `claude -p` directly on any runner (GitLab CI, Jenkins, CircleCI, Bitbucket, etc.), inject the credential as a masked secret env var, and parse the JSON output yourself. There is no Marketplace action for non-GitHub systems — you wire it together.

The unifying truth is that **any runner that can execute a shell command can run Claude via the headless `claude -p` CLI**; the differences are in triggers, auth, permission posture, and platform-specific plumbing ([gitlab-ci-cd](https://code.claude.com/docs/en/gitlab-ci-cd), [mehdi.cz](https://www.mehdi.cz/blog/claude-code-ci-automation), [circleci](https://circleci.com/blog/getting-started-with-claude-code-and-circleci)).

A separate, *managed* product — Anthropic's hosted Code Review — runs multi-agent analysis on Anthropic infrastructure rather than in your CI. It is a research preview, Team/Enterprise only, and not available with Zero Data Retention. Keep it mentally separate from the self-hosted Action; their cost and behavior differ sharply ([code-review](https://code.claude.com/docs/en/code-review)).

The recurring framing across sources: CI/CD was built for *deterministic* automation, while an LLM agent is *probabilistic* and treats natural-language input as instruction. That mismatch is the root of most failure modes in the later sections ([refactix](https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production)).

---

## Setup options

### Headless / print mode (`claude -p`) — the universal building block

`claude -p "<prompt>"` (alias `--print`) is the non-interactive entry point: Claude reads the prompt (as an arg or piped on stdin), runs the full agent loop to completion, prints the result, and exits — no REPL, no follow-up turn. All regular CLI options work with `-p` ([headless](https://code.claude.com/docs/en/headless)).

Key flags for CI use:

- **`--output-format json`** returns a structured payload with a `result` field (final text), `session_id`, and `total_cost_usd` plus a per-model cost breakdown. Parse it with `jq -r '.result'` instead of grepping human-readable text, which breaks when phrasing shifts. Add `--json-schema '<JSON Schema>'` for schema-constrained results (lands in a separate `structured_output` field). `--output-format stream-json --verbose` emits newline-delimited JSON events for live logs/audit ([headless](https://code.claude.com/docs/en/headless)).
- **`--bare`** skips auto-discovery of hooks, skills, plugins, MCP servers, auto-memory, and `CLAUDE.md`, so scripted runs are deterministic across machines. It also skips OAuth/keychain reads — auth must come from `ANTHROPIC_API_KEY` or an `apiKeyHelper` in `--settings`. Anthropic says `--bare` is recommended for scripts and will become the `-p` default in a future release ([headless](https://code.claude.com/docs/en/headless)).
- **Pipe in, redirect out** like any Unix tool: `git diff main | claude -p "you are a typo linter…"`. Piping the diff means Claude needs no Bash permission to read it. Caveat: as of v2.1.128, piped stdin is capped at 10MB — exceed it and Claude Code exits non-zero with a clear error; for bigger inputs, write to a file and reference its path ([headless](https://code.claude.com/docs/en/headless)).
- **Multi-stage runs:** capture `session_id=$(claude -p "stage 1" --output-format json | jq -r '.session_id')`, then `claude -p "stage 2" --resume "$session_id"` (run both from the same project dir — session lookup is directory-scoped), or `--continue` for the most recent conversation. Make pipeline tasks idempotent so a retry yields the same final state ([headless](https://code.claude.com/docs/en/headless), [hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).

> **SDK note / gap:** the research data does not contain a standalone Agent SDK setup recipe. The SDK appears only as the layer the GitHub Action wraps, and as a separate billing category (see Cost). Treat "SDK" and "headless `claude -p`" as the same execution surface for setup purposes here.

### GitHub Action / `@claude`

Setup is one command: run `/install-github-app` in a Claude Code terminal session. It installs the official Claude GitHub app, requests Read & write on Contents, Issues, and Pull requests, and adds the `ANTHROPIC_API_KEY` repo secret. You must be a repo admin. Manual fallback: install the app, add the secret, copy `examples/claude.yml` into `.github/workflows/` ([github-actions](https://code.claude.com/docs/en/github-actions)).

**One action, two modes, auto-detected.** With no `prompt` and a comment event, the Action runs interactively and responds to `@claude` mentions; with a `prompt` input it runs automation immediately. GA v1 removed the old `mode: tag|agent` input — you no longer choose by hand ([github-actions](https://code.claude.com/docs/en/github-actions), [claude-code-action](https://github.com/anthropics/claude-code-action)).

The `@claude` wiring is a literal string check, not magic: the example workflow triggers on `issue_comment`, `pull_request_review_comment`, `pull_request_review`, and `issues`, gated by an `if` like `contains(github.event.comment.body, '@claude')`. It must be `@claude`, not `/claude`. Override with the `trigger_phrase` input or trigger on assignment via `assignee_trigger` ([claude.yml](https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml), [github-actions](https://code.claude.com/docs/en/github-actions)).

**v1 breaking changes to migrate** ([github-actions](https://code.claude.com/docs/en/github-actions)): `@beta` → `@v1`; remove `mode`; `direct_prompt` → `prompt`; and `max_turns`/`model`/`custom_instructions`/`allowed_tools` all move into a single `claude_args` passthrough that accepts any CLI flag, e.g. `claude_args: "--max-turns 5 --model <model> --allowedTools \"Bash(gh issue view:*)\""`. They map to `--max-turns` / `--model` / `--append-system-prompt` / `--allowedTools`; `--max-turns` defaults to 10.

**`CLAUDE.md` at the repo root is the persistent control surface** — code style, review criteria, hard constraints (don't touch `/vendor/`, no direct commits to protected branches, require tests for new functions). It applies to both the `@claude` bot and automated reviews, keeping behavior consistent without re-prompting ([github-actions](https://code.claude.com/docs/en/github-actions), [groundy](https://groundy.com/articles/how-to-run-claude-code-as-a-github-actions-agent-for-automated-pr-fixes/)).

**Enterprise / self-hosted LLM.** The Action runs against Amazon Bedrock (`use_bedrock: true`, model id needs a region prefix like `us.anthropic.claude-sonnet-4-6`) or Google Vertex AI (`use_vertex: true`, `@date` suffix form like `claude-sonnet-4-5@20250929`) using GitHub OIDC / Workload Identity Federation instead of static keys. The workflow still needs `id-token: write` plus a GitHub App token (`actions/create-github-app-token`) so Claude's commits can re-trigger CI ([github-actions](https://code.claude.com/docs/en/github-actions)). *Caveat (single-source, practitioner):* under federation, inline-comment classification needs an API key and is skipped ([groundy](https://groundy.com/articles/how-to-run-claude-code-as-a-github-actions-agent-for-automated-pr-fixes/)).

### Other CI

- **GitLab CI** is the most turnkey non-GitHub path: add one job to `.gitlab-ci.yml` and a masked `ANTHROPIC_API_KEY` variable. The official job uses `image: node:24-alpine3.21`, installs Claude, then runs `claude -p "${AI_FLOW_INPUT}" --permission-mode acceptEdits --allowedTools "Bash Read Edit Write mcp__gitlab" --debug`. Triggers via `rules` on `$CI_PIPELINE_SOURCE == "merge_request_event"`/`"web"`, or a webhook listener. Keyless auth via OIDC→AWS STS (Bedrock) or Workload Identity Federation (Vertex). **Caveat: this integration is BETA and maintained by GitLab, not Anthropic** ([gitlab-ci-cd](https://code.claude.com/docs/en/gitlab-ci-cd)).
- **Jenkins** has a community "AI Agent" plugin adding a reusable `aiAgent(...)` build step that resolves the API key from Jenkins credentials, supports approval gates, and live-streams the conversation with per-invocation token/cost/duration stats. Requires Jenkins 2.528.3+ and Node.js on agents. *Third-party community plugin, not an Anthropic product; no documented adoption stats* ([jenkins ai-agent](https://plugins.jenkins.io/ai-agent/)).
- **Bitbucket** integration is done through MCP, not a native action: the Bitbucket Server MCP server is bundled into the agent container, and Claude is restricted with `--allowedTools "mcp__bitbucket__*"` ([sayan.nandi](https://medium.com/@sayan.nandi/building-an-ai-powered-automated-code-reviewer-with-claude-code-jenkins-and-bitbucket-b0e600d27b25)).
- **CircleCI** has no dedicated in-pipeline integration. Its official tutorial covers the *reverse* direction — running Claude locally and connecting to CircleCI via the `@circleci/mcp-server-circleci` MCP server. To run Claude *in* a CircleCI job, fall back to the generic shell pattern: install the CLI, set `ANTHROPIC_API_KEY` from a context, run `claude -p … --max-turns N --output-format json`. *No CircleCI-specific in-pipeline recipe exists in primary sources* ([circleci](https://circleci.com/blog/getting-started-with-claude-code-and-circleci)).
- **Self-hosted GitLab fork:** `RealMikeChong/claude-code-for-gitlab` ports the GitHub-Actions behavior to GitLab and self-hosted instances with a Docker webhook service. *Unofficial community fork; explicitly states GitHub users should not use it; cannot approve MRs by design* ([RealMikeChong](https://github.com/RealMikeChong/claude-code-for-gitlab)).

---

## Automated review & auto-fix

### Automated PR review (self-hosted Action)

Trigger on `pull_request: types: [opened, synchronize]` and pass a `prompt` — v1 auto-detects automation mode on non-comment events ([github-actions](https://code.claude.com/docs/en/github-actions), [solutions.md](https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md)). The docs' recommended pattern installs the official code-review plugin and invokes a skill, e.g. `prompt: '/code-review:code-review ${{ github.repository }}/pull/${{ github.event.pull_request.number }}'` ([github-actions](https://code.claude.com/docs/en/github-actions)).

- **Required permissions:** `contents: read`, `pull-requests: write`, `id-token: write`. Missing `pull-requests:write` blocks comment posting; missing `id-token:write` blocks auth — insufficient perms tend to fail silently. Checkout with `fetch-depth: 1` so Claude can read surrounding context ([solutions.md](https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md)).
- **Inline (line-specific) comments** require allowlisting the MCP tool: `--allowedTools "mcp__github_inline_comment__create_inline_comment,Bash(gh pr comment:*)"`. Without it, Claude falls back to a single top-level comment. Inline comments support `confirmed: true` to post immediately; omit it and comments are buffered and classified after the session so subagent probe comments don't leak (`classify_inline_comments: 'false'` disables this) ([solutions.md](https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md), [startdebugging](https://startdebugging.net/2026/05/how-to-run-claude-code-in-a-github-action-for-autonomous-pr-review/)).
- **Calibrate via `CLAUDE.md`:** define Important (production breakage, data leaks) vs nits (style/naming), exclude what CI already enforces (lint, generated code, lockfiles), and cap nit volume (e.g. "at most five nits per review") to avoid reviewer fatigue ([solutions.md](https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md), [startdebugging](https://startdebugging.net/2026/05/how-to-run-claude-code-in-a-github-action-for-autonomous-pr-review/)).
- **Local pre-CI shortcut:** run `/code-review` in a Claude Code session before opening a PR; `--comment` posts inline, `--fix` applies to the working tree, and effort level trades coverage for confidence ([code-review](https://code.claude.com/docs/en/code-review)).

A real GitLab CI review job saves output as Markdown + PDF artifacts and does **not** auto-post to the MR ([jinmiaoluo](https://blog.jinmiaoluo.com/en/posts/gitlab-ci-claude-code-review/)).

### Managed Code Review (contrast — different mechanism)

Admin enables it once for the org; multi-agent analysis runs on Anthropic infrastructure and posts inline comments tagged Important / Nit / Pre-existing, with a verification step that checks each candidate against actual code behavior to cut false positives. Per-repo trigger modes: once on PR open, every push, or manual via `@claude review` / `@claude review once`. Tune via `REVIEW.md` (highest-priority instructions) or `CLAUDE.md` (violations flagged as nits) ([code-review](https://code.claude.com/docs/en/code-review)).

**Critical gating gotcha:** the Code Review check run *always* completes NEUTRAL, so it never blocks a merge via branch protection. To gate merges you must parse the machine-readable severity counts yourself, e.g. `gh api repos/OWNER/REPO/check-runs/CHECK_RUN_ID --jq '…'` returning `{"normal":2,"nit":1,"pre_existing":0}` (`normal` = Important). Failed/timed-out reviews don't auto-retry; GitHub's Re-run button does not retrigger it — use `@claude review once`. Replying to a comment does nothing; to act, fix code and push ([code-review](https://code.claude.com/docs/en/code-review)).

### Auto-fix / "make it green"

Anthropic ships an official ready-to-use workflow, `examples/ci-failure-auto-fix.yml`: it triggers on `workflow_run` for the CI workflow with `types: [completed]` and condition `github.event.workflow_run.conclusion == 'failure'`, then invokes a `/fix-ci` command with the failed run URL, job names, PR number, branch, and error logs ([examples](https://github.com/anthropics/claude-code-action/tree/main/examples), [ci-failure-auto-fix.yml](https://raw.githubusercontent.com/anthropics/claude-code-action/main/examples/ci-failure-auto-fix.yml)).

- **The critical loop guard:** the workflow names fix branches with a `claude-auto-fix-ci-` prefix and gates the job on `!startsWith(github.event.workflow_run.head_branch, 'claude-auto-fix-ci-')`. Without it, every fix commit re-triggers CI, which re-triggers the fixer — an infinite loop. *Caveat: the official example itself does NOT set `--max-turns`, so add your own cap if cost/runaway is a concern* ([ci-failure-auto-fix.yml](https://raw.githubusercontent.com/anthropics/claude-code-action/main/examples/ci-failure-auto-fix.yml)).
- **Tight tool scope:** the example restricts Claude to `Edit,MultiEdit,Write,Read,Glob,Grep,LS,Bash(git:*),Bash(bun:*),Bash(npm:*),Bash(npx:*),Bash(gh:*)`. Least-privilege tooling doubles as a cost control by reducing unnecessary turns ([ci-failure-auto-fix.yml](https://raw.githubusercontent.com/anthropics/claude-code-action/main/examples/ci-failure-auto-fix.yml), [github-actions](https://code.claude.com/docs/en/github-actions)).
- **Required permissions:** `contents: write`, `pull-requests: write`, `issues: write`, `actions: read` (the example comments that `actions:read` is "Required for Claude to read CI results on PRs" — without it Claude can't see why your build failed), and `id-token: write` ([claude.yml](https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml), [ci-failure-auto-fix.yml](https://raw.githubusercontent.com/anthropics/claude-code-action/main/examples/ci-failure-auto-fix.yml)).
- **The #1 CI gotcha:** commits Claude makes don't re-trigger downstream CI when authored by the default `GITHUB_TOKEN`/Actions user. Fix: authenticate via the Claude GitHub App, or generate a token with `actions/create-github-app-token@v2` and pass it as `github_token` ([github-actions](https://code.claude.com/docs/en/github-actions), [groundy](https://groundy.com/articles/how-to-run-claude-code-as-a-github-actions-agent-for-automated-pr-fixes/)).
- **Human stays in the loop by design:** the Action does NOT auto-create or auto-merge PRs by default — Claude commits to a new branch and links the PR for you to open. This is the natural Share-phase boundary ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md), [github-actions](https://code.claude.com/docs/en/github-actions)).

A **third-party "PR Autofix"** Marketplace action shows complementary design ideas (iteration tagging `[N/M]`, bot allowlist, `max_iterations: 5`). *Not Anthropic-certified; the "under $5 for 50 PRs/month" figure is vendor marketing, unverified* ([pr-autofix](https://github.com/marketplace/actions/pr-autofix-with-claude-code)).

**Honest failure modes for auto-fix** ([paddo.dev](https://paddo.dev/blog/claude-code-auto-fix-pr-lifecycle/), [groundy](https://groundy.com/articles/how-to-run-claude-code-as-a-github-actions-agent-for-automated-pr-fixes/)):
- *Green is not correct.* An auto-fix loop optimizes for a passing check, which can mean Claude weakens a test or patches a symptom. Keep a human gate before merge. *(Practitioner framing; the cited "1.7x bug rate in AI-authored PRs" is a third-party figure I did not independently verify.)*
- *Non-deterministic failures cause cost spirals.* Flaky tests and infra failures aren't fixable from a stack trace; Claude can "spin hard," each speculative push triggering a new run. Mitigate by classifying flaky-vs-real before letting Claude edit code. *(Classifier-then-retry pattern is from the groundy blog; the official repo ships a separate `test-failure-analysis.yml` to verify the exact mechanism.)*

---

## Security & least-privilege

The most serious failure mode is **prompt injection + secret exfiltration**, documented by independent security research. Treat any agent that reads untrusted GitHub content as privileged automation that must be threat-modeled like a build script.

**The disclosed attack chains:**
- Microsoft Security disclosed that the action's Read tool ran as a direct in-process call that bypassed the sandbox isolation protecting Bash, so a prompt-injection payload hidden in an issue/PR/comment could read `/proc/self/environ` and exfiltrate `ANTHROPIC_API_KEY` and other workflow secrets — even instructing the model to trim the key's first characters to dodge GitHub's secret scanner, and in one chain appending malicious HTML and opening a PR (supply-chain compromise). Anthropic patched the Read-tool/`/proc` gap in the **Claude Code CLI/agent at v2.1.128** ([microsoft](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/)).
- Separately, GMO Flatt found that `checkWritePermissions` unconditionally trusted any actor ending in `[bot]`; since GitHub Apps can create issues on public repos without being installed, an external attacker bypassed the permission gate and used injection to leak OIDC vars exchangeable for a write-scoped token. Fixed in the **action wrapper at claude-code-action v1.0.94** ([flatt.tech](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/)).

> **Gap / nuance:** these are two different components with two version schemes — v1.0.94 (the Action) and v2.1.128 (the CLI/agent) — complementary, not the same patch. The `/proc` exfiltration path is patched, but the *underlying class* (prompt injection via untrusted content) is not solved by a version bump; the architectural guardrails below are the durable fix. **Pin and update both** — public repos on pre-1.0.94 Action versions are exploitable by unauthenticated external attackers.

**Practices:**
- **Never hardcode keys.** Pass via GitHub Secrets (`anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}`); for Bedrock/Vertex drop the static key entirely and use OIDC, which Anthropic notes is "more secure than static AWS access keys because credentials are temporary and automatically rotated." *Caveat: OIDC removes stored keys but the runner still exports the resulting token into the process environment — exactly the surface the injection attacks targeted* ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md), [github-actions](https://code.claude.com/docs/en/github-actions)).
- **Prefer the auto-rotating `secrets.GITHUB_TOKEN` over a static PAT** — Anthropic says "NEVER USE" a static PAT, which doesn't rotate and could be recovered over repeated injections. Scope top-level `permissions:` to the minimum; even read-only `gh issue view` was abused for exfiltration ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md), [flatt.tech](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/)).
- **Apply the "Agents Rule of Two"** (Microsoft): an AI workflow should never simultaneously (1) process untrusted input, (2) hold access to secrets/sensitive systems, and (3) have external-communication / state-changing tools. Drop at least one leg. *Caveat: it's a design heuristic, not an enforced control* ([microsoft](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/), [mehdi.cz](https://www.mehdi.cz/blog/claude-code-ci-automation)).
- **Harden the system prompt** to declare that anything in an issue, comment, commit message, PR description, or file contents is data from an untrusted author, not instructions. Never interpolate raw PR title/body/commit text into the prompt without sanitizing ([microsoft](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/)).
- **Constrain who/what can trigger.** Default trust boundary is repository write access. Bypass inputs (`allowed_non_write_users`, `allowed_bots`) are flagged as significant risks — `allowed_bots` are NOT permission-checked, and `'*'` on public repos lets external parties trigger via comments. Use explicit allowlists (`include_comments_by_actor`); never use `'*'` publicly. The action auto-strips HTML comments, invisible characters, image alt text, and hidden HTML attributes, but new bypasses can emerge — review raw external content ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md)).
- **Forked / untrusted PRs:** do NOT use `pull_request_target` for autonomous fork-PR review — it checks out the base branch (you review the wrong tree) and exposes base-repo secrets to attacker-controlled head tooling. Safe pattern: check out the base ref at root, and if you need the PR's files put them in a subdirectory and pass `--add-dir pr-head` ([startdebugging](https://startdebugging.net/2026/05/how-to-run-claude-code-in-a-github-action-for-autonomous-pr-review/), [security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md)).
- **Keep debug output off.** `show_full_output` defaults to `false`; enabling it (or `ACTIONS_STEP_DEBUG`) prints full tool outputs — `env`, file reads, API responses — which can leak tokens into publicly-readable logs ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md), [generalanalysis](https://generalanalysis.com/guides/how-to-secure-claude-code)).
- **Scope tools with explicit allow/ask/deny rules** where "deny wins over ask and allow" — e.g. deny `Read(./.env)`, `Read(~/.ssh/**)`, `Bash(curl *)`, `Bash(git push *)`. Note: allowlists like `Bash(gh issue view:*)` are still abusable for exfiltration if the tool emits attacker-readable output — restrict outbound/write capability, not just command names ([generalanalysis](https://generalanalysis.com/guides/how-to-secure-claude-code), [security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md)).
- **Subprocess env scrubbing is on by default** via `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` (leave enabled). The sandbox uses OS primitives — Linux `bubblewrap`, macOS `seatbelt`. *Caveat: scrubbing covers the Bash/subprocess path; the disclosed Read-tool leak proved it didn't cover every tool — scrubbing alone is not complete secret isolation* ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md), [generalanalysis](https://generalanalysis.com/guides/how-to-secure-claude-code)).
- **Run in an ephemeral, locked-down runner:** single worktree, non-root, mount only the task dir, restrict network egress, and lock MCP servers with `allowManagedMcpServersOnly` using exact URL/command matching (never name-only — server names are user-assigned labels). Reserve `bypassPermissions` strictly for isolated containers/VMs ([generalanalysis](https://generalanalysis.com/guides/how-to-secure-claude-code)).
- **Human review for high-impact changes:** gate deploy/release/infra behind approval, and require CODEOWNERS review for `.github`, `.claude`, infra, and auth files so the agent can't silently rewrite its own guardrails. Optionally `use_commit_signing: true` for attributable commits ([security.md](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md), [generalanalysis](https://generalanalysis.com/guides/how-to-secure-claude-code)).

---

## Cost & rate limits

**The spend is dual:** Claude API tokens **plus** GitHub Actions / CI runner minutes, since the Action runs on your runners ([github-actions](https://code.claude.com/docs/en/github-actions), [groundy](https://groundy.com/articles/how-to-run-claude-code-as-a-github-actions-agent-for-automated-pr-fixes/)).

**Cost levers:**
- **Cap iterations** with `--max-turns` — the single most-cited cost control; in headless mode each run prints `total_cost_usd` so a CI job can log exact dollar cost per run. `--allowedTools` indirectly cuts tokens by preventing unnecessary tool calls. `--max-budget-usd` sets a hard spend ceiling ([opentools](https://opentools.ai/resources/claude-code-cicd-github-actions), [hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)). *Caveat: `--max-turns` is the only documented CLI cost lever in some guides; don't assume a hard token budget exists as a flag.*
- **Set a hard ceiling, not just monitoring.** On the Claude API, set a workspace spend limit; on Pro/Max use `/usage-credits` to set a monthly cap. You can set a spend limit *below* your tier's ceiling purely for cost control. *Caveat: spend limits are not available on Bedrock (billed via AWS Marketplace, starts at Tier 1 with no auto tier advancement); on Bedrock/Vertex/Foundry, Claude Code does not emit cost metrics — teams reported using LiteLLM (third-party, unaudited) to track spend* ([costs](https://code.claude.com/docs/en/costs), [rate-limits](https://platform.claude.com/docs/en/api/rate-limits)).
- **Workflow-level controls:** `timeout-minutes` to kill runaway jobs, `concurrency` with `cancel-in-progress: true` so a new push cancels the in-flight run, and event/path filters (`types: [opened]`, `paths-ignore: ['*.md','docs/**']`) to avoid firing on every commit ([github-actions](https://code.claude.com/docs/en/github-actions), [groundy](https://groundy.com/articles/how-to-run-claude-code-as-a-github-actions-agent-for-automated-pr-fixes/)).
- **Model choice & thinking budget.** Use Sonnet for most CI tasks, reserve Opus for hard reasoning, `model: haiku` for cheap subagent steps. Extended thinking is on by default and bills as output tokens; for simple tasks lower `/effort`, disable thinking, or set `MAX_THINKING_TOKENS=8000` on fixed-budget models. *Caveat: adaptive-reasoning models ignore a nonzero `MAX_THINKING_TOKENS` (use effort levels); some models always use extended thinking* ([costs](https://code.claude.com/docs/en/costs)).
- **Cut input tokens before Claude sees them:** PreToolUse hooks to preprocess (grep a 10k-line log to ERROR lines), prefer CLI tools over MCP servers, keep `CLAUDE.md` under ~200 lines, and write specific prompts ([costs](https://code.claude.com/docs/en/costs)).
- **Practitioner review tuning** (one author, not Anthropic docs): Sonnet by default on all PRs, gate Opus to security-sensitive paths via `paths:` filters, baseline `--max-turns 8` (raising it rarely helps — narrow paths or change model on timeout). *Starting points only; measure for your repo* ([startdebugging](https://startdebugging.net/2026/05/how-to-run-claude-code-in-a-github-action-for-autonomous-pr-review/)).

**Rate limits:**
- **CI traffic shares your org's rate limits and can starve production.** Claude Code via Console runs in an auto-created "Claude Code" workspace whose traffic counts toward org-wide limits. Set a per-workspace rate-limit cap to protect production. *Org-wide limits always apply; you cannot set limits on the default workspace* ([costs](https://code.claude.com/docs/en/costs), [rate-limits](https://platform.claude.com/docs/en/api/rate-limits)).
- **Prompt caching is the biggest throughput lever.** For most models only *uncached* input counts toward ITPM; cache reads do not — the docs cite a 2,000,000 ITPM limit with 80% cache hit rate effectively processing 10,000,000 input tokens/min, and cached tokens bill at ~10% of base input price. *Caveat: Claude Haiku 3.5 is the exception — it DOES count cache reads toward ITPM* ([rate-limits](https://platform.claude.com/docs/en/api/rate-limits)).
- **Handle 429s defensively.** Token-bucket algorithm (continuously replenished); a 60 RPM limit may enforce ~1 req/sec, so bursts trigger 429s and sharp spikes can hit separate "acceleration limits" — ramp CI traffic gradually, read `retry-after`, back off, and monitor `anthropic-ratelimit-*-remaining` headers. Limits are per-model, so splitting work across model classes gives independent budgets. *Tier-1 Sonnet 4.x is only 30,000 ITPM / 8,000 OTPM / 50 RPM — a single large run can blow past it* ([rate-limits](https://platform.claude.com/docs/en/api/rate-limits)).
- **Use the Message Batches API for non-urgent CI work** (bulk doc gen, nightly refactors) — it has its own rate-limit pool separate from the Messages API (up to 100,000 requests/batch). *Caveat: I could not confirm a specific batch price-discount percentage from the fetched sources; don't quote one* ([rate-limits](https://platform.claude.com/docs/en/api/rate-limits)).

**Budgeting honesty:** enterprise *interactive*-developer averages run ~$13/active-day and ~$150–250/developer/month, with agent teams using ~7x more tokens. These are interactive-developer averages, not CI-specific; automated/multi-instance pipelines can run higher, and the `/usage` figure is a local estimate — the Console Usage page is authoritative ([costs](https://code.claude.com/docs/en/costs)).

> **Gaps to flag explicitly:**
> - The **subscription billing change (June 2026)** routes Agent SDK / `claude -p` / GitHub Actions usage to a *separate per-user monthly programmatic credit pool* that does not consume interactive Claude Code limits; these drain first, then fall back to usage credits at API rates or halt. API-key users stay pure pay-as-you-go. **The support page's dollar figures (Pro $20, Max 5x $100, etc.) read as plan prices, not clearly-labeled credit amounts — do not quote a specific credit figure on a slide without verifying which number is the credit** ([support](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)).
> - The **$15–25 per review** figure (averaging ~20 min, billed as separate usage credits, "after every push" multiplies by push count, hitting the org spend cap silently skips with one comment) applies to the **managed Code Review product only** — *not* the self-hosted Action, whose cost is raw API tokens plus runner minutes ([code-review](https://code.claude.com/docs/en/code-review)).
> - **Exit-code semantics beyond `0 = success` are not officially documented.** Treat exit status as binary success/failure and read the JSON output for the failure reason rather than hard-coding specific non-zero values. The "non-zero on `--max-turns` exceed" behavior comes from the docs noting max-turns aborts plus a practitioner blog ([hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html), [headless](https://code.claude.com/docs/en/headless)).

---

## When NOT to put Claude in CI/CD

The honest litmus test: *"If this runs at 3 a.m. and does exactly what the prompt literally says, am I comfortable with the result with no review?"* If no, it doesn't belong unattended yet ([hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).

- **Never let a non-deterministic check be the only gate.** Even with temperature locked, output varies run to run; if CI makes a hard decision (approve/block/merge) on a single Claude answer, you have a flaky gate. Wrap the model's output in deterministic validation — run the actual tests/linter and let *that* decide. Treat one Claude review as advisory signal, not an authoritative quality gate ([refactix](https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production)).
- **Don't start by automating code generation in CI.** Generation works best in an interactive, reviewed session. The recommended on-ramp is a read-only review job first; graduate to write/generation only once the verification harness is trusted ([refactix](https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production)).
- **Only automate tasks that are idempotent, bounded, and verifiable.** "Fix the failing test in `tests/auth/`" is automatable; "improve the codebase" is not. Keep production deploys, irreversible/destructive operations, and under-specified judgment calls human-in-the-loop ([hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).
- **Never grant write access or Bash to an agent processing untrusted PR/issue content** — whitelist to exactly what the task needs (`Read,Grep,Glob` for review), scope tokens to least privilege, use a permission mode that auto-denies (not hangs), and add PreToolUse hooks as deterministic vetoes. Do NOT use `--dangerously-skip-permissions` against a real repo ([refactix](https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production), [hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).
- **Don't write interactive-style prompts for headless runs.** A prompt ending in a question ("let me know if you'd like me to proceed…") has no responder in headless mode, so the run stalls or aborts. Write self-contained instructions with explicit success criteria ([hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).
- **Don't leave runs unbounded.** Without a turn cap, a stuck agent runs until the default Actions timeout (six hours), spending real money unwatched. Set three independent backstops: `--max-turns`, `--max-budget-usd`, and a workflow `timeout-minutes`. *Illustrative guidance: ~5 turns for reviews, ~10–15 for iterative fixes — practitioner numbers, not official limits; past that the problem is usually the prompt* ([refactix](https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production), [hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).
- **Lock scheduled/recurring jobs against overlap.** Concurrent runs on the same PR race each other's commits; use a CI concurrency group (keyed on PR number, `cancel-in-progress`) or a `flock`, and give runs a deterministic `--session-id` for addressable, idempotent retries ([backgroundclaude](https://backgroundclaude.com/blog/github-actions), [hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).
- **Feed concrete failure data back; don't blindly retry.** Injecting the actual test/linter output into the next turn converges far better than "try again." Infinite retry on a bad task is worse than failing cleanly — cap iterations and fail loudly ([refactix](https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production)).
- **Know when you've outgrown GitHub Actions.** The Actions pattern fits stateless, one-shot, single-agent work tied to a GitHub event. Once you need cross-tool triggers (Linear/GitLab/Slack), parallel isolated runs, mid-run approvals, or persistent sessions, move to the Agent SDK or a dispatch/orchestration layer rather than bolting complexity onto a workflow ([backgroundclaude](https://backgroundclaude.com/blog/github-actions), [refactix](https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production)).

> **Recurring honesty point:** prompt injection from untrusted content (covered in Security) remains the strongest reason to keep an agent *out* of any CI job that combines untrusted input, secret access, and write/egress capability — the architectural guardrails, not a version bump, are the durable fix ([microsoft](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/)).

---

## Sources

### Anthropic official — Claude Code docs
- https://code.claude.com/docs/en/headless
- https://code.claude.com/docs/en/github-actions
- https://code.claude.com/docs/en/gitlab-ci-cd
- https://code.claude.com/docs/en/code-review
- https://code.claude.com/docs/en/costs

### Anthropic official — platform & support
- https://platform.claude.com/docs/en/api/rate-limits
- https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan

### Anthropic official — claude-code-action repo (examples & docs)
- https://github.com/anthropics/claude-code-action
- https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml
- https://github.com/anthropics/claude-code-action/tree/main/examples
- https://raw.githubusercontent.com/anthropics/claude-code-action/main/examples/ci-failure-auto-fix.yml
- https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md
- https://github.com/anthropics/claude-code-action/blob/main/docs/security.md

### Independent security research
- https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/
- https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/
- https://generalanalysis.com/guides/how-to-secure-claude-code

### Practitioner blogs & guides (vet independently)
- https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html
- https://www.mehdi.cz/blog/claude-code-ci-automation
- https://groundy.com/articles/how-to-run-claude-code-as-a-github-actions-agent-for-automated-pr-fixes/
- https://startdebugging.net/2026/05/how-to-run-claude-code-in-a-github-action-for-autonomous-pr-review/
- https://paddo.dev/blog/claude-code-auto-fix-pr-lifecycle/
- https://refactix.com/ai-development-engineering/claude-code-ci-cd-headless-patterns-production
- https://backgroundclaude.com/blog/github-actions
- https://blog.jinmiaoluo.com/en/posts/gitlab-ci-claude-code-review/
- https://medium.com/@sayan.nandi/building-an-ai-powered-automated-code-reviewer-with-claude-code-jenkins-and-bitbucket-b0e600d27b25
- https://circleci.com/blog/getting-started-with-claude-code-and-circleci
- https://about.gitlab.com/blog/claude-code-and-gitlab/
- https://opentools.ai/resources/claude-code-cicd-github-actions
- https://agentpatterns.ai/workflows/headless-claude-ci/

### Third-party tools & plugins (not Anthropic-certified)
- https://github.com/marketplace/actions/pr-autofix-with-claude-code
- https://plugins.jenkins.io/ai-agent/
- https://github.com/RealMikeChong/claude-code-for-gitlab
