# ai-reviewer — AGENTS.md

## Background

GitHub Action para revisión automatizada de código con IA. Proporciona análisis línea por línea de Pull Requests, generando sugerencias para bugs, anti-patrones, typos, problemas de seguridad y rendimiento. Incluye resumen automático de PR, respuestas interactivas a comentarios, y revisiones incrementales que solo analizan cambios nuevos.

---

## Structure

```
src/
├── main.ts                    # GitHub Action entry point
│   ├── handlePullRequest()    # Routes pull_request events
│   └── handlePullRequestComment() # Routes pull_request_review_comment events
├── cli.ts                     # CLI entry point (--list-prs, --pr <num> --dry-run, --out)
├── context.ts                 # GitHub Actions context loading + debug mode
├── config.ts                  # Centralized config from env vars & action inputs
├── octokit.ts                 # Octokit client factory with retry & throttling
├── pull_request.ts            # Full PR review orchestration (371 lines)
│   ├── diff.ts                # Parses GitHub patches into hunks (234 lines)
│   ├── prompts.ts             # LLM prompt engineering (389 lines)
│   ├── messages.ts            # Markdown builders for comments
│   └── comments.ts            # GitHub comment thread utilities
├── pull_request_comment.ts    # Interactive comment reply handler (97 lines)
└── ai.ts                      # LLM provider abstraction & model registry (297 lines)
    └── providers/
        ├── ai-sdk.ts          # Vercel AI SDK provider (Anthropic, OpenAI, Google)
        └── sapaicore.ts       # SAP AI Core provider (OAuth, deployments, inference)
```

---

## Coding Style

- **TypeScript only**: Strict TypeScript with `tsconfig.json` targeting ES2022.
- **Naming**: camelCase for variables/functions, PascalCase for types/interfaces.
- **Architecture**: Clean separation between GitHub API, diff parsing, prompt engineering, and LLM providers.
- **Structured outputs**: All LLM outputs validated with Zod schemas.
- **Error handling**: Graceful degradation — if LLM fails, log and continue. Never block PR review.
- **Async/await**: Always use async/await.
- **Bundling**: esbuild for production bundles (`dist/index.js`, `dist/cli.js`).

---

## Dependencies

### Core
| Package | Purpose |
|---------|---------|
| `@actions/core` | GitHub Actions SDK (inputs, context, logging) |
| `@actions/github` | GitHub API context |
| `@octokit/action` | GitHub Actions API client |
| `@octokit/plugin-retry` | Retry logic for API calls |
| `@octokit/plugin-throttling` | Rate limit handling |
| `ai` (Vercel AI SDK) | Structured LLM inference with `generateObject` |
| `zod` | Runtime schema validation for LLM outputs |
| `esbuild` | Bundling |
| `dotenv` | Local environment loading |

### LLM Providers
| Package | Purpose |
|---------|---------|
| `@ai-sdk/anthropic` | Anthropic models (Claude) |
| `@ai-sdk/openai` | OpenAI models (GPT, o1, o3, o4) |
| `@ai-sdk/google` | Google models (Gemini) |
| `@langchain/core` | Output parsing for SAP AI Core |
| `axios` | HTTP client for SAP AI Core |

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | No | Anthropic API key (Claude models) |
| `OPENAI_API_KEY` | No | OpenAI API key (GPT models) |
| `GOOGLE_API_KEY` | No | Google API key (Gemini models) |
| `AI_REVIEWER_MODEL` | No | Model to use (default: `claude-3.5-sonnet`) |
| `GITHUB_TOKEN` | Yes | GitHub token for API access |
| `SAP_AI_CORE_*` | No | SAP AI Core provider config |

---

## Build & Run

```bash
# Install
npm install

# Build
npm run build

# Local CLI testing
npx ts-node src/cli.ts --list-prs
npx ts-node src/cli.ts --pr 42 --dry-run
npx ts-node src/cli.ts --pr 42 --dry-run --out review.json

# Run tests
npm test
```

---

## Testing

- **Framework**: Jest with `ts-jest`
- **Coverage thresholds**: 70% branches, 80% functions/lines/statements
- **Test files**: `src/__tests__/`
- **Mocks**: `src/__mocks__/@octokit/` for action, retry, throttling plugins

---

## Deployment

- **Platform**: GitHub Actions
- **Action manifest**: `action.yml`
- **Entry point**: `dist/index.js` (bundled, minified ~1.8MB)
- **Trigger**: `pull_request` / `pull_request_target` events
- **Example workflow**: `.github/workflows/presubmit-review.yml`

---

## LLM Model Registry

### AI SDK Provider (ai-sdk)

| Provider | Models |
|----------|--------|
| Anthropic | claude-3.5-sonnet, claude-3.7-sonnet, claude-4-opus |
| OpenAI | gpt-4o, gpt-5, o1, o3, o4 |
| Google | gemini-1.5-pro, gemini-2.0, gemini-2.5 |

### SAP AI Core Provider (sapaicore)

| Provider | Endpoints |
|----------|-----------|
| Anthropic | `/invoke`, `/converse` |
| OpenAI | `/chat/completions` |

---

## Common Tasks

### Add a new LLM model

1. Add model name to the registry in `src/ai.ts`.
2. Ensure the provider supports the model.
3. Add to the model whitelist in `src/config.ts`.

### Modify a prompt

1. Edit the prompt template in `src/prompts.ts`.
2. Update the corresponding Zod schema if the output structure changes.
3. Add test case in `src/__tests__/prompts.test.ts`.

### Add a new GitHub event handler

1. Add event route in `src/main.ts`.
2. Create handler function with proper context loading.
3. Add appropriate Octokit API calls in `src/octokit.ts`.

---

## Architecture Notes

### Incremental Review Algorithm

1. Fetch PR commits and existing overview comment.
2. Parse new commits since last review.
3. If no new commits, skip review (already reviewed).
4. If new commits exist, generate review only for changed files.
5. Post incremental review comment referencing previous review.

### Diff Parsing

- GitHub patches are parsed into `Hunk` objects with start/end line tracking.
- Diffs are formatted into `__new hunk__` / `__old hunk__` representation optimized for LLM consumption.
- Existing comment threads are associated with relevant hunks to avoid duplicate reviews.
