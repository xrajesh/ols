# OTel GenAI Semantic Conventions Alignment

Design for aligning OpenShift Lightspeed telemetry (traces, metrics, structured logs) with the [OpenTelemetry GenAI Semantic Conventions](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/README.md) (v1.41).

**Jira**: OLS-3493
**Related**: PR [openshift/lightspeed-agentic-operator#281](https://github.com/openshift/lightspeed-agentic-operator/pull/281) (RFC: audit logging and tracing improvements)

## Scope

**In scope** (OLS-3493 gap numbers):

- Gap 1: Span names and attributes → GenAI standard naming
- Gap 2: Span kind specification (`CLIENT`/`INTERNAL`)
- Gap 3: Metrics instrument types and naming (Histograms, `gen_ai.*` names)
- Gap 4: MCP semantic conventions (OLS now, sandbox [PLANNED])
- Gap 5: Structured log field names → GenAI attribute names
- Gap 6: Request vs. response model distinction
- Gap 10: Conversation/trace ID separation

**Deferred** (need their own Jira issues):

- Gap 7: Content capture controls (three-mode opt-in)
- Gap 8: Evaluation events (`gen_ai.evaluation.result`)
- Gap 9: Cache token attributes (`gen_ai.usage.cache_read.input_tokens`)

## Design Decisions

### 1. Correlation Model — Per-Phase Traces

**Agentic system**: Replace single-trace-per-lifecycle with per-phase traces linked by `proposal.uid` attribute.

- Each phase (analysis, execution, verification, escalation) gets its own trace with a fresh trace ID.
- `proposal.uid` (the CR's `metadata.uid`, hyphens stripped) becomes a span attribute on every span in every phase trace — the cross-trace correlation key.
- `proposal.name` and `proposal.namespace` also carried as span attributes.
- The operator creates the root span for each phase and propagates trace context to the sandbox via `traceparent` header — same propagation pattern as today.
- Human approval is recorded as a standalone trace (short-lived — just the approval event, not the wait time). Wait duration is derived from timestamps between analysis-completed and approval-received traces.
- Span Links: each phase trace's root span includes a Span Link back to the prior phase's root span.
- On retry (verification failure → re-execute), new traces are created. Retry index is a span attribute.
- Users query by `proposal.uid = X` to see all phase traces.

**OLS**: Each HTTP request gets its own trace ID. `conversation_id` becomes `gen_ai.conversation.id` span attribute.

- Each incoming request generates a fresh trace ID.
- `gen_ai.conversation.id` = `conversation_id` as a span attribute on every span.
- `user_id` remains a span attribute on every span.
- Users query by `gen_ai.conversation.id` to see all requests in a conversation.

**Rationale**: A single trace spanning hours/days (waiting for human approval) is pathological for trace backends. Per-phase traces are compact, meaningful, and render well. The same pattern applies to OLS — per-request traces with conversation linkage via attribute.

### 2. Span Naming and Attributes

**Operator spans**: Keep `proposal.*` naming. These are Kubernetes workflow orchestration spans, not GenAI inference spans. Add `gen_ai.request.model` and `gen_ai.provider.name` where the operator knows the model/provider being sent to the sandbox.

**Sandbox spans**: Full OTel GenAI naming.

Inference span (`agent.run` → `chat {model}`):

| Attribute | Requirement | Description |
|---|---|---|
| `gen_ai.operation.name` | Required | `"chat"` |
| `gen_ai.request.model` | Required | Model name requested |
| `gen_ai.response.model` | Recommended | Actual model from SDK response |
| `gen_ai.provider.name` | Required | Provider name |
| `gen_ai.usage.input_tokens` | Recommended | Input token count |
| `gen_ai.usage.output_tokens` | Recommended | Output token count |
| `proposal.uid` | Required (custom) | Correlation key |
| `server.address` | Recommended | LLM API endpoint |

Tool execution span (`tool.{name}` → `execute_tool {tool_name}`):

| Attribute | Requirement | Description |
|---|---|---|
| `gen_ai.operation.name` | Required | `"execute_tool"` |
| `gen_ai.tool.name` | Required | Tool name |
| `gen_ai.tool.call.id` | Recommended | Tool call ID from SDK |
| `gen_ai.tool.type` | Recommended | `"function"` |

**OLS spans**: The `llm.turn` span becomes `chat {model}` with full GenAI attributes (same as sandbox inference span). Tool spans become `execute_tool {tool_name}` (same as sandbox). The root `request.lifecycle` span keeps its current name — it encompasses the full HTTP request (auth, RAG, history, LLM turns, storage), not just the LLM call. Non-GenAI child spans (`request.auth`, `request.rag`, `request.history`, `request.store`) also keep their current names. All non-GenAI spans use span kind `INTERNAL`.

### 3. Span Kinds

| Span | Kind | Rationale |
|---|---|---|
| Operator phase spans (`proposal.*`) | `INTERNAL` | In-process workflow orchestration |
| Sandbox inference span (`chat {model}`) | `CLIENT` | Calling external LLM API |
| Sandbox tool span (`execute_tool {name}`) | `INTERNAL` | In-process tool execution |
| OLS request lifecycle (`request.lifecycle`) | `INTERNAL` | In-process HTTP request handler |
| OLS LLM turn (`chat {model}`) | `CLIENT` | Calling external LLM API |
| OLS tool span (`execute_tool {name}`) | `INTERNAL` | In-process tool execution |
| OLS non-GenAI spans (`request.auth`, `request.rag`, etc.) | `INTERNAL` | In-process operations |

### 4. Structured Log Format — OTel JSON

Audit structured log emission becomes an OTel JSON span/event payload. Two exporters on the same TracerProvider:

1. **OTLP exporter** → trace backend (when endpoint configured)
2. **Stdout exporter** → OTLP JSON to stdout (always, when audit enabled)

Both Go and Python OTel SDKs ship stdout/console exporters that produce OTLP JSON natively.

#### Single-Emission Rule

Each audit-significant datum is recorded exactly once, as an OTel span or span event. The stdout and OTLP exporters are two destinations for the same emission, not two separate emission paths. Application-level loggers (Go `logr`, Python `logging`) emit only developer-debugging messages and MUST NOT re-emit data that appears in spans or span events.

This collapses the sandbox's current triple-emission (standard logging, JSON audit, OTEL span) into:
- OTel spans/events → audit (two exporters, one emission)
- Standard logging → developer debugging only (non-audit, non-structured)

And the operator's dual-emission (JSON audit + span event) into:
- OTel spans/events → audit
- Standard logging → developer debugging only

#### CR Serialization Model

Operator CR payloads (AnalysisResult, ExecutionResult, etc.) use a split model:

- **Key fields → span attributes** (queryable): `result.name`, `result.uid`, `options.count`, `phase`, `terminal.reason`
- **Full CR serialization → event attributes** (viewable, full fidelity): complete `.spec` + `.status` + select metadata as a single event attribute. Event names: `proposal.analysis.completed`, `proposal.execution.completed`, etc.

The stdout exporter does NOT truncate — full fidelity is preserved. The OTLP exporter may truncate based on backend limits, but the stdout signal is the compliance record.

### 5. Metrics

**Agentic sandbox** (greenfield — no existing metrics):

| Metric | Type | Unit | Labels |
|---|---|---|---|
| `gen_ai.client.token.usage` | Histogram | `{token}` | `gen_ai.token.type` (input/output), `gen_ai.request.model`, `gen_ai.provider.name` |
| `gen_ai.client.operation.duration` | Histogram | `s` | `gen_ai.request.model`, `gen_ai.provider.name`, `gen_ai.operation.name` |
| `gen_ai.execute_tool.duration` | Histogram | `s` | `gen_ai.tool.name` |

Token usage histogram bucket boundaries: `[1, 4, 16, 64, 256, 1024, 4096, 16384, 65536]` (per semconv recommendation).

Sandbox will need a `/metrics` endpoint added.

**Agentic operator**: No new gen_ai.* metrics. The operator does not make LLM calls. Controller-runtime default metrics remain.

**OLS** (gen_ai.* alongside existing ols_*):

New metrics added:

| Metric | Type | Unit | Labels |
|---|---|---|---|
| `gen_ai.client.token.usage` | Histogram | `{token}` | `gen_ai.token.type` (input/output/reasoning), `gen_ai.request.model`, `gen_ai.provider.name` |
| `gen_ai.client.operation.duration` | Histogram | `s` | `gen_ai.request.model`, `gen_ai.provider.name`, `gen_ai.operation.name` |
| `gen_ai.execute_tool.duration` | Histogram | `s` | `gen_ai.tool.name` |

Existing `ols_*` metrics kept unchanged for backward compatibility. `gen_ai.client.token.usage` histogram supersedes `ols_llm_token_sent_total`/`ols_llm_token_received_total` for distribution analysis, but counters remain available.

OLS-1279 (planned LLM-only duration metric) is subsumed by `gen_ai.client.operation.duration`.

**Streaming metrics** (`gen_ai.client.operation.time_to_first_chunk`, `gen_ai.client.operation.time_per_output_chunk`): [PLANNED] for OLS when streaming is the default path. Not applicable to the sandbox.

### 6. MCP Semantic Conventions

**OLS** (active — MCP in use today):

Tool spans for MCP-sourced tools get additional attributes:

| Attribute | Requirement | Description |
|---|---|---|
| `mcp.method.name` | Recommended | MCP method invoked (e.g., `tools/call`) |
| `mcp.session.id` | Recommended | MCP session identifier |
| `mcp.protocol.version` | Recommended | MCP protocol version |
| `gen_ai.tool.call.id` | Recommended | Tool call ID from MCP response |
| `network.transport` | Recommended | `stdio` or `sse` |

Non-MCP tools get standard `gen_ai.tool.*` attributes only.

MCP-specific metrics (`mcp.client.operation.duration`, `mcp.client.session.duration`): [PLANNED] pending sufficient usage data.

**Sandbox** ([PLANNED] — no MCP implementation today):

When MCP support lands, MCP tool spans follow the same attribute schema as OLS. The conventions are defined now so MCP ships with correct instrumentation from day one.

### 7. Backward Compatibility

**Clean cut** on all naming. Audit logging has not shipped/GA'd, so there are no downstream consumers to break. Specs define only the new OTel GenAI names. Old custom names (`tokens_in`, `tokens_out`, `tool`, `provider`, `model`) are removed.

Exception: OLS `ols_*` Prometheus metrics are kept alongside new `gen_ai.*` metrics since existing OLS users may have dashboards/alerts on them.

## Affected Spec Files

| File | Changes |
|---|---|
| `.ai/spec/what/audit-logging.md` | Correlation model rewrite, span hierarchy rewrite, structured JSON format rewrite (OTel JSON), single-emission rule, OTel GenAI attribute reference |
| `lightspeed-service/.ai/spec/what/audit-logging.md` | Span names/kinds/attributes, per-request trace ID, MCP conventions |
| `lightspeed-service/.ai/spec/what/observability.md` | Add gen_ai.* histogram metrics, update OLS-1279 reference |
| `lightspeed-agentic-operator/.ai/spec/what/audit-logging.md` | Remove lifecycle root span, per-phase trace roots, span kinds, span links, CR serialization model, OTel JSON format |
| `lightspeed-agentic-sandbox/.ai/spec/what/audit-logging.md` | GenAI span names/attributes, span kinds, OTel JSON format, single-emission rule, gen_ai.* metrics, MCP [PLANNED] |

## References

- [OTel GenAI Semantic Conventions v1.41](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/README.md)
- [OTel MCP Semantic Conventions (v1.39+)](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/mcp.md)
- [OLS-3493 — Gap analysis](https://redhat.atlassian.net/browse/OLS-3493)
- [PR #281 — RFC: Audit logging and tracing improvements](https://github.com/openshift/lightspeed-agentic-operator/pull/281)
