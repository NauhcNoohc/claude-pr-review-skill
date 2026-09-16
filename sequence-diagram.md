# Claude PR Review — sequence diagram

How the automated review actually runs once set up (OAuth-token auth path).

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant GH as GitHub
    participant Runner as Actions Runner
    participant Action as claude-code-action
    participant API as Anthropic API

    Dev->>GH: Open / update PR
    GH->>Runner: Trigger claude-pr-review.yml (pull_request event)
    Runner->>Runner: Checkout repository (fetch-depth: 1)
    Runner->>Action: Run "Claude Code Review" step
    Action->>API: Authenticate with CLAUDE_CODE_OAUTH_TOKEN
    API-->>Action: Auth OK (billed against Pro/Max plan usage)
    Action->>Runner: Read files / git diff / git log (allowed-tools)
    Runner-->>Action: Diff + file contents
    Action->>API: Send prompt + gathered context
    API-->>Action: Review findings
    Action->>GH: POST inline comments (mcp github_inline_comment)
    Action->>GH: POST summary comment
    Action-->>Runner: Exit success
    Runner-->>GH: Report check status (pass/fail)
    GH-->>Dev: Notify — review comments + check result on PR
```

## Why this is simpler than WIF

The OAuth token is the *only* new credential in the whole chain — checkout,
diff reading, and comment posting all run under the runner's own GitHub-side
permissions. Anthropic never needs to know anything about the repo's
structure, organization, or workspace layout, which is exactly the surface
area that WIF's federation rule (issuer, subject pattern, claim conditions,
workspace targeting) has to get right instead.
