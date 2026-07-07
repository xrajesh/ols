# Script-Grounded RBAC for Agentic Runs

**Jira**: OLS-3441 — Execution phase fails when analysis agent generates incomplete RBAC permissions
**Date**: 2026-07-06

## Problem

The analysis agent predicts RBAC permissions the execution agent will need. These predictions are unreliable because:

1. The analysis agent generates abstract action descriptions ("patch the deployment") from which it guesses RBAC. It misses implicit sub-resources (e.g., `replicasets` when patching `deployments`), intermediate reads (checking secrets for configuration), and wait/status commands (listing pods for rollout status).

2. Analysis and execution are separate LLM sessions. Even if RBAC were predicted perfectly, the execution agent starts fresh and may take a different path — it interprets the approved option independently and can diverge from what analysis planned.

When execution encounters a missing permission, it fails with a 403 and no recovery mechanism exists.

## Root Cause

RBAC is derived from abstract predictions about future behavior, not from concrete planned operations. The analysis prompt asks the LLM to "include the RBAC permissions the execution agent will need" without requiring it to first produce the exact commands it's planning. This means RBAC is predicted independently from the action plan, rather than derived from it.

## Solution: Script-Grounded Analysis

Change the analysis prompt and output schema so that:

1. The analysis agent produces a **concrete remediation script** — an ordered list of exact bash commands (kubectl/oc) rather than abstract action descriptions.
2. RBAC is **derived from the script** — the agent examines each command and lists every Kubernetes resource it touches, including implicit sub-resources and intermediate operations.
3. The execution agent receives the **concrete script** in the approved option, reducing divergence from the approved plan.

This is a prompt-and-schema-level fix. No architectural changes to the operator, sandbox, or approval flow.

## Changes

### 1. Analysis Query Template

**File**: `agentic-operator/controller/proposal/templates/analysis_query.tmpl`

Replace the current template with:

See `agentic-operator/controller/proposal/templates/analysis_query.tmpl` for the full template. Key elements:

- Agent is told to use kubectl/oc to inspect cluster state before diagnosing (not guess from local files)
- Actions must be concrete bash commands, not descriptions
- Explicit instruction to include pre-checks, mutations, waits, and post-checks (intermediate operations)
- RBAC derivation is grounded in the script with a cross-check instruction
- Minimum read verbs: `get`, `list`, `watch` required for every resource in RBAC (kubectl uses these internally)
- Guidance on implicit sub-resources and side-effect resources

### 2. Analysis Output Schema

**File**: `agentic-operator/controller/proposal/schemas.go`

Change the `proposal.actions` items schema to require a `command` field:

Current:
```json
"actions": {
  "type": "array",
  "description": "Ordered list of discrete actions to perform",
  "items": {
    "properties": {
      "type": { "type": "string", "description": "Action category (e.g., 'patch', 'scale', ...)" },
      "description": { "type": "string", "description": "What this action does (e.g., 'Increase memory limit...')" }
    },
    "required": ["type", "description"]
  }
}
```

New:
```json
"actions": {
  "type": "array",
  "description": "Ordered list of exact bash commands to execute. Each action is one command.",
  "items": {
    "properties": {
      "command": { "type": "string", "description": "Exact executable bash command using kubectl or oc (e.g., 'kubectl set image deployment/foo container=registry/foo:v1.3 -n production')" },
      "type": { "type": "string", "description": "Action category (e.g., 'pre-check', 'mutation', 'wait', 'post-check')" },
      "description": { "type": "string", "description": "What this command does and why" }
    },
    "required": ["command", "type", "description"]
  }
}
```

The `command` field makes the script concrete and machine-parseable. The `type` field is updated to reflect the operation categories from the prompt (pre-check, mutation, wait, post-check).

The Go API type `ProposedAction` adds a `Command` field with validation: `MinLength=1`, `MaxLength=4096`, required. CRD manifests are regenerated to include this field.

### 3. Execution Query Template

**File**: `agentic-operator/controller/proposal/templates/execution_query.tmpl`

Updated to emphasize following the concrete script. Key additions beyond the original design:
- Agent must dry-run every mutation command with `--dry-run=server` before executing
- If dry-run fails, fix the command based on error and dry-run again before real execution
- Only execute once dry-run succeeds

### 4. Parent Spec Updates

**File**: `.ai/spec/what/agentic-runs.md`

Add to Planned Changes table:

```
| OLS-3441 | Script-grounded RBAC: analysis produces concrete bash scripts and derives RBAC from commands, replacing abstract action descriptions and independent RBAC prediction |
```

Update Phase 2 step 10 description:

Current: "The sandbox executes the request using the configured LLM provider and returns structured remediation options (diagnosis, proposed actions, RBAC requirements, verification plan)."

New: "The sandbox executes the request using the configured LLM provider and returns structured remediation options. Each option contains a concrete remediation script (ordered bash commands using kubectl/oc) and RBAC requirements derived from those commands."

Update Shared Data Formats bullet on AnalysisResult:

Add: "Each `RemediationOption.proposal.actions` entry includes the exact bash `command`, a `type` (pre-check, mutation, wait, post-check), and a `description`. RBAC requirements are derived from the commands in the script."

**File**: `.ai/spec/what/agentic-security.md`

No structural changes needed. The RBAC materialization rules (rules 7-15) are about how the operator creates K8s RBAC objects from the analysis output — they don't change. What changes is the quality of the RBAC data going into materialization.

### 5. Child Spec Updates

**File**: `agentic-operator/.ai/spec/what/sandbox-execution.md`

Update rule 12 (Execution query payload):

Current: "The `query` MUST include JSON describing the approved remediation option."

New: "The `query` MUST include JSON describing the approved remediation option, which contains a concrete remediation script (ordered bash commands). The execution prompt MUST instruct the agent to follow the script exactly, execute commands in order without substitution, and dry-run every mutation command with `--dry-run=server` before applying."

**File**: `agentic-sandbox/.ai/spec/what/run-api.md`

Update rule 16 (Context — approvedOption) to note that each action's `command` field contains the exact bash command to execute.

## What This Does NOT Fix

- **Session discontinuity**: The execution agent is still a separate LLM session. It receives the concrete script but could still deviate. This fix makes deviation less likely (concrete commands vs. abstract descriptions) but doesn't eliminate it.
- **Runtime state changes**: If cluster state changes between analysis and execution, commands may behave differently.
- **Order-dependent permission gaps**: If command 2 depends on command 1's output (e.g., command 1 creates a resource, command 2 modifies it), and both need RBAC, the cross-check instruction helps but can't guarantee the LLM catches every dependency.

For a complete solution to these gaps, see the design in `rbac-accuracy-via-single-pod-analysis.txt` which proposes empirical 403 collection and deterministic script replay.

## Verification

1. Deploy operator with updated template and schema
2. Trigger an analysis run for a known problem (e.g., CrashLoopBackOff due to OOMKilled)
3. Verify the analysis output contains:
   - Concrete bash commands in `proposal.actions[].command`
   - Pre-check, mutation, wait, and post-check command types
   - RBAC entries that trace to specific commands
4. Approve and execute — verify no 403 failures for the commands in the script
5. Compare RBAC completeness against the current prompt (run both side by side on the same problem)
