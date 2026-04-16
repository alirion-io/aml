# Memory Collection Specification

> **File naming**: `collections/<collection_id>.collection.md`

> **Audience**: Platform engineers

---

## Overview

A memory collection definition file describes one registered memory collection that agents are permitted to read from and write to. It is the single authoritative source of truth for that collection's backing store, scope, retrieval configuration, and writeback rules.

Collection files are authored by platform engineers and approved by the platform team. Agent files reference collections by `collection_id` in their `memory.read_collections` field — they do not inline storage configuration. When a collection's backend changes (for example, migrating from AgentCore Memory to a different store), only the collection file changes; the compiler re-validates all referencing agents automatically.

The collection file does not contain memory data. It describes the storage backend, the lifetime scope, and how the runtime should read from and write to it on the agent's behalf. Access control — which agents are permitted to use a collection — is enforced by the agent's IAM role, not by the collection file.

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

---

## How the compiler resolves a collection reference

When the compiler encounters `memory.read_collections: ["customer-preferences"]` in an agent definition, it:

1. Looks up `collections/customer-preferences.collection.md` in the platform registry.
2. Validates that the collection `status` is `active` or `deprecated` (a `disabled` collection is a hard error).
3. Checks that the agent's `memory.mode` is at least as broad as the collection's `scope.lifetime` (e.g., an agent with `memory.mode: "session"` cannot reference a collection with `scope.lifetime: "project"`).
4. Verifies that the agent's IAM role grants the required read or write permission on this collection.
5. Injects the compiled backend configuration into the runtime payload, including credentials references, namespace patterns, and retrieval defaults.

The runtime never re-resolves collection names at execution time. It uses only the compiled payload.

---

## YAML front matter — complete field reference

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

### `meta` — descriptive metadata (required)

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

### `scope` — lifetime and sharing (required)

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

### `backend` — storage configuration (required)

The `backend` section declares the physical storage system and the credentials or resource identifiers needed to access it. The `type` field selects which sub-fields apply.

#### `type: "agentcore_memory"` — Amazon Bedrock AgentCore Memory

```yaml
backend:
  type: "agentcore_memory"
  memory_id_secret: "secrets/agentcore/customer-preferences-memory-id"
  region: "us-east-1"
  strategies:
    - type: "userPreferenceMemoryStrategy"
      name: "PreferenceLearner"
      namespace: "/preferences/{actorId}"
      retrieval:
        top_k: 5
        relevance_score: 0.7
    - type: "semanticMemoryStrategy"
      name: "FactExtractor"
      namespace: "/facts/{actorId}"
      retrieval:
        top_k: 10
        relevance_score: 0.3
    - type: "summaryMemoryStrategy"
      name: "SessionSummarizer"
      namespace: "/summaries/{actorId}/{sessionId}"
      retrieval:
        top_k: 5
        relevance_score: 0.5
```

`memory_id_secret` is a reference to a secrets manager path that holds the AgentCore Memory resource ID (e.g., an AWS Secrets Manager ARN or a platform-internal secret key). The actual resource ID is never hardcoded in the collection file — it is resolved at compile time. The memory resource itself is provisioned once, separately from agent deployment.

`strategies` declares the retrieval namespaces and how each one is queried. Supported built-in strategy types:

| Type | What it stores |
|---|---|
| `userPreferenceMemoryStrategy` | User preferences learned across sessions |
| `semanticMemoryStrategy` | Factual information extracted from conversations |
| `summaryMemoryStrategy` | Session summaries for efficient context retrieval |

Namespace patterns support `{actorId}` (the user or caller identity) and `{sessionId}` (the current session identifier). These are automatically substituted by the runtime using the values from the active execution context — the agent author never constructs them manually.

#### `type: "valkey"` — Valkey or Redis

```yaml
backend:
  type: "valkey"
  endpoint_secret: "secrets/valkey/customer-preferences"
  port: 6379
  key_prefix: "session"
  ttl_seconds: 86400
```

`endpoint_secret` is a reference to a secrets manager path that holds the Valkey/Redis connection string. The runtime constructs keys using the pattern `<key_prefix>:<session_id>:agent:<agent_id>` with `{sessionId}` and `{actorId}` substituted automatically.

`ttl_seconds` sets the time-to-live for stored entries. Leave unset for project or user-scoped collections that should not expire automatically.

#### `type: "custom"` — custom backend

```yaml
backend:
  type: "custom"
  read_endpoint_secret: "secrets/memory-service/read"
  write_endpoint_secret: "secrets/memory-service/write"
  headers:
    Content-Type: "application/json"
  read_payload_template: |
    {
      "collection": "{{collection_id}}",
      "actor_id": "{{actorId}}",
      "session_id": "{{sessionId}}",
      "query": "{{query}}"
    }
  write_payload_template: |
    {
      "collection": "{{collection_id}}",
      "actor_id": "{{actorId}}",
      "session_id": "{{sessionId}}",
      "content": "{{content}}"
    }
```

For `custom` backends, the runtime calls the provided endpoint with the rendered payload template. Template variables (`{{collection_id}}`, `{{actorId}}`, `{{sessionId}}`, `{{query}}`, `{{content}}`) are substituted by the runtime. The endpoints must return a JSON array of memory entries on read, and a success/failure status on write.

Endpoint values are references to secrets manager paths, not literal URLs, to prevent credential exposure in definition files.

---

### `writeback` — write policy (required)

```yaml
writeback:
  enabled: true
  strategy: "user-preference"      # conversation-summary | user-preference | factual-extraction | raw-output
```

`enabled` controls whether agents are permitted to write to this collection at all. If `false`, the collection is read-only and any agent declaring this collection under a `write_collections` key causes a hard validation error.

`strategy` tells the runtime what to extract and persist from the model's output at the end of a run:

| Strategy | What gets written |
|---|---|
| `conversation-summary` | A summarized digest of the current conversation turn |
| `user-preference` | Preference signals explicitly or implicitly expressed by the user |
| `factual-extraction` | Named facts stated by the user (name, location, account details) |
| `raw-output` | The agent's full response, verbatim |

---

## Validation rules

### Hard validation failures

- Missing any required field (`collection_id`, `version`, `status`, `meta.name`, `meta.description`, `meta.owner`, `scope.lifetime`, `backend.type`, `writeback`).
- Invalid `collection_id` format.
- Unknown `backend.type` value.
- Unknown `scope.lifetime` value.
- An agent references a collection whose `scope.lifetime` exceeds its own `memory.mode`.
- An agent's IAM role does not grant the required read or write permission on a referenced collection.

### Recommended lint rules

- `backend.memory_id_secret` (or equivalent credential field) absent for non-`custom` backend types.
- `meta.last_updated` absent.

---

## Minimal complete example — AgentCore Memory backend

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
  strategies:
    - type: "userPreferenceMemoryStrategy"
      name: "PreferenceLearner"
      namespace: "/preferences/{actorId}"
      retrieval:
        top_k: 5
        relevance_score: 0.7

writeback:
  enabled: true
  strategy: "user-preference"
---

# Purpose

Provides persistent user preference memory for support and personalization agents. Preferences are stored per actor (user) and retrieved automatically at run start, so agents can tailor tone, language, and recommendations without requiring the user to re-state them in each session.

# Notes

This collection contains PII (actor identifiers mapped to preference data) and is subject to GDPR right-to-erasure requirements. The platform's memory service must support per-actor data deletion. Access rights are controlled by the IAM role assigned to each agent that references this collection.
```

---

## Minimal complete example — Valkey backend

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
  strategy: "conversation-summary"
---

# Purpose

Provides a distributed session cache so that agent runs across multiple Lambda instances can read and resume the same conversation context within a session boundary.
```
