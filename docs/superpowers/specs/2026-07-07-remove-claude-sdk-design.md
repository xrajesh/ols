# OLS-3473: Remove Claude SDK and Binaries from Agentic Sandbox

## Problem Statement

The agentic-sandbox Containerfile bundles `@anthropic-ai/claude-code` (~220MB proprietary Node.js binary) and installs Node.js solely to run it. The `claude-agent-sdk` Python package is a subprocess wrapper (`SubprocessCLITransport`) that spawns this binary — no pure-Python mode exists. This binary carries Anthropic's proprietary license (not OSI-approved), creating redistribution risk for a Red Hat product image shipped via `registry.redhat.io`.

OLS-3113 (remove Node.js and SDK packages from image) was closed as "Won't Do" because the Claude provider still depends on the binary. This story addresses the license concern by eliminating the claude-agent-sdk and its binary dependency entirely.

## Scope

**In scope:** Remove all claude-agent-sdk and @anthropic-ai/claude-code artifacts from the agentic-sandbox and related repos. Document broken config paths for future rerouting.

**Out of scope:** Replacement SDK selection for Anthropic model access on Vertex/Bedrock. That decision is deferred to implementation planning.

## What's Being Removed

### Artifacts

| Artifact | Location | Purpose being eliminated |
|---|---|---|
| `@anthropic-ai/claude-code` | `package.json`, `package-lock.json`, Containerfile npm install | Proprietary CLI binary (~220MB) |
| `claude-agent-sdk` | `pyproject.toml` dependencies, `requirements.*.txt` | Python subprocess wrapper for the binary |
| Node.js runtime | Containerfile `dnf install nodejs` | Only needed to run claude-code |
| `ClaudeProvider` | `src/lightspeed_agentic/providers/claude.py` | Provider adapter that uses claude-agent-sdk |
| Claude factory entry | `factory.py` case `"claude"` | Provider instantiation path |
| Claude symlink | Containerfile `ln -s .../claude.exe /usr/local/bin/claude` | Binary entrypoint |
| `node_modules` copy | Containerfile `COPY --from=builder /app/node_modules` | NPM artifacts in image |

### Config Paths Broken

Three config paths in `config.py` resolve to SDK name `"claude"` and will lose their backend:

| `LIGHTSPEED_PROVIDER` | `MODEL_PROVIDER` | Current SDK resolution | Impact |
|---|---|---|---|
| `anthropic` | — | `"claude"` via `_resolve_anthropic()` | Broken — no provider |
| `vertex` | `anthropic` | `"claude"` via `_resolve_vertex()` | Broken — no provider |
| `bedrock` | — | `"claude"` via `_resolve_bedrock()` | Broken — no provider |

These paths need rerouting to alternative agentic SDKs. The replacement approach is deferred to implementation planning. Candidate directions include rerouting through existing agentic SDKs (`google-adk` for Vertex, `openai-agents` for Bedrock) or building a custom agent loop on the `anthropic` Python SDK.

**Note:** The Gemini provider (`google-adk`) does not currently support Anthropic models on Vertex — the `vertex/anthropic` path has always routed through `ClaudeProvider`. Whether `google-adk` can be extended for this is a research question for implementation.

### Claude-Specific Environment Variables Removed

These env vars are only set by Claude config paths and have no other consumers:

- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_BEDROCK`
- `ANTHROPIC_MODEL`
- `ANTHROPIC_BASE_URL`
- `ANTHROPIC_VERTEX_PROJECT_ID`
- `CLOUD_ML_REGION`

## What's NOT Being Removed

### Untouched Providers

- `GeminiProvider` (`google-adk`) — no changes
- `OpenAIProvider` (`openai-agents`) — no changes

### Untouched Config Paths

| `LIGHTSPEED_PROVIDER` | `MODEL_PROVIDER` | SDK | Status |
|---|---|---|---|
| `vertex` | `google` | `"gemini"` | Unaffected |
| `vertex` | `openai` | `"openai"` | Unaffected |
| `openai` | — | `"openai"` | Unaffected |
| `azure` | — | `"openai"` | Unaffected |

### Anthropic Model Support

Anthropic model support remains available through:
- Classic OLS (`lightspeed-service`) which has its own LLM provider abstraction
- Future rerouting of `vertex/anthropic` and `bedrock` paths to alternative agentic SDKs (deferred to implementation planning)

### Unaffected Spec Files

- `what/run-api.md`, `what/health-probes.md`, `what/e2e-testing.md` — provider-agnostic
- Parent specs: `agentic-proposals.md`, `agentic-security.md`, `query-pipeline.md`

## Spec Files Requiring Updates

### In `lightspeed-agentic-sandbox`

| Spec File | Update |
|---|---|
| `what/provider-contract.md` | Remove Claude from provider list. Note three broken config paths pending rerouting as `[PLANNED: OLS-3473]`. |
| `what/configuration.md` | Remove Claude-specific env vars. Mark `anthropic`, `vertex/anthropic`, `bedrock` paths as `[PLANNED: OLS-3473]` pending rerouting. |
| `how/provider-architecture.md` | Remove `claude.py` from module map. Update data flow to reflect two providers (Gemini, OpenAI). |
| `CLAUDE.md` | Remove Claude from architecture table, env vars table, dependency files table (`package.json`/`package-lock.json`). |
| `AGENTS.md` | Remove Claude provider references. |

### In Parent Workspace (`.ai/spec/`)

| Spec File | Update |
|---|---|
| `what/system-overview.md` | Update agentic-sandbox description to note two providers (Gemini, OpenAI). Add `[PLANNED: OLS-3473]` for Anthropic model rerouting. |

## Acceptance Criteria

1. `@anthropic-ai/claude-code` npm package, `package.json`, `package-lock.json`, and Node.js runtime are removed from the Containerfile
2. `claude-agent-sdk` is removed from Python dependencies
3. `ClaudeProvider` and its factory/config entries are removed
4. No proprietary-licensed binaries remain in the final image layer
5. Container image builds successfully
6. Gemini and OpenAI providers continue to function (agent loop, tool calling, skills, structured output)
7. All spec files listed above are updated to reflect the removal
8. Broken config paths (`anthropic`, `vertex/anthropic`, `bedrock`) are marked `[PLANNED: OLS-3473]` in specs
