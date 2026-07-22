# AWS Bedrock Provider — Design Spec

**Epic:** [OLS-1680](https://redhat.atlassian.net/browse/OLS-1680) — Support AWS Bedrock as LLM provider in OLS
**Stories:**
- [OLS-1895](https://redhat.atlassian.net/browse/OLS-1895) — OLS service support for AWS Bedrock (`lightspeed-service`)
- [OLS-2605](https://redhat.atlassian.net/browse/OLS-2605) — OLSConfig CR support for Bedrock (`lightspeed-operator`)

**Date:** 2026-06-19
**Status:** Draft
**Research:** [`lightspeed-service/docs/superpowers/bedrock-provider-findings.md`](../../../lightspeed-service/docs/superpowers/bedrock-provider-findings.md)

---

## 1. Overview

Add AWS Bedrock as a supported LLM provider in OLS, using the Bedrock Mantle gateway. Mantle exposes multiple model families (Anthropic Claude, OpenAI GPT, DeepSeek, etc.) behind a single endpoint with a single Bearer token. The provider uses existing LangChain classes (`ChatAnthropic`, `ChatOpenAI`) pointed at Mantle — no `langchain-aws` or `boto3` required.

## 2. Approach

**Single `bedrock` provider with model-prefix routing.** One provider type in `olsconfig.yaml`. The provider detects the model name prefix and picks the right LangChain class and Mantle API path:

| Model prefix | LangChain class | Mantle path suffix | Extra params |
|---|---|---|---|
| `anthropic.*` | `ChatAnthropic` | `/anthropic` | — |
| `openai.*` | `ChatOpenAI` | `/openai/v1` | `use_responses_api=True` |
| Everything else | `ChatOpenAI` | `/v1` | — (standard Chat Completions) |

**Why this approach:** Mantle is a single gateway with one credential. A single provider type reflects this reality and gives users the simplest config. The model-prefix routing is modest (3 branches) and directly mirrors the Mantle endpoint matrix documented in the research findings.

**Alternatives considered:**
- *Separate providers per API family* (`bedrock_anthropic`, `bedrock_openai`, `bedrock_deepseek`) — rejected because it forces duplicate credentials and doesn't reflect the single-gateway reality.
- *Delegate to existing OpenAI/Anthropic providers* — rejected because it couples Bedrock to internal details of other providers and is fragile.

## 3. Authentication

### 3.1 Bearer Token (initial implementation)

A static Bearer token generated in the AWS Bedrock console. Same key works for all models on the account. Stored in a Kubernetes Secret and read from `credentials_path` using the existing `apitoken` pattern — identical to how OpenAI credentials work today.

- 30-day long-term keys for dev (generated in console)
- Short-term keys via `aws-bedrock-token-generator` for production
- Admin is responsible for rotation

### 3.2 STS / IAM Role (future work)

**[PLANNED]** The pod assumes an AWS IAM role and obtains temporary credentials via STS `AssumeRoleWithWebIdentity`. The design follows the same dual-auth pattern as the existing Azure OpenAI provider, where the absence of an `apitoken` triggers the alternative auth path.

When implemented, the `BedrockConfig` would accept `role_arn` and optional `region` fields (read from the credentials directory, analogous to Azure's `tenant_id`/`client_id`/`client_secret`). The provider would use `boto3` STS to obtain a temporary Bearer token, cache it with expiry-aware refresh (analogous to Azure's `TokenCache`), and pass it to the LangChain class.

### 3.3 Design for STS extensibility

The initial Bearer-only implementation structures the code so STS can be added without refactoring:

- `BedrockConfig` is defined as a provider-specific config class (even though Bearer auth doesn't strictly need it), so STS fields can be added later without changing the config schema structure.
- The credential detection logic in `default_params` follows the Azure pattern: if `self.credentials` (Bearer token) is present, use it directly; otherwise, fall through to STS auth. The `else` branch is a placeholder that raises a clear error until STS is implemented.
- The operator's `ProviderSpec` uses the standard `credentialsSecretRef` for Bearer tokens. For STS, a future `bedrockConfig` block on the CRD would carry `roleARN` and `region`, mirroring how `googleVertexConfig` carries provider-specific fields.

### 3.4 Bearer Token vs STS — Analysis for Customers

#### Bearer Token

| Aspect | Detail |
|---|---|
| **Setup complexity** | Low — generate key in AWS console, create K8s Secret |
| **AWS IAM knowledge** | Not required |
| **Works on** | Any OpenShift cluster (ROSA, on-prem, other clouds, disconnected) |
| **Credential lifetime** | Long-lived (30-day keys); manual rotation required |
| **Security risk** | If leaked, valid until manually revoked |
| **Blast radius** | Single key has broad permissions; no per-pod scoping |
| **Audit trail** | All requests appear as same API key identity |
| **Compliance** | May not meet enterprise policies prohibiting static long-lived credentials |

#### STS / IAM Role

| Aspect | Detail |
|---|---|
| **Setup complexity** | Moderate — requires IAM role, OIDC trust policy, SA annotation; one-time setup |
| **Works on** | ROSA with STS mode (automatic); OSD on AWS; self-managed OCP on AWS (IPI with STS mode); any cluster with manual AWS OIDC federation |
| **Does not work on** | Disconnected/air-gapped clusters; clusters without AWS IAM federation |
| **Credential lifetime** | Short-lived (~1 hour), auto-refreshed; leaked token expires quickly |
| **Secrets to manage** | None — no static key in etcd |
| **Scoping** | Each service account can assume a different IAM role with least-privilege |
| **Audit trail** | CloudTrail logs show which role/workload made each call |
| **Compliance** | Meets SOC2, FedRAMP, zero-trust requirements |

#### Recommendation

- **ROSA / AWS clusters (production):** STS is the expected pattern. These customers already use STS for every other AWS service (S3, RDS, etc.); a static key for Bedrock alone would be an outlier.
- **Non-AWS / on-prem clusters:** Bearer token is the only practical option.
- **Dev / test:** Bearer token for simplicity.

Both auth modes serve real customer segments. Bearer token ships first; STS follows.

## 4. Provider Implementation (OLS-1895)

### 4.1 Provider class

New file `ols/src/llms/providers/bedrock.py`:

```python
@register_llm_provider_as(constants.PROVIDER_BEDROCK)
class Bedrock(LLMProvider):

    @property
    def default_params(self) -> dict[str, Any]:
        # Read URL and credentials from provider_config
        # If credentials (Bearer token) present -> use directly
        # Else -> raise error (STS placeholder for future)
        # Build default params (model, httpx clients, max_tokens, etc.)

    def load(self) -> BaseChatModel:
        # Route based on model prefix:
        #   anthropic.* -> ChatAnthropic(base_url=url/anthropic, api_key=..., ...)
        #   openai.*    -> ChatOpenAI(base_url=url/openai/v1, use_responses_api=True, ...)
        #   *           -> ChatOpenAI(base_url=url/v1, ...)
```

The `default_params` property builds parameters common to all branches (httpx clients, model name). The `load()` method selects the LangChain class and constructs branch-specific parameters from `self.params`.

Auth detection in `default_params` follows the Azure pattern:

```python
if self.credentials is not None:
    # Bearer token auth — pass directly
    ...
else:
    # [PLANNED] STS auth — future implementation
    raise ValueError(
        "No Bearer token found. STS authentication is not yet supported. "
        "Provide a Bearer token via credentials_path."
    )
```

### 4.2 Parameter set

One `BedrockParameters` set registered in `available_provider_parameters` — the union of kwargs across all three LangChain classes (`ChatAnthropic` + `ChatOpenAI`). This matches the existing pattern where every provider type maps to exactly one parameter set. The `load()` method picks the subset relevant to each branch.

One `BedrockParametersMapping` registered in `generic_to_llm_parameters` — maps `max_tokens_for_response` to the appropriate LangChain param name. Since the Anthropic and OpenAI classes use different names (`max_tokens` vs `max_completion_tokens`), the mapping uses the OpenAI name and `load()` remaps for the Anthropic branch.

### 4.3 Config model

`BedrockConfig(ProviderSpecificConfig)` in `config.py`:

```python
class BedrockConfig(ProviderSpecificConfig, extra="forbid"):
    """Configuration specific to AWS Bedrock provider."""
    credentials_path: Optional[str] = None
    # [PLANNED] STS fields — added when STS auth is implemented:
    # role_arn: Optional[str] = None
    # region: Optional[str] = None
```

Added to `ProviderConfig`:
- `bedrock_config: Optional[BedrockConfig] = None` field
- `case constants.PROVIDER_BEDROCK:` in `set_provider_specific_configuration()`

Validation: if `type == "bedrock"` and no `url`, raise config error (Mantle URL is required — it's region-specific, no sensible default).

### 4.4 Constants

In `ols/constants.py`:
- Add `PROVIDER_BEDROCK = "bedrock"`
- Add to `SUPPORTED_PROVIDER_TYPES` frozenset

### 4.5 Dependencies

In `pyproject.toml`:
- Add `langchain-anthropic` as explicit production dependency (currently only transitive)
- No `langchain-aws` or `boto3` needed for Bearer token auth

### 4.6 Config shape

```yaml
llm_providers:
  - name: my-bedrock
    type: bedrock
    url: "https://bedrock-mantle.us-east-1.api.aws"
    credentials_path: /path/to/bedrock_api_key
    models:
      - name: anthropic.claude-opus-4-7
      - name: openai.gpt-5.4
      - name: deepseek.v3.1
```

### 4.7 Files to create/modify

| # | File | Action |
|---|---|---|
| 1 | `ols/constants.py` | Add `PROVIDER_BEDROCK`, update `SUPPORTED_PROVIDER_TYPES` |
| 2 | `ols/src/llms/providers/provider.py` | Add `BedrockParameters`, `BedrockParametersMapping`, register in dicts |
| 3 | `ols/src/llms/providers/bedrock.py` | New — provider class with model-prefix routing |
| 4 | `ols/app/models/config.py` | Add `BedrockConfig`, `bedrock_config` field, validation case |
| 5 | `pyproject.toml` | Add `langchain-anthropic` dependency |
| 6 | `tests/unit/llms/providers/test_bedrock.py` | New — unit tests |
| 7 | `tests/unit/llms/providers/test_providers.py` | Update registry assertion |
| 8 | `tests/config/with_bedrock.yaml` | New — test config |

## 5. Operator CRD Changes (OLS-2605)

### 5.1 CRD type

In `api/v1alpha1/olsconfig_types.go`, add `bedrock` to the kubebuilder Enum on `ProviderSpec.Type`:

```
// +kubebuilder:validation:Enum=azure_openai;bam;openai;watsonx;rhoai_vllm;rhelai_vllm;fake_provider;google_vertex;google_vertex_anthropic;bedrock
```

No new fields on `ProviderSpec` — Bedrock uses the existing `URL`, `CredentialsSecretRef`, and `Models` fields. No provider-specific CRD block needed for Bearer token auth. (A `bedrockConfig` block would be added when STS is implemented, following the `googleVertexConfig` pattern.)

No new CEL `XValidation` rules needed — Bedrock has no required provider-specific fields beyond the standard ones.

### 5.2 CRD to YAML mapping

Bedrock falls into the `default` case of `buildProviderConfigs()` in `internal/controller/appserver/assets.go`. No special-case `switch` branch needed. The default case generates:

```go
providerConfig = utils.ProviderConfig{
    Name: provider.Name, Type: provider.Type, URL: provider.URL,
    CredentialsPath: credentialPath, Models: modelConfigs,
}
```

This produces the YAML shape the Bedrock provider expects.

### 5.3 Credential validation

Bedrock uses the standard `apitoken` key pattern — falls into the existing `else` branch of `ValidateLLMCredentials()`. No changes needed. (When STS is added, a Bedrock-specific branch would accept either `apitoken` or STS-related keys, following the Azure pattern.)

### 5.4 Constants

Add `BedrockType = "bedrock"` in `internal/controller/utils/constants.go`.

### 5.5 CRD manifest regeneration

Run `make manifests` to update `bundle/manifests/ols.openshift.io_olsconfigs.yaml` with the new enum value.

### 5.6 CRD example

```yaml
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
spec:
  llm:
    providers:
      - name: my-bedrock
        type: bedrock
        url: "https://bedrock-mantle.us-east-1.api.aws"
        credentialsSecretRef:
          name: bedrock-api-key
        models:
          - name: anthropic.claude-opus-4-7
          - name: openai.gpt-5.4
          - name: deepseek.v3.1
```

### 5.7 Files to create/modify

| # | File | Action |
|---|---|---|
| 1 | `api/v1alpha1/olsconfig_types.go` | Add `bedrock` to kubebuilder Enum |
| 2 | `internal/controller/utils/constants.go` | Add `BedrockType` |
| 3 | `make manifests` | Regenerate CRD YAML |
| 4 | `internal/controller/utils/test_fixtures.go` | Add `WithBedrockProvider` helper |
| 5 | `internal/controller/appserver/assets_test.go` | Add Bedrock config gen test |

## 6. Testing

### 6.1 OLS-1895 (lightspeed-service)

Unit tests in `tests/unit/llms/providers/test_bedrock.py`:

- **Credential loading** — Bearer token read from `credentials_path` via `apitoken` file
- **Model routing** — all 3 branches:
  - `anthropic.*` → `ChatAnthropic` with `/anthropic` base URL suffix
  - `openai.*` → `ChatOpenAI` with `use_responses_api=True` and `/openai/v1`
  - `deepseek.*` (or any other) → `ChatOpenAI` with `/v1`
- **Parameter filtering** — only allowed params pass through validation
- **Default params** — correct defaults per branch (`max_tokens` vs `max_completion_tokens`)
- **Missing credentials** — clear error when neither Bearer token nor (future) STS config is present
- **Missing URL** — config validation rejects Bedrock provider without URL
- **TLS/proxy** — httpx client construction (inherited from base, same pattern as OpenAI tests)

Registration test update in `test_providers.py` — assert `PROVIDER_BEDROCK` is in the registry.

Test config file `tests/config/with_bedrock.yaml`.

### 6.2 OLS-2605 (lightspeed-operator)

- `assets_test.go` — verify `buildProviderConfigs()` generates correct YAML for Bedrock
- Test fixture `WithBedrockProvider` for use across test files
- CRD validation — verify `bedrock` type is accepted, no extra fields required

### 6.3 Not in scope

No integration or e2e tests — those require a live Bedrock account.

## 7. Scope Summary

| Item | Status |
|---|---|
| Single `bedrock` provider type with model-prefix routing | In scope (OLS-1895) |
| Bearer token auth via `credentials_path` | In scope (OLS-1895) |
| `BedrockConfig` structured for future STS extension | In scope (OLS-1895) |
| `langchain-anthropic` explicit dependency | In scope (OLS-1895) |
| Operator CRD enum + default mapping | In scope (OLS-2605) |
| Unit tests (service + operator) | In scope |
| STS / IAM role auth | **[PLANNED]** — future work |
| Operator `bedrockConfig` CRD block for STS | **[PLANNED]** — added with STS |
| `boto3` production dependency | **[PLANNED]** — added with STS |
| Integration / e2e tests | Not in scope |
