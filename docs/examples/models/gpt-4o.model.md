# Example — GPT-4o (OpenAI)

---

```markdown
---
spec_version: "1.2"
model_id: "gpt-4o"
version: "1.0.0"
status: "active"

meta:
  name: "GPT-4o (OpenAI)"
  owner: "platform-ml-team"
  last_updated: "2026-01-15"

provider:
  type: "openai"
  config:
    model: "gpt-4o"
    credentials:
      source: "aws_secrets_manager"
      secret_id: "prod/openai/api-key"
      region: "us-east-1"
      key: "api_key"

capabilities:
  context_window: 128000
  max_output_tokens: 16384
  supports_tools: true
  supports_system_prompt: true
  supports_streaming: true
  modalities: ["text", "image"]

defaults:
  temperature: 0.7
  max_tokens: 4096
---

# Description

GPT-4o is OpenAI's flagship multimodal model, accessed directly through the OpenAI API. It supports text and image inputs, tool use, and a 128,000-token context window.

Use this model when agents must interoperate with OpenAI-specific features, when your organization has an existing OpenAI contract, or when evaluating output quality across providers.

The credentials are fetched at runtime from the backend declared in `credentials.source`. They must never appear in any AML file. To use an environment variable during local development, set `source: "env"` and add `name: "OPENAI_API_KEY"`.
```

---

## Notes

- **Credential model**: The runtime resolves `credentials` at agent startup. The example above reads the API key from AWS Secrets Manager (`prod/openai/api-key`, key `api_key`). For development, substitute `source: "env"` and `name: "OPENAI_API_KEY"` to read from an environment variable instead. The key value must never appear in any AML file.
- **Secret rotation**: Rotate the secret in AWS Secrets Manager (or whichever backend is declared). No AML files need to change.
- **Rate limits**: OpenAI rate limits apply per organization and tier. For high-throughput agents, consider `litellm` with load balancing across multiple keys.
- **Alternative endpoint**: To use an Azure OpenAI deployment instead, set `base_url` to your Azure endpoint and update `credentials` accordingly. The `model` field should match your Azure deployment name.
