# ROSA-Aware Answering in OpenShift Lightspeed

**Epic:** [OLS-1679](https://redhat.atlassian.net/browse/OLS-1679)
**Date:** 2026-07-06
**Status:** Draft

## Problem

OpenShift Lightspeed currently retrieves documentation only for `openshift_container_platform` from OKP (via the Solr `chunk_filter_query`). Users on ROSA clusters receive generic OCP answers, even though OKP already contains ROSA-specific documentation under two separate product identifiers. ROSA users need answers that account for their managed-service environment (different networking defaults, IAM integration, STS, service quotas, cluster lifecycle, etc.).

## OKP Product Catalog (relevant subset)

| Solr `product` identifier | Description |
|---|---|
| `openshift_container_platform` | Base OCP docs (always included) |
| `red_hat_openshift_service_on_aws` | ROSA HCP (Hosted Control Plane) docs |
| `red_hat_openshift_service_on_aws_classic_architecture` | ROSA Classic docs |

## Approach: OKP Product Filter Extension

No new RAG images, no separate BYOK indexes, no system prompt changes. The feature is implemented entirely by extending the existing OKP Solr `chunk_filter_query` to include the appropriate ROSA product when the operator detects a ROSA cluster.

### Detection (Operator — OLS-1894)

The operator detects the cluster platform and ROSA variant at reconcile time using two standard OpenShift API resources:

#### Step 1: ROSA Detection via Console Brand

```
apiVersion: operator.openshift.io/v1
kind: Console
metadata:
  name: cluster
spec:
  customization:
    brand: ROSA          # "ROSA" on ROSA 4.16+, "OKD"/"online"/"dedicated"/"ocp" on others
```

Read `.spec.customization.brand` (note: singular `customization`, not `customizations`). Value `ROSA` indicates a ROSA cluster. Available on all ROSA clusters running OCP 4.16+.

**Verified on live ROSA HCP cluster:**
```
$ oc get console.operator.openshift.io cluster -o jsonpath='{.spec.customization.brand}'
ROSA
```

#### Step 2: Classic vs HCP Detection via Infrastructure Topology

```
apiVersion: config.openshift.io/v1
kind: Infrastructure
metadata:
  name: cluster
status:
  controlPlaneTopology: External    # "External" = HCP, "HighlyAvailable" = Classic
```

Read `.status.controlPlaneTopology`:
- **`External`** → ROSA HCP (control plane hosted in Red Hat's AWS account)
- **`HighlyAvailable`** → ROSA Classic (control plane nodes in customer's AWS account)

**Verified on live ROSA HCP cluster:**
```
$ oc get infrastructure cluster -o jsonpath='{.status.controlPlaneTopology}'
External
```

#### Step 3: Pass Platform Info to Service

When ROSA is detected, the operator sets a new environment variable on the app-server container:

| Env var | Value | When set |
|---|---|---|
| `OLS_ROSA_PRODUCT` | `red_hat_openshift_service_on_aws` | ROSA HCP detected (brand=ROSA, topology=External) |
| `OLS_ROSA_PRODUCT` | `red_hat_openshift_service_on_aws_classic_architecture` | ROSA Classic detected (brand=ROSA, topology=HighlyAvailable) |
| *(not set)* | | Non-ROSA cluster or `byokRAGOnly=true` |

The env var is only set when OKP is active (`!byokRAGOnly`). On non-ROSA clusters (brand != `ROSA`), the env var is absent and the service behaves exactly as today.

#### RBAC

The operator already has ClusterRole permissions for `config.openshift.io` resources (Infrastructure) via its existing cluster-version lookup. Console operator resources (`operator.openshift.io/v1` Console) require adding a `get` verb for `consoles` in the `operator.openshift.io` API group to the operator's ClusterRole.

ROSA detection follows the same pattern as OCP version detection — determined once and passed to the service as an environment variable.

### Service-Side Filter Extension (extends PR #2926)

The `SolrHybridSearch._resolve_chunk_filter_query()` method currently builds:

```
is_chunk:true AND product:openshift_container_platform AND product_version:<resolved>
```

With ROSA awareness, when `OLS_ROSA_PRODUCT` is set, it builds:

```
is_chunk:true AND (
  (product:openshift_container_platform AND product_version:<ocp_resolved>)
  OR
  (product:<rosa_product> AND product_version:<rosa_resolved>)
)
```

Where `<rosa_product>` is the value of `OLS_ROSA_PRODUCT` (either `red_hat_openshift_service_on_aws` or `red_hat_openshift_service_on_aws_classic_architecture`). Each product gets its own version filter because OCP uses minor versions (e.g. `4.22`) while ROSA uses major versions (e.g. `4`).

The OCP base product is **always** included — ROSA clusters are OCP clusters and need OCP docs too. The ROSA product is additive.

#### Version Filtering for ROSA Products

ROSA product version resolution uses the same mechanism as OCP: derive the major version from `OCP_CLUSTER_VERSION`, query Solr for available versions of the ROSA product, and clamp to the nearest available version.

Concretely, `_resolve_chunk_filter_query()` already does this for OCP:
1. Read `OCP_CLUSTER_VERSION` (e.g. `4.22`)
2. Query Solr facets for available `product_version` values for `openshift_container_platform`
3. Clamp to the nearest available version

For the ROSA product, the same logic applies:
1. Extract the major version from `OCP_CLUSTER_VERSION` (e.g. `4.22` → `4`)
2. Query Solr facets for available `product_version` values for the ROSA product
3. Clamp to the nearest available version (currently resolves to `4`; when OCP 5 ships and OKP adds version `5`, it will automatically resolve to `5`)

The filter query with ROSA becomes:

```
is_chunk:true AND (
  (product:openshift_container_platform AND product_version:<ocp_resolved>)
  OR
  (product:<rosa_product> AND product_version:<rosa_resolved>)
)
```

This approach is future-proof — no hardcoded version, automatic adaptation when OKP adds new ROSA versions.

### No Changes to OLS-2603

OLS-2603 ("OLS RAG image to include ROSA documents") was created when the assumption was that ROSA docs would need a separate RAG image. Since the ROSA documentation is already in OKP's Solr corpus, this story is **not needed** in its current form. It can be:
- Closed as "not needed" (docs already in OKP), or
- Repurposed to "Verify ROSA product coverage in OKP corpus"

## Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Operator (reconcile)                                         │
│                                                              │
│  1. GET console.operator.openshift.io/cluster                │
│     → .spec.customization.brand == "ROSA"?                   │
│                                                              │
│  2. GET infrastructure.config.openshift.io/cluster            │
│     → .status.controlPlaneTopology == "External"?            │
│        → HCP: red_hat_openshift_service_on_aws               │
│        → Classic: ..._classic_architecture                   │
│                                                              │
│  3. Set env var OLS_ROSA_PRODUCT on app-server container     │
│  4. Set OCP_CLUSTER_VERSION (existing)                       │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Service (startup)                                            │
│                                                              │
│  _resolve_chunk_filter_query():                              │
│    ocp_version = env(OCP_CLUSTER_VERSION)  → e.g. "4.22"    │
│    rosa_product = env(OLS_ROSA_PRODUCT)    → e.g. "red_..."  │
│                                                              │
│    Resolve OCP version: facet query → clamp → "4.22"         │
│    if rosa_product:                                          │
│      Resolve ROSA version: major("4.22")→"4", facet → "4"   │
│      fq = "is_chunk:true AND ("                              │
│           "  (product:ocp AND product_version:4.22)"         │
│           "  OR"                                             │
│           "  (product:<rosa> AND product_version:4)"         │
│           ")"                                                │
│    else:                                                     │
│      fq = "is_chunk:true AND product:ocp"                    │
│           " AND product_version:4.22"                        │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Solr (RHOKP sidecar)                                         │
│                                                              │
│  hybrid-search with fq filter                                │
│  → Returns OCP + ROSA passages ranked by relevance           │
│  → Solr's scoring naturally ranks the most relevant docs     │
│     for the user's actual question                           │
└─────────────────────────────────────────────────────────────┘
```

## Stories Mapping

| Story | Scope | What changes |
|---|---|---|
| **OLS-1894** | Operator | Read Console brand + Infrastructure topology → set `OLS_ROSA_PRODUCT` env var. Add RBAC for Console read. |
| **OLS-2603** | RAG content | Not needed — ROSA docs already in OKP. Close or repurpose. |
| *(new, if needed)* | Service | Extend `_resolve_chunk_filter_query` to include ROSA product from env var. (Could be part of OLS-1894 or a separate service story.) |

## Spec Updates Required

### lightspeed-operator

- `.ai/spec/what/app-server.md` — Add behavioral rule for ROSA detection and `OLS_ROSA_PRODUCT` env var
- `.ai/spec/what/crd-api.md` — No CRD changes (auto-detect, no new fields)
- `.ai/spec/how/config-generation.md` — Document `OLS_ROSA_PRODUCT` env var generation
- `.ai/spec/how/deployment-generation.md` — Add ROSA detection to deployment generation flow

### lightspeed-service

- `.ai/spec/what/rag.md` — Add OKP behavioral rule for ROSA-aware `chunk_filter_query`

## Testing Strategy

### Operator
- Unit: mock Console resource with `brand: ROSA` + Infrastructure with `controlPlaneTopology: External` → verify `OLS_ROSA_PRODUCT=red_hat_openshift_service_on_aws` in generated deployment env
- Unit: mock Classic topology → verify `OLS_ROSA_PRODUCT=red_hat_openshift_service_on_aws_classic_architecture`
- Unit: mock non-ROSA brand → verify `OLS_ROSA_PRODUCT` absent
- Unit: `byokRAGOnly=true` → verify `OLS_ROSA_PRODUCT` absent regardless of brand

### Service
- Unit: `OLS_ROSA_PRODUCT` set → verify `chunk_filter_query` includes both OCP and ROSA product clauses
- Unit: `OLS_ROSA_PRODUCT` unset → verify `chunk_filter_query` is OCP-only (no regression)
- Integration: on a ROSA cluster, ask a ROSA-specific question → verify response cites ROSA docs

## Edge Cases

1. **OSD clusters**: Brand is `dedicated`, not `ROSA`. No ROSA products added. Could be extended later if OSD-specific docs exist in OKP (`openshift_dedicated` product is already in the catalog).
2. **Console resource unavailable**: If the operator cannot read the Console resource (RBAC failure, resource missing), it should log a warning and continue without setting `OLS_ROSA_PRODUCT`. No failure.
3. **ROSA product version changes in OKP**: If OKP starts indexing ROSA docs with fine-grained versions (4.18, 4.19, etc.), the service-side filter needs updating. Currently uses `product_version:4` as a catch-all.
