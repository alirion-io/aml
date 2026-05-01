---
spec_version: "1.2"
guardrail_id: "azure-content-moderation"
version: "2.0.0"
status: "active"

behaviour:
  result_type: "score"
  content_types: ["text"]

meta:
  name: "Content Moderation (Azure AI Content Safety)"
  owner: "ai-safety-team"
  last_updated: "2026-04-27"
  description: >
    Scores agent inputs and outputs against four harm categories — Hate, SelfHarm,
    Sexual, and Violence — using Azure AI Content Safety. Returns a normalized 0–10
    AML severity score. Suitable for customer-facing agents where harmful content
    must be detected before reaching the model or the caller.
  tags: ["content-moderation", "hate-speech", "violence", "safety"]

transport:
  type: "lambda"
  function_arn: "arn:aws:lambda:eu-west-1:123456789:function:content-mod-azure-v2"
  invocation_type: "RequestResponse"
  payload_format: "json"
  credentials:
    scheme: "iam-role"          # Runtime uses the agent's declared IAM role

invocation:
  timeout_ms: 400
  on_timeout:
    severity: 10                # If Azure does not respond in time, return maximum severity
  on_provider_error:
    severity: 10
  retry_policy:
    max_attempts: 2
    backoff_ms: 150

fallback:
  enabled: true
  fallback_guardrail_id: "content-mod-lite"   # Platform-native fallback if Azure is unavailable
  emit_warning: true
---

# Content Moderation (Azure AI Content Safety)

Uses [Azure AI Content Safety](https://learn.microsoft.com/azure/ai-services/content-safety/overview)
to evaluate agent inputs and outputs against four harm categories: **Hate**, **SelfHarm**,
**Sexual**, and **Violence**. Each category is scored on a 0–6 severity scale; this
guardrail blocks any request or response that meets or exceeds the configured threshold
for any category.

## When to use this guardrail

Attach to the `input` position to refuse harmful or off-topic user requests before they
reach the model. Attach to the `output` position to catch any harmful content the model
may generate, for example from adversarial prompts or retrieved document fragments.

Use both positions for customer-facing agents (support bots, public chat interfaces)
where any exposure to harmful output carries reputational or legal risk.

## Recommended agent configuration

```yaml
guardrails:
  input:
    - ref: "azure-content-moderation"
      severity_threshold: 5     # Trigger on moderate+ content (AML normalized 0–10)
      on_fail: "block"
  output:
    - ref: "azure-content-moderation"
      severity_threshold: 5
      on_fail: "block"
```

Adjust `severity_threshold` to match your product's content policy:
- `3` — strict: block anything above very mild
- `5` — balanced: block moderate and above (recommended default)
- `7` — permissive: block only severe content

## Severity scale mapping

Azure Content Safety returns a 0–6 severity per category. The Lambda wrapper normalizes this to AML's 0–10 scale before returning the result:

| Azure severity | AML severity | Meaning |
|---|---|---|
| 0 | 0–2 | Safe |
| 2 | 3–4 | Low |
| 4 | 5–6 | Medium |
| 6 | 8–10 | High |

## Relationship to other guardrails

This guardrail and `bedrock-pii-scan` serve different purposes and can be stacked:

```yaml
guardrails:
  input:
    - ref: "bedrock-pii-scan"
      on_fail: "apply"            # Redact PII before content moderation runs
    - ref: "azure-content-moderation"
      severity_threshold: 5
      on_fail: "block"            # Block if moderation score >= 5
```

PII redaction runs first to strip personal data, then content moderation checks the
sanitised input for harmful content. Both guardrails must pass before the model
receives the request.
