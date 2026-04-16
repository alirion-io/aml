# Example — GPT-4o (OpenAI)

---

```markdown
---
spec_version: "1.2"
model_id: "gpt-4o"
display_name: "GPT-4o (OpenAI)"
provider: "openai"
status: "active"
owner: "platform-ml-team"
last_updated: "2026-01-15"

provider_config:
  openai:
    model: "gpt-4o"
    api_key:
      secret_ref:
        source: "aws_secrets_manager"
        secret_id: "prod/openai/api-key"
        region: "us-east-1"
        key: "api_key"

capabilities:
  context_window: 128000
  max_output_tokens: 16384
  supports_tools: true
  supports_vision: true
  supports_system_prompt: true
  supports_streaming: true
  modalities: ["text", "image"]

defaults:
  temperature: 0.7
  max_tokens: 4096
---

# GPT-4o (OpenAI)

GPT-4o is OpenAI's flagship multimodal model, accessed directly through the OpenAI API. It supports text and image inputs, tool use, and a 128,000-token context window.

Use this model when agents must interoperate with OpenAI-specific features, when your organization has an existing OpenAI contract, or when evaluating output quality across providers.

The API key is fetched at runtime from the secret manager backend declared in `api_key.secret_ref`. It must never appear in any AML file. To use an environment variable during local development, change the `secret_ref` source to `env` and provide the variable name.
```

---

## Notes

- **Credential model**: The runtime resolves the `secret_ref` at agent startup. The example above reads the API key from AWS Secrets Manager (`prod/openai/api-key`, key `api_key`). For development, substitute `source: "env"` and `name: "OPENAI_API_KEY"` to read from an environment variable instead. The key value must never appear in any AML file.
- **Secret rotation**: Rotate the secret in AWS Secrets Manager (or whichever backend is declared). No AML files need to change.
- **Rate limits**: OpenAI rate limits apply per organization and tier. For high-throughput agents, consider `litellm` with load balancing across multiple keys.
- **Alternative endpoint**: To use an Azure OpenAI deployment instead, set `base_url` to your Azure endpoint and update `api_key_secret` accordingly. The `model` field should match your Azure deployment name.
