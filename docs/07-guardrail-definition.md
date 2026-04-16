# Guardrail Definition Specification

> **File naming**: `guardrails/<guardrail_id>.guardrail.md`

> **Audience**: Platform engineers, AI safety teams, product owners

---

## Overview

A guardrail definition file declares one named guardrail check that agents can reference. It maps a stable, human-readable identifier (e.g., `pii-scan`) to a specific provider, invocation details, and failure policy. The compiler resolves guardrail references from agent definitions and injects the complete provider configuration into the compiled payload. The runtime never reads raw guardrail identifiers — it only executes compiled payloads.

This indirection serves the same purpose as model definition files: the agent author writes `ref: "pii-scan"`, not a provider ARN, a version number, a region, or a set of credentials. The platform team controls guardrail backends — including provider-specific settings, environment routing, and secret references — in one place without touching any agent file.

Guardrail definition files are only required when the guardrail is backed by an external provider service (e.g., AWS Bedrock Guardrails, Azure AI Content Safety) or when the check requires any configuration beyond its identifier. For simple platform-built checks, the inline `id` field in the agent's `guardrails` section is sufficient.

---

## File structure

```
---
[YAML front matter — all structured fields]
---

# Description
[optional prose — what this guardrail checks, when to use it, usage guidance]
```

The prose body is optional and intended for the guardrail registry documentation UI. It does not affect runtime behavior.

---

## YAML front matter — complete field reference

### Top-level required fields

```yaml
spec_version: "1.2"
```
The AML format version. Must equal a platform-approved version string.

```yaml
guardrail_id: "pii-scan"
```
Stable, immutable identifier for this guardrail entry. Used as the value of `ref` in agent definition `guardrails` sections. Lowercase kebab-case. Must match `^[a-z0-9_-]{3,64}$`. Once published, the `guardrail_id` cannot change — create a new entry for a different check.

```yaml
display_name: "PII Scan (Bedrock)"
```
Human-readable name shown in the guardrail registry UI and compiler output.

```yaml
provider: "aws-bedrock-guardrail"
```
Provider type. Determines which `provider_config` sub-key is required. Enum:

| Value | Provider |
|---|---|
| `aws-bedrock-guardrail` | Amazon Bedrock Guardrails |
| `azure-content-safety` | Azure AI Content Safety |
| `gcp-natural-language` | Google Cloud Natural Language API |
| `platform-native` | Built-in platform check (no external call) |
| `custom` | Custom provider — requires a `custom` config block |

```yaml
status: "active"
```
Lifecycle state. Enum: `active` | `deprecated` | `disabled`. A `deprecated` guardrail triggers a lint warning on any agent that references it. A `disabled` guardrail causes a hard compile error.

```yaml
owner: "ai-safety-team"
```
Team or individual responsible for maintaining this guardrail entry.

```yaml
last_updated: "2026-04-12"
```
ISO 8601 date of the last update to this file.

---

### `meta` — descriptive metadata (recommended)

```yaml
meta:
  description: >
    Detects and redacts personally identifiable information in agent inputs
    and outputs using AWS Bedrock Guardrails.
  tags: ["pii", "privacy", "gdpr"]
  applies_to: ["input", "output"]    # Which pipeline positions this guardrail is designed for
```

`applies_to` is documentation-only — it does not restrict where agents may attach this guardrail. An agent may attach a guardrail to a position not listed in `applies_to`; the compiler emits a lint warning in that case.

---

### `provider_config` — provider-specific settings (required)

Exactly one sub-key matching the `provider` value must be present.

#### `aws-bedrock-guardrail` — Amazon Bedrock Guardrails

```yaml
provider_config:
  aws-bedrock-guardrail:
    guardrail_id: "abc123xyz"       # Required — Bedrock guardrail ID from AWS console
    guardrail_version: "1"          # Required — exact version; use a numeric string, not "DRAFT" in production
    region: "eu-west-1"             # Required — AWS region where the guardrail is deployed
    credentials:
      source: "iam_role"            # iam_role | env | aws_profile (default: iam_role)
```

**Why each field is required:**

- **`guardrail_id`** — the opaque identifier assigned by AWS at guardrail creation. It is environment-specific: a guardrail deployed in a dev account has a different ID from its production counterpart. This value never belongs in the agent file itself.
- **`guardrail_version`** — Bedrock Guardrails are explicitly versioned. Omitting this or using `"DRAFT"` in production means the guardrail policy may change without notice (Bedrock promotes draft to a new numeric version only on explicit action). Pin to a numeric version and update this file when a new version is published.
- **`region`** — Bedrock Guardrails are regional resources. A guardrail provisioned in `us-east-1` cannot be invoked from `eu-west-1`. This field must be consistent with the agent's `policies.data_residency` setting; a mismatch is a hard compile error.
- **`credentials.source`** — `iam_role` is the recommended production setting — the runtime uses the IAM role declared in the agent definition. `env` reads `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from the environment (development only). `aws_profile` uses a named profile (development only).

#### `azure-content-safety` — Azure AI Content Safety

```yaml
provider_config:
  azure-content-safety:
    endpoint: "https://<resource-name>.cognitiveservices.azure.com"  # Required
    api_version: "2024-09-01"       # Required — pinned API version
    subscription_key:               # Required — secret reference
      secret_ref:
        source: "azure_key_vault"
        vault_url: "https://myvault.vault.azure.net"
        secret_name: "azure-content-safety-key"
    categories:                     # Optional — which categories to evaluate
      - name: "Hate"
        severity_threshold: 2       # 0–6; block if severity >= threshold
      - name: "SelfHarm"
        severity_threshold: 2
      - name: "Sexual"
        severity_threshold: 2
      - name: "Violence"
        severity_threshold: 2
```

`severity_threshold` maps to Azure's 0–6 severity scale. Azure returns a severity value per category; the guardrail triggers if any category meets or exceeds its threshold. Omitting `categories` applies the provider's default moderation settings.

For secret references, see the [`secret_ref` pattern](#secret_ref----secret-reference-pattern) below.

#### `gcp-natural-language` — Google Cloud Natural Language API

```yaml
provider_config:
  gcp-natural-language:
    project_id: "my-gcp-project"    # Required
    location: "eu"                  # Required — "global" | "eu" | "us"
    credentials:
      source: "workload_identity"   # workload_identity | service_account_key | env
      service_account_key:          # Only if source is service_account_key — secret reference
        secret_ref:
          source: "gcp_secret_manager"
          project: "my-gcp-project"
          secret: "nlp-api-service-account"
    classify_text: true             # Enable content classification
    moderate_text: true             # Enable text moderation endpoint
```

#### `platform-native` — built-in platform check

```yaml
provider_config:
  platform-native:
    check_id: "jailbreak_check"     # Required — ID in the platform's built-in check registry
    parameters:                     # Optional — check-specific configuration
      sensitivity: "high"
```

Use `platform-native` for all platform-built checks, whether or not they require parameters. This ensures every guardrail in the platform has a definition file, a clear owner, a version history, and is discoverable in the registry.

#### `custom` — custom provider

```yaml
provider_config:
  custom:
    endpoint:                       # Required — the URL of the custom guardrail service
      secret_ref:
        source: "aws_secrets_manager"
        secret_id: "prod/guardrails/custom-pii-endpoint"
        region: "eu-west-1"
    auth:
      method: "bearer"              # bearer | api_key | none
      token:
        secret_ref:
          source: "env"
          name: "CUSTOM_GUARDRAIL_TOKEN"
    request_schema:                 # Shape of the request payload expected by the service
      type: "object"
      properties:
        content:
          type: "string"
        source:
          type: "string"
          enum: ["input", "output"]
    response_schema:                # Shape of the response the platform should parse
      type: "object"
      properties:
        triggered:
          type: "boolean"
        findings:
          type: "array"
```

---

### `secret_ref` — secret reference pattern

Any field that requires a secret value (API key, bearer token, endpoint URL) uses a `secret_ref` object. This allows the runtime to fetch the secret from different backends without the value ever appearing in a definition file.

```yaml
# Environment variable
token:
  secret_ref:
    source: "env"
    name: "MY_GUARDRAIL_API_KEY"

# AWS Secrets Manager
token:
  secret_ref:
    source: "aws_secrets_manager"
    secret_id: "prod/guardrails/pii-scan-key"
    region: "eu-west-1"            # Optional — defaults to the provider's region
    key: "api_key"                 # Optional — JSON key if the secret value is a JSON object

# GCP Secret Manager
token:
  secret_ref:
    source: "gcp_secret_manager"
    project: "my-gcp-project"
    secret: "guardrail-api-key"
    version: "latest"              # Optional — defaults to 'latest'

# Azure Key Vault
token:
  secret_ref:
    source: "azure_key_vault"
    vault_url: "https://myvault.vault.azure.net"
    secret_name: "guardrail-api-key"
    version: ""                    # Optional — omit or leave empty for latest
```

`source` enum: `env` | `aws_secrets_manager` | `gcp_secret_manager` | `azure_key_vault`.

The runtime resolves the `secret_ref` once at agent startup and caches the value for the duration of the run. A `secret_ref` that fails to resolve is a hard runtime error — the agent will not start.

---

### `invocation` — execution settings (required for external providers)

```yaml
invocation:
  timeout_ms: 300                  # Hard timeout for the guardrail call
  on_timeout: "fail_closed"        # fail_open | fail_closed
  on_provider_error: "fail_closed" # fail_open | fail_closed — behavior on provider 5xx
  retry_policy:
    max_attempts: 2
    backoff_ms: 100
```

| Field | Default | Description |
|---|---|---|
| `timeout_ms` | `500` | Maximum time to wait for the provider response. If exceeded, `on_timeout` applies. |
| `on_timeout` | `fail_closed` | `fail_closed` — treat as triggered (block/refuse); `fail_open` — treat as passed (allow through). |
| `on_provider_error` | `fail_closed` | Same as `on_timeout` but triggered by a provider error (5xx, network failure). |
| `retry_policy.max_attempts` | `1` | How many times to retry a failed call before applying `on_provider_error`. |
| `retry_policy.backoff_ms` | `100` | Initial backoff in milliseconds (exponential). |

**`fail_open` vs. `fail_closed`**

This is a deliberate safety policy choice. `fail_closed` means "if we cannot verify safety, block the request." `fail_open` means "if we cannot verify safety, allow the request." The right default depends on the check:

- For security-critical checks (PII, prompt injection, unsafe content): use `fail_closed`. Allowing an unverified request is worse than a false block.
- For quality checks (schema validation): `fail_open` may be acceptable if the check is supplementary and a false block would degrade user experience unacceptably.

Never set `fail_open` for guardrails with `on_fail: "refuse"` on the agent side — the fail-open behavior would silently bypass the refuse action.

---

### `fallback` — degraded-mode behavior (recommended for external providers)

```yaml
fallback:
  enabled: true
  fallback_guardrail_id: "pii-scan-lite"   # Platform-native fallback guardrail_id
  fallback_mode: "platform-native"         # platform-native | skip
  emit_warning: true                       # Log a warning event when fallback activates
```

| Field | Default | Description |
|---|---|---|
| `enabled` | `false` | Whether to activate a fallback when the primary provider is unavailable. |
| `fallback_guardrail_id` | — | The `guardrail_id` of a platform-native fallback check. Must resolve to a registered guardrail with `provider: "platform-native"`. |
| `fallback_mode` | `platform-native` | `platform-native` — invoke the fallback guardrail; `skip` — skip the check entirely (only for non-critical checks). |
| `emit_warning` | `true` | Emit a structured warning event to the audit log when fallback activates. |

`skip` should only be used for supplementary, non-security checks. Using `skip` on a security-critical guardrail effectively acts as `fail_open` and must be explicitly justified.

---

### `scope` — applicable pipeline positions (required)

```yaml
scope:
  positions: ["input", "output"]   # input | output | tool_call | tool_response
  content_types: ["text"]          # text | image | audio | structured
```

`positions` declares which pipeline positions this guardrail supports. Agents may attach the guardrail to any of the listed positions. Attaching it to an unlisted position is a lint warning.

`content_types` declares which content types this guardrail can evaluate. If an agent routes content of an unsupported type to this guardrail, it is a hard validation error.

---

## Full example: AWS Bedrock PII guardrail

```yaml
---
spec_version: "1.2"
guardrail_id: "pii-scan"
display_name: "PII Scan (Bedrock)"
provider: "aws-bedrock-guardrail"
status: "active"
owner: "ai-safety-team"
last_updated: "2026-04-12"

meta:
  description: >
    Detects and redacts personally identifiable information in agent inputs
    and outputs using AWS Bedrock Guardrails.
  tags: ["pii", "privacy", "gdpr"]
  applies_to: ["input", "output"]

provider_config:
  aws-bedrock-guardrail:
    guardrail_id: "abc123xyz"
    guardrail_version: "3"
    region: "eu-west-1"
    credentials:
      source: "iam_role"

invocation:
  timeout_ms: 300
  on_timeout: "fail_closed"
  on_provider_error: "fail_closed"
  retry_policy:
    max_attempts: 2
    backoff_ms: 100

fallback:
  enabled: true
  fallback_guardrail_id: "pii-scan-lite"
  fallback_mode: "platform-native"
  emit_warning: true

scope:
  positions: ["input", "output"]
  content_types: ["text"]
---

# PII Scan (Bedrock)

Uses AWS Bedrock Guardrails to detect and redact personally identifiable information
(names, email addresses, phone numbers, national identifiers, and similar) before the
model processes input and before responses are returned to callers.

**Recommended agent usage:**

```yaml
guardrails:
  input:
    - ref: "pii-scan"
      mode: "detect"
      on_fail: "redact"
  output:
    - ref: "pii-scan"
      mode: "block"
      on_fail: "redact"
```

Use `mode: "detect"` on input to redact before the model sees the data, and
`mode: "block"` on output to catch any PII the model reproduced from tool results
or retrieved context.

This guardrail is required for all agents operating under GDPR, HIPAA, or any
data-handling policy that restricts exposure of personal data.
```

---

## Relationship to `policies.pii_redaction`

The `policies.pii_redaction: true` flag in the agent definition and a `ref: "pii-scan"` guardrail can coexist and both run. They serve complementary roles:

| | `policies.pii_redaction: true` | `ref: "pii-scan"` guardrail |
|---|---|---|
| **What runs it** | Platform-built redaction pass (always available) | External provider (Bedrock, Azure, etc.) |
| **Configuration** | None — on or off | Full provider config, version pinning, fallback |
| **Precision** | Broad coverage, heuristic-based | Provider-specific, often tunable |
| **Failure mode** | Never errors (platform-built) | Can fail — governed by `on_provider_error` |

In a defense-in-depth configuration, set both: `pii_redaction: true` for baseline coverage that never fails, and a provider-backed guardrail for higher-precision detection on sensitive workflows.
