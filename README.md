# claude-pr-review-skill

A [Claude Code](https://claude.com/claude-code) **skill** that sets up automated,
Claude-powered pull request review on a GitHub repo — via a `claude-code-action`
GitHub Actions workflow, authenticated with a Claude Pro/Max OAuth token (no
pay-as-you-go API billing required).

Ask Claude Code something like *"set up Claude PR review on this repo"* and,
with this skill installed, it will:

1. Add a `.github/workflows/claude-pr-review.yml` workflow that runs on every
   PR (open/sync/reopen) and posts inline + summary review comments.
2. Walk you through minting a `CLAUDE_CODE_OAUTH_TOKEN` via `claude setup-token`
   and storing it as a repo secret — without the token ever touching the
   conversation or shell history.
3. Flag known gotchas (default-branch workflow sync requirements, why the
   workflow must stay on `pull_request` and never `pull_request_target`) and
   verify the check runs end-to-end on a test PR.

## Files

- **[SKILL.md](./SKILL.md)** — the full skill definition: step-by-step
  instructions, the workflow YAML, and an alternative Workload Identity
  Federation (WIF) setup for org/team use cases.
- **[sequence-diagram.md](./sequence-diagram.md)** — a mermaid sequence
  diagram showing how a PR review actually flows end-to-end once the
  workflow is installed (Developer → GitHub → Actions Runner →
  `claude-code-action` → Anthropic API → back to the PR).

## Using this skill

Drop the `setup-claude-pr-review/` folder (rename this repo's contents into
that folder, or clone it directly) into a project's `.claude/skills/`
directory. Claude Code will pick it up automatically and offer to use it
when you ask for PR review automation.
