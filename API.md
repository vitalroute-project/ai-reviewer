# ai-reviewer — API Reference

## Overview

ai-reviewer es una GitHub Action que se activa en eventos de Pull Request. No expone APIs REST tradicionales — su interfaz es a través de:
1. **GitHub Action inputs** (para configuración)
2. **CLI flags** (para testing local)
3. **LLM structured outputs** (Zod schemas)

---

## GitHub Action Inputs

### action.yml Configuration

```yaml
name: 'Presubmit AI Reviewer'
inputs:
  github_api_url:
    description: 'Custom GitHub API URL (Enterprise Server)'
    required: false
  github_server_url:
    description: 'Custom GitHub server URL (Enterprise Server)'
    required: false
```

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GITHUB_TOKEN` | Sí | GitHub token for API access |
| `LLM_API_KEY` | Sí (one of) | API key for selected LLM provider |
| `LLM_MODEL` | No | Model to use (default: `claude-sonnet-4-5`) |
| `LLM_PROVIDER` | No | Provider: `ai-sdk` (default) or `sap-ai-sdk` |
| `LLM_BASE_URL` | No | Base URL for OpenAI-compatible providers |
| `GITHUB_API_URL` | No | Custom GitHub API URL |
| `GITHUB_SERVER_URL` | No | Custom GitHub server URL |

### Workflow Configuration

```yaml
name: Presubmit.ai

permissions:
  contents: read
  pull-requests: write
  issues: write

on:
  pull_request_target:
    types: [opened, synchronize]
  pull_request_review_comment:
    types: [created]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: presubmit/ai-reviewer@latest
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_MODEL: "claude-sonnet-4-5"
```

---

## CLI Interface

### Usage

```bash
# List open PRs
pnpm review -- --list-prs --state open --limit 5

# Review a PR (dry-run, no GitHub writes)
pnpm review -- --pr 123 --dry-run

# Review and save to file
pnpm review -- --pr 123 --dry-run --out review.txt

# Specify repository
pnpm review -- --pr 123 --owner myorg --repo myrepo --dry-run
```

### CLI Flags

| Flag | Description | Example |
|------|-------------|---------|
| `--list-prs` | List open PRs | `--list-prs --state open --limit 5` |
| `--pr <number>` | PR number to review | `--pr 123` |
| `--dry-run` | Skip GitHub API writes | `--dry-run` |
| `--out <file>` | Save output to file | `--out review.json` |
| `--owner <string>` | Repository owner | `--owner myorg` |
| `--repo <string>` | Repository name | `--repo myrepo` |

---

## LLM Structured Outputs (Zod Schemas)

### PR Summary Schema

```typescript
const SummarySchema = z.object({
  title: z.string().describe("PR title"),
  description: z.string().describe("PR description"),
  fileSummaries: z.array(z.object({
    file: z.string().describe("File path"),
    summary: z.string().describe("File change summary"),
  })),
});
```

### Review Comment Schema

```typescript
const ReviewCommentSchema = z.object({
  comments: z.array(z.object({
    file: z.string().describe("File path"),
    line: z.number().describe("Starting line number"),
    lineEnd: z.number().describe("Ending line number"),
    message: z.string().describe("Review comment text"),
    label: z.enum(["bug", "security", "performance", "style", "docs"])
      .describe("Issue category"),
    critical: z.boolean().describe("Whether this is a critical issue"),
  })),
});
```

### Comment Reply Schema

```typescript
const CommentReplySchema = z.object({
  reply: z.string().describe("AI reply to the review comment"),
});
```

---

## GitHub API Interactions

### Events Handled

| Event | Handler | Action |
|-------|---------|--------|
| `pull_request_target` (opened, synchronize) | `handlePullRequest()` | Full or incremental review |
| `pull_request_review_comment` (created) | `handlePullRequestComment()` | Interactive reply |

### API Calls Made

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/repos/{owner}/{repo}/pulls/{prNumber}/commits` | Fetch PR commits |
| GET | `/repos/{owner}/{repo}/pulls/{prNumber}/files` | Fetch changed files |
| GET | `/repos/{owner}/{repo}/issues/{prNumber}/comments` | Fetch existing comments |
| GET | `/repos/{owner}/{repo}/pulls/{prNumber}/comments` | Fetch review comments |
| GET | `/repos/{owner}/{repo}/git/blobs/{sha}` | Fetch file content |
| POST | `/repos/{owner}/{repo}/commits/{sha}/comments` | Post review comment |
| POST | `/repos/{owner}/{repo}/pulls/{prNumber}/reviews` | Submit review |
| PATCH | `/repos/{owner}/{repo}/issues/comments/{commentId}` | Update overview comment |

---

## LLM API Interactions

### Provider Abstraction

All providers use the Vercel AI SDK `generateObject()` interface:

```typescript
const result = await generateObject({
  model: ai(model),
  schema: ReviewCommentSchema,
  prompt: reviewPrompt,
});
```

### Supported Providers

| Provider | SDK Package | Models |
|----------|------------|--------|
| Anthropic | `@ai-sdk/anthropic` | claude-3.5-sonnet, claude-3.7-sonnet, claude-4-opus |
| OpenAI | `@ai-sdk/openai` | gpt-4o, gpt-5, o1, o3, o4 |
| Google | `@ai-sdk/google` | gemini-1.5-pro, gemini-2.0, gemini-2.5 |
| SAP AI Core | `@langchain/core` | Anthropic + OpenAI via SAP endpoints |

---

## Output Format

### Review Comment Format

Reviews are posted to GitHub as structured markdown:

```markdown
## PR Summary
**Title:** Add user authentication endpoint
**Description:** Implements JWT-based authentication for the API...

### Files Changed
- `src/routes/authRoutes.js` — Added login, logout, refresh endpoints
- `src/middleware/auth.js` — Added JWT verification middleware

## Code Review

### src/routes/authRoutes.js:15-20
> ⚠️ **bug**: Missing error handling for `jwt.sign()` failure
> ```diff
> - const token = jwt.sign(payload, secret);
> + let token;
> + try {
> +   token = jwt.sign(payload, secret);
> + } catch (err) {
> +   throw new Error('Token generation failed');
> + }
> ```
```
