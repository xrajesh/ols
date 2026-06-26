# Risk Level Rubric

## Risk Levels

| Level | Customer Impact | Review Requirements | Automation |
|-------|----------------|---------------------|------------|
| Risk 1 | Very little impact if change goes wrong | No human code review required | Fully automated implementation |
| Risk 2 | Medium impact if change causes problems | 1 human reviewer required | Automated implementation with human review gate |
| Risk 3 | Major impact — risk of losing customers if a bug is introduced | 2+ human reviewers required | Human-driven implementation |

## Classification Examples

| Change Type | Risk Level |
|-------------|------------|
| Dependency version bump | 1 |
| Doc/comment updates, test-only changes | 1 |
| Localization/translation updates | 1 |
| Metadata-only changes (CSV version, labels) | 1 |
| Internal refactor with no API or behavior change | 2 |
| New component or adapter (non-critical path) | 2 |
| Pipeline or calculation logic changes | 2 |
| API contract changes (endpoints, schemas, CRDs — spec fields) | 3 |
| Additive CRD status/condition changes (no spec field changes) | 2 |
| Authentication/authorization/RBAC changes | 3 |
| User-facing UI flow changes | 3 |
| Data export schema or credential handling changes | 3 |
| Changes that mutate cluster state | 3 |

## Decision Tree

1. **Does the change touch an external contract?** (API endpoints, CRD spec fields, auth/RBAC, data export formats, credential handling)
   - Yes → **Risk 3**
   - Exception: additive CRD status/condition changes (no spec field changes) → **Risk 2** — status subresources are operator-managed, not user-facing input

2. **Does the change affect user-visible behavior?** (UI flows, user-facing error messages, cluster state mutations)
   - Yes → **Risk 3**

3. **Does the change alter internal logic?** (refactors, new non-critical-path components, pipeline/calculation changes)
   - Yes → **Risk 2**

4. **Is the change mechanical or cosmetic?** (dep bumps, doc/comment edits, test-only, metadata, localization)
   - Yes → **Risk 1**

5. **When in doubt**, bias UP — a Risk 2 that should have been Risk 3 causes more damage than a Risk 3 that could have been Risk 2.

## Edge Cases

- **Cross-repo changes:** If the change spans multiple repos, treat each repo's portion independently but note the cross-repo dependency — this often pushes toward Risk 3.
- **CVE fixes:** Dep bumps for CVEs are still Risk 1 if the bump is the only change. If a code fix accompanies the bump, assess the code fix separately.
- **Spikes / investigations:** Risk 1 — no production code changes result from the spike itself.
- **Feature flags:** Adding a new feature behind a flag is Risk 2 (the flag mechanism itself). Removing a flag to expose a feature is Risk 3 (user-visible behavior change).
