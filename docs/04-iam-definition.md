# IAM Role Definition Specification

> **File naming**: `iam/<iam_id>.iam.md`

> **Audience**: Security team

---

## Overview

An IAM role definition file describes a named **execution role** that an agent assumes at runtime. It declares which tools the agent is permitted to call and which knowledge bases the agent may query, serving as the ceiling of the agent's tool and knowledge base access.

This is **indicative at this stage** — the compiler uses the role as a documentation and validation surface, not to create or verify the actual cloud role. The connection between an AML role and its cloud counterpart (e.g., an AWS Lambda execution role) is a future capability. For now, the role helps authors and reviewers reason about least-privilege intent and enables the compiler to flag agents referencing tools outside their declared scope.

This is deliberately scoped to *what the agent can do*, not *who can call the agent*. Caller access — who or what is allowed to invoke the Lambda function or API endpoint that runs the agent — is an infrastructure concern managed outside AML, using native cloud IAM tools (AWS resource policies, API Gateway authorizers, Cognito, etc.).

The mental model maps directly to AWS:

```
AML IAM role  →  documents intent for the AWS Lambda execution role
Caller access →  Lambda resource policy / API GW authorizer  (managed outside AML)
```

An agent references exactly one IAM role. The agent's `tools` section may further restrict which tools from the role it actually uses, but it cannot reference tools that are not listed in the role. Similarly, the agent's `knowledge` section may only reference KBs listed in the role's `knowledge_bases` grants.

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

### `tools` — tool grants (optional)

Declares the set of tools this role permits an agent to use. An agent referencing this role may only call tools listed here. References to tools outside this set are hard validation errors.

```yaml
tools:
  - ref: "search-product-kb"
    access: "call"

  - ref: "get-account-info"
    access: "call"

  - ref: "send-email"
    access: "call"
    approval_required: true        # Override: always require approval for this tool
    max_calls_per_run: 1           # Override: tighten from tool's default of 3

  - ref: "create-ticket"
    access: "call"
    approval_required: true

  - ref: "delete-account"
    access: "deny"
    notes: "Explicitly denied. Support agents may never delete accounts."
```

#### Tool access levels

| Level | Description |
|---|---|
| `call` | agent may invoke this tool |
| `call-read-only` | agent may invoke only if the tool's type is `retrieval` or `function`; denied for `action` types |
| `deny` | explicitly denied; overrides any looser grant inherited from another role (if role composition is added in a future version) |

**Tightening only.** The role may impose stricter constraints than the tool's registry defaults (e.g., add `approval_required: true`, reduce `max_calls_per_run`). It may not loosen them. Attempting to set `approval_required: false` for a tool that declares `approval_required: true` in its registry definition is a hard validation error.

`ref` must resolve to a registered `.tool.md` file with `status: active` or `status: deprecated`. References to `disabled` tools are hard validation errors.

---

### `knowledge_bases` — KB access grants (optional)

Declares the set of knowledge bases this role permits an agent to query. An agent referencing this role may only retrieve from knowledge bases listed here. References to KBs outside this set are hard validation errors.

```yaml
knowledge_bases:
  - ref: "product-docs"
    access: "read"

  - ref: "brand-guidelines"
    access: "read"

  - ref: "legal-policies"
    access: "deny"
    notes: "Legal KB is restricted to compliance and privacy roles only."
```

#### KB access levels

| Level | Description |
|---|---|
| `read` | Agent may retrieve from this knowledge base |
| `deny` | Explicitly denied; overrides any looser grant inherited from another role |

`ref` must resolve to a registered `.kb.md` file with `status: active` or `status: deprecated`. References to `disabled` KBs are hard validation errors.

If `knowledge_bases` is omitted, the agent has no KB access by default.

---

### `collections` — memory collection grants (optional)

Declares the set of memory collections this role permits an agent to read from or write to. An agent referencing this role may only access collections listed here. References to collections outside this set are hard validation errors.

```yaml
collections:
  - ref: "customer-preferences"
    access: "read-write"

  - ref: "session-state-cache"
    access: "read"

  - ref: "audit-log"
    access: "deny"
    notes: "Audit log is read-only for compliance roles only."
```

#### Collection access levels

| Level | Description |
|---|---|
| `read` | Agent may read from this collection |
| `write` | Agent may write to this collection (subject to collection's own `writeback` policy) |
| `read-write` | Agent may both read from and write to this collection |
| `deny` | Explicitly denied; overrides any looser grant inherited from another role |

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
    Grants read access to product documentation and the ability to send
    confirmation emails and create support tickets.
    Billing tools and account-deletion tools are not in scope for this role.
  owner: "platform-security"
  tags: ["support", "cx"]
  last_updated: "2025-04-01"

tools:
  - ref: "search-product-kb"
    access: "call"

  - ref: "get-account-info"
    access: "call-read-only"

  - ref: "send-email"
    access: "call"
    approval_required: true
    max_calls_per_run: 1

  - ref: "create-ticket"
    access: "call"
    approval_required: true

knowledge_bases:
  - ref: "product-docs"
    access: "read"

  - ref: "brand-guidelines"
    access: "read"

collections:
  - ref: "customer-preferences"
    access: "read-write"
---

# Support Agent Role

This role is the minimal execution role for agents handling first-line customer support.
It covers reading product knowledge and taking low-risk actions (email, ticket creation)
that require user confirmation before execution.

Billing queries, refund issuance, and account management are handled by separate roles
with stricter approval chains.
```

---

## Validation rules

### Hard validation failures

- Missing any required field (`iam_id`, `version`, `status`, `meta.name`, `meta.description`, `meta.owner`).
- Invalid `iam_id` format.
- Any `ref` in `tools` that does not resolve to a registered `.tool.md` file.
- A `ref` in `tools` that resolves to a `disabled` tool.
- `approval_required: false` declared for a tool whose registry definition sets `approval_required: true`.
- Any `ref` in `knowledge_bases` that does not resolve to a registered `.kb.md` file.
- A `ref` in `knowledge_bases` that resolves to a `disabled` KB.
- Any `ref` in `collections` that does not resolve to a registered `.collection.md` file.
- A `ref` in `collections` that resolves to a `disabled` collection.
- `access: "write"` or `access: "read-write"` granted on a collection whose `writeback.enabled` is `false`.

### Recommended lint rules

- `access: "call"` for an `action`-type tool without `approval_required: true`.
- `meta.last_updated` older than 180 days — flag for review.
- Agent referencing this role declares tools outside the role's granted set.
