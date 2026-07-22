# OLS-3696: Templog Phase Storage

**Epic:** [OLS-3696](https://redhat.atlassian.net/browse/OLS-3696)
**Status:** Design
**Risk Level:** 3 (High) — data export schema change + API contract change + cross-repo

## Problem

The `templogs.logs` table has no phase information. The console cannot display or filter agentic run logs by lifecycle phase (analysis, execution, verification, etc.). Additionally, the `trace_id` column name is misleading — it stores the AgenticRun UUID, not an OTel trace ID.

A secondary issue: the current `postgresexporter` extracts `trace_id` from the OTel log record's native `TraceID` field. In the per-phase audit trace model, each phase gets a fresh auto-generated OTel trace ID (`WithNewRoot()`), so the log record's `TraceID` is the per-phase trace ID — not the AgenticRun UUID. This means cleanup (which deletes by AgenticRun UUID) would not match logs. Today this is latent because sandbox OTLP log emission is specified but not yet implemented, so the table is effectively empty.

## Design Decisions

1. **Rename `trace_id` → `agentic_run_id`.** The column stores a Kubernetes UID, not an OTel trace ID. The name should reflect that.

2. **Add `phase` column.** Stores the audit phase name so the console can filter logs within a run by phase.

3. **Extract from log attributes, not OTel TraceID.** The `postgresexporter` maps `agenticrun.uid` log attribute → `agentic_run_id` column and `agenticrun.phase` log attribute → `phase` column. The OTel log record's native `TraceID` field (which carries the per-phase trace ID) is ignored for column mapping.

4. **UUID normalization in collector.** Callers (agentic-operator, console) pass the natural Kubernetes `metadata.uid` (with hyphens). The collector normalizes it (strips hyphens) internally — both in `postgresexporter` (on INSERT) and `postgres_admin` (on GET/DELETE). Callers never need to know about the 32-char hex format.

5. **Composite index.** `(agentic_run_id, phase)` enables efficient console queries like `SELECT ... WHERE agentic_run_id = $1 AND phase = $2`.

6. **No per-phase trace ID in templog.** The per-phase OTel trace ID is useful for distributed tracing backends (Jaeger, Tempo) but not for templog. Correlation between a templog entry and its distributed trace is done via `agenticrun.uid` span attribute in the trace backend, not by storing the trace ID in Postgres.

## Schema

```sql
CREATE TABLE IF NOT EXISTS templogs.logs (
    id              BIGSERIAL    PRIMARY KEY,
    agentic_run_id  TEXT         NOT NULL,
    phase           TEXT         NOT NULL DEFAULT '',
    timestamp       TIMESTAMPTZ  NOT NULL,
    event           TEXT         NOT NULL,
    body            JSONB
);

CREATE INDEX idx_logs_run_id    ON templogs.logs (agentic_run_id);
CREATE INDEX idx_logs_run_phase ON templogs.logs (agentic_run_id, phase);
CREATE INDEX idx_logs_timestamp ON templogs.logs (timestamp);
```

- **`agentic_run_id`** — AgenticRun `metadata.uid` with hyphens stripped (32-char hex). Primary query and cleanup key.
- **`phase`** — Audit phase name. See Phase Vocabulary below.

## Phase Vocabulary

Uses the audit trace phase names, normalized to lowercase nouns:

| Phase value | Operator span name | When |
|---|---|---|
| `analysis` | `agenticrun.analyze` | Sandbox analysis call |
| `approval` | `agenticrun.human_approval` | Approval event |
| `execution` | `agenticrun.execute` | Sandbox execution call |
| `verification` | `agenticrun.verify` | Sandbox verification call |
| `escalation` | `agenticrun.escalate` | Escalation |
| `terminal` | `agenticrun.terminal` | Terminal phase |

The sandbox's `derive_phase()` already produces `analysis`, `execution`, `verification` — these align directly.

## Column Mapping (postgresexporter)

| Column | Source | OTel field |
|---|---|---|
| `agentic_run_id` | Log attribute `agenticrun.uid` | `LogRecord.Attributes["agenticrun.uid"]` |
| `phase` | Log attribute `agenticrun.phase` | `LogRecord.Attributes["agenticrun.phase"]` |
| `timestamp` | Log record timestamp | `LogRecord.TimeUnixNano` (unchanged) |
| `event` | Log attribute `event` | `LogRecord.Attributes["event"]` (unchanged) |
| `body` | Log record body | `LogRecord.Body` (unchanged) |

The OTel log record's native `TraceID` field is ignored for column mapping. It may carry the per-phase trace ID, which is useful for distributed tracing but not for templog queries.

## UUID Normalization

Callers pass raw Kubernetes `metadata.uid` (with hyphens, e.g., `550e8400-e29b-41d4-a716-446655440000`). The collector normalizes in two places:

1. **`postgresexporter`** (on INSERT) — reads `agenticrun.uid` log attribute, strips hyphens, writes 32-char hex to `agentic_run_id` column.
2. **`postgres_admin`** (on GET/DELETE) — reads `agentic_run_id` query parameter, strips hyphens, uses in `WHERE` clause.

The column stores the compact 32-char hex format. Callers never need to pre-normalize.

## Admin API

| Method | Before | After |
|---|---|---|
| GET | `/api/v1/logs?trace_id=<uid>&limit=N&after=M` | `/api/v1/logs?agentic_run_id=<uid>&limit=N&after=M&phase=<phase>` |
| DELETE | `/api/v1/logs?trace_id=<uid>` | `/api/v1/logs?agentic_run_id=<uid>` |

- `phase` is an **optional** filter on GET — omit to fetch all phases.
- DELETE always deletes by `agentic_run_id` (all phases for a run).
- `agentic_run_id` accepts raw UUID with hyphens; collector normalizes internally.

## Data Flow

```
┌──────────────────────────────┐     ┌──────────────────────────────┐
│  agentic-operator            │     │  agentic-sandbox             │
│  EmitLog(event, body)        │     │  (OTLP log when implemented) │
│  log attrs:                  │     │  log attrs:                  │
│    agenticrun.uid   = <UID>  │     │    agenticrun.uid   = <UID>  │
│    agenticrun.phase = exec   │     │    agenticrun.phase = exec   │
│    event = audit.run.exec    │     │    event = gen_ai.choice     │
└──────────┬───────────────────┘     └──────────┬───────────────────┘
           │ OTLP/gRPC log                      │ OTLP/gRPC log
           └───────────┬────────────────────────┘
                       ▼
            ┌──────────────────────┐
            │  OTEL Collector      │
            │  postgresexporter    │
            │  attr → column map:  │
            │    uid   → agentic_run_id (normalize)
            │    phase → phase     │
            │    event → event     │
            │    body  → body      │
            └──────────┬───────────┘
                       ▼
            ┌──────────────────────┐
            │  PostgreSQL          │
            │  templogs.logs       │
            │  (agentic_run_id,    │
            │   phase, timestamp,  │
            │   event, body)       │
            └──────────────────────┘
```

## Changes by Repository

### lightspeed-otel-collector

| Component | Change |
|---|---|
| `postgres_admin` extension | Update `ensureTable()` DDL: rename `trace_id` → `agentic_run_id`, add `phase TEXT NOT NULL DEFAULT ''`, update indexes to include composite `(agentic_run_id, phase)` |
| `postgres_admin` admin API | Rename query param `trace_id` → `agentic_run_id`. Add optional `phase` query param on GET. Normalize UUID (strip hyphens) before querying. |
| `postgresexporter` | Extract `agenticrun.uid` log attribute → `agentic_run_id` column (with UUID normalization). Extract `agenticrun.phase` log attribute → `phase` column. Stop using `LogRecord.TraceID` for the `agentic_run_id` column. |

### lightspeed-agentic-operator

| Component | Change |
|---|---|
| `EmitLog()` | Add `agenticrun.phase` as a log record attribute, set from the current phase context (e.g., `analysis`, `execution`). `agenticrun.uid` is already emitted. |
| `DeleteLogs()` | Change admin API call from `?trace_id=<uid>` to `?agentic_run_id=<uid>`. Pass raw UUID (with hyphens) — collector normalizes. |

### lightspeed-agentic-sandbox

| Component | Change |
|---|---|
| OTLP log emission | Spec-only update: when OTLP log emission is implemented (§22–25 in audit-logging spec), log records must include `agenticrun.uid` and `agenticrun.phase` as log attributes. `agenticrun.phase` is derived from `derive_phase()`. No code change needed now. |

### lightspeed-operator

No changes. The operator generates collector config but does not reference the Postgres schema directly. The collector manages its own schema via `postgres_admin`.

## Migration

The `postgres_admin` extension's `ensureTable()` runs at collector startup. It currently uses `CREATE TABLE IF NOT EXISTS`. For the schema change:

- **New installations:** `ensureTable()` creates the table with the new schema directly.
- **Existing installations:** `ensureTable()` must handle the migration: add `phase` column if missing, rename `trace_id` → `agentic_run_id` if old column exists. This is a one-time idempotent migration in the `postgres_admin` extension.

Existing rows (if any) get `phase = ''` (the column default). Since sandbox OTLP log emission is not yet implemented, existing rows are unlikely in practice.

## Acceptance Criteria

1. `templogs.logs` table has `agentic_run_id` and `phase` columns with composite index `(agentic_run_id, phase)`.
2. Operator `EmitLog()` sets `agenticrun.phase` attribute on all log records.
3. Collector `postgresexporter` maps `agenticrun.uid` → `agentic_run_id` and `agenticrun.phase` → `phase` from log attributes.
4. Admin API GET supports `?agentic_run_id=<uid>&phase=<phase>` filtering.
5. Admin API DELETE uses `?agentic_run_id=<uid>`.
6. UUID normalization (strip hyphens) happens in collector — callers pass raw Kubernetes UIDs.
7. Agentic-operator cleanup works with the renamed parameter.
8. Schema migration handles rename from `trace_id` → `agentic_run_id` for existing installations.

## Risk Assessment

Per the risk-level rubric:
- **Data export schema change** (column rename + addition in `templogs.logs`) → Risk 3
- **API contract change** (admin API query parameter rename) → Risk 3
- **Cross-repo change** (collector + agentic-operator + sandbox specs) → reinforces Risk 3

**Risk Level: 3 (High)** — 2+ human reviewers required.
