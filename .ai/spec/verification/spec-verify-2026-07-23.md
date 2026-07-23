# Verification Report: OLS Main Spec
Verified: 2026-07-23
Spec root: /Users/xavi/street/github.com/AI/ols/.ai/spec/

## Summary
- 2 constraint violations
- 5 broken or inaccurate references
- 3 cross-spec inconsistencies
- 4 completeness gaps

## Constraint Violations

1. **Non-standard `[PENDING]` marker in `rag-pipeline.md`.**
Line 68 uses `[PENDING]` — not a recognized marker. README.md line 39 defines only `[PLANNED]` and `[PLANNED: TICKET-XXXX]`.

2. **Non-standard `[DEFERRED]` marker in `audit-logging.md`.**
Lines 249, 379, 384-386 use `[DEFERRED]` and `[DEFERRED: needs Jira]`. Neither marker is defined in the README convention. Appears 4 times across the file.

## Reference Issues

1. **`lightspeed-hub/AGENTS.md` — does not exist.**
Referenced in `what/system-overview.md` line 29. The `lightspeed-hub` directory is not present in this workspace at all.

2. **`lightspeed-hub-ui/AGENTS.md` — does not exist.**
Referenced in `what/system-overview.md` line 30. The `lightspeed-hub-ui` directory is not present in this workspace.

3. **`lightspeed-otel-collector/AGENTS.md` — does not exist.**
Referenced in `what/system-overview.md` line 31. The repo exists but has no `AGENTS.md`. Should point to `.ai/spec/README.md` instead.

4. **`lightspeed-agentic-console/.ai/spec/what/dynamic-components.md` — does not exist.**
Referenced in `how/repo-map.md` line 81 ("Dynamic configuration components | lightspeed-agentic-console | `what/dynamic-components.md`").

5. **All `lightspeed-hub` and `lightspeed-hub-ui` spec files in `repo-map.md` — do not exist.**
`how/repo-map.md` lines 108-123 references 10 spec files across hub and hub-ui repos (system-overview.md, spoke-lifecycle.md, fleet-coordination.md, fleet-dashboard.md, spoke-management.md, etc.). Neither repository exists in the workspace.

## Cross-Spec Inconsistencies

1. **"Proposal" vs "AgenticRun" naming conflict in `audit-logging.md`.**
`agentic-runs.md` and `agentic-security.md` consistently use "AgenticRun"/"AgenticRunApproval". But `audit-logging.md` body text and event catalog still use old "Proposal"/"ProposalApproval" naming (lines 31, 33, 67, 69, 74, 193, 318-320). Span attribute names (`proposal.uid`, `proposal.name`) and event names (`proposal.received`, `proposal.terminal`) also use old naming. OLS-3295 tracks the rename; the what/ spec is internally inconsistent in the interim.

2. **`templog.md` uses `agentic_run_id`; `audit-logging.md` correlation model uses `proposal.uid`.**
Same underlying concept (the AgenticRun UID as a 32-char hex correlation key), different naming. `templog.md` line 92 correctly uses `agentic_run_id`; `audit-logging.md` line 33 says `Proposal CR's metadata.uid`.

3. **`system-overview.md` otel-collector description vs actual collector role.**
`system-overview.md` line 31 describes otel-collector as "Collects and forwards observability data (metrics, traces, logs) across the OLS fleet" — describing a fleet-wide telemetry forwarder. But the actual collector repo spec describes it as a custom OCB-built collector with a postgres exporter for temporary audit log (templog) storage. The repo-map also lists it under "Multicluster OLS" with fleet telemetry concerns, which may be correct for future state but creates confusion with the current templog purpose.

## Completeness Gaps

1. **`templog.md` missing from README quick-start table and `system-overview.md` cross-repo features table.**
README.md lists 7 what/ specs in Quick Start — `what/templog.md` is absent. `system-overview.md` lists 5 cross-repo features — templog is absent. Only `how/repo-map.md` line 153 references it.

2. **`agentic-security.md` missing from `system-overview.md` cross-repo features table.**
`system-overview.md` lines 42-48 lists 5 cross-repo features but omits agentic-security. README quick-start (line 33) and repo-map (line 148) both include it.

3. **`lightspeed-hub` and `lightspeed-hub-ui` repos not in workspace; README "13 repositories" claim is inaccurate.**
`system-overview.md` lines 27-30 and `repo-map.md` lines 106-123 describe these repos with specific spec files. Currently 11 repo directories are present (12 including `konflux-release-data`). README line 16 claims "13 repositories in this workspace."

4. **No parent-level what/ spec for multicluster hub features.**
`system-overview.md` describes Multicluster OLS as a product layer. No `what/multicluster.md` or `what/hub.md` exists at the parent level. Child repos don't exist either, leaving multicluster with no verifiable spec coverage.

## Files Checked

### Spec files read
- README.md, constraints.md
- what/system-overview.md, query-pipeline.md, agentic-runs.md, agentic-security.md, audit-logging.md, deployment-lifecycle.md, rag-pipeline.md, templog.md
- how/repo-map.md

### Reference targets verified (existence checks)
| Target | Status |
|---|---|
| lightspeed-service/.ai/spec/README.md | EXISTS |
| lightspeed-operator/.ai/spec/README.md | EXISTS |
| lightspeed-console/AGENTS.md | EXISTS |
| lightspeed-rag-content/.ai/spec/README.md | EXISTS |
| lightspeed-agentic-operator/.ai/spec/README.md | EXISTS |
| lightspeed-agentic-console/.ai/spec/README.md | EXISTS |
| lightspeed-agentic-sandbox/.ai/spec/README.md | EXISTS |
| lightspeed-agentic-alerts-adapter/AGENTS.md | EXISTS |
| lightspeed-hub/AGENTS.md | **MISSING** (repo not cloned) |
| lightspeed-hub-ui/AGENTS.md | **MISSING** (repo not cloned) |
| lightspeed-otel-collector/AGENTS.md | **MISSING** (repo exists, no AGENTS.md) |
| lightspeed-team-harness/AGENTS.md | EXISTS |
| All 20 lightspeed-service child specs in repo-map | ALL EXIST |
| All 13 lightspeed-operator child specs in repo-map | ALL EXIST |
| All 8 lightspeed-rag-content child specs in repo-map | ALL EXIST |
| All 7 lightspeed-agentic-operator child specs in repo-map | ALL EXIST |
| lightspeed-agentic-console/what/dynamic-components.md | **MISSING** |
| All 5 lightspeed-agentic-sandbox child specs in repo-map | ALL EXIST |
| docs/superpowers/specs/2026-07-22-templog-phase-storage.md | EXISTS |
