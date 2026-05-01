# Tool Definition Specification

> **File naming**: `tools/<tool_id>.tool.md`

> **Audience**: Infrastructure engineers

---

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

### `interface` — input and output contract (required)

Both `interface.input` and `interface.output` are required. They follow JSON Schema syntax and are written from the model's perspective. `interface.input` is used by the compiler to validate agent configurations and by the model at runtime to construct correct tool calls. `interface.output` is required so the model can interpret responses without ambiguity — a missing output schema is a hard validation failure. See [JSON Schema in YAML](08-json-schema.md) for the full field reference, supported types, constraints, and worked examples.

#### `interface.input` — input parameters (required)

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

#### `interface.output` — expected output shape (required)

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

### `transport` — invocation details (required unless `type` is `function`)

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
  function_arn: "arn:aws:lambda:eu-west-1:123456789:function:send-email-v2"
  invocation_type: "RequestResponse"
  payload_format: "json"
  credentials:
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
  provider: "aws-sqs"
  queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789/email-outbox"
  message_format: "json"
  response_queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789/email-results"
  credentials:
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
  credentials:
    scheme: "service-account"
    source: "aws_secrets_manager"
    secret_id: "prod/db/credentials"
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

Credentials (`username` / `password`) are intentionally kept separate in `transport.credentials` so they can rotate independently of the connection details.

**`transport.credentials` secret format** (when `credentials.scheme` is `service-account` for a database) — a JSON object:

```json
{
  "username": "agent_read_user",
  "password": "s3cr3t"
}
```

If the secret uses different key names, override them with `username_key` and `password_key`:

```yaml
credentials:
  scheme: "service-account"
  source: "aws_secrets_manager"
  secret_id: "prod/db/credentials"
  username_key: "user"
  password_key: "pass"
```

| Field | Required | Description |
|---|---|---|
| `username_key` | no | Key name for the username in the resolved JSON object. Default: `username`. |
| `password_key` | no | Key name for the password in the resolved JSON object. Default: `password`. |

---

#### `transport.credentials` — authentication

`credentials` is a required sub-block of every transport type except `function`. It declares how the runtime obtains and presents credentials to the remote system.

The `credentials` block follows the same structure as model provider credentials: `scheme` selects the authentication method, and `source` (plus its associated fields) specifies how the runtime resolves the secret value. This pattern is consistent across all AML definitions.

**`credentials.scheme` values:**

| Value | Description |
|---|---|
| `none` | No authentication. Internal trusted-network services only. |
| `api-key` | Static API key sent in an HTTP header. Requires `source` + `header`. |
| `bearer-token` | Bearer token sent in the `Authorization` header. Requires `source`. |
| `oauth2` | OAuth 2.0 client credentials flow. Requires `token_url` + `client_id` + `client_secret`. |
| `service-account` | Named service identity whose credential lives in the secrets manager. Requires `source`. |
| `iam-role` | Cloud platform identity (AWS IAM Role, GCP Workload Identity, Azure Managed Identity). No extra fields needed. |

**`credentials.source` values** (same as model provider credentials):

| Value | Description |
|---|---|
| `env` | Environment variable named in `name`. |
| `env_aws` | Two AWS credential env vars: `access_key_id` and `secret_access_key`. |
| `aws_secrets_manager` | AWS Secrets Manager; requires `secret_id`. |
| `gcp_secret_manager` | GCP Secret Manager; requires `project` and `secret`. |
| `azure_key_vault` | Azure Key Vault; requires `vault_url` and `secret_name`. |

When `scheme` is `iam-role`, `source` is not required — the runtime uses the attached identity automatically.

**Secret value shapes** — what the runtime expects to find at the resolved path:

| Scheme | Expected secret value |
|---|---|
| `api-key` | Plain string — the key value. |
| `bearer-token` | Plain string — the token value. |
| `oauth2` | Client ID and client secret resolved separately via `client_id` and `client_secret` (each a `secret_ref`). |
| `service-account` (non-database) | Plain string — the service account token. |
| `service-account` (database) | JSON object — `{ "username": "...", "password": "..." }`. |

##### `scheme: "none"`

No authentication. Only valid for internal services operating inside a trusted network boundary.

```yaml
credentials:
  scheme: "none"
```

##### `scheme: "api-key"`

```yaml
credentials:
  scheme: "api-key"
  source: "env"
  name: "SERVICE_API_KEY"
  header: "X-Api-Key"          # Header name to send the key in. Default: X-Api-Key.
```

| Field | Required | Description |
|---|---|---|
| `source` / `name` | yes | Where to resolve the key at runtime. |
| `header` | yes | HTTP header name. Default: `X-Api-Key`. |

##### `scheme: "bearer-token"`

```yaml
credentials:
  scheme: "bearer-token"
  source: "aws_secrets_manager"
  secret_id: "prod/service/token"
```

| Field | Required | Description |
|---|---|---|
| `source` / … | yes | Where to resolve the token at runtime. |

##### `scheme: "oauth2"`

```yaml
credentials:
  scheme: "oauth2"
  token_url: "https://auth.example.com/oauth/token"
  client_id:
    source: "env"
    name: "OAUTH_CLIENT_ID"
  client_secret:
    source: "aws_secrets_manager"
    secret_id: "prod/oauth/client-secret"
  scopes: ["mail:send", "audit:write"]
  grant_type: "client_credentials"
```

| Field | Required | Description |
|---|---|---|
| `token_url` | yes | OAuth 2.0 token endpoint. |
| `client_id` | yes | Where to resolve the client ID (a `secret_ref` object). |
| `client_secret` | yes | Where to resolve the client secret (a `secret_ref` object). |
| `scopes` | no | List of OAuth scopes to request. |
| `grant_type` | no | OAuth grant type. Default: `client_credentials`. |

##### `scheme: "service-account"`

A named identity created explicitly for a non-human caller. Its credential (token, key, or username/password) is stored in the secrets manager, provisioned manually, and must be rotated by the owning team or a secrets manager policy.

Use this when the target system does not participate in cloud IAM — a Postgres database, an internal REST API with its own auth, a third-party SaaS service, etc.

The expected secret value shape depends on the transport: a plain string token for HTTP-based transports, a JSON credentials object for `database` transport (see the secret value shapes table above).

```yaml
credentials:
  scheme: "service-account"
  source: "aws_secrets_manager"
  secret_id: "prod/agent/service-token"
```

| Field | Required | Description |
|---|---|---|
| `source` / … | yes | Where to resolve the service account token or credentials. |

##### `scheme: "iam-role"`

The cloud platform assigns an identity to the execution environment automatically (AWS IAM Role, GCP Workload Identity, Azure Managed Identity). The runtime obtains a short-lived token from the local cloud metadata service at call time — no credential is stored or provisioned.

Use this when both the agent runtime and the target service are within the same cloud ecosystem (e.g. a Lambda calling an S3 bucket, or a Cloud Run service calling a Cloud SQL instance with IAM authentication enabled). It cannot be used for systems outside the cloud provider's identity boundary.

| | `service-account` | `iam-role` |
|---|---|---|
| Credential stored in secrets manager? | Yes | No |
| Rotation managed by? | Owning team or policy | Cloud platform, automatically |
| Works with | Any system that accepts a token, key, or password | Cloud-native services in the same provider ecosystem |

```yaml
credentials:
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
- `transport.credentials.client_id` or `transport.credentials.client_secret` missing when `scheme` is `oauth2`.

### Recommended lint rules

- `type: "action"` tool with `side_effects` absent or set to `"None"`.
- `use_guidance.use_when` or `avoid_when` absent.
- `use_guidance.side_effects` absent for a `type: "action"` tool.
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
