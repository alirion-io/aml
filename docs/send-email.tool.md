# Example — Send Email Tool

---

```markdown
---
spec_version: "1.2"
tool_id: "send-email"
version: "2.1.0"
status: "active"

meta:
  name: "Send Email"
  description: >
    Send a transactional email to one or more recipients using the internal email service.
    Use to notify customers of case resolutions, send document attachments,
    or deliver confirmation messages. Do not use for bulk, marketing,
    or unsolicited communications.
  owner: "comms-platform-team"
  tags: ["email", "write", "comms", "action"]
  last_updated: "2026-04-15"

type: "action"

parameters:
  type: object
  properties:
    to:
      type: array
      items: { type: string, format: "email" }
      description: "Recipient email addresses. Maximum 5."
      maxItems: 5
    subject:
      type: string
      description: "Email subject line. Keep under 80 characters."
      maxLength: 80
    body:
      type: string
      description: "Email body in plain text or Markdown. Maximum 10,000 characters."
      maxLength: 10000
    cc:
      type: array
      items: { type: string, format: "email" }
      description: "CC addresses (optional). Maximum 3."
      maxItems: 3
    template_id:
      type: string
      description: "Optional. Platform template ID. When provided, body is merged into the template."
    reply_to:
      type: string
      format: "email"
      description: "Optional reply-to address. Defaults to the service no-reply address."
  required: ["to", "subject", "body"]

transport:
  type: "rest-api"
  base_url: "https://mail.internal.example.com/v2"
  endpoint: "POST /messages/send"
  timeout_ms: 10000
  retry_policy:
    max_attempts: 1
    on_status: []
  auth:
    scheme: "bearer-token"
    token_source: "secrets:prod/agent/mail-api-token"

use_guidance:
  use_when:
    - "the customer has confirmed they want an email sent to them"
    - "a support case is resolved and the customer requires written confirmation"
    - "a document or attachment needs to be delivered to the customer's registered address"
  avoid_when:
    - "the customer has not explicitly requested or confirmed the email"
    - "the recipient address has not been verified or confirmed"
    - "the email content would constitute marketing or promotional communication"
    - "the body contains raw PII that has not been reviewed by the agent"
  side_effects:
    - "Sends an email to the specified recipients. This cannot be undone."
    - "Writes an audit log entry to agent_email_sends on every call."
    - "Triggers a Slack notification to #cx-alerts if more than 3 recipients are specified."

response_schema:
  type: object
  properties:
    message_id:
      type: string
      description: "Unique identifier of the sent message in the email service."
    status:
      type: string
      enum: ["sent", "queued", "failed"]
      description: "Delivery status of the message."
    queued_at:
      type: ["string", "null"]
      format: "date-time"
      description: "Timestamp when the message entered the queue, or null if sent immediately."
  required: ["status"]

error_codes:
  400:
    meaning: "Malformed request — invalid address format or missing required field"
    agent_action: "Review and correct the request. Do not retry automatically."
  401:
    meaning: "Authentication failure — bad or expired token"
    agent_action: "Escalate to platform. Do not retry."
  403:
    meaning: "Forbidden — caller is not authorised to send to this recipient domain"
    agent_action: "Do not retry. Inform the user that the send was blocked."
  429:
    meaning: "Rate limited"
    agent_action: "Wait for the Retry-After header value, then retry once."
  503:
    meaning: "Mail service unavailable"
    agent_action: "Retry once after 5 seconds. If still failing, escalate to human queue."
---

# Purpose
Enables agents to send transactional emails to customers and internal stakeholders.
This is a write-action tool — every call sends a real email. It must always be
called with user confirmation obtained first.

# Side effects
Every call results in an email being sent to real recipients. Calls cannot be undone.
All calls are logged to the compliance audit table regardless of success or failure.

# Notes
- Templates are managed by the comms-platform team. Request new templates via #comms-platform.
- The `reply_to` field should be set when the customer should be able to reply directly.
- Attachments are not currently supported. Use the `document-share` tool for file delivery.
- Changelog:
  - 2.1.0 — Added `reply_to` field and Slack notification on bulk sends.
  - 2.0.0 — Migrated to v2 endpoint with structured response schema.
  - 1.0.0 — Initial release.
```
