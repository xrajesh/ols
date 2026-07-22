# Dynamic Product Filtering in OKP Solr Search Tool

**Story:** [OLS-3432](https://redhat.atlassian.net/browse/OLS-3432)
**Date:** 2026-07-06
**Status:** Draft
**Depends on:** [OLS-3310](https://redhat.atlassian.net/browse/OLS-3310) (OKP Solr hybrid RAG — PR #2926)

## Problem

The OKP Solr index contains ~1.1 million documents across ~98 Red Hat products. The `search_openshift_documentation` tool (OLS-3310) filters to `openshift_container_platform` at a resolved version. When a user asks about a layered product — e.g. "how to configure a Tekton pipeline" or "set up ArgoCD with gitops" — relevant documentation from the specialized product is excluded, forcing the LLM to rely on general OCP docs or its own training data.

28 OpenShift-related products exist in the corpus (e.g. `red_hat_openshift_pipelines`, `red_hat_openshift_gitops`, `red_hat_openshift_service_mesh`, `red_hat_openshift_ai_self-managed`). Dynamically including additional products based on query intent would improve answer quality for layered-product questions while keeping OCP as the baseline.

## Approach: LLM-Driven Product Selection via Tool Argument

The `search_openshift_documentation` tool gains an optional `additional_products` parameter. At startup, available OpenShift-related products are discovered from Solr. The LLM selects relevant products per query based on intent. OCP remains the always-included baseline.

### 1. Product Discovery at Startup

During `SolrHybridSearch.__init__()`, alongside OCP version resolution:

1. Query Solr facets for all products matching `product:*openshift*`:
   ```
   q=*:*  fq=product:*openshift*  rows=0  facet=true  facet.field=product  facet.mincount=1
   ```
2. Exclude `openshift_container_platform` (already the baseline).
3. **Always exclude ROSA products** (`red_hat_openshift_service_on_aws`, `red_hat_openshift_service_on_aws_classic_architecture`) — ROSA products are handled by the separate ROSA detection mechanism (OLS-1894) and must never appear in the LLM-selectable list.
4. Store the resulting list as `self._additional_products: list[str]`.
5. If the facet query fails, log a warning and set the list to empty — graceful degradation, OCP-only search still works.

Uses the same retry logic as `_fetch_available_ocp_versions` (startup retries with backoff) since it runs in the same startup window.

### 2. Tool Schema Extension

The `get_openshift_docs_tool()` factory receives the discovered product list and modifies the tool:

- **New optional argument:** `additional_products: list[str] = []`
- **Tool description** is dynamically generated to include the discovered products:
  ```
  Search published Red Hat OpenShift and related product documentation
  (not the live cluster). Returns JSON: an array of {text, score, title, docs_url},
  or [] if no hits, or an object with error on failure. Cite docs_url when using
  a passage.

  By default, searches OpenShift Container Platform docs only. To include
  documentation from other OpenShift-related products, pass their identifiers
  in additional_products. Available products: red_hat_openshift_pipelines,
  red_hat_openshift_gitops, red_hat_openshift_service_mesh, ...
  ```
- When no additional products were discovered, the argument is omitted from the schema entirely (no empty list in the tool description).

### 3. Dynamic Filter Query Construction

`_build_hybrid_form()` accepts an `additional_products` parameter:

- **No additional products** → use the static `chunk_filter_query` (OCP-only, unchanged from OLS-3310):
  ```
  is_chunk:true AND product:openshift_container_platform AND product_version:4.22
  ```

- **With additional products** → validate each name against `self._additional_products` (reject unknown names), then build a compound filter:
  ```
  is_chunk:true AND (
    (product:openshift_container_platform AND product_version:4.22)
    OR product:red_hat_openshift_pipelines
    OR product:red_hat_openshift_gitops
  )
  ```

**No version filter on additional products** — layered products use their own versioning schemes unrelated to OCP minor versions. Including all versions of a selected product maximizes documentation coverage.

### 4. Interaction with ROSA (OLS-1894)

ROSA products are handled by a separate, deterministic mechanism:

- The operator detects ROSA clusters and sets `OLS_ROSA_PRODUCT` as an env var.
- ROSA products are included in the **static base filter** (always searched, no LLM decision).
- ROSA products are **excluded from the dynamic product list** (never appear in the tool schema).

On a ROSA cluster with dynamic products selected, the filter combines all three layers:

```
is_chunk:true AND (
  (product:openshift_container_platform AND product_version:4.22)
  OR (product:red_hat_openshift_service_on_aws AND product_version:4)
  OR product:red_hat_openshift_pipelines
)
```

- OCP: always present, version-pinned (static)
- ROSA: always present on ROSA clusters, version-pinned (static, from OLS-1894)
- Additional products: per-query, no version filter (dynamic, from LLM)

### 5. Prompt Guidance Update

The `SOLR_DOCS_TOOL_SUPPLEMENT` is updated to guide the LLM on product selection:
- When a question mentions a specific OpenShift layered product (Pipelines, GitOps, Service Mesh, Virtualization, AI/ML, etc.), include the corresponding product identifier in `additional_products`.
- For general OCP questions, omit `additional_products` to keep results focused.
- Do not include products speculatively — only when the query clearly relates to that product.

### 6. Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Startup (SolrHybridSearch.__init__)                          │
│                                                              │
│  1. Resolve OCP version (existing, OLS-3310)                 │
│     → chunk_filter_query = "product:ocp AND version:4.22"    │
│                                                              │
│  2. If OLS_ROSA_PRODUCT set (OLS-1894):                      │
│     → include ROSA product in static filter                  │
│                                                              │
│  3. Discover additional products (OLS-3432):                  │
│     → facet query product:*openshift* minus OCP, ROSA        │
│     → store as _additional_products list                     │
│     → inject into tool description                           │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Per-Request (tool calling loop)                              │
│                                                              │
│  LLM decides to call search_openshift_documentation:         │
│    search_query: "configure tekton pipeline triggers"        │
│    additional_products: ["red_hat_openshift_pipelines"]       │
│                                                              │
│  Tool validates additional_products against discovered list  │
│  Builds dynamic fq:                                          │
│    is_chunk:true AND (                                       │
│      (product:ocp AND product_version:4.22)                  │
│      OR product:red_hat_openshift_pipelines                  │
│    )                                                         │
│  Executes hybrid-search against Solr                         │
│  Returns ranked passages from OCP + Pipelines docs           │
└─────────────────────────────────────────────────────────────┘
```

## Acceptance Criteria

1. Tool description dynamically lists available OpenShift-related products from Solr (excluding OCP, ROSA)
2. Tool schema includes optional `additional_products` argument
3. OCP is always in the filter; additional products are additive with no version constraint
4. ROSA products are never in the dynamic list; on ROSA clusters, ROSA is in the static base filter
5. Invalid product names in `additional_products` are ignored (OCP-only filter used)
6. LLM correctly selects relevant products for layered-product queries
7. When no additional products are selected, behavior is identical to OLS-3310

## Testing Strategy

- **Unit:** Facet query returns N products → verify `_additional_products` excludes OCP and ROSA products
- **Unit:** Tool schema includes `additional_products` with discovered product names in description
- **Unit:** Empty `additional_products` → static OCP-only filter (no regression from OLS-3310)
- **Unit:** Valid `additional_products` → compound filter with unversioned OR clauses
- **Unit:** Invalid product name → rejected, OCP-only filter used
- **Unit:** ROSA env var set + additional products → three-layer compound filter (OCP + ROSA + dynamic)
- **Unit:** Facet query fails → empty list, tool works without additional products
- **Integration:** Ask "how to configure a Tekton pipeline" → verify LLM selects `red_hat_openshift_pipelines` and response cites Pipelines docs

## Risk Assessment

**Risk Level: 3 (Low)**

- No customer data at risk — additive filter only broadens search results
- No breaking changes — OCP baseline is unchanged when no additional products selected
- Automated tests cover filter construction, facet parsing, and validation
- Graceful degradation — facet discovery failure means OCP-only (same as today)
- Rollback — removing `additional_products` argument reverts to OLS-3310 behavior

## Spec Updates

### lightspeed-service

- `.ai/spec/what/rag.md` — Add behavioral rule for dynamic product filtering in OKP retrieval, update planned changes
