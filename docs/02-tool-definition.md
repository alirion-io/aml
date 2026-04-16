# Tool Definition Specification

> **File naming**: `tools/<tool_id>.tool.md`

> **Audience**: Infrastructure engineers

---

## Overview

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

---

## YAML front matter — complete field reference

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

### `meta` — descriptive metadata (required)

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

### `type` — behavioral contract (required)

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

### `parameters` — call contract (required)

```yaml
parameters:
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

Parameters follow JSON Schema. The schema is used by the compiler to validate agent configurations and by the model at runtime to construct correct tool calls. All parameter descriptions should be written from the model's perspective — what does the model need to know to use this parameter correctly? See [JSON Schema in YAML](08-json-schema.md) for the full field reference, supported types, constraints, and worked examples.

---

### `transport` — invocation details (required unless `type` is `function`)

The transport block defines **how** the tool is called: the protocol, endpoint, call parameters, and authentication. It is orthogonal to `type`, which defines behavioral semantics. A `retrieval` tool and an `action` tool can share the same transport protocol.

For `function` type tools, the transport is managed by the platform SDK and this section may be omitted. For all other types, the transport block is required.

All transport variants share the `type` discriminator field and a nested `auth` block. The remaining fields are specific to each protocol.

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
  auth:
    scheme: "bearer-token"
    token_source: "env:MAIL_API_TOKEN"
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
  function_arn: "arn:aws:lambda:eu-west-1:123456789:function:send-email-v2"
  invocation_type: "RequestResponse"
  payload_format: "json"
  auth:
    scheme: "iam-role"
```

| Field | Required | Description |
|---|---|---|
| `function_arn` | yes | Fully qualified ARN of the Lambda function. |
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
  auth:
    scheme: "bearer-token"
    token_source: "secrets:prod/mcp/comms-token"
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
  provider: "aws-sqs"
  queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789/email-outbox"
  message_format: "json"
  response_queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789/email-results"
  auth:
    scheme: "iam-role"
```

| Field | Required | Description |
|---|---|---|
| `provider` | yes | Queue provider. Enum: `aws-sqs` \| `gcp-pubsub` \| `azure-servicebus` \| `kafka`. |
| `queue_url` | yes | Full URL or topic path of the target queue. |
| `message_format` | no | Message serialisation format. `json` (default) or `avro`. |
| `response_queue_url` | no | Queue from which to read the async response. Omit for fire-and-forget. |

---

#### `transport.type: "database"`

```yaml
transport:
  type: "database"
  engine: "postgresql"
  connection_source: "secrets:prod/db/connection"
  query_method: "parameterised-sql"
  auth:
    scheme: "service-account"
    token_source: "secrets:prod/db/credentials"
```

| Field | Required | Description |
|---|---|---|
| `engine` | yes | Database engine. Enum: `postgresql` \| `mysql` \| `mssql` \| `bigquery` \| `snowflake`. |
| `connection_source` | yes | Where the runtime fetches the connection details. Format: `env:<VAR>` or `secrets:<path>`. The referenced value must be a JSON object — see secret format below. |
| `query_method` | yes | How queries are issued. `parameterised-sql` (recommended) or `orm`. Never use string interpolation. |

**`connection_source` secret format** — a JSON object with the following fields:

```json
{
  "host": "db.internal.example.com",
  "port": 5432,
  "database": "mydb"
}
```

Credentials (`username` / `password`) are intentionally kept separate in `auth.token_source` so they can rotate independently of the connection details.

**`auth.token_source` secret format** (when `auth.scheme` is `service-account` for a database) — a JSON object:

```json
{
  "username": "agent_read_user",
  "password": "s3cr3t"
}
```

---

#### `transport.auth` — authentication

`auth` is a required sub-block of every transport type except `function`. It declares how the runtime obtains and presents credentials to the remote system.

**Source field format**: all `*_source` fields (`token_source`, `client_id_source`, `client_secret_source`) accept the same two prefixes:

| Prefix | Example | Description |
|---|---|---|
| `env:` | `env:MY_API_KEY` | Read from an environment variable. Value is a plain string. |
| `secrets:` | `secrets:prod/service/token` | Read from the platform secrets manager at the given path. |

**Secret value shapes** — what the runtime expects to find at the referenced path:

| Scheme | Field | Expected secret value |
|---|---|---|
| `api-key` | `token_source` | Plain string — the key value. |
| `bearer-token` | `token_source` | Plain string — the token value. |
| `oauth2` | `client_id_source` | Plain string — the OAuth client ID. |
| `oauth2` | `client_secret_source` | Plain string — the OAuth client secret. |
| `service-account` (non-database) | `token_source` | Plain string — the service account token. |
| `service-account` (database) | `token_source` | JSON object — `{ "username": "...", "password": "..." }`. |

When `scheme` is `iam-role`, no `token_source` is required — the runtime uses the execution environment's attached identity.

##### `scheme: "none"`

No authentication. Only valid for internal services operating inside a trusted network boundary.

```yaml
auth:
  scheme: "none"
```

##### `scheme: "api-key"`

```yaml
auth:
  scheme: "api-key"
  token_source: "env:SERVICE_API_KEY"
  header: "X-Api-Key"          # Header name to send the key in. Default: X-Api-Key.
```

| Field | Required | Description |
|---|---|---|
| `token_source` | yes | Where to read the key at runtime. |
| `header` | yes | HTTP header name. Default: `X-Api-Key`. |

##### `scheme: "bearer-token"`

```yaml
auth:
  scheme: "bearer-token"
  token_source: "secrets:prod/service/token"
```

| Field | Required | Description |
|---|---|---|
| `token_source` | yes | Where to read the token at runtime. |

##### `scheme: "oauth2"`

```yaml
auth:
  scheme: "oauth2"
  token_url: "https://auth.example.com/oauth/token"
  client_id_source: "env:OAUTH_CLIENT_ID"
  client_secret_source: "secrets:prod/oauth/client-secret"
  scopes: ["mail:send", "audit:write"]
  grant_type: "client_credentials"
```

| Field | Required | Description |
|---|---|---|
| `token_url` | yes | OAuth 2.0 token endpoint. |
| `client_id_source` | yes | Where to read the client ID. |
| `client_secret_source` | yes | Where to read the client secret. |
| `scopes` | no | List of OAuth scopes to request. |
| `grant_type` | no | OAuth grant type. Default: `client_credentials`. |

##### `scheme: "service-account"`

A named identity created explicitly for a non-human caller. Its credential (token, key, or username/password) is stored in the secrets manager, provisioned manually, and must be rotated by the owning team or a secrets manager policy.

Use this when the target system does not participate in cloud IAM — a Postgres database, an internal REST API with its own auth, a third-party SaaS service, etc.

The expected secret value shape depends on the transport: a plain string token for HTTP-based transports, a JSON credentials object for `database` transport (see the secret value shapes table above).

```yaml
auth:
  scheme: "service-account"
  token_source: "secrets:prod/agent/service-token"
```

| Field | Required | Description |
|---|---|---|
| `token_source` | yes | Path to the service account token or credentials in the secrets manager. |

##### `scheme: "iam-role"`

The cloud platform assigns an identity to the execution environment automatically (AWS IAM Role, GCP Workload Identity, Azure Managed Identity). The runtime obtains a short-lived token from the local cloud metadata service at call time — no credential is stored or provisioned.

Use this when both the agent runtime and the target service are within the same cloud ecosystem (e.g. a Lambda calling an S3 bucket, or a Cloud Run service calling a Cloud SQL instance with IAM authentication enabled). It cannot be used for systems outside the cloud provider's identity boundary.

| | `service-account` | `iam-role` |
|---|---|---|
| Credential stored in secrets manager? | Yes | No |
| Rotation managed by? | Owning team or policy | Cloud platform, automatically |
| Works with | Any system that accepts a token, key, or password | Cloud-native services in the same provider ecosystem |

```yaml
auth:
  scheme: "iam-role"
```

No additional fields are required.

---

### `use_guidance` — when to use (required)

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

### `response_schema` — expected output shape (optional but recommended)

```yaml
response_schema:
  type: object
  properties:
    message_id:
      type: string
      description: "Unique identifier of the sent message."
    status:
      type: string
      enum: ["sent", "queued", "failed"]
  required: ["status"]

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

---

!!! tip
    `response_schema` accepts the same JSON Schema syntax as `parameters`. See [JSON Schema in YAML](08-json-schema.md) for the full field reference, supported types, constraints, and worked examples.

---

## Validation rules

### Hard validation failures

- Missing any required field (`tool_id`, `version`, `status`, `meta.name`, `meta.description`, `meta.owner`, `type`, `parameters`).
- Invalid `tool_id` format.
- Invalid semantic version.
- `type` is not one of `retrieval` | `action` | `function` | `human`.
- `transport.type` is not a recognised protocol value when `transport` is present.
- `parameters` is not a valid JSON Schema subset.
- Transport block missing when `type` is not `function`.
- `transport.auth` missing when `transport` is present.
- `transport.auth.scheme` is an unknown value.
- `transport.auth.token_source` missing when `scheme` requires it (`api-key`, `bearer-token`, `service-account`, `oauth2`).

### Recommended lint rules

- `type: "action"` tool with `side_effects` absent or set to `"None"`.
- `use_guidance.use_when` or `avoid_when` absent.
- `use_guidance.side_effects` absent for a `type: "action"` tool.
- `response_schema` absent for a tool that returns structured data.
- `status: "deprecated"` without a `meta.last_updated` date.

---

## Minimal complete example — read-only retrieval tool

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

parameters:
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

transport:
  type: "rest-api"
  base_url: "https://kb.internal.example.com/v2"
  endpoint: "POST /search"
  timeout_ms: 8000
  auth:
    scheme: "service-account"
    token_source: "secrets:prod/agent/kb-token"

use_guidance:
  use_when:
    - "question asks for specific product feature behavior or configuration"
    - "troubleshooting requires authoritative internal documentation"
  avoid_when:
    - "question is purely conversational or general knowledge"
    - "question is outside product or technical support scope"
  side_effects: "None. Read-only."

response_schema:
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
---

# Purpose
Gives agents access to the internal product knowledge base for authoritative answers
to product and technical questions.

# Notes
Managed by the platform-engineering team. Contact #platform-ai-tools for access requests
or to report indexing issues. The index refreshes every 6 hours.
```
