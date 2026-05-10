# Memory Collection Specification

## Overview

A memory collection definition file describes one registered memory collection that agents are permitted to read from and write to. It is the single authoritative source of truth for that collection's backing store, scope, retrieval configuration, and writeback rules.

Collection files are authored by platform engineers and approved by the platform team. Agent files reference collections by `collection_id`. When a collection's backend changes (for example, migrating from AgentCore Memory to a different store), only the collection file changes; the compiler re-validates all referencing agents automatically.

The collection file does not contain memory data. It describes the storage backend, the lifetime scope, and how the runtime should read from and write to it on the agent's behalf. Access control — which agents are permitted to use a collection — is enforced by the agent's IAM role, not by the collection file.

### Memory tiers and when to use each

Agent memory is not a single concept. It spans four distinct tiers, each with a different lifetime and purpose. Understanding the distinction helps you choose the right `scope.lifetime` value and the right backend type for a given collection.

**In-turn context** is the conversation history accumulated during a single agent loop execution — the messages, tool calls, and results the model sees within one run. On a serverless platform like Lambda, it is gone when the function returns. This tier is configurable through AML via the `state` section of the agent definition (`session_history`, `working_state`, `max_state_tokens`, `scratchpad_visible_to_model`). Collections play no role here — there is no `scope.lifetime` value for this tier.

**Session persistence** is state that must survive beyond a single execution but is scoped to one logical conversation. A user pausing and resuming a chat, a Step Functions workflow pausing for human-in-the-loop approval and then resuming — both require the agent to restore the conversation exactly where it left off. This maps to `scope.lifetime: "session"` and is typically backed by a low-latency store like Valkey or Redis with a TTL.

**Project memory** is state that persists across all sessions belonging to the same project, but is not shared across projects or users. This is useful for shared context within a bounded product scope — for example, an ongoing analysis that multiple sessions contribute to, or shared configuration for all agents running under the same project. This maps to `scope.lifetime: "project"` and persists until the project itself is deleted. Namespace patterns for project-scoped collections should use `{projectId}` to scope data to the project (e.g., `/context/{projectId}`).

**User lifetime memory** is state that persists across all sessions and all projects for a given user, indefinitely. This is where learned preferences, extracted facts, and interaction summaries live. The agent starts each session knowing something about the user without the user having to repeat themselves. This maps to `scope.lifetime: "user"` and is typically backed by a semantic store like AgentCore Memory or S3 Vectors, which supports relevance-based retrieval rather than simple key lookup.

| Tier | Lifetime | Typical use case | AML configuration |
|---|---|---|---|
| In-turn context | One execution | Tool call history, reasoning trace | `state` section in agent definition |
| Session persistence | One conversation | Resume after interrupt, distributed Lambda instances | `scope.lifetime: "session"` + `backend.type: "valkey"` or `"s3"` |
| Project memory | Until project is deleted | Shared context across agents within a project | `scope.lifetime: "project"` + `backend.type: "agentcore_memory"` or `"custom"` |
| User lifetime memory | Indefinitely, per user | Preferences, facts, interaction summaries | `scope.lifetime: "user"` + `backend.type: "agentcore_memory"` or `"custom"` |

A single agent may reference collections from multiple tiers simultaneously — for example, a session cache for resumability and a user preference collection for personalisation.

---

## File structure

```
---
[YAML front matter — all structured fields]
---

# Purpose           (optional editorial section)
# Content coverage  (optional editorial section)
# Notes             (optional editorial section)
```

The Markdown body is entirely editorial. The compiler ignores it. Runtime behavior is determined solely by the YAML front matter.

### Top-level fields at a glance

| Field | Required | Description |
|---|---|---|
| `spec_version` | Required | AML version string. Must match a platform-approved value. |
| `collection_id` | Required | Stable, immutable identifier. Lowercase kebab-case. |
| `version` | Required | Semantic version of this collection definition. |
| `status` | Required | Lifecycle state: `draft` \| `active` \| `deprecated` \| `disabled`. |
| `meta` | Required | Descriptive metadata. `name`, `description`, and `owner` are required inside. |
| `scope` | Required | Lifetime and sharing policy (`session`, `project`, or `user`). |
| `backend` | Required | Storage configuration. `type` determines required sub-fields. |
| `writeback` | Required | Write policy; controls whether agents may write to this collection. |

---

## YAML front matter

### Top-level required fields

```yaml
spec_version: "1.2"
```
Must equal a platform-approved AML version string.

```yaml
collection_id: "customer-preferences"
```
Stable, immutable identifier. Lowercase kebab-case. Must match `^[a-z0-9_-]{3,64}$`. Immutable once any agent references this collection in a published definition.

```yaml
version: "1.0.0"
```
Semantic version of this collection definition. Patch for metadata corrections; minor for scope or retrieval changes; major for a backend migration or breaking schema change.

```yaml
status: "active"
```
Enum: `draft` | `active` | `deprecated` | `disabled`. Agents referencing a `disabled` collection fail hard validation. Agents referencing a `deprecated` collection receive a lint warning.

---

### `meta`

Descriptive metadata of a collection (required).

```yaml
meta:
  name: "Customer Preferences"       # Human-readable display name (required)
  description: >                     # Two to four sentences (required)
    Stores and retrieves per-user preferences learned across sessions,
    such as language, communication style, and product interests.
    Used by support and personalization agents to tailor responses
    without requiring users to repeat their preferences.
  owner: "cx-platform-team"          # Team responsible for this collection (required)
  tags: ["preferences", "customer", "personalization"]
  last_updated: "2026-04-01"
```

---

### `scope`

Lifetime and sharing of a collection (required).

```yaml
scope:
  lifetime: "project"        # session | project | user
```

`lifetime` defines how long data in this collection persists and who may share it:

| Lifetime | Persistence | Sharing |
|---|---|---|
| `session` | Deleted at session end | This agent run only |
| `project` | Until project is deleted | All sessions and agents under the same project |
| `user` | Until user data is deleted | All sessions and agents for the same user (actor), across projects |

An agent referencing this collection must have `memory.mode` set to the same or broader lifetime. For example, a collection with `lifetime: "project"` cannot be accessed by an agent whose `memory.mode` is `"session"`. This is a hard validation error.

---

### `backend`

The `backend` section declares the physical storage system and the credentials or resource identifiers needed to access it (required). The `type` field selects which sub-fields apply.

**Backend type is constrained by `scope.lifetime`.** Not all backends are appropriate for all scopes. Using the wrong backend for a scope (e.g., a low-latency key store for user lifetime memory that needs semantic retrieval) is a hard validation error:

| `scope.lifetime` | Permitted `backend.type` values | Rationale |
|---|---|---|
| `session` | `valkey`, `s3` | Low-latency or durable snapshot store; TTL-bounded; no semantic retrieval needed |
| `project` | `agentcore_memory`, `custom` | Shared cross-session context; requires durable, actor-scoped storage |
| `user` | `agentcore_memory`, `custom` | Long-lived user preferences and facts; requires semantic retrieval across sessions |

`agentcore_memory` is the recommended choice for `project` and `user` scopes. `custom` is available when you need to integrate an existing external memory service or a specialised store not covered by built-in types. `valkey` is the preferred session backend when sub-millisecond restore latency is required; `s3` when cost and durability are the priority.

#### `type: "agentcore_memory"` — Amazon Bedrock AgentCore Memory

```yaml
backend:
  type: "agentcore_memory"
  memory_id_secret: "secrets/agentcore/customer-preferences-memory-id"
  region: "us-east-1"
  retrieval_config:
    "/preferences/{actorId}":
      top_k: 5
      relevance_score: 0.7
    "/facts/{actorId}":
      top_k: 10
      relevance_score: 0.3
    "/summaries/{actorId}/{sessionId}":
      top_k: 5
      relevance_score: 0.5
  batch_size: 1
```

`memory_id_secret` is a reference to a secrets manager path that holds the AgentCore Memory resource ID. The actual resource ID is never hardcoded in the collection file — it is resolved at compile time. The AgentCore Memory resource itself (including its strategies and namespaces) is provisioned once, separately from agent deployment, via the AWS Console, CLI, or a setup script. AML does not manage or describe the resource configuration — it only references it.

`retrieval_config` maps namespace patterns to retrieval parameters. The namespaces must match those configured on the AgentCore Memory resource. The following placeholders are substituted automatically by the runtime from the active execution context:

| Placeholder | Resolved from | Applicable scope |
|---|---|---|
| `{actorId}` | Identity of the current user or caller | `user` |
| `{projectId}` | Identifier of the current project | `project` |
| `{sessionId}` | Identifier of the current session | `session`, `project`, `user` |

Omit `retrieval_config` entirely to use the AgentCore resource's defaults.

| Field | Default | Description |
|---|---|---|
| `top_k` | 10 | Number of top-scoring records to return from semantic search (1–1000) |
| `relevance_score` | 0.2 | Minimum relevance threshold for filtering results (0.0–1.0) |

`batch_size` controls how many messages are buffered locally before being flushed to AgentCore Memory. Default `1` sends each message immediately. Set higher (e.g., `10`) to reduce API call volume for high-throughput agents. When `batch_size > 1`, the runtime flushes remaining messages at the end of the invocation — no messages are lost across Lambda invocations.

#### `type: "valkey"` — Valkey or Redis

```yaml
backend:
  type: "valkey"
  endpoint_secret: "secrets/valkey/customer-preferences"
  port: 6379
  key_prefix: "session"
  ttl_seconds: 86400
```

`endpoint_secret` is a reference to a secrets manager path that holds the Valkey/Redis connection string. The runtime constructs keys using the pattern `<key_prefix>:<session_id>:agent:<agent_id>` with `{sessionId}` substituted automatically.

`ttl_seconds` sets the time-to-live for stored entries. Only valid for `scope.lifetime: "session"`.

#### `type: "s3"` — Amazon S3

```yaml
backend:
  type: "s3"
  bucket_secret: "secrets/s3/session-snapshots-bucket"
  region: "us-east-1"
  prefix: "sessions"
  ttl_days: 7
```

`bucket_secret` is a reference to a secrets manager path that holds the S3 bucket name. The runtime stores session snapshots as JSON objects under the key pattern `<prefix>/<session_id>/agent/<agent_id>/snapshots/`. `{sessionId}` is substituted automatically.

`prefix` is an optional key prefix. Omit for no prefix.

`ttl_days` optionally configures an S3 object lifecycle rule applied by the runtime at write time. Only valid for `scope.lifetime: "session"`.

`s3` is appropriate when low-latency sub-millisecond reads are not required and durability or cost are the primary concern. Prefer `valkey` when the session must be restored quickly (e.g., synchronous Lambda chains).

#### `type: "custom"` — custom backend

A `custom` backend delegates all memory management to an external endpoint you own and operate. The runtime calls it using the same `transport` model as tools and guardrails — either a REST API or a Lambda function. Your endpoint must implement the **Strands `SnapshotStorage` interface** to be compatible with AML runtime: the runtime will call it for every snapshot read, write, list, and delete operation, using a fixed JSON contract for each operation type.

Two transport types are supported. Both follow the definitions in [Transport & Credentials](09-transport-credentials.md):

| Transport | Description |
|---|---|
| `rest-api` | HTTP/REST endpoint. The runtime calls different HTTP paths per operation (see interface below). |
| `lambda` | AWS Lambda function. The runtime passes an `operation` field in the payload to distinguish calls. |

**Example — REST API transport:**

```yaml
backend:
  type: "custom"
  transport:
    type: "rest-api"
    base_url: "https://memory.internal.example.com/v1"
    timeout_ms: 3000
    retry_policy:
      max_attempts: 2
      on_status: [429, 503]
    credentials:
      scheme: "bearer-token"
      source: "aws_secrets_manager"
      secret_id: "prod/memory-service/token"
```

**Example — Lambda transport:**

```yaml
backend:
  type: "custom"
  transport:
    type: "lambda"
    function_arn: "arn:aws:lambda:eu-west-1:123456789:function:memory-service"
    invocation_type: "RequestResponse"
    credentials:
      scheme: "iam-role"
```

Refer to [Transport & Credentials](09-transport-credentials.md) for the full `credentials` block field reference.

---

##### SnapshotStorage interface contract

Your endpoint must implement the following five operations. The runtime calls them automatically — your implementation decides where the data is stored (DynamoDB, PostgreSQL, Redis, a third-party service, etc.).

**`save_snapshot`** — persist a snapshot of the agent's current state.

Runtime calls `POST /snapshots` (REST) or passes `"operation": "save_snapshot"` (Lambda).

Request payload:
```json
{
  "operation": "save_snapshot",
  "session_id": "sess-123",
  "scope": { "type": "agent", "scope_id": "support-agent" },
  "snapshot_id": "latest",
  "is_latest": true,
  "snapshot": {
    "data": {
      "messages": [...],
      "state": {},
      "system_prompt": "..."
    },
    "schema_version": "1",
    "created_at": "2026-05-01T10:00:00Z"
  }
}
```

Expected response: `{ "ok": true }`.

---

**`load_snapshot`** — retrieve a stored snapshot.

Runtime calls `GET /snapshots` (REST) or passes `"operation": "load_snapshot"` (Lambda).

Request payload:
```json
{
  "operation": "load_snapshot",
  "session_id": "sess-123",
  "scope": { "type": "agent", "scope_id": "support-agent" },
  "snapshot_id": "latest"
}
```

`snapshot_id` is `"latest"` when the runtime wants the most recent state, or a specific UUID v7 for time-travel restore. Expected response: the `snapshot` object (same shape as above), or `null` if nothing exists yet.

---

**`list_snapshot_ids`** — list all immutable checkpoint IDs for a location.

Runtime calls `GET /snapshots/list` (REST) or passes `"operation": "list_snapshot_ids"` (Lambda). Only used when immutable snapshots are enabled.

Request payload:
```json
{
  "operation": "list_snapshot_ids",
  "session_id": "sess-123",
  "scope": { "type": "agent", "scope_id": "support-agent" },
  "limit": 10,
  "start_after": null
}
```

Expected response: `{ "snapshot_ids": ["<uuid7>", ...] }` — sorted chronologically ascending.

---

**`delete_session`** — remove all data for a session.

Runtime calls `DELETE /sessions/{session_id}` (REST) or passes `"operation": "delete_session"` (Lambda).

Request payload:
```json
{
  "operation": "delete_session",
  "session_id": "sess-123"
}
```

Expected response: `{ "ok": true }`.

---

**`save_manifest` / `load_manifest`** — persist and retrieve a small metadata record alongside snapshots (schema version, last updated timestamp). Used by the runtime for forward-compatibility checks.

Request payload for save:
```json
{
  "operation": "save_manifest",
  "session_id": "sess-123",
  "scope": { "type": "agent", "scope_id": "support-agent" },
  "manifest": { "schema_version": "1", "updated_at": "2026-05-01T10:00:00Z" }
}
```

Request payload for load:
```json
{
  "operation": "load_manifest",
  "session_id": "sess-123",
  "scope": { "type": "agent", "scope_id": "support-agent" }
}
```

Expected response for load: the `manifest` object, or `null` if none exists.

---

All `{actorId}`, `{projectId}`, and `{sessionId}` substitutions are performed by the runtime before calling your endpoint — your implementation receives already-resolved values. Endpoint URLs and function ARNs are never hardcoded in the collection file; credentials are always resolved through the secrets manager at compile time.

---

### `writeback`

Write policy (required).

```yaml
writeback:
  enabled: true
```

`enabled` controls whether agents are permitted to write to this collection at all. If `false`, the collection is read-only and any agent declaring this collection under a `write_collections` key causes a hard validation error.

For `agentcore_memory` backends, writeback behavior (what to extract, how to summarize, which strategy to apply) is governed entirely by the strategies configured on the AgentCore Memory resource itself — not by AML. The runtime flushes conversation turns to AgentCore at the end of each invocation and AgentCore applies its configured strategies automatically.

For `custom` backends, the endpoint receives the full conversation payload and is responsible for its own extraction and persistence logic.

---

## Validation rules

### Hard validation failures

- Missing any required field (`collection_id`, `version`, `status`, `meta.name`, `meta.description`, `meta.owner`, `scope.lifetime`, `backend.type`, `writeback`).
- Invalid `collection_id` format.
- Unknown `backend.type` value.
- Unknown `scope.lifetime` value.
- An agent references a collection whose `scope.lifetime` exceeds its own `memory.mode`.
- An agent's IAM role does not grant the required read or write permission on a referenced collection.
- `backend.type` is `valkey` or `s3` but `scope.lifetime` is not `session`.
- `backend.type` is `agentcore_memory` or `custom` but `scope.lifetime` is `session`.

### Recommended lint rules

- `backend.memory_id_secret` absent for `agentcore_memory` backend.
- `backend.endpoint_secret` absent for `valkey` backend.
- `backend.bucket_secret` absent for `s3` backend.
- `backend.transport` absent for `custom` backend.
- `meta.last_updated` absent.
- `backend.ttl_seconds` or `backend.ttl_days` absent for `session`-scoped collections (unbounded session data is likely a mistake).
- `backend.retrieval_config` absent for `agentcore_memory` backend (defaults will be used — verify they match the resource's namespace configuration).

---

## Example — AgentCore Memory backend

```markdown
---
spec_version: "1.2"
collection_id: "customer-preferences"
version: "1.0.0"
status: "active"

meta:
  name: "Customer Preferences"
  description: >
    Stores and retrieves per-user preferences learned across sessions.
    Used by support agents to personalize responses without requiring users
    to repeat their preferences on each interaction.
  owner: "cx-platform-team"
  tags: ["preferences", "customer"]
  last_updated: "2026-04-01"

scope:
  lifetime: "project"

backend:
  type: "agentcore_memory"
  memory_id_secret: "secrets/agentcore/customer-preferences-memory-id"
  region: "us-east-1"
  retrieval_config:
    "/preferences/{actorId}":
      top_k: 5
      relevance_score: 0.7

writeback:
  enabled: true
---

# Purpose

Provides persistent user preference memory for support and personalization agents. Preferences are stored per actor (user) and retrieved automatically at run start, so agents can tailor tone, language, and recommendations without requiring the user to re-state them in each session.

# Notes

This collection contains PII (actor identifiers mapped to preference data) and is subject to GDPR right-to-erasure requirements. The platform's memory service must support per-actor data deletion. Access rights are controlled by the IAM role assigned to each agent that references this collection.
```

---

## Example — Valkey backend

```markdown
---
spec_version: "1.2"
collection_id: "session-state-cache"
version: "1.0.0"
status: "active"

meta:
  name: "Session State Cache"
  description: >
    Short-lived session conversation cache backed by Valkey.
    Enables agents to resume interrupted sessions and share
    session context across distributed runtime instances.
  owner: "platform-infra-team"
  tags: ["session", "cache", "distributed"]
  last_updated: "2026-04-01"

scope:
  lifetime: "session"

backend:
  type: "valkey"
  endpoint_secret: "secrets/valkey/session-state"
  port: 6379
  key_prefix: "session"
  ttl_seconds: 3600

writeback:
  enabled: true
---

# Purpose

Provides a distributed session cache so that agent runs across multiple Lambda instances can read and resume the same conversation context within a session boundary.
```
