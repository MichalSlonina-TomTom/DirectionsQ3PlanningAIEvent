# Claude in CI/CD — primer

A short, plain-language intro for someone who has never wired Claude into a build pipeline. About a 5-minute read.

## What it is

Claude can run inside your CI/CD pipeline as an automated teammate: it reads code, reviews pull requests, and pushes fixes — triggered either by a comment you write or automatically on a pull request. The official building block is the **`anthropics/claude-code-action@v1`** GitHub Action, which runs Claude Code inside a GitHub-hosted runner. (The same engine also runs from the command line as `claude -p "<prompt>"` on any runner, including GitLab CI or Jenkins, but GitHub is the easiest place to start.)

## The simplest way to start

1. From a repo where you're an admin, open Claude Code in your terminal and run **`/install-github-app`**. It installs the Claude GitHub app and adds your `ANTHROPIC_API_KEY` as a repo secret for you.
2. In any issue or pull request, leave a comment that mentions **`@claude`** — for example, "@claude fix the failing test in tests/auth/". Claude checks out the branch, makes the change on a new branch, and gives you a link to open the PR.

Two things to know up front: it must be `@claude`, not `/claude`; and Claude does **not** auto-merge — a human still reviews and merges. The action runs on your CI, so it spends both your Claude API tokens and your GitHub Actions runner minutes.

## What to use it for in a feature build

- **Pull request review (Verify).** Trigger on `pull_request` and pass a `prompt` (no `@claude` needed) so every PR gets reviewed automatically against your standards. Put your review rules in a **`CLAUDE.md`** file at the repo root — Claude reads it on every run, so you don't re-explain your conventions each time.
- **Auto-fix / "make it green" (Build).** When CI fails, have Claude read the failure and push a fix. Anthropic ships a ready-made example workflow for this. The most common task, though, is simply commenting `@claude` on a failing PR and asking it to fix the problem.

Start with read-only review; graduate to letting Claude write code only once you trust the setup.

## Safety must-dos

- **Secrets:** never hardcode the key in a workflow file. Pass it from GitHub Secrets (`${{ secrets.ANTHROPIC_API_KEY }}`), and use the auto-rotating `${{ secrets.GITHUB_TOKEN }}` rather than a long-lived personal access token.
- **Least privilege:** grant the workflow only the permissions the job needs (a review-only job needs far less than an auto-fixer). Keep an explicit allowlist of who can trigger Claude — **never use `'*'`** for allowed users or bots, especially on public repos.
- **Untrusted / fork PRs:** treat anything in an issue, comment, or PR description as untrusted data, not instructions — attackers hide prompts there. For autonomous review of fork PRs, do **not** use `pull_request_target` (it exposes your secrets); skip fork reviews or have a maintainer trigger them manually after eyeballing the diff.
- **Stay current:** pin the action and keep both it and Claude Code updated — past security fixes (e.g. Claude Code v2.1.128) closed real secret-leak holes.

## Cost note

For the GitHub Action, your cost is your Claude API token usage plus your GitHub Actions runner minutes — there's no per-review price tag. Keep it predictable by capping each run with `--max-turns`, adding a workflow `timeout-minutes`, using Sonnet for everyday work, and filtering events so it doesn't fire on every tiny push. (Note: Anthropic's separate *managed* Code Review product, which runs on Anthropic's infrastructure, is billed differently — don't confuse the two.)

## Pitfalls to watch

1. **Claude's commits may not re-trigger CI.** Commits authored by the default Actions user don't kick off downstream checks. Fix: authenticate through the Claude GitHub App (or a GitHub App token) so its commits re-run CI.
2. **Green is not the same as correct.** An auto-fix loop optimizes for a passing check — it can weaken or skip a test instead of fixing the real bug. Always keep a human gate before merge.
3. **Runaway cost.** A run with no turn cap and no timeout can spin until the job times out, billing the whole time. Always set `--max-turns` and a workflow timeout.
4. **Prompt injection from untrusted text.** Malicious instructions hidden in a PR or issue (even in invisible HTML) can try to make Claude leak secrets. The action sanitizes known tricks, but the durable defense is least privilege plus never combining untrusted input, secret access, and write/external tools in the same job.

## Sources

- Claude Code GitHub Actions docs — https://code.claude.com/docs/en/github-actions
- claude-code-action security guide — https://github.com/anthropics/claude-code-action/blob/main/docs/security.md
- Example workflow (`@claude` wiring) — https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml
- Claude Code cost controls — https://code.claude.com/docs/en/costs
- Managed Code Review docs — https://code.claude.com/docs/en/code-review
- Microsoft Security: securing CI/CD in an agentic world — https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/
