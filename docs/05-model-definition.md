# Model Definition Specification

> **File naming**: `models/<model_id>.model.md`

> **Audience**: Platform engineers, ML engineers, product owners

---

## Overview

A model definition file declares one named model that agents can reference. It maps a stable, human-readable model identifier (e.g., `claude-4-sonnet`) to a specific provider, provider-level model name, and configuration. The compiler resolves model references from agent definitions and injects the correct provider configuration into the compiled payload. The runtime never reads raw model identifiers — it only executes compiled payloads.

This indirection serves two purposes. First, it decouples agent files from provider-specific naming conventions: an agent author writes `model: "claude-4-sonnet"` rather than `model_id: "us.anthropic.claude-sonnet-4-20250514-v1:0"`. Second, it allows the platform team to update provider configurations, swap regions, or rotate API key references in a single file without touching any agent definition.

API keys and other secrets are never stored in model definition files. Instead, each provider config that requires a secret uses a structured `secret_ref` block that tells the runtime *where* to fetch the value at execution time — either from an environment variable or from a secret manager such as AWS Secrets Manager, GCP Secret Manager, or Azure Key Vault.

---

## File structure

```
---
[YAML front matter — all structured fields]
---

# Description
[optional prose — what this model is, when to use it, usage guidance]
```

The prose body is optional and intended for the model registry documentation UI. It does not affect runtime behavior.

---

## YAML front matter — complete field reference

### Top-level required fields

```yaml
spec_version: "1.2"
```
The AML format version. Must equal a platform-approved version string.

```yaml
model_id: "claude-4-sonnet"
```
Stable, immutable identifier for this model entry. Used as the value of `runtime.model` in agent definition files. Lowercase kebab-case. Must match `^[a-z0-9_-]{3,64}$`. Once published, the `model_id` cannot change — create a new entry for a different model.

```yaml
display_name: "Claude 4 Sonnet (Bedrock)"
```
Human-readable name shown in the model registry UI and compiler output.

```yaml
provider: "bedrock"
```
Provider type. Determines which `provider_config` sub-key is required. Enum:

| Value | Provider |
|---|---|
| `bedrock` | Amazon Bedrock |
| `anthropic` | Anthropic direct API |
| `openai` | OpenAI or OpenAI-compatible endpoint |
| `litellm` | LiteLLM unified interface |
| `ollama` | Ollama (local models) |
| `llamaapi` | Llama API (Meta) |
| `mistral` | Mistral AI direct API |
| `writer` | Writer (Palmyra models) |
| `custom` | Custom provider — requires a `custom` config block |

```yaml
status: "active"
```
Lifecycle state. Enum: `active` | `deprecated` | `disabled`. A `deprecated` model can still be compiled but triggers a lint warning on any agent that references it. A `disabled` model causes a hard compile error.

```yaml
owner: "platform-ml-team"
```
Team or individual responsible for maintaining this model entry.

```yaml
last_updated: "2026-01-15"
```
ISO 8601 date of the last update to this file.

---

### `secret_ref` — secret reference pattern

Any field in a `provider_config` block that requires a secret value (API key, bearer token, etc.) uses a `secret_ref` object instead of a plain string. This allows the runtime to fetch the secret from different backends without the value ever appearing in a definition file.

```yaml
# Shorthand — environment variable (equivalent to the full form below)
api_key:
  secret_ref:
    source: "env"
    name: "OPENAI_API_KEY"         # Name of the environment variable

# AWS Secrets Manager
api_key:
  secret_ref:
    source: "aws_secrets_manager"
    secret_id: "prod/openai/api-key"  # Secret name or full ARN
    region: "us-east-1"               # Optional — defaults to the provider's region
    key: "api_key"                    # Optional — JSON key if the secret value is a JSON object

# GCP Secret Manager
api_key:
  secret_ref:
    source: "gcp_secret_manager"
    project: "my-gcp-project"
    secret: "openai-api-key"
    version: "latest"                  # Optional — defaults to 'latest'

# Azure Key Vault
api_key:
  secret_ref:
    source: "azure_key_vault"
    vault_url: "https://myvault.vault.azure.net"
    secret_name: "openai-api-key"
    version: ""                        # Optional — omit or leave empty for latest
```

`source` enum: `env` | `aws_secrets_manager` | `gcp_secret_manager` | `azure_key_vault`.

The runtime resolves the `secret_ref` once at agent startup and caches the value for the duration of the run. The resolved value is never written to logs or persisted. A `secret_ref` that fails to resolve at startup is a hard runtime error — the agent will not start.

---

### `provider_config` — provider-specific settings (required)

Exactly one sub-key matching the `provider` value must be present.

#### `bedrock` — Amazon Bedrock

```yaml
provider_config:
  bedrock:
    model_id: "us.anthropic.claude-sonnet-4-20250514-v1:0"   # Required — Bedrock model ID
    region: "us-east-1"                                       # Required — AWS region
    credentials:
      source: "iam_role"    # iam_role | env | aws_profile (default: iam_role)
```

`credentials.source: "iam_role"` is the recommended production setting — the runtime assumes the IAM role declared in the agent definition. `env` reads `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from the environment. 

Model access must be enabled in Amazon Bedrock for the specified `model_id` and region. See the [AWS documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access-modify.html).

#### `anthropic` — Anthropic direct API

```yaml
provider_config:
  anthropic:
    model: "claude-sonnet-4-5"   # Required — Anthropic model name
    api_key:                     # Required — secret reference for the API key
      secret_ref:
        source: "aws_secrets_manager"
        secret_id: "prod/anthropic/api-key"
        region: "us-east-1"
        key: "api_key"           # Optional — if the secret is a JSON object
```

Alternatively, using an environment variable:

```yaml
provider_config:
  anthropic:
    model: "claude-sonnet-4-5"
    api_key:
      secret_ref:
        source: "env"
        name: "ANTHROPIC_API_KEY"
```

The literal key value must never appear in this file. See the [`secret_ref` pattern](#secret_ref----secret-reference-pattern) section for all supported backends.

#### `openai` — OpenAI or OpenAI-compatible endpoint

```yaml
provider_config:
  openai:
    model: "gpt-4o"                       # Required — model name
    api_key:                              # Required — secret reference
      secret_ref:
        source: "aws_secrets_manager"
        secret_id: "prod/openai/api-key"
        region: "us-east-1"
    base_url: "https://api.openai.com/v1" # Optional — override for compatible endpoints
    organization: "org-abc123"            # Optional — OpenAI organization ID
```

`base_url` can point to any OpenAI-compatible API (Azure OpenAI, vLLM, LM Studio, etc.). If omitted, the official OpenAI endpoint is used.

#### `litellm` — LiteLLM unified interface

```yaml
provider_config:
  litellm:
    model: "openai/gpt-4o"   # Required — LiteLLM model string (provider/model format)
    api_key:                 # Required — secret reference for the underlying provider key
      secret_ref:
        source: "aws_secrets_manager"
        secret_id: "prod/openai/api-key"
        region: "us-east-1"
    api_base: "https://..."  # Optional — override base URL
    extra_params:            # Optional — passed through to LiteLLM
      drop_params: true
```

LiteLLM model strings follow the `provider/model` convention (e.g., `anthropic/claude-sonnet-4-5`, `mistral/mistral-large-latest`). See the [LiteLLM provider docs](https://docs.litellm.ai/docs/providers) for the full list.

#### `ollama` — Ollama (local models)

```yaml
provider_config:
  ollama:
    model: "llama3.2"                     # Required — Ollama model name
    host: "http://localhost:11434"         # Optional — Ollama host (default: localhost:11434)
```

`Ollama` runs models locally. No API key is required. `host` must be accessible from the runtime environment. Recommended for development and offline/privacy scenarios only.

#### `llamaapi` — Llama API (Meta)

```yaml
provider_config:
  llamaapi:
    model: "Llama-4-Scout-17B-16E-Instruct-FP8"  # Required — model name
    api_key:                                      # Required — secret reference
      secret_ref:
        source: "aws_secrets_manager"
        secret_id: "prod/llamaapi/api-key"
        region: "us-east-1"
```

#### `mistral` — Mistral AI

```yaml
provider_config:
  mistral:
    model: "mistral-large-latest"  # Required — Mistral model name
    api_key:                       # Required — secret reference
      secret_ref:
        source: "gcp_secret_manager"
        project: "my-gcp-project"
        secret: "mistral-api-key"
        version: "latest"
```

#### `writer` — Writer (Palmyra)

```yaml
provider_config:
  writer:
    model: "palmyra-x5"  # Required — model name
    api_key:             # Required — secret reference
      secret_ref:
        source: "azure_key_vault"
        vault_url: "https://myvault.vault.azure.net"
        secret_name: "writer-api-key"
```

#### `custom` — custom provider

```yaml
provider_config:
  custom:
    class_path: "myorg.models.CustomProvider"  # Required — fully qualified Python class
    config:                                     # Passed as kwargs to the provider constructor
      model: "my-model"
      endpoint: "https://models.myorg.internal/v1"
      api_key:                                    # Secret reference supported in custom config too
        secret_ref:
          source: "aws_secrets_manager"
          secret_id: "prod/custom-model/api-key"
          region: "us-east-1"
```

Custom providers must implement the Strands `Model` interface. They are registered by platform engineers and are not available to self-service authors.

---

### `capabilities` — model capability declaration (recommended)

```yaml
capabilities:
  context_window: 200000        # Max input context in tokens
  max_output_tokens: 16000      # Max tokens the model can generate per call
  supports_tools: true          # Whether the model supports tool/function calling
  supports_vision: true         # Whether the model accepts image inputs
  supports_system_prompt: true  # Whether the model accepts a system prompt
  supports_streaming: true      # Whether the model supports token streaming
  modalities: ["text", "image"] # Input modalities: text | image | audio | document
```

The compiler uses `capabilities` to validate agent definitions. For example, if an agent declares knowledge bases with document retrieval but the model does not support `"document"` in `modalities`, the compiler emits a lint warning. If `supports_tools: false`, any agent referencing this model with a non-empty `tools` section is a hard validation error.

---

### `defaults` — default inference parameters (optional)

```yaml
defaults:
  temperature: 0.7          # Default temperature; overridden by agent runtime.temperature
  max_tokens: 4096          # Default max output tokens; overridden by agent runtime.max_output_tokens
  top_p: 1.0                # Nucleus sampling parameter
  stop_sequences: []        # Token sequences that stop generation
```

Agent-level `runtime` fields override these defaults. Provider-level limits take precedence over both — a model that supports a maximum of 16,000 output tokens cannot be forced beyond that limit regardless of what the agent or defaults declare.

---

## How agents reference model definitions

In an agent definition file, the `runtime.model` and `runtime.fallback_model` fields hold a `model_id` string that the compiler resolves to a `.model.md` file:

```yaml
runtime:
  model: "claude-4-sonnet"          # Resolved to models/claude-4-sonnet.model.md
  fallback_model: "claude-4-haiku"  # Resolved to models/claude-4-haiku.model.md
```

An unresolvable `model_id` (no matching `.model.md` file with `status: active`) is a **hard compile error**. A reference to a model with `status: deprecated` is a **lint warning**. A reference to a model with `status: disabled` is a **hard compile error**.

---

## Validation rules

| Rule | Severity |
|---|---|
| `model_id` does not match `^[a-z0-9_-]{3,64}$` | Hard error |
| `provider` value is not in the supported enum | Hard error |
| Required `provider_config` sub-key for the declared `provider` is absent | Hard error |
| A secret field contains a literal secret value instead of a `secret_ref` block | Hard error |
| `secret_ref.source` is not in the supported enum | Hard error |
| `secret_ref` for `aws_secrets_manager` is missing `secret_id` | Hard error |
| `secret_ref` for `gcp_secret_manager` is missing `project` or `secret` | Hard error |
| `secret_ref` for `azure_key_vault` is missing `vault_url` or `secret_name` | Hard error |
| `secret_ref` for `env` is missing `name` | Hard error |
| `status: disabled` and the model is referenced by a compiled agent | Hard error |
| `capabilities.supports_tools: false` and the referencing agent has tools | Hard error |
| `status: deprecated` and the model is referenced by an active agent | Lint warning |
| `provider: ollama` used in an agent with `status: active` in production | Lint warning |
| `credentials.source` is not `iam_role` in a production-targeted definition | Lint warning |
| `secret_ref.source: env` used in a production-targeted definition (prefer a secret manager) | Lint warning |
| `capabilities` block is absent | Lint warning |
| `last_updated` is more than 180 days ago | Lint warning |

---

## Complete example

```yaml
spec_version: "1.2"
model_id: "claude-4-sonnet"
display_name: "Claude 4 Sonnet (Bedrock, us-east-1)"
provider: "bedrock"
status: "active"
owner: "platform-ml-team"
last_updated: "2026-01-15"

provider_config:
  bedrock:
    model_id: "us.anthropic.claude-sonnet-4-20250514-v1:0"
    region: "us-east-1"
    credentials:
      source: "iam_role"

capabilities:
  context_window: 200000
  max_output_tokens: 16000
  supports_tools: true
  supports_vision: true
  supports_system_prompt: true
  supports_streaming: true
  modalities: ["text", "image"]

defaults:
  temperature: 0.7
  max_tokens: 4096
```
