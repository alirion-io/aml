---
spec_version: "1.2"
guardrail_id: "bedrock-pii-scan"
version: "2.0.0"
status: "active"

behaviour:
  result_type: "transform"
  content_types: ["text"]

meta:
  name: "PII Scan (Bedrock Guardrails)"
  owner: "ai-safety-team"
  last_updated: "2026-04-27"
  description: >
    Detects and redacts personally identifiable information — names, email addresses,
    phone numbers, national identifiers, and similar — using AWS Bedrock Guardrails.
    Suitable for agents operating under GDPR, HIPAA, or any data policy that restricts
    exposure of personal data.
  tags: ["pii", "privacy", "gdpr", "hipaa"]

transport:
  type: "lambda"
  function_arn: "arn:aws:lambda:eu-west-1:123456789:function:pii-scan-bedrock-v3"
  invocation_type: "RequestResponse"
  payload_format: "json"
  credentials:
    scheme: "iam-role"          # Runtime uses the agent's declared IAM role

invocation:
  timeout_ms: 300
  on_timeout:
    severity: 10                # If Bedrock does not respond in time, return maximum severity
  on_provider_error:
    severity: 10
  retry_policy:
    max_attempts: 2
    backoff_ms: 100

fallback:
  enabled: true
  fallback_guardrail_id: "pii-scan-lite"   # Platform-native fallback if Bedrock is unavailable
  emit_warning: true
---

# PII Scan (Bedrock Guardrails)

Uses AWS Bedrock Guardrails to detect and redact personally identifiable information
before the model processes input and before responses are returned to callers.

## When to use this guardrail

Attach to the `input` position to ensure PII is stripped before it reaches the model.
Attach to the `output` position to catch any PII the model may have reproduced from
tool results or retrieved knowledge base chunks.

Use both positions for agents that handle customer data in regulated contexts (GDPR, HIPAA).

## Recommended agent configuration

```yaml
guardrails:
  input:
    - ref: "bedrock-pii-scan"
      on_fail: "apply"          # Apply the redacted version before the model sees the data
  output:
    - ref: "bedrock-pii-scan"
      on_fail: "reject"         # If output redaction fails, halt rather than leak PII
```

## Updating the guardrail version

When a new version of the Bedrock guardrail policy is published in the AWS console:

1. Note the new numeric version (visible in the Bedrock console under **Guardrails > Versions**).
2. Update `guardrail_version` in this file.
3. Increment `last_updated`.
4. Open a pull request — all agents referencing `bedrock-pii-scan` will automatically pick up the new version after the next compile.

Do not update `guardrail_id`. This value is immutable once the guardrail is created in AWS.

## Relationship to `policies.pii_redaction`

This guardrail and `policies.pii_redaction: true` can coexist and both run. The policy provides platform-built baseline coverage that never fails; this guardrail adds Bedrock-backed precision and configurability. Use both in depth-in-defense configurations.
