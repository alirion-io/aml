# IAM Role Definition Specification

> **File naming**: `iam/<iam_id>.iam.md`

> **Audience**: Security team

---

## Overview

An IAM role definition file describes a named **execution role** that an agent assumes at runtime. It declares which tools the agent is permitted to call and which knowledge bases the agent may query, serving as the ceiling of the agent's tool and knowledge base access.

The `cloud` block links an AML role to its counterpart in the cloud provider — an AWS IAM role ARN, an Azure managed-identity client ID, or a GCP service account. This lets the compiler resolve which actual execution identity an agent will assume at runtime and enables tooling to cross-reference the AML definition against the live cloud configuration.

This is deliberately scoped to *what the agent can do*, not *who can call the agent*. Caller access — who or what is allowed to invoke the function or API endpoint that runs the agent — is an infrastructure concern managed outside AML, using native cloud IAM tools (AWS resource policies, API Gateway authorizers, Cognito, etc.).

An agent references exactly one IAM role. The agent's `tools` section may further restrict which tools from the role it actually uses, but it cannot reference tools that are not listed in the role. Similarly, the agent's `knowledge` section may only reference KBs listed in the role's `knowledge_bases` grants, and its `collections` section may only reference collections listed in the role's `collections` grants.

IAM role files are authored and owned by the platform security team and approved before any agent may reference them.

---

## File structure

```
---
[YAML front matter — all structured fields]
---

# Purpose           (optional editorial section)
# Notes             (optional editorial section)
```

The Markdown body is entirely editorial. The compiler ignores it.

---

## YAML front matter — complete field reference

### Top-level required fields

```yaml
spec_version: "1.2"
```

```yaml
iam_id: "support-agent-role"
```
Stable, immutable identifier for this role. Lowercase kebab-case. Must match `^[a-z0-9_-]{3,64}$`. Immutable once referenced by any published agent definition.

```yaml
version: "1.0.0"
```
Semantic version. Patch for descriptions or annotation changes; minor for adding tool grants; major for removing tool grants (breaking change for referencing agents).

```yaml
status: "active"
```
Lifecycle state. Enum: `draft` | `active` | `deprecated` | `disabled`. Agents referencing a `disabled` role fail hard validation. Agents referencing a `deprecated` role receive a lint warning.

---

### `meta` — descriptive metadata (required)

```yaml
meta:
  name: "Support Agent Role"            # Human-readable display name (required)
  description: >                         # Two to four sentences (required)
    Execution role for first-line customer support agents.
    Grants read access to product knowledge bases and the ability to send
    transactional email and create support tickets.
    Does not grant access to billing, financial, or admin tools.
  owner: "platform-security"            # Team responsible for this role (required)
  tags: ["support", "cx", "read-write"] # Optional searchable labels
  last_updated: "2025-04-01"           # ISO 8601 date (recommended)
```

---

### `cloud` — cloud role binding (optional, recommended)

Links this AML role to its actual execution identity in the cloud provider. The structure varies by provider.

**AWS** — reference an IAM role by ARN:

```yaml
cloud:
  provider: "aws"
  role_arn: "arn:aws:iam::123456789012:role/SupportAgentRole"
```

**Azure** — reference a managed identity by client ID:

```yaml
cloud:
  provider: "azure"
  client_id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

**GCP** — reference a service account by email:

```yaml
cloud:
  provider: "gcp"
  service_account: "support-agent@my-project.iam.gserviceaccount.com"
```

`provider` is an enum: `aws` | `azure` | `gcp`. When `cloud` is present, the provider-specific identifier field is required (`role_arn` for AWS, `client_id` for Azure, `service_account` for GCP). Omitting `cloud` is permitted but produces a lint warning.

---

### `tools` — tool grants (optional)

Declares the set of tools this role permits an agent to use. An agent referencing this role may only call tools listed here. References to tools outside this set are hard validation errors.

```yaml
tools:
  - ref: "search-product-kb"
  - ref: "get-account-info"
  - ref: "send-email"
```

`ref` must resolve to a registered `.tool.md` file with `status: active` or `status: deprecated`. References to `disabled` tools are hard validation errors.

---

### `knowledge_bases` — KB access grants (optional)

Declares the set of knowledge bases this role permits an agent to query. An agent referencing this role may only retrieve from knowledge bases listed here. References to KBs outside this set are hard validation errors.

```yaml
knowledge_bases:
  - ref: "product-docs"
  - ref: "brand-guidelines"
```

`ref` must resolve to a registered `.kb.md` file with `status: active` or `status: deprecated`. References to `disabled` KBs are hard validation errors.

If `knowledge_bases` is omitted, the agent has no KB access by default.

---

### `collections` — memory collection grants (optional)

Declares the set of memory collections this role permits an agent to access. An agent referencing this role may only access collections listed here. References to collections outside this set are hard validation errors.

```yaml
collections:
  - ref: "customer-preferences"
  - ref: "session-state-cache"
```

`ref` must resolve to a registered `.collection.md` file with `status: active` or `status: deprecated`. References to `disabled` collections are hard validation errors.

If `collections` is omitted, the agent has no memory collection access by default.

---

## What this role does NOT control

The following are explicitly out of scope for AML IAM roles:

- **Who can invoke the agent** — handled by Lambda resource policies, API Gateway authorizers, or equivalent. Configure these in your Terraform / CDK / CloudFormation infrastructure.
- **Authentication of end users** — handled by Cognito, Auth0, or your identity provider at the API layer.
- **Network-level access** — VPC configuration, security groups, and NACLs are infrastructure concerns.
- **Cross-account access** — AWS STS assume-role chains are infrastructure concerns.

None of these should appear in an AML file.

---

## Example — complete IAM role file

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
    Billing tools and account-deletion tools are not in scope for this role.
  owner: "platform-security"
  tags: ["support", "cx"]
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

## Validation rules

### Hard validation failures

- Missing any required field (`iam_id`, `version`, `status`, `meta.name`, `meta.description`, `meta.owner`).
- Invalid `iam_id` format.
- Any `ref` in `tools` that does not resolve to a registered `.tool.md` file.
- A `ref` in `tools` that resolves to a `disabled` tool.
- Any `ref` in `knowledge_bases` that does not resolve to a registered `.kb.md` file.
- A `ref` in `knowledge_bases` that resolves to a `disabled` KB.
- Any `ref` in `collections` that does not resolve to a registered `.collection.md` file.
- A `ref` in `collections` that resolves to a `disabled` collection.
- `cloud.provider` present but the required provider-specific identifier field is absent (e.g., `cloud.provider: "aws"` without `role_arn`).
- `cloud.provider` set to an unrecognised value.

### Recommended lint rules

- `cloud` block is absent — the AML role cannot be linked to a live cloud identity.
- `meta.last_updated` older than 180 days — flag for review.
- Agent referencing this role declares tools outside the role's granted set.
