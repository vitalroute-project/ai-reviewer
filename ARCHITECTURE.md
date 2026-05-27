# ai-reviewer — ARCHITECTURE.md

## System Overview

GitHub Action que proporciona revisión automatizada de código con IA para Pull Requests. Analiza diffs, genera sugerencias línea por línea, crea resúmenes automáticos y responde a comentarios interactivos.

---

## C4 Context Diagram

```mermaid
C4Context
    title System Context - ai-reviewer
    Person(dev, "Developer", "Crea Pull Requests y recibe revisiones automatizadas")
    System(github, "GitHub", "PR management + Actions runner")
    System(ai_reviewer, "ai-reviewer", "AI-powered PR review GitHub Action")
    System(llm, "LLM Provider", "Anthropic / OpenAI / Google / SAP AI Core")

    Rel(dev, github, "Creates PR")
    github --> ai_reviewer: Triggers on pull_request
    ai_reviewer --> llm: Sends diff + context
    llm -->> ai_reviewer: Review suggestions
    ai_reviewer --> github: Posts review comments
```

---

## C4 Container Diagram

```mermaid
C4Container
    title Container Diagram - ai-reviewer
    Container_Boundary(action, "GitHub Actions Runner") {
        Component(entry, "main.ts", "GitHub Action entry point", "TypeScript")
        Component(pr_review, "pull_request.ts", "Full PR review logic", "TypeScript")
        Component(pr_comment, "pull_request_comment.ts", "Interactive comment replies", "TypeScript")
        Component(cli, "cli.ts", "Local CLI for dry-run testing", "TypeScript")
        Component(diff, "diff.ts", "Patch parsing into hunks", "TypeScript")
        Component(prompts, "prompts.ts", "LLM prompt engineering", "TypeScript")
        Component(ai, "ai.ts", "LLM provider abstraction", "TypeScript")
        Component(config, "config.ts", "Configuration from env/action inputs", "TypeScript")
        Component(octokit, "octokit.ts", "GitHub API client with retry/throttle", "TypeScript")
    }

    System_Ext(github_api, "GitHub API", "PR data, comments, reviews")
    System_Ext(llm_api, "LLM API", "Claude, GPT, Gemini inference")

    Rel(entry, pr_review, "Calls")
    Rel(entry, pr_comment, "Calls")
    Rel(pr_review, diff, "Uses")
    Rel(pr_review, prompts, "Uses")
    Rel(pr_review, ai, "Uses")
    Rel(pr_review, octokit, "Uses")
    Rel(ai, llm_api, "HTTP calls")
    Rel(octokit, github_api, "REST API")
```

---

## PR Review Flow

```mermaid
sequenceDiagram
    participant GH as GitHub
    participant ACT as Action (main.ts)
    participant PR as PR Review (pull_request.ts)
    participant DIFF as Diff Parser (diff.ts)
    participant LLM as LLM Provider
    participant OUT as Output (messages.ts)
    participant GHA as GitHub API

    GH->>ACT: pull_request_target event
    ACT->>PR: handlePullRequest()
    PR->>GHA: Fetch PR commits, files, existing comments
    GHA-->>PR: PR data
    PR->>DIFF: Parse patches into hunks
    DIFF-->>PR: Formatted hunks with line tracking
    PR->>PR: Check incremental review (new commits?)
    alt New commits
        PR->>LLM: runSummaryPrompt → PR summary
        LLM-->>PR: Title + description + file summaries
        PR->>LLM: runReviewPrompt → line-by-line review
        LLM-->>PR: Structured review comments (Zod validated)
        PR->>OUT: Build review comment markdown
        PR->>GHA: Post review comments (createReview + submitReview)
        PR->>GHA: Post overview comment
    else No new commits
        PR->>PR: Skip review (already done)
    end
```

---

## Incremental Review Algorithm

```mermaid
graph TD
    A[PR Event Received] --> B[Fetch Existing Overview Comment]
    B --> C{Has Overview?}
    C -->|No| D[Full Review]
    C -->|Yes| E[Fetch PR Commits]
    E --> F{New Commits Since Last Review?}
    F -->|No| G[Skip - Already Reviewed]
    F -->|Yes| H[Parse New File Diffs]
    H --> I[Run LLM Review on New Changes]
    I --> J[Post Incremental Review]
    J --> K[Update Overview Comment]

    D --> L[Parse All File Diffs]
    L --> M[Run LLM Review on All Changes]
    M --> N[Post Full Review]
    N --> O[Create Overview Comment]
```

---

## Diff Parsing

### Hunk Structure

GitHub patches are parsed into `Hunk` objects:

```typescript
interface Hunk {
  file: string;
  newHunk: string;   // Format: __new hunk__\n<line_number>: <content>\n
  oldHunk: string;   // Format: __old hunk__\n<line_number>: <content>\n
  startLine: number;
  endLine: number;
  existingThreads: ReviewCommentThread[];
}
```

### LLM-Optimized Format

Diffs are reformatted for optimal LLM consumption:

```
__new hunk__
42: const result = await fetch(url);
43: const data = JSON.parse(result);  // ← potential issue here
```

---

## Prompt Engineering

Three main prompts in `src/prompts.ts`:

### 1. Summary Prompt

Generates PR title, description, and per-file summaries.

**Output Schema (Zod):**
```typescript
{
  title: string;
  description: string;
  fileSummaries: { file: string; summary: string }[];
}
```

### 2. Review Prompt

Generates line-by-line review comments.

**Output Schema (Zod):**
```typescript
{
  comments: {
    file: string;
    line: number;
    lineEnd: number;
    message: string;
    label: string;  // bug | security | performance | style | docs
    critical: boolean;
  }[];
}
```

### 3. Comment Reply Prompt

Generates replies to user review comments.

---

## LLM Provider Abstraction

```mermaid
graph TD
    A[ai.ts] --> B{Provider Type}
    B -->|ai-sdk| C[ai-sdk.ts]
    B -->|sap-ai-sdk| D[sapaicore.ts]

    C --> E[Anthropic]
    C --> F[OpenAI]
    C --> G[Google]

    D --> H[Anthropic via SAP]
    D --> I[OpenAI via SAP]

    E --> J[generateObject with Zod schema]
    F --> J
    G --> J
    H --> J
    I --> J
```

### Model Registry

| Provider | Models |
|----------|--------|
| Anthropic | claude-3.5-sonnet, claude-3.7-sonnet, claude-4-opus |
| OpenAI | gpt-4o, gpt-5, o1, o3, o4 |
| Google | gemini-1.5-pro, gemini-2.0, gemini-2.5 |

---

## Security Architecture

### GitHub Action Permissions

```yaml
permissions:
  contents: read
  pull-requests: write
  issues: write
```

### Why `pull_request_target` vs `pull_request`

`pull_request_target` is used because:
- It runs in the context of the base branch (safe for this use case — no code execution from PR)
- Allows posting comments to PRs from forked repositories
- The action only reads PR data and posts comments; it never executes PR code

### Security Best Practices

- API keys stored as GitHub secrets, never in code
- LLM prompts sanitized to prevent prompt injection from code comments
- Octokit configured with retry + throttling to prevent rate limit issues

---

## CLI Architecture

```mermaid
graph LR
    A[CLI Entry] --> B{Command}
    B -->|--list-prs| C[List open PRs]
    B -->|--pr N --dry-run| D[Review PR locally]
    B -->|--pr N --out FILE| E[Review + save to file]

    D --> F[Fetch PR data via Octokit]
    F --> G[Parse diff]
    G --> H[Run LLM review]
    H --> I[Print to stdout]

    E --> J[Save to file]
```

---

## Deployment

### GitHub Action

```yaml
# action.yml
name: 'Presubmit AI Reviewer'
runs:
  using: 'node20'
  main: 'dist/index.js'
```

### Build Output

| File | Size | Purpose |
|------|------|---------|
| `dist/index.js` | ~1.8MB | Bundled GitHub Action entry point |
| `dist/cli.js` | ~1.8MB | Bundled CLI entry point |

Built with esbuild for production optimization.
