# Alerts Adapter — Multicluster Support

How the alerts-adapter supports polling AlertManager on multiple spoke clusters from a single adapter instance on the hub.

> **Parent spec:** [multicluster-ops.md](multicluster-ops.md) — overall multicluster architecture.

## Repos Involved

| Repo | Role |
|---|---|
| lightspeed-agentic-alerts-adapter | Adapter changes: SpokeCluster watch, multi-spoke polling, spoke-scoped AgenticRun creation |
| lightspeed-hub | Hub controller: provisions `spoke-alert-kubeconfig-{spoke-name}` Secrets, sets label on SpokeCluster CR |
| lightspeed-agentic-operator | AgenticRun CRD: `spec.targetCluster` field |

## Design Overview

A single alerts-adapter instance runs on the hub cluster. It watches `SpokeCluster` CRs to discover managed clusters, reads a per-spoke AlertManager kubeconfig Secret, and polls each spoke's AlertManager independently. AgenticRuns created on the hub carry `spec.targetCluster` and a spoke label for per-spoke dedup and fleet visibility.

The adapter operates in one of two modes, controlled by config:

| Mode | Behavior |
|---|---|
| Single-cluster (`multicluster: false`, default) | Current behavior. Polls the local in-cluster AlertManager. No SpokeCluster watch. No `spec.targetCluster` or spoke label on AgenticRuns. |
| Multi-cluster (`multicluster: true`) | Watches SpokeCluster CRs. Polls spoke AlertManagers via per-spoke kubeconfig Secrets. Sets `spec.targetCluster` and `hub.openshift.io/spoke-cluster` label on every AgenticRun. |

## Behavioral Rules

### Configuration

1. The adapter config (YAML from ConfigMap) gains a `multicluster` boolean field. Default: `false`.
2. When `multicluster: false`, the adapter behaves exactly as today — single-cluster mode. No SpokeCluster watch, no spoke-scoped behavior.
3. When `multicluster: true`, the adapter MUST NOT poll a local AlertManager. All AlertManager sources come from SpokeCluster CRs.
4. Global config fields (`pollInterval`, `preRunDelay`, `postRunDelay`, `allowedReceivers`, `ignoredLabels`, `tools`, `agent`) apply uniformly to all spokes.

### SpokeCluster Watch (multi-cluster mode only)

5. The adapter MUST register a controller-runtime watch on `SpokeCluster` CRs (`hub.openshift.io/v1alpha1`).
6. On SpokeCluster **create or update**: read the AlertManager kubeconfig Secret name from the `hub.openshift.io/alert-kubeconfig-secret` label on the CR. Fetch the Secret. Build a `rest.Config` from the kubeconfig. Construct an AlertManager client for that spoke. Start (or restart) a polling loop for that spoke.
7. On SpokeCluster **delete**: immediately stop the polling loop for that spoke. Discard the cached AlertManager client and config.
8. The adapter MUST NOT create, stop, or delete AgenticRuns on SpokeCluster deletion. Adapters only create AgenticRuns — the hub controller owns AgenticRun lifecycle (stopping, deleting, cleanup).

### Credential Flow

9. The hub controller provisions a per-spoke AlertManager kubeconfig Secret on the hub with the naming pattern `spoke-alert-kubeconfig-{spoke-name}`. This Secret contains a kubeconfig scoped to AlertManager read-only access on the spoke.
10. The hub controller sets the label `hub.openshift.io/alert-kubeconfig-secret` on the SpokeCluster CR with the Secret name as the value.
11. The Secret is separate from the standing kubeconfig `spoke-kubeconfig-{spoke-name}` used by the agentic-operator. The alert adapter SA has a narrower scope — AlertManager read-only access only.
12. On credential rotation, the hub controller updates the Secret. The adapter picks up the new credentials on the next SpokeCluster reconcile or on AlertManager connection error (re-fetch and retry).

### AlertManager Polling

13. The adapter maintains an in-memory map of `spokeName → pollingLoop` (goroutine + cancel context). This map is the only mutable state.
14. Each spoke polling loop runs independently on the global `pollInterval` tick.
15. The adapter accesses each spoke's AlertManager through the kube-api service proxy: `/api/v1/namespaces/openshift-monitoring/services/alertmanager-main:web/proxy/`. This works uniformly in both MCE mode (cluster-proxy) and secret mode (direct kube-api) because the spoke kubeconfig already provides kube-api access — no external Route or internal DNS resolution required.
16. The spoke SA MUST have RBAC to proxy to services in the `openshift-monitoring` namespace in addition to `cluster-monitoring-view`.
17. If a spoke's AlertManager is unreachable, the adapter MUST log an error and continue polling other spokes. One spoke failure MUST NOT affect other spokes.

### AgenticRun Creation (multi-cluster mode)

18. Every AgenticRun created in multi-cluster mode MUST set `spec.targetCluster` to the SpokeCluster name.
19. Every AgenticRun created in multi-cluster mode MUST carry the label `hub.openshift.io/spoke-cluster: {spoke-name}`.
20. The `hub.openshift.io/spoke-cluster` label MUST NOT be set in single-cluster mode.
21. AgenticRuns are created on the hub cluster using the adapter's own pod SA (in-cluster config). The spoke kubeconfig is only used for AlertManager access.
22. AgenticRuns are created in the `openshift-lightspeed` namespace on the hub.

### Deduplication

23. In multi-cluster mode, `ListAgenticRuns` MUST include the `hub.openshift.io/spoke-cluster` label in its label selector, scoping dedup to the originating spoke.
24. The same alert firing on two different spokes MUST produce two separate AgenticRuns — no cross-spoke dedup.
25. In single-cluster mode, dedup behavior is unchanged (no spoke label filter).

### Hub SA (hub-side operations)

26. The adapter uses its own pod ServiceAccount for all hub-side operations: creating AgenticRuns, listing AgenticRuns (dedup), reading Secrets, watching SpokeCluster CRs.
27. The pod SA requires RBAC on the hub: create/list AgenticRuns in `openshift-lightspeed`, get Secrets in `openshift-lightspeed`, watch/list SpokeCluster CRs (cluster-scoped).

## Spec Changes Required in Other Files

The following existing specs describe a one-adapter-per-spoke model and MUST be updated to reflect the single multi-cluster adapter design:

| File | Section | Change |
|---|---|---|
| `.ai/spec/what/multicluster-ops.md` | Standalone Adapter Path, step 1 | Update: single adapter watches SpokeCluster CRs and polls all spokes (not one pod per spoke) |
| `lightspeed-hub/.ai/spec/what/fleet-coordination.md` | Rule 9 | Update: single alerts-adapter instance handles all spokes (not one pod per spoke) |
| `lightspeed-hub/.ai/spec/what/system-overview.md` | Rule 8, Adapter Orchestrator | Update: orchestrator configures a single adapter with `multicluster: true` (not one Deployment per spoke) |
| `lightspeed-hub/.ai/spec/what/spoke-lifecycle.md` | Rule 7 | Update: hub operator provisions `spoke-alert-kubeconfig-{spoke-name}` Secret and labels SpokeCluster CR (not deploy adapter pods per spoke) |

## What Does NOT Change

- Single-cluster deployment mode (continues to work with `multicluster: false`)
- AgenticRun CRD schema beyond the already-specified `spec.targetCluster` field
- Config fields: `allowedReceivers`, `ignoredLabels`, `tools`, `agent`, `pollInterval`, `preRunDelay`, `postRunDelay`
- AgenticRun namespace (`openshift-lightspeed`)
- Adapter responsibility boundary: adapters only create AgenticRuns, never stop or delete them
