# Example — Support Agent IAM Role

---

```markdown
---
spec_version: "1.2"
iam_id: "support-agent-role"
version: "1.0.0"
status: "active"

meta:
  name: "Support Agent Role"
  description: >
    Execution role for first-line customer support agents.
    Grants access to the CRM account API, the product knowledge base,
    and the ticketing system for opening and updating cases.
    Does not grant access to billing, financial, admin, or account-deletion tools.
  owner: "platform-security"
  tags: ["support", "cx", "crm", "ticketing"]
  last_updated: "2026-04-17"

cloud:
  provider: "aws"
  role_arn: "arn:aws:iam::123456789012:role/SupportAgentRole"

tools:
  - ref: "get-account-info"
  - ref: "create-ticket"
  - ref: "update-ticket"
  - ref: "send-email"

knowledge_bases:
  - ref: "product-docs"
  - ref: "brand-guidelines"

collections:
  - ref: "customer-preferences"
---
```

---

## Purpose

This role defines the **permission ceiling** for any agent handling first-line customer support interactions. It covers two external API surfaces:

- **CRM read** (`get-account-info`) — agents may look up account status, contact details, and subscription tier to personalise responses and route escalations correctly.
- **Ticketing write** (`create-ticket`, `update-ticket`) — agents may open new support cases and add notes to existing ones.

Email notifications (`send-email`) are permitted for automated case-confirmation messages.

## Notes

This role does not grant access to billing, financial, or account-deletion tools. Any agent referencing `support-agent-role` is limited to the tools, knowledge bases, and collections listed above.
