# Transport & Credentials Reference

This page describes the transport and credentials for all artefacts having such sections. It is the reference (as is the [`schemas/transport-credentials.schema.json`](../schemas/transport-credentials.schema.json) file for JSON schema).

---

## Transport × artefact matrix

Not every artefact type supports every transport protocol. The table below shows which transports are valid for each artefact.

| Transport | Tool | Guardrail | Collection |
|---|---|---|---|
| `rest-api` | ✅ | ✅ | ✅ |
| `lambda` | ✅ | ✅ | ✅ |
| `mcp` | ✅ | ✗ | ✗ |
| `message-queue` | ✅ | ✗ | ✗ |
| `database` | ✅ | ✗ | ✗ |


---

## Credential scheme × transport matrix

Credential scheme availability per transport type. Restrictions exist where the auth model is incompatible with the transport protocol.

| Scheme | `rest-api` | `lambda` | `mcp` | `message-queue` | `database` |
|---|---|---|---|---|---|
| `none` | ✅ | ✗ | ✅ | ✗ | ✗ |
| `iam-role` | ✗ | ✅ | ✗ | ✅ | ✅ |
| `api-key` | ✅ | ✗ | ✅ | ✗ | ✗ |
| `bearer-token` | ✅ | ✗ | ✅ | ✗ | ✗ |
| `oauth2` | ✅ | ✗ | ✅ | ✗ | ✗ |
| `service-account` | ✗ | ✅ | ✗ | ✅ | ✅ |

---

## Transport types

### `rest-api`

HTTP/REST endpoint. Invokes the target via an HTTPS call.

```yaml
transport:
  type: "rest-api"
  base_url: "https://api.example.com/v1"
  endpoint: "POST /check"
  timeout_ms: 5000
  retry_policy:
    max_attempts: 3
    on_status: [429, 503]
    backoff_ms: 200
  credentials:
    scheme: "bearer-token"
    source: "aws_secrets_manager"
    secret_id: "prod/service/token"
```

| Field | Required | Description |
|---|---|---|
| `base_url` | yes | Base URL of the API. No trailing slash. |
| `endpoint` | yes | HTTP method and path (e.g. `POST /send`). |
| `timeout_ms` | no | Request timeout in milliseconds. Default: `5000`. |
| `retry_policy.max_attempts` | no | Max attempts including the first. Set to `1` to disable retries. |
| `retry_policy.on_status` | no | HTTP status codes that trigger a retry (e.g. `[429, 503]`). |
| `retry_policy.backoff_ms` | no | Initial backoff in milliseconds between retries (exponential). |

---

### `lambda`

Serverless cloud function transport. Covers AWS Lambda, Azure Functions, GCP Cloud Functions, and any equivalent managed function service. It is recommended to always use `iam-role` for `lambda` to avoid manipulating credentials if on the same cloud.

```yaml
transport:
  type: "lambda"
  provider: "aws"
  function_id: "arn:aws:lambda:eu-west-1:123456789012:function:my-function"
  invocation_type: "RequestResponse"
  payload_format: "json"
  credentials:
    scheme: "iam-role"
```

| Field | Required | Description |
|---|---|---|
| `provider` | yes | Cloud provider running the function. `aws` \| `gcp` \| `azure`. |
| `function_id` | yes | Provider-specific function identifier. AWS: full ARN (`arn:aws:lambda:…`). Azure: resource path or function URL. GCP: resource name (`projects/…/functions/…`). |
| `invocation_type` | yes | `RequestResponse` (synchronous) or `Event` (fire-and-forget). |
| `payload_format` | no | `json` (default) or `raw`. |

---

### `mcp`

Model Context Protocol server. 

```yaml
transport:
  type: "mcp"
  url: "https://mcp.example.com/tools"
  tool_name: "search_documents"
  protocol_version: "2024-11-05"
  credentials:
    scheme: "bearer-token"
    source: "env"
    name: "MCP_TOKEN"
```

| Field | Required | Description |
|---|---|---|
| `url` | yes | Full URL of the MCP server. |
| `tool_name` | yes | Name of the tool as registered on the MCP server. |
| `protocol_version` | no | MCP protocol version to negotiate. Defaults to the platform's current supported version. |

---

### `message-queue`

Async message queue. Provider-specific auth is expressed via the credential scheme and source — for example, `iam-role` for SQS. It is recommended to always use `iam-role` for `message-queue` to avoid manipulating credentials if on the same cloud.

```yaml
transport:
  type: "message-queue"
  provider: "aws"
  queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789012/my-queue"
  message_format: "json"
  response_queue_url: "https://sqs.eu-west-1.amazonaws.com/123456789012/my-queue-response"
  credentials:
    scheme: "iam-role"
```

| Field | Required | Description |
|---|---|---|
| `provider` | yes | `aws` \| `gcp` \| `azure` \| `kafka` |
| `queue_url` | yes | URL, ARN, or topic path of the target queue. |
| `message_format` | no | `json` (default) or `avro`. |
| `response_queue_url` | no | Queue to read responses from. Omit for fire-and-forget. |

---

### `database`

Relational database (or database API). All connection parameters live inside the credential — there is no separate connection source. Use `service-account` to store all connection details in a secret JSON object; use `iam-role` for platform-managed auth where connection parameters are specified directly in the credential.

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
| `engine` | yes | `postgresql` \| `mysql` \| `mssql` \| `bigquery` \| `snowflake` \| `rds-data-api` |
| `query_method` | yes | `parameterised-sql` (recommended) or `orm`. Never use string interpolation. |

---

#### Connection objects by engine

When using `service-account`, the resolved secret **must** be a JSON object containing all connection parameters for the engine. When using `env` as the source, the variable value must be a stringified JSON object. Use the `key` field on the credential to read a sub-object within a larger secret.

| Field | `postgresql` | `mysql` | `mssql` | `bigquery` | `snowflake` | `rds-data-api` |
|---|---|---|---|---|---|---|
| `host` | req | req | req | — | — | — |
| `port` | req | req | req | — | — | — |
| `database` | req | req | req | — | — | req |
| `username` | req | req | req | — | req | req |
| `password` | req | req | req | — | req | req |
| `project` | — | — | — | req | — | — |
| `dataset` | — | — | — | opt | — | — |
| `account` | — | — | — | — | req | — |
| `warehouse` | — | — | — | — | req | — |
| `schema` | — | — | — | — | opt | — |
| `role` | — | — | — | — | opt | — |
| `cluster_arn` | — | — | — | — | — | req |

> `rds-data-api` is the AWS RDS Data API (Aurora Serverless v2). It is not a database engine per se but a query interface — for AML it is treated as an engine.

**`service-account` examples:**

```json
// postgresql / mysql / mssql
{
  "host": "db.example.com",
  "port": 5432,
  "database": "mydb",
  "username": "agent_user",
  "password": "s3cr3t"
}
```

```json
// snowflake
{
  "account": "xy12345.eu-west-1",
  "warehouse": "COMPUTE_WH",
  "database": "PROD_DB",
  "schema": "PUBLIC",
  "username": "agent_user",
  "password": "s3cr3t"
}
```

```json
// bigquery
{
  "project": "my-gcp-project",
  "dataset": "my_dataset"
}
```

```json
// rds-data-api
{
  "cluster_arn": "arn:aws:rds:eu-west-1:123456789012:cluster:my-cluster",
  "database": "mydb",
  "username": "agent_user",
  "password": "s3cr3t"
}
```

When the secret contains a larger JSON object, use `key` to point to the connection sub-object:

```yaml
credentials:
  scheme: "service-account"
  source: "aws_secrets_manager"
  secret_id: "prod/services/config"
  key: "database"   # secret JSON is { "database": { "host": "...", ... } }
```

---

#### `iam-role` for database

When using `iam-role`, no secret is resolved. The platform identity handles authentication and connection parameters are placed directly in the credential object. Required fields depend on the engine.

```yaml
# PostgreSQL / MySQL / MSSQL — RDS IAM token auth
transport:
  type: "database"
  engine: "postgresql"
  query_method: "parameterised-sql"
  credentials:
    scheme: "iam-role"
    host: "db.example.com"
    port: 5432
    database: "mydb"
    username: "agent_iam_user"   # DB user configured for IAM auth
    region: "eu-west-1"          # needed to generate the RDS IAM token
```

```yaml
# RDS Data API — IAM auth
transport:
  type: "database"
  engine: "rds-data-api"
  query_method: "parameterised-sql"
  credentials:
    scheme: "iam-role"
    cluster_arn: "arn:aws:rds:eu-west-1:123456789012:cluster:my-cluster"
    database: "mydb"
```

```yaml
# BigQuery — Workload Identity / ADC
transport:
  type: "database"
  engine: "bigquery"
  query_method: "parameterised-sql"
  credentials:
    scheme: "iam-role"
    project: "my-gcp-project"
    dataset: "my_dataset"
```

| Field | Description |
|---|---|
| `host` | Database host. Required for `postgresql`, `mysql`, `mssql`. |
| `port` | Database port. Required for `postgresql`, `mysql`, `mssql`. |
| `database` | Database name. Required for `postgresql`, `mysql`, `mssql`, `rds-data-api`. |
| `username` | DB user configured for IAM auth. Required for `postgresql`, `mysql`, `mssql` on RDS. |
| `region` | AWS region. Required for RDS IAM token generation. |
| `cluster_arn` | Aurora Serverless cluster ARN. Required for `rds-data-api`. |
| `project` | GCP project ID. Required for `bigquery`. |
| `dataset` | BigQuery dataset name. Optional for `bigquery`. |

---

## Credential schemes

### `none`

No authentication. Only valid for internal services operating inside a fully trusted network boundary.

```yaml
credentials:
  scheme: "none"
```

No additional fields.

---

### `iam-role`

Platform-managed identity. The runtime uses the cloud execution environment's **attached identity** — no credential is stored, fetched, or rotated by you. The platform handles everything automatically.

| Cloud | Mechanism |
|---|---|
| AWS | IAM Role attached to the execution environment (EC2 instance profile, ECS task role, Lambda execution role) |
| GCP | Workload Identity / Application Default Credentials (ADC) on Compute, Cloud Run, GKE |
| Azure | Managed Identity (system-assigned or user-assigned) on VM, Container App, Azure Functions |

> **Key point**: `iam-role` means "trust the platform — I don't manage a credential". The calling service and the called service must be within the same cloud ecosystem, or cross-cloud federation (e.g. AWS → GCP Workload Identity Federation) must be configured.

```yaml
credentials:
  scheme: "iam-role"
```

No additional fields. 

---

### `api-key`

Static API key sent in an HTTP header. `header` is required — the field has no default to avoid silent misconfiguration.

```yaml
credentials:
  scheme: "api-key"
  header: "X-Api-Key"
  source: "aws_secrets_manager"
  secret_id: "prod/service/api-key"
```

| Field | Required | Description |
|---|---|---|
| `header` | **yes** | HTTP header name (e.g. `X-Api-Key`, `Authorization`). |
| `source` | **yes** | Secret backend — see [Secret sources](#secret-sources). |
| *(source fields)* | conditional | See [Secret sources](#secret-sources). |

Depending on the source of the secret, the key will be accessed differently:

| Source | Description |
|---|---|
| Any secret manager | The secret must contain only the API key as a string, or use `key` to read a field from a JSON map. |
| `env` | `name` gives the environment variable to read, which contains the API key as a string. |

---

### `bearer-token`

Bearer token sent in the `Authorization: Bearer <token>` header.

```yaml
credentials:
  scheme: "bearer-token"
  source: "env"
  name: "SERVICE_TOKEN"
```

| Field | Required | Description |
|---|---|---|
| `source` | **yes** | Secret backend — see [Secret sources](#secret-sources). |
| *(source fields)* | conditional | See [Secret sources](#secret-sources). |

Depending on the source of the secret, the bearer token will be accessed differently:

| Source | Description |
|---|---|
| Any secret manager | The secret must contain only the bearer token as a string, or use `key` to read a field from a JSON map. |
| `env` | `name` gives the environment variable to read, which contains the bearer token as a string. |


---

### `oauth2`

OAuth 2.0 dynamic token via a **token broker lambda** on the same cloud platform. The AML runtime invokes the lambda using its platform identity (`iam-role` — no additional credential required) and receives a fresh bearer token in return. The runtime then sends that token as `Authorization: Bearer` on every API or MCP call.

The lambda owns all token lifecycle concerns — acquisition, caching, refresh, and user-delegated lookup (e.g. reading a stored token from a user database). This design keeps all OAuth business logic outside AML and avoids embedding `client_id`/`client_secret` pairs in configuration, which provide no real security improvement over a static API key.

> **Opinionated constraint**: the token broker lambda must live on the same cloud platform as the AML runtime so it can be invoked via `iam-role`. Cross-cloud token brokers are not supported.

```yaml
credentials:
  scheme: "oauth2"
  provider: "aws"
  function_id: "arn:aws:lambda:eu-west-1:123456789012:function:token-broker"
  scopes: ["api:read", "api:write"]
```

| Field | Required | Description |
|---|---|---|
| `provider` | **yes** | Cloud provider hosting the token broker lambda. `aws` \| `gcp` \| `azure`. Must match the AML runtime's cloud. |
| `function_id` | **yes** | Provider-specific function identifier. AWS: full ARN (`arn:aws:lambda:…`). GCP: resource name (`projects/…/functions/…`). Azure: resource path or function URL. |
| `scopes` | no | OAuth 2.0 scopes to pass as context to the token broker. The lambda may use or ignore them. |

**Token broker contract**: the lambda receives an invocation payload (including `scopes` if provided) and must return a JSON object with at minimum:

```json
{ "access_token": "eyJ...", "expires_in": 3600 }
```

All other concerns — which OAuth flow to use, token caching, refresh, user identity resolution — are the lambda's responsibility.


---

### `service-account`

An explicitly managed service identity whose credential is stored in a secret backend and resolved at runtime. You (or your secrets rotation policy) are responsible for provisioning and rotating it.

> **Key point**: `service-account` is different from cloud-native IAM. Despite GCP naming their platform workload identity "Service Account", the `service-account` scheme here means the opposite: an **explicit credential you manage and store**, regardless of cloud provider. If the credential lives in a vault (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, or a set of env vars), use `service-account`.

Valid for **cloud transports only** (`lambda`, `message-queue`, `database`). The resolved secret must be a JSON object with provider-specific or engine-specific connection parameters. For `source: env`, the variable value must be a stringified JSON object. Use the `key` field to read a sub-object within a larger secret.

```yaml
credentials:
  scheme: "service-account"
  source: "aws_secrets_manager"
  secret_id: "prod/agent/aws-credentials"
```

| Field | Required | Description |
|---|---|---|
| `source` | **yes** | Secret backend — see [Secret sources](#secret-sources). |
| `name` | conditional | Env var name. Required with `source: env`. The variable value must be a stringified JSON object with provider-specific or engine-specific connection fields. |
| `key` | no | Key within the secret JSON object to read a sub-object. |

---

#### Connection objects by provider (lambda and message-queue)

When using `service-account` for `lambda` or `message-queue` transport, the resolved secret must be a JSON object with provider-specific fields. The `provider` field on the transport determines which structure is expected.

| Field | AWS | GCP | Azure |
|---|---|---|---|
| `access_key_id` | req | — | — |
| `secret_access_key` | req | — | — |
| `region` | opt | — | — |
| `type` | — | req (`"service_account"`) | — |
| `project_id` | — | req | — |
| `private_key` | — | req | — |
| `client_email` | — | req | — |
| `tenant_id` | — | — | req |
| `client_id` | — | — | req |
| `client_secret` | — | — | req |

**AWS:**

```json
{
  "access_key_id": "AKIAIOSFODNN7EXAMPLE",
  "secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "region": "eu-west-1"
}
```

**GCP** (standard service account key file — store the full JSON as the secret):

```json
{
  "type": "service_account",
  "project_id": "my-gcp-project",
  "private_key_id": "key-id",
  "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...",
  "client_email": "my-sa@my-gcp-project.iam.gserviceaccount.com",
  "client_id": "123456789",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

**Azure:**

```json
{
  "tenant_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "client_secret": "your-client-secret"
}
```

> The database connection objects by engine are documented in [Connection objects by engine](#connection-objects-by-engine) under the `database` transport section.


---

## Secret sources

A `source` field appears in every credential scheme that requires a stored secret (all except `none`, `iam-role`, and `oauth2`). It tells the runtime which backend holds the credential and which additional fields identify it.

### `env`

Credential value is in a single environment variable.

| Field | Required | Description |
|---|---|---|
| `source` | — | `"env"` |
| `name` | **yes** | Environment variable name. |
| `key` | no | Key within the secret JSON object if the variable's value is a JSON map. |

```yaml
source: "env"
name: "SERVICE_API_KEY"
```

---

### `aws_secrets_manager`

Credential is stored as a secret in AWS Secrets Manager.

| Field | Required | Description |
|---|---|---|
| `source` | — | `"aws_secrets_manager"` |
| `secret_id` | **yes** | AWS Secrets Manager secret ID (name or full ARN). |
| `region` | no | AWS region where the secret is stored. |
| `key` | no | Key within the secret JSON object if the secret value is a JSON map. |

```yaml
source: "aws_secrets_manager"
secret_id: "prod/service/api-key"
region: "eu-west-1"
```

---

### `gcp_secret_manager`

Credential is stored as a secret in GCP Secret Manager.

| Field | Required | Description |
|---|---|---|
| `source` | — | `"gcp_secret_manager"` |
| `project` | **yes** | GCP project ID. |
| `secret` | **yes** | GCP secret name. |
| `version` | no | Secret version. Default: `latest`. |
| `key` | no | Key within the secret JSON object if the secret value is a JSON map. |

```yaml
source: "gcp_secret_manager"
project: "my-gcp-project"
secret: "service-api-key"
version: "latest"
```

---

### `azure_key_vault`

Credential is stored as a secret in Azure Key Vault.

| Field | Required | Description |
|---|---|---|
| `source` | — | `"azure_key_vault"` |
| `vault_url` | **yes** | Azure Key Vault URL (e.g. `https://my-vault.vault.azure.net`). |
| `secret_name` | **yes** | Secret name in the Key Vault. |
| `key` | no | Key within the secret JSON object if the secret value is a JSON map. |

```yaml
source: "azure_key_vault"
vault_url: "https://my-vault.vault.azure.net"
secret_name: "service-api-key"
```

---

## Schema maintenance

The canonical `$defs` live in [`schemas/transport-credentials.schema.json`](../schemas/transport-credentials.schema.json). Each artefact schema (`tool.schema.json`, `guardrail.schema.json`, `collection.schema.json`) contains a verbatim copy of these `$defs`.

When a change is required:
1. Update `transport-credentials.schema.json`.
2. Copy the changed `$defs` into every affected artefact schema.
3. Update this documentation page if the change affects the field reference or behaviour.
4. Validate all artefact schemas with `python3 -c "import json; json.load(open('schemas/SCHEMA.json'))"` before committing.

Keeping the definitions as inline copies (rather than cross-file `$ref`) ensures artefact schemas are self-contained and work with any JSON Schema validator without requiring URI resolution.
