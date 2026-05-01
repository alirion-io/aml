# Model Definition Specification

> **File naming**: `models/<model_id>.model.md`

> **Audience**: Platform engineers, ML engineers, product owners

---

## Overview

A model definition file declares one named model that agents can reference. It maps a stable, human-readable model identifier (e.g., `claude-4-sonnet`) to a specific provider, provider-level model name, and configuration. The compiler resolves model references from agent definitions and injects the correct provider configuration into the compiled payload. The runtime never reads raw model identifiers — it only executes compiled payloads.

This indirection serves two purposes. First, it decouples agent files from provider-specific naming conventions: an agent author writes `model: "claude-4-sonnet"` rather than `model_id: "us.anthropic.claude-sonnet-4-20250514-v1:0"`. Second, it allows the platform team to update provider configurations, swap regions, or rotate API key references in a single file without touching any agent definition.

Credentials are never stored in model definition files. Instead, each provider config uses a `credentials` block with a `source` field that tells the runtime *where* to fetch the value at execution time — environment variable, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, or AWS IAM role.

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

## Description (prose body)

The `# Description` heading opens the optional Markdown body that follows the closing `---` of the YAML front matter. It has no effect on compilation or runtime behavior — it exists exclusively for human readers and the model registry documentation UI.

**What to include:**

- **What the model is** — provider, model family, and any notable architecture notes (e.g., multimodal, reasoning model, instruction-tuned variant).
- **When to use it** — recommended scenarios, relative strengths compared to other registered models, and workload types it is optimized for.
- **When not to use it** — known limitations, cost or latency trade-offs, or cases where a different registered model is a better fit.
- **Operational notes** — how credentials are resolved at runtime, any region or quota constraints, and local-development alternatives (e.g., swapping `source: "iam_role"` for `source: "env"`).

Keep the description focused and actionable. Aim for two to four short paragraphs. Avoid duplicating the one-to-two sentence summary already in `meta.description` — the prose body is the right place to expand on context, trade-offs, and usage guidance that would not fit there.

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
version: "1.0.0"
```
Semantic version of this model definition (MAJOR.MINOR.PATCH). Increment whenever provider config, capabilities, or defaults change.

```yaml
status: "active"
```
Lifecycle state. Enum: `active` | `deprecated` | `disabled`. A `deprecated` model can still be compiled but triggers a lint warning on any agent that references it. A `disabled` model causes a hard compile error.

---

### `meta` — descriptive metadata (required)

```yaml
meta:
  name: "Claude 4 Sonnet (Bedrock, us-east-1)"  # Required — display name in the registry UI
  description: "Balanced model for intelligence and speed via Amazon Bedrock."  # Optional
  owner: "platform-ml-team"                      # Optional
  tags: ["bedrock", "anthropic", "vision"]        # Optional — searchable labels
  last_updated: "2026-01-15"                      # Optional — ISO 8601 date
```

`meta.name` is the only required sub-field. All others are optional but recommended for registry discoverability.

---

### `provider` — provider declaration (required)

A single `provider` object with two required sub-fields: `type` selects the provider, `config` holds the provider-specific settings whose shape is validated against the declared `type`. Any secret values inside `config` (API keys, tokens, endpoint URLs) must use a `credentials` block — literal values are a hard validation error.

```yaml
provider:
  type: "bedrock"       # Required — selects the provider; determines the required shape of config
  config: { ... }       # Required — provider-specific settings; validated against type
```

`type` enum:

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

#### `bedrock` — Amazon Bedrock

```yaml
provider:
  type: "bedrock"
  config:
    model_id: "us.anthropic.claude-sonnet-4-20250514-v1:0"   # Required — Bedrock model ID
    region: "us-east-1"                                       # Required — AWS region
    credentials:
      source: "iam_role"    # iam_role | env_aws (default: iam_role)
```

Or with explicit environment variables:

```yaml
provider:
  type: "bedrock"
  config:
    model_id: "us.anthropic.claude-sonnet-4-20250514-v1:0"
    region: "us-east-1"
    credentials:
      source: "env_aws"
      access_key_id: "MY_AWS_KEY_ID"         # Name of the env var holding the access key ID
      secret_access_key: "MY_AWS_SECRET_KEY" # Name of the env var holding the secret access key
```

`credentials.source: "iam_role"` is the recommended production setting — the runtime assumes the IAM role declared in the agent definition and requires no extra fields. `env_aws` reads credentials from the environment variables named in `access_key_id` and `secret_access_key`. These are env var *names*, not the values themselves — the runtime resolves them at startup.

Model access must be enabled in Amazon Bedrock for the specified `model_id` and region. See the [AWS documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access-modify.html).

#### `anthropic` — Anthropic direct API

```yaml
provider:
  type: "anthropic"
  config:
    model: "claude-sonnet-4-5"   # Required — Anthropic model name
    credentials:                 # Required — credentials
      source: "aws_secrets_manager"
      secret_id: "prod/anthropic/api-key"
      region: "us-east-1"
      key: "api_key"           # Optional — if the secret is a JSON object
```

Alternatively, using an environment variable:

```yaml
provider:
  type: "anthropic"
  config:
    model: "claude-sonnet-4-5"
    credentials:
      source: "env"
      name: "ANTHROPIC_API_KEY"
```

The literal key value must never appear in this file. See [`credentials` — authenticating to providers](#credentials----authenticating-to-providers) below for all supported backends.

#### `openai` — OpenAI or OpenAI-compatible endpoint

```yaml
provider:
  type: "openai"
  config:
    model: "gpt-4o"                       # Required — model name
    credentials:                          # Required — credentials
      source: "aws_secrets_manager"
      secret_id: "prod/openai/api-key"
      region: "us-east-1"
    base_url: "https://api.openai.com/v1" # Optional — override for compatible endpoints
    organization: "org-abc123"            # Optional — OpenAI organization ID
```

`base_url` can point to any OpenAI-compatible API (Azure OpenAI, vLLM, LM Studio, etc.). If omitted, the official OpenAI endpoint is used.

#### `litellm` — LiteLLM unified interface

```yaml
provider:
  type: "litellm"
  config:
    model: "openai/gpt-4o"   # Required — LiteLLM model string (provider/model format)
    credentials:             # Required — credentials
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
provider:
  type: "ollama"
  config:
    model: "llama3.2"                     # Required — Ollama model name
    host: "http://localhost:11434"         # Optional — Ollama host (default: localhost:11434)
```

`Ollama` runs models locally. No API key is required. `host` must be accessible from the runtime environment. Recommended for development and offline/privacy scenarios only.

#### `llamaapi` — Llama API (Meta)

```yaml
provider:
  type: "llamaapi"
  config:
    model: "Llama-4-Scout-17B-16E-Instruct-FP8"  # Required — model name
    credentials:                                  # Required — credentials
      source: "aws_secrets_manager"
      secret_id: "prod/llamaapi/api-key"
      region: "us-east-1"
```

#### `mistral` — Mistral AI

```yaml
provider:
  type: "mistral"
  config:
    model: "mistral-large-latest"  # Required — Mistral model name
    credentials:                   # Required — credentials
      source: "gcp_secret_manager"
      project: "my-gcp-project"
      secret: "mistral-api-key"
      version: "latest"
```

#### `writer` — Writer (Palmyra)

```yaml
provider:
  type: "writer"
  config:
    model: "palmyra-x5"  # Required — model name
    credentials:         # Required — credentials
      source: "azure_key_vault"
      vault_url: "https://myvault.vault.azure.net"
      secret_name: "writer-api-key"
```

#### `custom` — custom provider

```yaml
provider:
  type: "custom"
  config:
    class_path: "myorg.models.CustomProvider"  # Required — fully qualified Python class
    config:                                     # Passed as kwargs to the provider constructor
      model: "my-model"
      endpoint: "https://models.myorg.internal/v1"
      credentials:                                # Credentials supported in custom config too
        source: "aws_secrets_manager"
        secret_id: "prod/custom-model/api-key"
        region: "us-east-1"
```

Custom providers must implement the Strands `Model` interface. They are registered by platform engineers and are not available to self-service authors.

#### `credentials` — authenticating to providers

Every `credentials` field across all providers uses the same structure: a `source` field selects the backend; all other fields depend on it. This applies uniformly whether the provider uses an API key, a cloud secret manager, or AWS identity-based auth.

```yaml
# AWS IAM role — no secret needed (Bedrock only)
credentials:
  source: "iam_role"

# Two AWS credential env vars (Bedrock with explicit keys)
credentials:
  source: "env_aws"
  access_key_id: "MY_AWS_KEY_ID"          # Name of the env var holding the access key ID
  secret_access_key: "MY_AWS_SECRET_KEY" # Name of the env var holding the secret access key

# Single environment variable (API key providers)
credentials:
  source: "env"
  name: "OPENAI_API_KEY"         # Name of the environment variable

# AWS Secrets Manager
credentials:
  source: "aws_secrets_manager"
  secret_id: "prod/openai/api-key"  # Secret name or full ARN
  region: "us-east-1"               # Optional — defaults to the provider's region
  key: "api_key"                    # Optional — JSON key if the secret value is a JSON object

# GCP Secret Manager
credentials:
  source: "gcp_secret_manager"
  project: "my-gcp-project"
  secret: "openai-api-key"
  version: "latest"                  # Optional — defaults to 'latest'

# Azure Key Vault
credentials:
  source: "azure_key_vault"
  vault_url: "https://myvault.vault.azure.net"
  secret_name: "openai-api-key"
  version: ""                        # Optional — omit or leave empty for latest
```

`source` enum: `iam_role` | `env_aws` | `env` | `aws_secrets_manager` | `gcp_secret_manager` | `azure_key_vault`.

The runtime resolves `credentials` once at agent startup and caches the value for the duration of the run. The resolved value is never written to logs or persisted. A `credentials` block that fails to resolve at startup is a hard runtime error — the agent will not start.

---

### `capabilities` — model capability declaration (recommended)

```yaml
capabilities:
  context_window: 200000        # Max input context in tokens
  max_output_tokens: 16000      # Max tokens the model can generate per call
  supports_tools: true          # Whether the model's API natively supports the tool-calling protocol
  supports_system_prompt: true  # Whether the model accepts a system prompt as a separate input
  supports_streaming: true      # Whether the model supports token-by-token streaming output
  modalities: ["text", "image"] # Enum array — allowed values below
```

| Field | Type | Description |
|---|---|---|
| `context_window` | integer | Maximum context window size in tokens. |
| `max_output_tokens` | integer | Maximum tokens the model can generate in a single call. |
| `supports_tools` | boolean | Whether the model's API **natively supports the tool/function-calling protocol** — i.e., the runtime can pass structured tool definitions in the request and receive a structured `tool_use` / `function_call` response block. This is a model-level API capability, not a statement about agent behaviour. Models with `supports_tools: false` cannot be used with agents that have a non-empty `tools` section. |
| `supports_system_prompt` | boolean | Whether the model accepts a system prompt as a distinct input, separate from the user turn. |
| `supports_streaming` | boolean | Whether the model supports token-by-token streaming output. |
| `modalities` | string\[\] | Input modalities supported by this model. Enum: `text` \| `image` \| `audio` \| `document`. Use this to declare vision (`image`), audio, or document understanding support rather than separate boolean flags. |

The compiler uses `capabilities` to validate agent definitions. For example, if an agent declares knowledge bases with document retrieval but the model does not list `"document"` in `modalities`, the compiler emits a lint warning. If `supports_tools: false`, any agent referencing this model with a non-empty `tools` section is a hard validation error.

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
| `provider.type` value is not in the supported enum | Hard error |
| `provider.config` shape does not match the declared `provider.type` | Hard error |
| A `credentials` field contains a literal value instead of a structured block with `source` | Hard error |
| `credentials.source` is not in the supported enum | Hard error |
| `credentials` for `aws_secrets_manager` is missing `secret_id` | Hard error |
| `credentials` for `gcp_secret_manager` is missing `project` or `secret` | Hard error |
| `credentials` for `azure_key_vault` is missing `vault_url` or `secret_name` | Hard error |
| `credentials` for `env` is missing `name` | Hard error |
| `credentials` for `env_aws` is missing `access_key_id` or `secret_access_key` | Hard error |
| `status: disabled` and the model is referenced by a compiled agent | Hard error |
| `capabilities.supports_tools: false` and the referencing agent has tools | Hard error |
| `status: deprecated` and the model is referenced by an active agent | Lint warning |
| `provider: ollama` used in an agent with `status: active` in production | Lint warning |
| `credentials.source` is not `iam_role` in a production-targeted Bedrock definition | Lint warning |
| `credentials.source: env` or `credentials.source: env_aws` used in a production-targeted definition (prefer a secret manager) | Lint warning |
| `capabilities` block is absent | Lint warning |
| `last_updated` is more than 180 days ago | Lint warning |

---

## Complete example

```yaml
spec_version: "1.2"
model_id: "claude-4-sonnet"
version: "1.0.0"
status: "active"

meta:
  name: "Claude 4 Sonnet (Bedrock, us-east-1)"
  owner: "platform-ml-team"
  last_updated: "2026-01-15"

provider:
  type: "bedrock"
  config:
    model_id: "us.anthropic.claude-sonnet-4-20250514-v1:0"
    region: "us-east-1"
    credentials:
      source: "iam_role"

capabilities:
  context_window: 200000
  max_output_tokens: 16000
  supports_tools: true
  supports_system_prompt: true
  supports_streaming: true
  modalities: ["text", "image"]

defaults:
  temperature: 0.7
  max_tokens: 4096
```

# Description

Claude 4 Sonnet is Anthropic's balanced model for intelligence and speed, accessed via Amazon Bedrock in `us-east-1`. It supports text and image inputs, native tool use, and a 200,000-token context window.

Use this model for general-purpose agent workloads that require strong reasoning, tool orchestration, or document understanding without the latency or cost of a larger frontier model. Prefer `claude-4-opus` when maximum reasoning depth matters more than throughput.

Credentials use `source: "iam_role"` — the runtime assumes the IAM role declared in the agent definition and requires no extra secret configuration. For local development, set `source: "env_aws"` and export `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
