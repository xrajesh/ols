# Temporary Audit Log Storage

Stopgap audit log persistence for environments without cluster logging or SIEM. Stores agentic system audit events in the existing PostgreSQL instance via a custom OpenTelemetry Collector built with OCB. Logs are tied to AgenticRun lifecycle — deleted when the AgenticRun CR is deleted.

## Requirements & Principles

1. **Agentic system only.** This feature stores audit events from the agentic-operator and agentic-sandbox. OLS service audit events are out of scope.

2. **Collector always deployed.** The OTel Collector is always deployed by the lightspeed-operator. All agentic audit events and traces flow through it. `OLSConfig.spec.templog` controls only whether the Collector's postgres exporter is active — it does not control the Collector's existence.

3. **Templog default on.** `OLSConfig.spec.templog` defaults to `true`. When `true` (or absent), the Collector writes logs to PostgreSQL. When `false`, the Collector runs but does not write to PostgreSQL.

4. **Stdout always emits.** Structured JSON audit events to stdout are non-negotiable. Every agentic component always logs to stdout regardless of Collector or templog state. This is required for Kubernetes-level log access (`oc logs`, container log infrastructure).

5. **Collector is the telemetry hub.** All OTLP telemetry (logs and traces) from agentic components flows to the Collector. If the user configures a tracing endpoint (`OLSConfig.spec.audit.otel.endpoint`), the Collector forwards traces there. The Collector routes; the components emit to one place.

6. **Run-scoped lifecycle.** Audit logs are tied to their AgenticRun CR. When an AgenticRun is deleted, a finalizer ensures all associated rows are deleted from PostgreSQL before the CR is removed. No separate retention policy, TTL, or eviction logic.

7. **Configuration lives on OLSConfig.** All audit, tracing, and templog configuration lives on the `OLSConfig` CR. The lightspeed-operator owns all infrastructure deployment (Collector, Postgres, agentic pods) and propagates config to agentic components via env vars. `AgenticOLSConfig` has no audit/tracing/templog fields.

8. **Stopgap, not strategic.** This feature exists for customers who lack external log aggregation. It is not a replacement for a proper SIEM or log management solution. The schema and query surface are intentionally minimal.

## Architecture

```
┌──────────────────────┐    ┌──────────────────────┐
│  agentic-operator    │    │  agentic-sandbox     │
│  (stdout + OTLP)     │    │  (stdout + OTLP)     │
└─────────┬────────────┘    └─────────┬────────────┘
          │ OTLP/gRPC                 │ OTLP/gRPC
          │ (logs + traces)           │ (logs + traces)
          └──────────┬────────────────┘
                     ▼
          ┌──────────────────────────────────┐
          │  Custom OTel Collector (OCB)     │
          │  ┌────────────┐  ┌────────────┐  │
          │  │ logs       │  │ traces     │  │
          │  │ pipeline   │  │ pipeline   │  │
          │  └─────┬──────┘  └─────┬──────┘  │
          │        │               │         │
          │   postgresexp     otlpexporter   │
          │   (if templog)    (if endpoint)   │
          └────────┼───────────────┼─────────┘
                   ▼               ▼
          ┌──────────────┐  ┌──────────────┐
          │  PostgreSQL  │  │  External    │
          │  templogs    │  │  tracing     │
          │  schema      │  │  (Jaeger,    │
          └──────────────┘  │   Tempo)     │
                            └──────────────┘
```

Components:
- **Agentic-operator** and **agentic-sandbox** always emit to stdout AND send OTLP (logs + traces) to the Collector.
- **Custom OTel Collector** is always deployed. Routes telemetry based on operator-generated config:
  - **Logs pipeline:** OTLP receiver → `postgresexporter` (active when `spec.templog: true`, inactive when `false`)
  - **Traces pipeline:** OTLP receiver → `otlpexporter` (active when `spec.audit.otel.endpoint` is set, inactive when absent)
- **PostgreSQL** stores audit logs in the `templogs` schema when templog is enabled.

## Configuration Surface

### OLSConfig CR

```yaml
spec:
  audit:                                   # optional block; omitting = audit enabled
    enabled: true                          # default: true
    otel:
      endpoint: ""                         # optional external tracing endpoint; Collector forwards traces here
  templog: true                            # default: true. Controls postgres exporter in the Collector.
```

All telemetry configuration is on `OLSConfig`. The lightspeed-operator reads it and:
- Generates the Collector ConfigMap (enabling/disabling exporters based on `spec.templog` and `spec.audit.otel.endpoint`)
- Sets env vars on agentic-operator and sandbox pods (Collector endpoint, audit enabled flag)

`AgenticOLSConfig` has no audit, tracing, or templog fields.

### Interaction Matrix

| `spec.templog` | `spec.audit.enabled` | `spec.audit.otel.endpoint` | Behavior |
|---|---|---|---|
| true (or absent) | true (or absent) | absent | Collector deployed. Logs → Postgres. No trace forwarding. Stdout always. |
| true (or absent) | true (or absent) | set | Collector deployed. Logs → Postgres. Traces → external endpoint. Stdout always. |
| true (or absent) | false | any | Collector deployed. No audit events emitted by components. Postgres empty. |
| false | true (or absent) | absent | Collector deployed. Postgres exporter inactive. Logs received but not stored. Stdout always. |
| false | true (or absent) | set | Collector deployed. Postgres exporter inactive. Traces → external endpoint. Stdout always. |
| false | false | any | Collector deployed. No audit events emitted. |

## Schema & Data Model

Single table in a `templogs` schema:

```sql
CREATE SCHEMA IF NOT EXISTS templogs;

CREATE TABLE templogs.logs (
    id         BIGSERIAL    PRIMARY KEY,
    trace_id   CHAR(32)     NOT NULL,
    timestamp  TIMESTAMPTZ  NOT NULL,
    event      VARCHAR(128) NOT NULL,
    body       JSONB        NOT NULL
);

CREATE INDEX idx_logs_trace_id ON templogs.logs (trace_id);
```

- **`trace_id`** — AgenticRun `metadata.uid` with hyphens stripped (32-char hex). Primary query and cleanup key. Deterministically derived from the AgenticRun CR, not dependent on any OTEL infrastructure.
- **`timestamp`** — Event timestamp from the OTLP log record.
- **`event`** — Event discriminator (`audit.agenticrun.received`, `audit.agent.tool.call`, etc.). Extracted from log record attributes for filtering without parsing JSONB.
- **`body`** — Full structured JSON audit event as-is. Same content that goes to stdout. No transformation or field extraction beyond the dedicated columns.

The `templogs` schema is always created by the Postgres bootstrap script (regardless of `spec.templog` value).

## Custom OTel Collector

### Build (OCB)

Built with the OpenTelemetry Collector Builder (ocb). The build manifest includes:

- **Receiver:** `otlpreceiver` (standard OTLP gRPC receiver)
- **Exporters:**
  - `postgresexporter` (custom) — for the logs pipeline
  - `otlpexporter` (standard) — for the traces pipeline (forwarding to external endpoint)

### Collector Configuration

Generated by the lightspeed-operator and mounted as ConfigMap. The operator enables/disables pipelines based on `OLSConfig`:

**When `spec.templog: true` and `spec.audit.otel.endpoint` is set:**

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  postgres:
    dsn: "${POSTGRES_DSN}"
    schema: templogs
    table: logs
  otlp/traces:
    endpoint: "${TRACE_ENDPOINT}"

service:
  pipelines:
    logs:
      receivers: [otlp]
      exporters: [postgres]
    traces:
      receivers: [otlp]
      exporters: [otlp/traces]
```

**When `spec.templog: false` and no trace endpoint:**

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

service:
  pipelines:
    logs:
      receivers: [otlp]
      exporters: []
```

The operator regenerates and remounts the ConfigMap when `OLSConfig` changes.

### Deployment

- Single-replica Deployment, always deployed by the lightspeed-operator
- Same management patterns as PostgreSQL (operator-managed image, TLS to Postgres via service-ca certs)
- NetworkPolicy allowing ingress from agentic-operator and sandbox pods on port 4317
- Service exposing port 4317 for OTLP gRPC

### Container Image

Built and shipped from the `lightspeed-otel-collector` repository (shared with multicluster telemetry). Single container image containing the custom Collector binary.

## Collector Repository

**Repo:** `lightspeed-otel-collector`

Contents:
- OCB build manifest (`builder-config.yaml`)
- Custom exporter Go code (`postgresexporter/`)
- Dockerfile
- Build pipeline (Konflux)

Ships one artifact: a container image with the custom Collector binary.

## Operator Wiring (lightspeed-operator)

### Always (Collector is always deployed)

1. Deploy the custom Collector: Deployment, Service, ConfigMap, NetworkPolicy
2. Set OTLP endpoint env var on agentic-operator and sandbox pods, pointing at the Collector service: `<collector-service>.<namespace>.svc:4317`
3. Create `templogs` schema in Postgres bootstrap script

### Based on OLSConfig

4. Generate Collector ConfigMap with pipelines enabled/disabled based on `spec.templog` and `spec.audit.otel.endpoint`
5. Set audit enabled/disabled env var on agentic-operator and sandbox pods based on `spec.audit.enabled`
6. Regenerate Collector ConfigMap when `OLSConfig` changes (triggers Collector restart via annotation tracking)

## AgenticRun Finalizer & Cleanup

### Agentic-operator responsibility

- **Finalizer name:** `agentic.openshift.io/templog-cleanup`
- **Added when:** AgenticRun CR is created and `templog` is enabled (agentic-operator reads this from an env var set by the lightspeed-operator)
- **On AgenticRun deletion:**
  1. Finalizer fires
  2. Operator connects to Postgres: `DELETE FROM templogs.logs WHERE trace_id = $1`
  3. On success, removes the finalizer — CR deletion proceeds
  4. On failure (Postgres unreachable), finalizer blocks deletion and requeues with standard controller-runtime retry and backoff

### Edge cases

- **`templog` disabled after logs were written.** Finalizer was already added at AgenticRun creation. It still fires on deletion. The operator connects directly to Postgres (which it manages) to delete the rows. The finalizer does not depend on the Collector being present.
- **Postgres unavailable.** Finalizer blocks. AgenticRun CR cannot be deleted until cleanup succeeds. Correct behavior for a compliance-adjacent feature.

## Agentic Component Changes

### Agentic-operator

- Always emit audit events to stdout (existing behavior, unchanged)
- Always emit OTLP (logs + traces) to the Collector endpoint (env var set by lightspeed-operator)
- Add `agentic.openshift.io/templog-cleanup` finalizer to new AgenticRuns when templog is enabled
- Finalizer handler: delete audit log rows from Postgres on AgenticRun deletion

### Agentic-sandbox

- Always emit audit events to stdout (existing behavior, unchanged)
- Always emit OTLP (logs + traces) to the Collector endpoint (env var set by lightspeed-operator)

## Repo Ownership

| Repo | Templog Responsibilities |
|---|---|
| **lightspeed-otel-collector** | OCB manifest, custom `postgresexporter` Go code, standard `otlpexporter` for trace forwarding, Dockerfile, Konflux build pipeline. Ships the Collector container image. |
| **lightspeed-operator** | Read `OLSConfig.spec.templog` and `OLSConfig.spec.audit`. Always deploy Collector. Generate Collector ConfigMap with appropriate pipelines. Add `templogs` schema to Postgres bootstrap. Wire OTLP endpoint and audit config env vars to agentic pods. CRD change: add `spec.templog` to `OLSConfig`. |
| **lightspeed-agentic-operator** | Always emit OTLP to Collector. Add `agentic.openshift.io/templog-cleanup` finalizer to AgenticRuns. Finalizer handler: delete rows from `templogs.logs` on AgenticRun deletion. |
| **lightspeed-agentic-sandbox** | Always emit OTLP to Collector. |

## Child Spec Updates Required

| Repo | File | Content |
|---|---|---|
| lightspeed-otel-collector | `what/collector.md` | OCB build, `postgresexporter` implementation, Collector configuration, trace forwarding |
| lightspeed-otel-collector | `what/postgres-exporter.md` | Custom exporter Go implementation, batch insert strategy, schema interaction |
| lightspeed-operator | `what/templog.md` | Collector lifecycle reconciliation, schema bootstrap, pod wiring, ConfigMap generation |
| lightspeed-operator | `what/crd-api.md` (update) | Add `spec.templog` to `OLSConfig`. Remove audit/tracing from `AgenticOLSConfig` docs if present. |
| lightspeed-operator | `what/postgres.md` (update) | Add `templogs` schema to bootstrap |
| lightspeed-agentic-operator | `what/templog.md` | Finalizer implementation, Postgres cleanup |
| lightspeed-agentic-operator | `what/audit-logging.md` (update) | Always emit OTLP to Collector. Remove references to reading audit config from AgenticOLSConfig. |
| lightspeed-agentic-sandbox | `what/audit-logging.md` (update) | Always emit OTLP to Collector. Config received via env vars from lightspeed-operator. |

## Cross-References

- `audit-logging.md` — Audit event catalog, correlation model, structured JSON format
- `agentic-runs.md` — AgenticRun lifecycle, CRD definitions, phase transitions
- Lightspeed-operator `postgres.md` — PostgreSQL deployment, bootstrap, credentials

## Planned Changes

| Ticket | Summary |
|---|---|
| OLS-3328 | Implement temporary audit log storage |
