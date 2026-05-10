# Knowledge Base Definition Specification


## Overview

A knowledge base definition file describes one approved retrieval source that agents are permitted to use. It declares the content scope, storage type, retrieval defaults, freshness requirements, and usage guidance for a single knowledge source.

KB files are authored by knowledge owners — typically the team responsible for the underlying content (e.g., the brand team for brand guidelines, the legal team for legal policies) — and approved by the platform team. Agent files reference knowledge bases by ID and may override retrieval settings, but only in a more restrictive direction.

The KB file does not contain the knowledge itself. It describes the knowledge source's location, content contract, and the data sensitivity properties of its content.

### Knowledge bases vs tools

A knowledge base is a specific type of tool, and this difference drives several design decisions in AML.

**Knowledge bases are always co-located on the agent platform.** They are managed, indexed resources that run inside the same platform infrastructure as the agents that query them. The platform controls both the index and the retrieval pipeline. This is why access control is delegated entirely to IAM role definitions — there is no separate network boundary to authenticate across, no external API to authorize. The platform enforces KB access directly.

KBs are read-only from the agent's perspective. Agents retrieve from them; they never write to them. State that agents need to persist across turns belongs in memory, not in a knowledge base.

**Tools are mainly remote endpoints.** Tool types span a spectrum: `action` tools invoke external capabilities (an HTTP API, a Lambda function, an external service) over a network boundary; `retrieval` tools query an external data source; `function` tools are local computations that are part of the framework itself and do not cross a network boundary. The tool owns its own error contract and, for action types, its own side-effect model.

This distinction also means a KB cannot be used as a tool, and a tool cannot substitute for a KB. Retrieval from a KB is a first-class platform primitive (vector search, hybrid search, semantic ranking, chunking). Calling a retrieval tool that internally queries a knowledge store is a different pattern: the tool mediates the retrieval logic and returns a result, but the agent does not benefit from the platform's native retrieval features.

| Dimension | Knowledge Base | Tool |
|---|---|---|
| Location | Co-located on the agent platform | `function`: framework-local; `retrieval`/`action`: remote |
| Interaction pattern | Read-only retrieval (vector search, hybrid search) | Depends on type: local computation, data query, or external invocation |
| Access control | IAM role grants (`knowledge_bases`) | IAM role grants (`tools`) + per-tool approval |
| Side effects | None — retrieval only | `action` tools may mutate external state; `function` and `retrieval` tools do not |
| Grounding / citations | Managed at the agent level | Managed at the agent level |
| Observability | Managed at the agent level | Managed at the tool and agent level |
| Auth model | Platform-internal (IAM role) | `function`: none; `retrieval`/`action`: tool defines its own auth |

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

The Markdown body is entirely editorial. The compiler ignores it.

### Top-level fields at a glance

| Field | Required | Description |
|---|---|---|
| `spec_version` | Required | AML version string. Must match a platform-approved value. |
| `kb_id` | Required | Stable, immutable identifier. Lowercase kebab-case. |
| `version` | Required | Semantic version of this KB definition. |
| `status` | Required | Lifecycle state: `draft` \| `active` \| `deprecated` \| `disabled`. |
| `meta` | Required | Descriptive metadata. `name`, `description`, and `owner` are required inside. |
| `source` | Required | Storage type, URI, and freshness settings. |
| `scope` | Required | Content boundaries (domains, include/exclude, languages). |
| `classification` | Required | Trust level and data sensitivity properties. |
| `retrieval_defaults` | Required | Default retrieval parameters inherited by agents. |
| `retrieval` | Required | `use_when` and `avoid_when` hints for the model. |
| `freshness` | Optional | Staleness policy and fallback behavior. |

---

## YAML front matter

### Top-level required fields

```yaml
spec_version: "1.2"
```

```yaml
kb_id: "product-docs"
```
Stable, immutable identifier. Lowercase kebab-case. Must match `^[a-z0-9_-]{3,64}$`.

```yaml
version: "1.0.0"
```
Semantic version of this KB definition. Patch for metadata corrections; minor for scope or retrieval changes; major for a complete content restructuring or URI change.

```yaml
status: "active"
```
Enum: `draft` | `active` | `deprecated` | `disabled`. Agents referencing a `disabled` KB fail hard validation.

---

### `meta`

Descriptive metadata (required)

```yaml
meta:
  name: "Product Documentation"       # Human-readable name (required)
  description: >                      # Two to four sentences (required)
    Internal product documentation covering feature behavior, API references,
    release notes, configuration guides, and troubleshooting procedures.
    Used by support and developer agents to answer product questions
    with authoritative sourcing.
  owner: "docs-team"                  # Team responsible for this KB (required)
  tags: ["product", "docs", "support"] # Optional searchable labels
  last_updated: "2025-04-01"
```

---

### `source`

Storage and location (required).

```yaml
source:
  type: "vector_store"            # See type taxonomy below
  uri: "kb://product-docs"        # Stable, platform-resolved address the runtime uses to query this KB (required if available)
  freshness_window_days: 7        # Content older than this may be stale
  index_refresh: "every-6h"       # How often the index is rebuilt
```

`uri` is the stable, immutable address the runtime uses to locate and query this KB when an agent invokes retrieval. Once a KB is registered, its `uri` is immutable — changing it is a major version bump. Use the `kb://` scheme for platform-managed stores, or a fully-qualified address for other source types.

How content is ingested and indexed into the KB (connectors, crawlers, pipelines) is an infrastructure concern managed outside AML.

#### Source type taxonomy

| Type | Description |
|---|---|
| `vector_index` | Dense vector index (e.g., FAISS, Pinecone, Weaviate) |
| `vector_store` | Managed vector store with metadata filtering |
| `document_set` | Flat collection of documents without vector indexing |
| `sql_table` | Relational database table queried by the retrieval layer |
| `api` | External API that returns documents or structured content |
| `graph_db` | Graph database for relationship-aware retrieval |
| `hybrid_store` | Combined dense vector and keyword index |

`freshness_window_days` is optional but required (lint warning if absent) for time-sensitive domains such as `legal`, `pricing`, `compliance`, `regulatory`, and `product_releases`.

---

### `scope`

Content boundaries (required).

```yaml
scope:
  domains:
    - "product_features"
    - "api_usage"
    - "troubleshooting"
    - "release_notes"
  include:
    - "feature behavior descriptions"
    - "API endpoint documentation"
    - "known issues and workarounds"
    - "configuration and setup guides"
  exclude:
    - "internal engineering roadmaps"
    - "legal and compliance interpretations"
    - "pricing and commercial terms"
    - "HR and people policies"
  languages: ["en"]               # Content languages available
  version_coverage: "current"     # current | all | range:<from>-<to>
```

`domains` is a controlled vocabulary maintained by the platform team. The compiler validates that all listed domains are registered. `include` and `exclude` are editorial guidance to the model — they help the model decide whether a particular question is answerable from this KB. They are compiled into the retrieval context.

---

### `classification`

Content trust and data properties (required).

Access control — which agents may query this KB — is declared in IAM role definitions, not here. This section captures the trustworthiness of the content and its data sensitivity properties.

```yaml
classification:
  trust_level: "authoritative"    # authoritative | supplemental | unverified
  data_classification:
    contains_pii: false
    compliance: []
    confidentiality: "internal"   # public | internal | confidential | restricted
```

`trust_level` affects how the model is instructed to treat retrieved content:

| Level | Meaning | Model behavior |
|---|---|---|
| `authoritative` | Content is the official source of truth | Model must not contradict it; must cite it |
| `supplemental` | Content is useful but not definitive | Model may supplement with reasoning |
| `unverified` | Content is user-contributed or unmoderated | Model must caveat retrieved claims |

---

### `retrieval_defaults`

Retrieval policy (required).

```yaml
retrieval_defaults:
  search_mode: "hybrid"           # keyword | semantic | hybrid
  max_chunks: 5                   # Max retrieved chunks per query
  max_tokens: 1800                # Max tokens of retrieved content per query
  result_granularity: "section"   # chunk | section | document
  reranking: true                 # Apply a re-ranking pass on retrieved results
  citation_preference: "required" # required | optional | none
  deduplicate: true               # Remove near-duplicate results
  min_score: 0.65                 # Minimum relevance score threshold (0.0–1.0)
```

These are the defaults an agent inherits when it references this KB. An agent may override `max_chunks`, `max_tokens`, `citation_preference`, and `min_score`, but only in the more restrictive direction (lower `max_chunks`, higher `min_score`, etc.).

`result_granularity` controls the unit of retrieved content:
- `chunk`: fixed-size text windows from the index
- `section`: natural document sections (e.g., heading-delimited blocks)
- `document`: full documents (expensive; appropriate only for short documents)

---

### `retrieval`

When to use (required).

```yaml
retrieval:
  use_when:
    - "question requires product feature details or API behavior"
    - "troubleshooting steps are needed"
    - "release notes or known issues are relevant"
  avoid_when:
    - "question is purely conversational or general knowledge"
    - "question is outside product or technical support scope"
    - "answer is already present in the conversation context"
```

`use_when` and `avoid_when` are natural-language hints compiled into the agent's retrieval context. They guide the agent in deciding whether querying this KB is appropriate for a given turn. They are not hard rules enforced by the runtime — they are instructions to the model, so they should be written as clear, specific conditions that a reasoning model can evaluate against the current conversation.

`use_when` entries describe situations where this KB is likely to contain a relevant and useful answer. `avoid_when` entries describe situations where retrieval would be wasteful, misleading, or out of scope — for example, when the question falls outside the KB's content boundaries, or when the answer is already present in the conversation context and a retrieval round-trip adds no value.

---

### `freshness`

Staleness policy (optional)

```yaml
freshness:
  window_days: 7
  on_stale: "warn"                # warn | refuse | fallback
  fallback_kb: "general-web-search"
  last_verified: "2025-03-25"
```

`on_stale` controls the runtime's behavior when retrieved content exceeds `window_days`:
- `warn`: include the content but add a staleness warning in the agent's context
- `refuse`: do not return stale content; treat as a retrieval miss
- `fallback`: query the `fallback_kb` instead

---

## Validation rules

### Hard validation failures

- Missing any required field (`kb_id`, `version`, `status`, `meta.name`, `meta.description`, `meta.owner`, `source.type`, `scope`, `classification`).
- Invalid `kb_id` format.
- Unknown `source.type` value.
- Unknown `classification.trust_level` value.
- Unknown `retrieval_defaults.search_mode` or `retrieval_defaults.result_granularity` value.

### Recommended lint rules

- `freshness_window_days` absent for a domain tagged as `legal`, `pricing`, `compliance`, or `regulatory`.
- `scope.exclude` absent — at least some exclusions help the model avoid misuse.
- `source.uri` absent for a `vector_store` or `vector_index` type.

---

## Example

```markdown
---
spec_version: "1.2"
kb_id: "product-docs"
version: "1.0.0"
status: "active"

meta:
  name: "Product Documentation"
  description: >
    Internal product documentation covering feature behavior, API references,
    release notes, configuration guides, and troubleshooting procedures.
  owner: "docs-team"
  tags: ["product", "docs", "support"]
  last_updated: "2025-04-01"

source:
  type: "vector_store"
  uri: "kb://product-docs"
  freshness_window_days: 7

scope:
  domains: ["product_features", "api_usage", "troubleshooting"]
  include:
    - "feature behavior descriptions"
    - "API endpoint documentation"
    - "known issues and workarounds"
  exclude:
    - "legal interpretation"
    - "pricing and commercial terms"
    - "internal engineering roadmaps"
  languages: ["en"]

classification:
  trust_level: "authoritative"
  data_classification:
    contains_pii: false
    confidentiality: "internal"

retrieval_defaults:
  search_mode: "hybrid"
  max_chunks: 5
  max_tokens: 1800
  result_granularity: "section"
  citation_preference: "required"
  min_score: 0.65

retrieval:
  use_when:
    - "question requires product feature details or API behavior"
    - "troubleshooting steps are needed"
  avoid_when:
    - "question is purely conversational or general knowledge"
    - "question is outside product or technical scope"
---

# Purpose
The primary product documentation source for all support and developer agents.
Covers the current released version of all products.

# Notes
Maintained by the docs team. Index refreshes every 6 hours.
Contact #docs-platform for corrections or to report missing content.
```
