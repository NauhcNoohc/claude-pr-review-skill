---
name: setup-claude-pr-review
description: Set up automated Claude Code PR review on a GitHub repo via GitHub Actions, authenticated with a Claude Pro/Max OAuth token (no API billing needed). Use when the user asks to add/replicate automated PR review, "claude review my PRs", or set up claude-code-action in a repo.
---

# Set up Claude Code PR review

Adds a GitHub Actions workflow that runs `anthropics/claude-code-action` on every
PR, posting inline + summary review comments. Authenticates via
`CLAUDE_CODE_OAUTH_TOKEN` (uses the user's Claude Pro/Max subscription usage,
not pay-as-you-go API credits) — this is simpler than Workload Identity
Federation and doesn't require a funded Anthropic Console workspace.

See `sequence-diagram.md` in this folder for how the whole flow fits together.

## Steps

1. **Confirm prerequisites.** Check this is a git repo with a GitHub remote
   (`git remote -v`, `gh repo view`) and that `gh auth status` is logged in.

2. **Create `.github/workflows/claude-pr-review.yml`:**

   ```yaml
   name: Claude PR Review

   on:
     pull_request:
       types: [opened, synchronize, reopened]

   concurrency:
     group: claude-review-${{ github.event.pull_request.number }}
     cancel-in-progress: true

   jobs:
     claude-review:
       runs-on: ubuntu-latest
       permissions:
         contents: read
         pull-requests: write
         issues: read
         id-token: write

       steps:
         - name: Checkout repository
           uses: actions/checkout@v4
           with:
             fetch-depth: 1

         - name: Run Claude Code Review
           uses: anthropics/claude-code-action@v1
           with:
             claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
             prompt: |
               REPO: ${{ github.repository }}
               PR NUMBER: ${{ github.event.pull_request.number }}

               Review this pull request as a senior reviewer. Focus on:
               - Correctness and edge cases
               - Bugs and logic errors
               - Security issues (injection, unsafe deserialization, secrets)
               - Performance issues
               - Test coverage for the changed code
               - Consistency with existing conventions in this repo

               Skip nitpicks about formatting that a linter would catch. Leave
               inline comments on specific lines where possible, and end with a
               short summary comment (what's good, what must be fixed before
               merge, optional suggestions).
             claude_args: |
               --allowed-tools "Read,Grep,Glob,Bash(git diff:*),Bash(git log:*),mcp__github_inline_comment__create_inline_comment"
   ```

   Tailor the bullet list in `prompt` to the target repo's language/stack
   (e.g. name the specific concerns for that ecosystem — nulls/async for .NET,
   goroutine leaks for Go, etc.) — don't leave it generic if the repo has an
   obvious primary language.

3. **Get the OAuth token — this step needs the user, not you.** You cannot run
   this yourself: it's an interactive browser OAuth flow tied to the user's own
   account, and the token must never pass through the conversation. Tell the
   user to run, in their own terminal (suggest the `!` prefix if they're in
   Claude Code):

   ```
   claude setup-token
   ```

   Then, to store it as a secret **without it touching shell history or this
   conversation**, and to sidestep a known gotcha (long tokens can pick up a
   stray embedded line break when copied from a terminal, which breaks auth
   with a cryptic "invalid" error) — redirect straight to a file and strip
   newlines before setting the secret:

   ```
   claude setup-token > /tmp/claude-token.txt
   tr -d '\n\r' < /tmp/claude-token.txt | gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo OWNER/REPO && rm /tmp/claude-token.txt
   ```

   Verify with `gh secret list --repo OWNER/REPO` — do not ask the user to
   paste the token value to you at any point.

4. **Commit and push** the workflow file (ask before pushing, per normal
   git safety practice).

5. **Known gotcha — mention this to the user:** `claude-code-action` (and
   possibly GitHub itself) will silently skip running the review — not fail,
   just never create a check — if the PR branch's copy of
   `claude-pr-review.yml` differs from the **default branch's** copy at push
   time. In practice this means: once this workflow exists on the default
   branch, any *future* edits to it made only on a feature branch may not
   trigger a real run until the default branch's copy is updated to match too.
   This mostly matters when iterating on the workflow itself, not for normal
   PRs that don't touch it.

6. **Never trigger this workflow on `pull_request_target`.** It must stay on
   `pull_request` — `pull_request_target` runs with base-branch trust while
   checking out the PR's (possibly malicious, if the repo is public) code,
   which combined with any secret access is the classic "pwn request"
   vulnerability.

7. **Verify end-to-end:** open a small test PR (a trivial doc fix is enough)
   and confirm the `claude-review` check appears and passes:
   `gh pr checks <PR#> --repo OWNER/REPO`.

## Alternative: Workload Identity Federation (WIF)

For an org/team setting where PR review shouldn't be tied to one person's
subscription, `claude-code-action` also supports WIF (no stored API key,
short-lived tokens minted per-run). This is meaningfully more setup and more
prone to subtle misconfiguration than the OAuth-token path above — from
experience, the failure modes to watch for if going this route:

- The federation rule's `event_name` claim condition (if set) must match the
  actual triggering event (`pull_request`, not `push`) — GitHub's OIDC token
  carries a different `sub` claim shape per event type
  (`repo:OWNER/REPO:pull_request` vs `repo:OWNER/REPO:ref:refs/heads/BRANCH`).
- If the rule is enabled across multiple workspaces ("Enable in all
  workspaces"), the token exchange is ambiguous unless the workflow also
  passes `anthropic_workspace_id` — the 401 error message names this
  explicitly when it's the cause.
- This still needs a *funded* API workspace (separate billing from any
  claude.ai subscription) — a `"Credit balance is too low"` 400 error means
  billing, not a config problem.

Default to the OAuth-token approach unless the user specifically asks for WIF
or has a multi-person/CI-fleet use case where tying auth to one person's
subscription doesn't make sense.
