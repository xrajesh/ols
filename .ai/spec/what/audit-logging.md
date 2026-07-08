# Compliance Audit Logging

Durable, reconstructable audit trail of AI actions across the OpenShift Lightspeed system. Required by EU AI Act and similar regulations. The agentic system (takes cluster actions) is highest priority; OLS (makes recommendations via troubleshoot mode) is also in scope.

## Requirements & Principles

1. **Dual emission.** Both OTEL spans and structured JSON to stdout on every audit event. OTEL provides distributed tracing when a collector is available; structured JSON provides durable logs via OpenShift log aggregation (always available).

2. **Graceful degradation.** OTEL exporter endpoint is optional on both `OLSConfig` and `AgenticOLSConfig` CRs. When unconfigured, a no-op exporter is used. Structured JSON always emits regardless.

3. **On/off, full fidelity.** Audit logging is enabled or disabled per CR (`OLSConfig`, `AgenticOLSConfig`). When enabled, everything emits at full fidelity — every LLM turn, every tool call input/output, every thinking block. No verbosity dial. This is an audit trail, not operational logging.

4. **No redaction of audit logs.** Redaction is an input-to-LLM concern, not a logging concern.

5. **CR serialization is the compliance record.** Ephemeral Kubernetes CRs are serialized into the log stream at creation (immutable Result CRs) and at mutation (AgenticRunApproval on PATCH, AgenticRun status at phase transitions). Serialization includes `.spec` plus `metadata.name`, `metadata.namespace`, `metadata.creationTimestamp`, and `metadata.uid` — not the full Kubernetes metadata. The log system is the durable record, not etcd. CR serialization always emits when audit logging is enabled.

6. **Sandbox/service logs are the forensic record.** Real-time agent events capture the process (LLM text output, thinking, tool calls). CR serialization captures the decisions. Both are required; overlap is intentional; they serve different audiences.

7. **Human approval identity.** Mutating admission webhook on AgenticRunApproval PATCH injects authenticated user identity (`uid`, `username`) from the admission review. Authoritative for all paths (console, kubectl, API). Console populates approval decision fields; webhook adds/overwrites identity fields. See Mutating Admission Webhook section.

8. **No console-side audit events.** Both consoles are presentation layers. Every consequential action creates a CR or makes an API call captured by the receiving backend.

9. **Log size is the aggregator's problem.** Full CR content (`.spec` + select metadata) is serialized without truncation.

## Correlation Model

### Agentic System

One key on every audit log line and span:

- **`trace_id`** — the AgenticRun CR's `metadata.uid` with hyphens stripped to produce a 32-char hex string. Serves as unique identity, OTEL trace ID, and sole correlation key. Survives AgenticRun name reuse after deletion. Persists across operator restarts (read from the CR). Propagated to sandbox via W3C `traceparent` header on `/v1/agent/run` calls.

Note: agentic events do not carry a `user_id` — AgenticRuns are created by the alerts-adapter (a service account), not a human. The human identity enters the audit trail at approval time via the mutating webhook (`audit.approval.received`).

### OLS (lightspeed-service)

Two keys on every audit log line and span:

- **`trace_id`** — the `conversation_id` UUID with hyphens stripped. Links all events within a conversation across turns. Also used as OTEL trace ID.
- **`user_id`** — authenticated user identity from k8s token validation. Present on every audit event.

### CR Serialization Metadata Fields

All serialized CRs include: `metadata.name`, `metadata.namespace`, `metadata.creationTimestamp`, `metadata.uid`, plus `.spec`.

## Agentic Audit Event Catalog

### Operator Events

Emitted at each phase transition during AgenticRun reconciliation. Each carries `trace_id` (= AgenticRun `metadata.uid`, hyphens stripped) and the serialized CR content.

| Event | When | Payload |
|---|---|---|
| `audit.agenticrun.received` | New AgenticRun CR detected | AgenticRun `.spec` + select metadata |
| `audit.analysis.completed` | AnalysisResult CR created | AnalysisResult serialization (all RemediationOptions) |
| `audit.approval.received` | AgenticRunApproval PATCH observed | Approver `uid`/`username` (webhook-injected), selected option, full text of selected option |
| `audit.execution.completed` | ExecutionResult CR created | ExecutionResult serialization (all ActionsTaken) |
| `audit.verification.completed` | VerificationResult CR created, checks passed | VerificationResult serialization |
| `audit.verification.retry` | Verification failed, retrying execution+verification | VerificationResult serialization, retry count |
| `audit.escalation.completed` | EscalationResult CR created | EscalationResult serialization |
| `audit.agenticrun.terminal` | AgenticRun reaches terminal phase (Completed, Failed, Denied, Escalated) | Final phase, terminal reason |

### Sandbox Events

Emitted in real-time from the SDK event stream during agent execution. Each carries `trace_id` (received via `traceparent` header from operator). The sandbox does not run its own agent loop — it consumes events from the provider SDK's internal agentic loop (Claude `query()`, OpenAI `Runner.run_streamed()`, Gemini `Runner.run_async()`).

| Event | When | Payload | Notes |
|---|---|---|---|
| `audit.agent.started` | Before SDK agent call begins | Phase, model, provider | |
| `audit.agent.text` | SDK yields complete text block | Text content | LLM's visible reasoning between tool calls. Buffered per-message, not per-token. |
| `audit.agent.thinking` | SDK yields thinking delta | Thinking content | Claude only; OpenAI/Gemini do not expose thinking blocks. |
| `audit.agent.tool.call` | SDK yields tool call event | Tool name, input arguments | All three SDKs expose this. |
| `audit.agent.tool.result` | SDK yields tool result event | Tool name, output, success/failure | All three SDKs expose this. |
| `audit.agent.completed` | SDK run finishes | Success/failure, total tokens, total cost | Token counts at run level only — per-turn counts not available from SDKs. |

## OLS Audit Event Catalog

Every event carries `trace_id` (= `conversation_id` hyphens stripped) and `user_id`.

| Event | When | Payload |
|---|---|---|
| `audit.request.started` | Request enters the service | Mode (ask/troubleshooting), query text, attachments, provider/model |
| `audit.request.auth` | User authenticated | User identity from k8s token |
| `audit.rag.retrieved` | RAG chunks retrieved | Chunk count, similarity scores, source documents |
| `audit.history.retrieved` | Conversation history loaded | Turn count, compressed (yes/no) |
| `audit.llm.turn` | Each LLM turn completes | Turn index, token counts (input/output), cost |
| `audit.llm.thinking` | LLM emits reasoning/thinking | Thinking content |
| `audit.llm.text` | LLM emits text output | Text content |
| `audit.tool.call` | Tool invocation starts | Tool name, MCP server, input arguments |
| `audit.tool.result` | Tool invocation completes | Tool name, output, success/failure, duration |
| `audit.tool.approval.requested` | Tool requires human approval | Tool name, approval ID |
| `audit.tool.approval.decision` | User approves/denies tool | Approval ID, decision, tool name |
| `audit.request.completed` | Response fully streamed | Total turns, total tokens, total cost, referenced documents |

Note: OLS runs its own tool-calling loop (not an SDK agentic loop), so per-turn token counts are available.

## OTEL Span Hierarchy

### Agentic System

```
agenticrun.lifecycle            [operator, root, trace_id = AgenticRun metadata.uid]
├── agenticrun.analyze          [operator]
│   └── agent.run               [sandbox, via traceparent header]
│       └── agent.turn          [sandbox]
│           └── tool.{name}     [sandbox]
├── agenticrun.human_approval   [operator]
├── agenticrun.execute          [operator]
│   └── agent.run               [sandbox, via traceparent header]
│       └── agent.turn          [sandbox]
│           └── tool.{name}     [sandbox]
├── agenticrun.verify           [operator]
│   └── agent.run               [sandbox, via traceparent header]
│       └── agent.turn          [sandbox]
│           └── tool.{name}     [sandbox]
└── agenticrun.escalate         [operator]
    └── agent.run               [sandbox, via traceparent header]
```

On retry (verification failure → re-execute), new `agenticrun.execute` and `agenticrun.verify` child spans are created under the same root. The retry index is a span attribute.

`agenticrun.human_approval` is a span that starts when the operator begins waiting for approval and ends when the AgenticRunApproval PATCH is observed. Duration = human decision time.

### OLS (lightspeed-service)

```
request.lifecycle               [service, root, trace_id = conversation_id]
├── request.auth                [service]
├── request.rag                 [service]
├── request.history             [service]
├── llm.turn                    [service, repeats per turn]
│   └── tool.{name}             [service, repeats per tool call]
└── request.store               [service]
```

For multi-turn conversations, each request is a separate trace sharing the same `conversation_id` as trace ID. Multiple requests in a conversation produce traces with the same trace ID — spans are uniquely identified by trace_id + span_id, so querying by trace ID returns the full conversation.

## Mutating Admission Webhook

### Purpose

Inject authenticated user identity into AgenticRunApproval on PATCH. Serves two needs: audit logging (emit `audit.approval.received` with identity) and UI display (persist identity on the CR).

### Mechanics

- **Resource:** `agenticrunapprovals.agentic.openshift.io/v1alpha1`
- **Operation:** `PATCH`
- **Action:**
  1. Read `request.userInfo.username` and `request.userInfo.uid` from the AdmissionReview.
  2. Write `spec.approver.uid`, `spec.approver.username`, `spec.approver.timestamp` into the CR, overwriting any client-submitted values.
  3. Emit `audit.approval.received` log event with user identity and `trace_id` (AgenticRun's `metadata.uid`, read from the CR's owner reference).
- **Hosted by:** The agentic-operator controller-manager (same process, same logging/OTEL infrastructure).
- **Failure mode:** Fail-closed — if the webhook is unavailable, the API server rejects the PATCH. Correct default for a compliance-critical path.
- **TLS:** Webhook certificate managed by the operator's existing cert infrastructure.

### CRD Change Required

Add `spec.approver` to AgenticRunApproval:

```yaml
spec:
  approver:
    uid: ""         # from userInfo.uid — webhook-authoritative
    username: ""    # from userInfo.username — webhook-authoritative
    timestamp: ""   # server-side time.Now() — webhook-authoritative
```

### Console Responsibility

The agentic console populates approval decision fields on the PATCH request (selected option, max retries, stage). It does not need to populate identity fields — the webhook handles that. If the console does populate them, the webhook overwrites them.

## Configuration Surface

### OLSConfig CR (single source for all telemetry config)

```yaml
spec:
  audit:                                   # optional block; omitting = audit enabled, no OTEL export
    enabled: true                          # default: true (audit on even if spec.audit is absent)
    otel:
      endpoint: ""                         # optional external tracing endpoint; Collector forwards traces here
  templog: true                            # default: true. Controls postgres exporter in the Collector.
```

All audit, tracing, and templog configuration lives on `OLSConfig`. `AgenticOLSConfig` has no audit, tracing, or templog fields — it is limited to agentic-specific concerns (agents, policies, suspension).

### Defaults

If `spec.audit` is absent entirely, behavior is `enabled: true` with no trace forwarding. Structured JSON audit events emit to stdout. The user must explicitly set `enabled: false` to disable.

### Propagation

- The lightspeed-operator reads `OLSConfig.spec.audit` and generates the corresponding config in `olsconfig.yaml` for lightspeed-service to consume.
- The lightspeed-operator propagates audit config to agentic-operator and sandbox pods via env vars (audit enabled flag, Collector endpoint). The agentic-operator does not read audit config from `AgenticOLSConfig`.
- The lightspeed-operator generates the OTel Collector ConfigMap, enabling/disabling the postgres exporter (based on `spec.templog`) and the trace forwarding exporter (based on `spec.audit.otel.endpoint`).
- Structured JSON always goes to stdout — this is what any log aggregator (Loki, Splunk, Fluentd, etc.) reads from container logs.
- OTLP telemetry (logs + traces) always flows to the Collector. The Collector routes based on its config.

### Auto-Detection

Auto-detection of OpenShift logging OTLP endpoints: [DEFERRED].

## Structured JSON Log Format

Every audit event emits as a single JSON line to stdout. Consistent format across all components.

### Agentic Operator Event

```json
{
  "timestamp": "2026-06-11T14:30:00.000Z",
  "level": "info",
  "event": "audit.analysis.completed",
  "trace_id": "f47ac10b58cc4372a5670e02b2c3d479",
  "payload": {
    "metadata": {
      "name": "fix-nginx-abc12345-analysis-1",
      "namespace": "openshift-monitoring",
      "creationTimestamp": "2026-06-11T14:30:00Z",
      "uid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
    },
    "spec": {}
  }
}
```

### Sandbox Agent Events

```json
{
  "timestamp": "2026-06-11T14:30:00.000Z",
  "level": "info",
  "event": "audit.agent.started",
  "trace_id": "f47ac10b58cc4372a5670e02b2c3d479",
  "phase": "analysis",
  "model": "claude-sonnet-4-20250514",
  "provider": "anthropic"
}
```

```json
{
  "timestamp": "2026-06-11T14:30:01.000Z",
  "level": "info",
  "event": "audit.agent.text",
  "trace_id": "f47ac10b58cc4372a5670e02b2c3d479",
  "phase": "analysis",
  "content": "The pod is in CrashLoopBackOff due to OOMKilled. Let me check the resource limits."
}
```

```json
{
  "timestamp": "2026-06-11T14:30:01.500Z",
  "level": "info",
  "event": "audit.agent.thinking",
  "trace_id": "f47ac10b58cc4372a5670e02b2c3d479",
  "phase": "analysis",
  "content": "I should check memory limits on the container spec..."
}
```

```json
{
  "timestamp": "2026-06-11T14:30:02.000Z",
  "level": "info",
  "event": "audit.agent.tool.call",
  "trace_id": "f47ac10b58cc4372a5670e02b2c3d479",
  "phase": "analysis",
  "tool": "Bash",
  "input": "kubectl get pods -n openshift-monitoring"
}
```

```json
{
  "timestamp": "2026-06-11T14:30:03.000Z",
  "level": "info",
  "event": "audit.agent.tool.result",
  "trace_id": "f47ac10b58cc4372a5670e02b2c3d479",
  "phase": "analysis",
  "tool": "Bash",
  "output": "NAME                    READY   STATUS             RESTARTS\nnginx-7d4b8c6f-x2k9p   0/1     CrashLoopBackOff   5",
  "success": true,
  "duration_ms": 820
}
```

```json
{
  "timestamp": "2026-06-11T14:30:10.000Z",
  "level": "info",
  "event": "audit.agent.completed",
  "trace_id": "f47ac10b58cc4372a5670e02b2c3d479",
  "phase": "analysis",
  "success": true,
  "total_tokens_in": 12600,
  "total_tokens_out": 2400,
  "total_cost": 0.068
}
```

### OLS Service Event

```json
{
  "timestamp": "2026-06-11T14:30:00.000Z",
  "level": "info",
  "event": "audit.llm.turn",
  "trace_id": "d4e5f6a7b8c90d1e2f3a4b5c6d7e8f9a",
  "user_id": "admin@corp",
  "turn_index": 1,
  "tokens_in": 4200,
  "tokens_out": 850,
  "cost": 0.023
}
```

### Conventions

- All components use the same top-level fields: `timestamp`, `level`, `event`.
- `event` is the type discriminator — consumers filter on this (e.g. `event =~ "audit.agenticrun.*"` for compliance, `event =~ "audit.agent.*"` for forensics).
- `trace_id` on every event across all components.
- OLS events additionally carry `user_id`.
- Payloads vary by event type; the event catalogs define what each carries.
- Output format is mandated; logging library choice is left to each repo (Go: logr+zap already produces structured JSON; Python: repo discretion).

## Repo Ownership

| Repo | Audit Responsibilities |
|---|---|
| **lightspeed-agentic-operator** | Emit `audit.agenticrun.*` events at phase transitions with CR serialization. Host mutating admission webhook for AgenticRunApproval PATCH (inject identity, emit `audit.approval.received`). Create OTEL root span (`agenticrun.lifecycle`) using `metadata.uid` as trace ID. Propagate trace context to sandbox via `traceparent` header. Read audit config from env vars set by lightspeed-operator (not from `AgenticOLSConfig`). CRD change: add `spec.approver` to AgenticRunApproval. |
| **lightspeed-agentic-sandbox** | Emit `audit.agent.*` events from SDK event stream (text, thinking, tool calls, tool results, started, completed). Receive trace context from operator via `traceparent` header. Use `trace_id` on all events. |
| **lightspeed-service** | Emit `audit.request.*`, `audit.llm.*`, `audit.rag.*`, `audit.history.*`, `audit.tool.*` events. Ensure `conversation_id` and `user_id` on every event. Create OTEL root span using `conversation_id` as trace ID. Read audit config from `olsconfig.yaml`. |
| **lightspeed-otel-collector** | Custom OTel Collector built with OCB. Receives all OTLP telemetry from agentic components. Routes logs to Postgres (when templog enabled) and forwards traces to external endpoint (when configured). |
| **lightspeed-operator** | CRD: `spec.audit` and `spec.templog` on `OLSConfig` (single source for all telemetry config). Deploy Collector. Generate Collector ConfigMap. Propagate audit config to `olsconfig.yaml` for lightspeed-service and via env vars to agentic pods. |
| **lightspeed-agentic-console** | Populate approval decision fields on AgenticRunApproval PATCH (selected option, max retries, stage). Display `spec.approver` fields in UI. No audit emission responsibility. |
| **lightspeed-console** | No changes. No audit emission responsibility. |

## Child Spec Updates Required

Each child repo needs an audit logging spec with implementation details. The parent spec (this file) is authoritative for the "what" (requirements, event semantics, correlation contract). Child specs are authoritative for the "how" (implementation within that repo).

| Repo | Child Spec File | Content |
|---|---|---|
| lightspeed-agentic-operator | `what/audit-logging.md` | Operator audit event implementation, webhook implementation, OTEL span creation, CR serialization logic, CRD changes |
| lightspeed-agentic-sandbox | `what/audit-logging.md` | SDK event stream instrumentation per provider (Claude, OpenAI, Gemini), trace context reception, audit event emission |
| lightspeed-service | `what/audit-logging.md` | Request lifecycle instrumentation, conversation_id/user_id propagation, OTEL setup, audit config consumption |
| lightspeed-operator | `what/audit-logging.md` | OLSConfig CRD audit fields, olsconfig.yaml generation for audit config |

## Cross-References

- `agentic-runs.md` — AgenticRun lifecycle, CRD definitions, phase transitions
- `agentic-security.md` — Approval authorization (cluster-admin gate), per-run SA isolation
- `query-pipeline.md` — OLS request processing stages, streaming events

## Planned Changes

| Ticket | Summary |
|---|---|
| [PLANNED] | Auto-detection of OpenShift logging OTLP endpoint |
| OLS-3295 | Rename `Proposal` → `AgenticRun`, `ProposalApproval` → `AgenticRunApproval` across audit events and OTEL spans |
| OLS-3328 | Temporary audit log storage in PostgreSQL via custom OTel Collector (see `templog.md`) |
