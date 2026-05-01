# Example — Claude 4 Sonnet (Amazon Bedrock)


---

```markdown
---
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
---

# Description

Claude 4 Sonnet is Anthropic's balanced model for intelligence and speed, accessed here through Amazon Bedrock in `us-east-1`. It is the default model for most production agents on this platform.

Use this model for agents requiring multi-step reasoning, tool use, and document understanding with a large context window. For high-volume, low-latency agents where response quality requirements are lower, prefer `claude-4-haiku`. For the most complex reasoning tasks, prefer `claude-4-opus`.

Model access must be enabled in the AWS console for `us.anthropic.claude-sonnet-4-20250514-v1:0` in `us-east-1` before deploying any agent that references this model.
```

---

## Notes

- **Credential model**: The runtime assumes the IAM role declared in the agent's `role` field. No API key is needed — access is controlled through AWS IAM.
- **Cross-region inference**: The `us.` prefix in the Bedrock `model_id` enables cross-region inference, routing requests across US regions for availability.
- **Fallback**: Agents that use this as their primary model typically use `claude-4-haiku` as `fallback_model` for cost or availability reasons.
