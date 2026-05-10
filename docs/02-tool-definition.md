# Tool Definition Specification


## Overview

Think of a tool as a specific action an AI agent can take in the real world — like sending an email, looking up a customer record, or submitting a form. Just as a human employee needs to know which systems they are allowed to use and how to use them, an agent needs a clear description of each tool at its disposal. A tool definition is that description: it tells the agent what the tool does, what information it needs to run it, and any rules or limits that apply.

More technically, a tool is any callable unit the agent can invoke at runtime — a function, an external API, or a backend service.

A tool definition file describes one registered tool that agents are permitted to use. It is the single authoritative source of truth for that tool's capabilities, call contract, authentication requirements, invocation policy, and usage guidance.

Tool files are authored by engineers and approved by the platform team. Agent files reference tools by ID — they do not copy tool definitions inline. When a tool's API changes, only the tool file changes; the compiler re-validates all referencing agents automatically.

---

## File structure

```
---
[YAML front matter — all structured fields]
---

# Purpose           (optional editorial section)
# Side effects      (optional editorial section)
# Notes             (optional editorial section)
```

The Markdown body is entirely editorial. The compiler ignores it. Runtime behavior is determined solely by the YAML front matter.

### Top-level fields at a glance

| Field | Required | Description |
|---|---|---|
| `spec_version` | Required | AML version string. Must match a platform-approved value. |
| `tool_id` | Required | Stable, immutable identifier. Lowercase kebab-case. |
| `version` | Required | Semantic version of this tool definition. |
| `status` | Required | Lifecycle state: `draft` \| `active` \| `deprecated` \| `disabled`. |
| `meta` | Required | Descriptive metadata. `name`, `description`, and `owner` are required inside. |
| `type` | Required | Behavioral contract: `retrieval` \| `action` \| `function` \| `human`. |
| `interface` | Required | Input/output schema. Both `input` and `output` sub-fields are required. |
| `transport` | Conditional | Invocation protocol and credentials. Required for all types except `function`. |
| `error_codes` | Optional | Per-status-code handling hints for the model. |
| `use_guidance` | Required | Behavioral hints (`use_when`, `avoid_when`, `side_effects`). |

---

## YAML front matter

### Top-level required fields

```yaml
spec_version: "1.2"
```
Must equal a platform-approved AML version string.

```yaml
tool_id: "send-email"
```
Stable, immutable identifier. Lowercase kebab-case. Must match `^[a-z0-9_-]{3,64}$`. Immutable once any agent references this tool in a published definition.

```yaml
version: "1.0.0"
```
Semantic version of this tool definition. Increment on every published change. Major version bump required for breaking changes to the parameter schema.

```yaml
status: "active"
```
Lifecycle state. Enum: `draft` | `active` | `deprecated` | `disabled`. Agents referencing a `disabled` tool fail hard validation. Agents referencing a `deprecated` tool receive a lint warning.

---

### `meta`

Descriptive metadata of the tool (required)

```yaml
meta:
  name: "Send Email"                # Human-readable display name (required)
  description: >                    # Two to five sentences, agent's perspective (required)
    Send a transactional email to one or more recipients using the internal email service.
    Use this to notify users of case resolutions, send confirmation messages,
    or deliver documents. Do not use for bulk or marketing email.
  owner: "comms-platform-team"      # Team responsible for this tool (required)
  tags: ["email", "write", "comms"] # Searchable labels (optional)
  last_updated: "2025-04-01"        # ISO 8601 date (recommended)
```

---

### `type`

Behavioral contract of the tool (required).

```yaml
type: "action"
```

`type` describes the **behavioral semantics** of the tool — what it means for the runtime and the model. It is strictly separate from the invocation protocol, which is declared in the `transport` block.

| Type | Description |
|---|---|
| `retrieval` | Read-only data lookup; no side effects. The runtime may cache results and parallelize calls. |
| `action` | Causes an external side effect (write, send, delete). |
| `function` | In-process computation; no external call and no side effects. `transport` may be omitted. |
| `human` | Routes to a human reviewer or queue. The runtime pauses agent execution until a human response is received. |

!!! note
    The invocation protocol (`rest-api`, `mcp`, `lambda`, `grpc`, etc.) belongs in `transport.type`, not here. A `retrieval` tool can be backed by a REST API, an MCP server, or a database — `type` does not constrain that choice.

---

### `interface`

Input and output contract (required).

Both `interface.input` and `interface.output` are required. They follow JSON Schema syntax and are written from the model's perspective. `interface.input` is used by the compiler to validate agent configurations and by the model at runtime to construct correct tool calls. `interface.output` is required so the model can interpret responses without ambiguity — a missing output schema is a hard validation failure. See [JSON Schema in YAML](08-json-schema.md) for the full field reference, supported types, constraints, and worked examples.

#### `interface.input`

```yaml
interface:
  input:
    type: object
    properties:
      to:
        type: array
        items: { type: string }
        description: "Recipient email addresses."
      subject:
        type: string
        description: "Email subject line."
      body:
        type: string
        description: "Email body in plain text or Markdown."
      cc:
        type: array
        items: { type: string }
        description: "CC addresses (optional)."
      template_id:
        type: string
        description: "Optional template ID to use instead of a free-form body."
    required: ["to", "subject", "body"]
```

#### `interface.output`

```yaml
interface:
  output:
    type: object
    properties:
      message_id:
        type: string
        description: "Unique identifier of the sent message."
      status:
        type: string
        enum: ["sent", "queued", "failed"]
    required: ["status"]
```

Pair `interface.output` with `error_codes` to document how the model should react to non-success responses:

```yaml
error_codes:
  400:
    meaning: "Malformed request"
    agent_action: "Fix request parameters, do not retry"
  401:
    meaning: "Bad or expired token"
    agent_action: "Refresh token, retry once"
  429:
    meaning: "Rate limited"
    agent_action: "Wait Retry-After seconds, then retry"
  503:
    meaning: "Service unavailable"
    agent_action: "Retry with exponential backoff"
```

!!! tip
    `interface.output` accepts the same JSON Schema syntax as `interface.input`. See [JSON Schema in YAML](08-json-schema.md) for the full field reference, supported types, constraints, and worked examples.

---

### `transport`

Required when `type` is not `function`.

The transport block defines **how** the tool is called: the protocol, endpoint, call parameters, and authentication. It is orthogonal to `type`, which defines behavioral semantics. A `retrieval` tool and an `action` tool can share the same transport protocol.

For `function` type tools, the transport is managed by the platform SDK and this section may be omitted. For all other types, the transport block is required.

All transport variants share the `type` discriminator field and a nested `credentials` block. The remaining fields are specific to each protocol.

---

#### `transport.type: "rest-api"`

```yaml
transport:
  type: "rest-api"
  base_url: "https://mail.internal.example.com/v1"
  endpoint: "POST /send"
  timeout_ms: 5000
  retry_policy:
    max_attempts: 1
    on_status: []
  credentials:
    scheme: "bearer-token"
    source: "env"
    name: "MAIL_API_TOKEN"
```

| Field | Required | Description |
|---|---|---|
| `base_url` | yes | Base URL of the service. No trailing slash. |
| `endpoint` | yes | HTTP method and path, e.g. `POST /send`. |
| `timeout_ms` | no | Request timeout in milliseconds. Default: `5000`. |
| `retry_policy.max_attempts` | no | Maximum call attempts including the first. Default: `3`. Set to `1` to disable retries for non-idempotent calls. |
| `retry_policy.on_status` | no | HTTP status codes that trigger a retry. E.g. `[429, 503]`. Empty list disables retries. |

---

#### `transport.type: "lambda"`

```yaml
transport:
  type: "lambda"
  provider: "aws"
  function_id: "arn:aws:lambda:eu-west-1:123456789:function:send-email-v2"
  invocation_type: "RequestResponse"
  payload_format: "json"
  credentials:
    scheme: "iam-role"
```

| Field | Required | Description |
|---|---|---|
| `provider` | yes | Cloud provider: `aws` \| `gcp` \| `azure`. |
| `function_id` | yes | Provider-specific function identifier. AWS: full ARN (`arn:aws:lambda:…`). GCP: resource name (`projects/…/functions/…`). Azure: resource path or function URL. |
| `invocation_type` | yes | `RequestResponse` (synchronous) or `Event` (fire-and-forget). |
| `payload_format` | no | Serialisation format for the input payload. `json` (default) or `raw`. |

---

#### `transport.type: "mcp"`

```yaml
transport:
  type: "mcp"
  url: "https://mcp.internal.example.com/comms"
  tool_name: "send_email"
  protocol_version: "2024-11-05"
  credentials:
    scheme: "bearer-token"
    source: "aws_secrets_manager"
    secret_id: "prod/mcp/comms-token"
```

| Field | Required | Description |
|---|---|---|
| `url` | yes | Full URL of the MCP server endpoint. |
| `tool_name` | yes | Name of the tool as exposed by the MCP server. |
| `protocol_version` | no | MCP protocol version to negotiate. Defaults to the platform's current supported version. |

---

#### `transport.type: "message-queue"`

```yaml
transport:
  type: "message-queue"
  provider: "aws"
  queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789/email-outbox"
  message_format: "json"
  response_queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789/email-results"
  credentials:
    scheme: "iam-role"
```

| Field | Required | Description |
|---|---|---|
| `provider` | yes | Queue provider. Enum: `aws` \| `gcp` \| `azure` \| `kafka`. |
| `queue_url` | yes | Full URL or topic path of the target queue. |
| `message_format` | no | Message serialisation format. `json` (default) or `avro`. |
| `response_queue_url` | no | Queue from which to read the async response. Omit for fire-and-forget. |

---

#### `transport.type: "database"`

```yaml
transport:
  type: "database"
  engine: "postgresql"
  query_method: "parameterised-sql"
  credentials:
    scheme: "service-account"
    source: "aws_secrets_manager"
    secret_id: "prod/db/connection"
```

| Field | Required | Description |
|---|---|---|
| `engine` | yes | Database engine. Enum: `postgresql` \| `mysql` \| `mssql` \| `bigquery` \| `snowflake` \| `rds-data-api`. |
| `query_method` | yes | How queries are issued. `parameterised-sql` (recommended) or `orm`. Never use string interpolation. |

All connection parameters live inside `credentials`. With `service-account`, the resolved secret must be a JSON object with engine-specific connection fields. With `iam-role`, those parameters are placed directly in the credential object.

---

#### `transport.credentials` — authentication

See [Transport & Credentials](09-transport-credentials.md) for the full `credentials` block reference, including all credential schemes (`none`, `iam-role`, `api-key`, `bearer-token`, `oauth2`, `service-account`), secret sources (`env`, `aws_secrets_manager`, `gcp_secret_manager`, `azure_key_vault`), and connection objects for database and cloud transports.

---

### `use_guidance`

Explain to the agent when to use or when to avoid the tool (required).

```yaml
use_guidance:
  use_when:
    - "user has confirmed they want a confirmation email sent"
    - "a case is resolved and the customer should be notified"
    - "a document needs to be delivered to the user's registered email"
  avoid_when:
    - "user has not explicitly requested or confirmed an email"
    - "the recipient address has not been verified"
    - "the content would constitute bulk or marketing communication"
  side_effects:
    - "Sends an email to the specified recipients — this cannot be undone."
    - "Writes an audit log entry on every call."
```

`use_when` and `avoid_when` are behavioral hints compiled into the agent's instruction context. They help the model decide when to invoke the tool and when to prefer an alternative. `side_effects` must explicitly describe any real-world consequences of calling the tool.

---

## Validation rules

### Hard validation failures

- Missing any required field (`spec_version`, `tool_id`, `version`, `status`, `meta.name`, `meta.description`, `meta.owner`, `type`, `interface.input`, `interface.output`, `use_guidance`).
- Invalid `tool_id` format.
- Invalid semantic version.
- `type` is not one of `retrieval` | `action` | `function` | `human`.
- `transport.type` is not a recognised protocol value when `transport` is present.
- `interface.input` is not a valid JSON Schema subset.
- `interface.output` is not a valid JSON Schema subset.
- Transport block missing when `type` is not `function`.
- `transport.credentials` missing when `transport` is present.
- `transport.credentials.scheme` is an unknown value.
- `transport.credentials.source` missing when `scheme` requires it (`api-key`, `bearer-token`, `service-account`).
- `transport.credentials.provider` or `transport.credentials.function_id` missing when `scheme` is `oauth2`.

### Recommended lint rules

- `type: "action"` tool with `side_effects` absent or set to `"None"`.
- `use_guidance.use_when` or `avoid_when` absent.
- `use_guidance.side_effects` absent for a `type: "action"` tool.
- `status: "deprecated"` without a `meta.last_updated` date.

---

## Example

A read-only retrieval tool.

```markdown
---
spec_version: "1.2"
tool_id: "search-product-kb"
version: "1.0.0"
status: "active"

meta:
  name: "Search Product Knowledge Base"
  description: >
    Search the internal product documentation for feature behavior, API details,
    release notes, known issues, and troubleshooting steps.
    Use for product or technical questions requiring authoritative internal documentation.
    Do not use for HR, finance, legal, or general web questions.
  owner: "platform-engineering"
  tags: ["search", "product", "retrieval"]
  last_updated: "2025-04-01"

type: "retrieval"

interface:
  input:
    type: object
    properties:
      query:
        type: string
        description: "Search query in natural language."
      top_k:
        type: integer
        description: "Maximum results to return. Default 5, max 20."
        default: 5
    required: ["query"]
  output:
    type: object
    properties:
      results:
        type: array
        description: "List of matching documents."
        items:
          type: object
          properties:
            id:
              type: string
              description: "Unique document identifier."
            title:
              type: string
              description: "Document title."
            snippet:
              type: string
              description: "Relevant excerpt from the document."
            url:
              type: string
              format: "uri"
              description: "Link to the full document."
            score:
              type: number
              description: "Relevance score, 0.0–1.0."
          required: ["id", "title", "snippet"]
      total:
        type: integer
        description: "Total number of matching documents."
    required: ["results", "total"]

transport:
  type: "rest-api"
  base_url: "https://kb.internal.example.com/v2"
  endpoint: "POST /search"
  timeout_ms: 8000
  credentials:
    scheme: "service-account"
    source: "aws_secrets_manager"
    secret_id: "prod/agent/kb-token"

use_guidance:
  use_when:
    - "question asks for specific product feature behavior or configuration"
    - "troubleshooting requires authoritative internal documentation"
  avoid_when:
    - "question is purely conversational or general knowledge"
    - "question is outside product or technical support scope"
  side_effects: "None. Read-only."
---

# Purpose
Gives agents access to the internal product knowledge base for authoritative answers
to product and technical questions.

# Notes
Managed by the platform-engineering team. Contact #platform-ai-tools for access requests
or to report indexing issues. The index refreshes every 6 hours.
```
