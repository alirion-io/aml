# Agent Definition Specification

> **File naming**: `agents/<agent_id>.agent.md`

> **Audience**: Product owners

---

## Overview

An agent definition file is one Markdown file that fully describes a single agent. It has two parts: a YAML front matter block (between `---` delimiters) that carries all machine-readable structured fields, and a Markdown body that carries the prose behavioral instructions the model will follow.

The compiler reads both parts, validates them, resolves all external references to tools and knowledge bases, check rights, and emits an immutable compiled JSON payload. The runtime executes only compiled payloads — never raw Markdown.

---

## File structure

```
---
[YAML front matter — all structured fields]
---

# Purpose
[one paragraph]

# System behavior
[core model instructions]

# Rules
[non-negotiable rules]

# Escalation          (optional)
# Examples            (optional)
# Non-goals           (optional)
# Output style notes  (optional)
```

---

## YAML front matter — complete field reference

### Top-level required fields

```yaml
spec_version: "1.2"
```
The AML format version. Must equal a platform-approved version string. The runtime rejects unknown spec versions. This describes the definition format, not the agent's behavior version.

```yaml
agent_id: "support-agent"
```
Stable, immutable identifier for the agent lineage. Lowercase kebab-case or snake_case. Must match `^[a-z0-9_-]{3,64}$`. Once any version of this agent is published, the `agent_id` cannot change. Use a new `agent_id` for a fundamentally different agent.

```yaml
version: "1.0.0"
```
Semantic version of this definition. Increment on every published change. Patch for wording or examples; minor for backward-compatible behavior changes; major for breaking changes to input/output schemas or tool access.

```yaml
status: "active"
```
Lifecycle state. Enum: `draft` | `active` | `deprecated` | `disabled`. Draft definitions are not runnable in production. Deprecated definitions are still runnable but hidden for new use. Disabled definitions cannot be run.

---

### `meta` — business metadata (required)

```yaml
meta:
  name: "Customer Support Agent"      # Human-readable display name (required)
  category: "support"                 # From a platform-controlled list (required)
  description: >                      # One to two sentences (recommended)
    Handles first-line customer support for product and billing questions.
  owner: "cx-product-team"            # Team or individual responsible (recommended)
  tags: ["support", "customer", "billing"]  # Searchable labels (optional)
```

The `category` field should come from a controlled vocabulary maintained by the platform team to support routing, discovery, and reporting. Examples: `support`, `language`, `finance`, `hr`, `operations`, `developer`, `internal`. Not that the compiler will not enforce validation of categories.

---

### `runtime` — model and execution settings (required)

```yaml
runtime:
  model: "claude-4-sonnet"            # Model identifier (required)
  fallback_model: "claude-4-haiku"    # Used if primary is unavailable (optional)
  reasoning_effort: "medium"          # low | medium | high (recommended)
  temperature: 0.3                    # 0.0–1.0; lower for deterministic tasks
  timeout_seconds: 60                 # Hard timeout for a single run
  max_input_chars: 20000              # Input size guard
  max_output_tokens: 2000             # Output size guard
  max_turns: 8                        # Max conversation turns for loop-based agents
  max_tool_calls: 12                  # Max tool calls per run
  retry_policy:
    max_attempts: 2
    backoff_seconds: 2
```

The `model` value must be a `model_id` that resolves to a registered [model definition file](05-model-definition.md) (`models/<model_id>.model.md`). The compiler looks up the referenced file and injects the provider-specific configuration into the compiled payload. An unresolvable `model_id` or a reference to a model with `status: disabled` is a hard compile error; a reference to a `deprecated` model is a lint warning.

This indirection decouples agent files from provider-specific naming: authors write `claude-4-sonnet`, not `us.anthropic.claude-sonnet-4-20250514-v1:0`. The platform team controls which providers and model versions are available by managing model definition files.

`reasoning_effort` is an editorial hint to the platform — the platform maps it to model-specific parameters. `high` is appropriate for complex multi-step reasoning; `low` for simple extraction or formatting tasks.

High temperature (above 0.7) for business-critical deterministic workflows is a lint warning.

---

### `input` — input contract (required)

```yaml
input:
  schema:
    type: object
    properties:
      question:
        type: string
        description: "The customer's question or request."
      account_id:
        type: string
        description: "Customer account identifier, if known."
    required: ["question"]
  examples:
    - question: "How do I reset my password?"
    - question: "I was charged twice for my subscription."
      account_id: "acct-00123"
```

The `schema` must be a valid JSON Schema subset. The top-level `type` should almost always be `object`. Per-field rendering hints are declared in the `ui.ui_hints` section rather than here, to keep all UI concerns in one place. See [JSON Schema in YAML](08-json-schema.md) for the full field reference, supported types, constraints, and worked examples.

---

### `output` — output contract (required)

```yaml
output:
  schema:
    type: object
    properties:
      answer:
        type: string
        description: "The agent's response to the customer."
      escalated:
        type: boolean
        description: "True if the agent escalated the case."
      suggested_articles:
        type: array
        items:
          type: string
        description: "IDs of relevant help articles, if any."
    required: ["answer", "escalated"]
  render:
    format: "markdown"
  provenance:
    citations_required: true
```

The runtime must validate every response against the output schema. A response that fails validation is a run error (trigger retry or fallback). A loose schema — for example, allowing `additionalProperties: true` without justification — is a lint warning. See [JSON Schema in YAML](08-json-schema.md) for the full field reference, supported types, constraints, and worked examples.

---

### `tools` — tool access (required)

```yaml
tools:
  # Reference to a shared registered tool (preferred)
  - ref: "search-product-kb"

  # Reference with agent-level overrides (can only tighten, never loosen)
  - ref: "send-email"
    approval_required: true        # Registry default was false; this agent requires approval
    max_calls_per_run: 1           # Registry allowed 3; this agent restricts to 1

  tool_choice: "auto"              # none | auto | required
```

`ref:` must resolve to a registered `.tool.md` file. Unresolvable references are hard validation errors.

`tool_choice` is an LLM API parameter sent with every request alongside the tool list. It does not select which tool to call — the model always decides that. It constrains whether the model is permitted or required to call any tool at all:

| Value | Behaviour |
|---|---|
| `none` | Tools are advertised to the model but it is **forbidden** from calling any of them. The model must respond in plain text. Useful for temporarily disabling tools without removing them from the manifest. |
| `auto` | The model **decides** whether to call a tool or respond directly. This is the standard agentic mode. |
| `required` | The model **must** invoke at least one tool in its response. Useful when you need guaranteed structured output via a tool call on every turn. |

Agent-level overrides can only make constraints stricter than the registry defaults. Attempting to loosen a registry constraint (for example, removing `approval_required: true`) is a hard validation error.

---

### `knowledge` — knowledge source access (recommended)

```yaml
knowledge:
  # Reference to a shared KB (preferred)
  - ref: "product-docs"

  # Reference with agent-level retrieval overrides
  - ref: "brand-guidelines"
    required: true             # Inject directly into context on every run
    citations_required: true   # Tighten from registry default of false

  retrieval:                   # Agent-level retrieval policy (applies to all refs unless overridden per ref)
    search_mode: "hybrid"      # keyword | semantic | hybrid
    trigger: "auto"
    grounding_required: true
```

The `knowledge` section is optional but strongly recommended for any agent that makes factual claims about mutable information. If `required: true`, the KB is injected directly into the model context on every run. If `required: false` (the default), the agent decides when to retrieve from it.

If `required: true` then the `retrieval` section of the KB definition will not be considered by the LLM as the context is always added to its context.

---

### `memory` — longer-lived memory policy (optional)

```yaml
memory:
  mode: "project"             # none | session | project | user
  read_collections: ["customer-preferences"]
  write_collections: ["customer-preferences"]
```

**What memory is.** Memory is a reference to named, external storage collections that persist beyond a single run. It is not in-process data — it lives in a platform-managed store whose type and configuration are fully described in a [collection definition file](06-collection-definition.md) (`collections/<collection_id>.collection.md`).

**`mode`** controls the maximum scope of collections this agent is permitted to access:

| Mode | Scope | Lifetime |
|---|---|---|
| `none` | No memory access | — |
| `session` | Current conversation only | Deleted at session end |
| `project` | All sessions under the same project | Until project is deleted |
| `user` | All sessions across all projects for the same user | Until user data is deleted |

**`read_collections` and `write_collections`** are lists of `collection_id` values — the same identifiers used as file names in `collections/`. Each value must resolve to a registered `.collection.md` file with `status: active` or `status: deprecated`. An unresolvable ID or a reference to a `disabled` collection is a hard validation error. The compiler reads the referenced collection file and injects the backend configuration (store type, credentials references, namespace patterns, retrieval defaults) into the compiled runtime payload. AML authors never write connection strings, function ARNs, or storage keys directly in the agent file.

**How reads work.** At run start, the runtime calls each collection's configured backend (for example, the AgentCore Memory service or a Valkey instance) using the compiled credentials and namespace configuration. It passes the current conversation turn as the retrieval query, and `{actorId}` and `{sessionId}` are substituted automatically from the active execution context. Retrieved entries are injected into the model context before the model generates a response. Authors can influence what gets retrieved by choosing collection names precisely — `customer-preferences` will pull more targeted results than a broadly named collection.

**How writes work.** Writeback is fully automated — there is no API for the agent to call manually. When `write_collections` is declared and the collection's `writeback.enabled: true`, the runtime extracts the relevant content from the model's output at the end of the run (according to the collection's configured `writeback.strategy`) and writes it back to the collection. The agent author controls *which* collections the agent may write to; write behavior (what to extract, whether human approval is required) is governed by the collection definition itself, not the agent file.

---

### `state` — session-local working state (optional)

```yaml
state:
  session_history: "ephemeral"    # ephemeral | session
  working_state: "session"        # ephemeral | session
  scratchpad_visible_to_model: true
  max_state_tokens: 1200
```

`state` is distinct from `memory`. It describes short-term, session-local scratchpad behavior — not shared or persistent storage. Use it for temporary reasoning state within a single conversation.

**`session_history`** controls how much conversation history the runtime retains across turns within a session:

- `ephemeral`: history exists only for the current turn and is discarded immediately after the model responds. Each turn starts with no prior context. Use for stateless tasks where history adds no value and wastes context tokens.
- `session`: history is accumulated across all turns for the duration of the session. The model receives the full prior conversation on each subsequent turn. This is the default for conversational agents.

**`working_state`** controls whether intermediate working state — tool call results, partial outputs, reasoning traces produced between turns — is retained between turns:

- `ephemeral`: intermediate state is discarded after each turn. The runtime does not carry forward any working context beyond the conversation messages.
- `session`: intermediate state is preserved across turns for the session. Use when later turns need to reference or build on earlier intermediate results (e.g., a multi-step research task where the agent accumulates partial findings across turns).

**`scratchpad_visible_to_model`** controls whether the accumulated `working_state` is injected into the model's context window on subsequent turns:

- `true`: the scratchpad content is included in the context the model receives at the start of each turn. The model can read its own prior intermediate reasoning and build on it explicitly.
- `false`: the scratchpad is retained by the runtime for operational purposes (logging, debugging, crash recovery) but is **not** fed into the model. The model sees only conversation messages.

Setting `scratchpad_visible_to_model: true` improves reasoning continuity across turns but consumes context window tokens. Set it to `false` for agents where intermediate state is only needed for observability, or to conserve the context budget.

**`max_state_tokens`** caps the total number of tokens the runtime may allocate to state (history + working state) when constructing the model context for a turn. If accumulated state exceeds this limit, the runtime trims older entries. This prevents unbounded context growth in long sessions. Must be less than the agent's `runtime.max_input_chars` equivalent token budget.

**Memory vs. state — at a glance.**

| | `memory` | `state` |
|---|---|---|
| **What it is** | Named external storage collections | In-process working scratchpad |
| **Where it lives** | Platform-managed store (outside the run) | Runtime process memory (inside the run) |
| **Who can access it** | Other agents or sessions sharing the same collection | This run only — never exposed externally |
| **Max durability** | User-wide, indefinite (`user`) | End of session at most |
| **Governance required** | Yes — collection registration, writeback approval | No |

Both can be scoped to a session, which is a source of confusion. The key distinction: `memory.mode: "session"` means "use a session-scoped external store that the platform manages and that all agents called during that session can access"; `state.working_state: "session"` means "keep my in-process scratchpad alive across turns but never persist or share it".

---

### `artifacts` — storage and retention (required)

```yaml
artifacts:
  enabled: true
  save_input: false              # Do not persist raw user inputs (PII risk)
  save_output: true
  save_intermediate: false
  retention_class: "standard"    # standard | extended | ephemeral | restricted
```

#### What artifacts are

An **artifact** is a stored record of something that happened during an agent run. Concretely, the platform can persist three kinds of data from any given run:

- **Input** — the raw payload sent to the agent (user message, `account_id`, any other input fields).
- **Output** — the final structured response the agent returned (the validated object matching `output.schema`).
- **Intermediate** — everything that happened in between: tool call arguments and responses, KB retrieval results, reasoning traces, retry attempts.

None of these are persisted by default unless you declare them here. The `artifacts` section is where you opt in explicitly and tell the platform how long to keep what.

#### Why artifacts exist

Agents are opaque by nature — a model reasons internally, calls tools, and returns a result. Without stored artifacts, you have no way to answer questions like:

- "Why did this agent escalate this ticket?" → requires the intermediate tool call trace.
- "What exactly did the agent tell the customer?" → requires the saved output.
- "Did this run contain PII we need to delete under a right-to-erasure request?" → requires knowing what was stored and under which retention class.
- "We had a production incident last Tuesday — what were the inputs?" → requires saved inputs.

Artifacts make agents **auditable**, **debuggable**, and **compliant**. They are the evidence trail that operations, legal, and security teams need to operate AI in production.

That said, storing more data carries its own risks. Raw user inputs frequently contain personal information. Intermediate traces can expose internal tool parameters or system details that should not leave the system. This is why each category is independently controlled and why `save_input` defaults to `false`.

#### Field reference

| Field | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `false` | Master switch. When `false`, the platform writes no artifacts for this agent regardless of other settings. |
| `save_input` | bool | `false` | Persist the raw input payload. Disable unless you have a specific audit or replay need, as inputs are the most likely source of PII. |
| `save_output` | bool | `false` | Persist the final validated output. Usually safe to enable; outputs are already schema-validated and PII-redacted if `pii_redaction: true` is set in `policies`. |
| `save_intermediate` | bool | `false` | Persist tool arguments/responses, KB chunks, and reasoning traces. Useful for debugging and compliance, but produces the largest artifacts. |
| `retention_class` | enum | `standard` | How long artifacts are kept and who can access them. See below. |

#### Retention classes

| Class | Retention period | Access | When to use |
|---|---|---|---|
| `ephemeral` | Never persisted — artifacts are generated in-memory for this run only (e.g., for guardrail checks) and then discarded | N/A | Agents handling highly sensitive data where no record should ever be written to storage |
| `standard` | Platform default (typically 30–90 days, configurable per tenant) | Owning team and platform operators | The right default for most production agents |
| `extended` | Longer than standard (typically 1–7 years, configurable per tenant) | Owning team, platform operators, compliance tooling | Regulated industries (finance, healthcare) where long-term audit logs are required |
| `restricted` | Same as `standard` | **Owning agent and tenant only** — no cross-team or operator access by default | Agents processing commercially sensitive or legally privileged data |

Retention class is enforced by the platform storage layer, not by the agent. Declaring `retention_class: "ephemeral"` does not mean "try not to save" — it means the platform runtime is prohibited from writing any artifact for this agent to durable storage.

#### Where artifacts are stored

The agent definition file does **not** reference a storage location. This is intentional.

The artifact store is configured at the **tenant level** by the platform team, not chosen per agent. Every agent in a tenant writes artifacts to the same platform-managed store (for example, an S3 bucket, a cloud object store, or a compliance vault). The platform team provisions and configures this store once; agent authors never write bucket names, connection strings, or credentials into their agent definitions.

At compile time, the compiler reads the tenant's artifact store configuration and injects the resolved storage backend — including credentials references, bucket/namespace, and encryption settings — into the compiled payload. The agent file itself never contains this detail.

This is different from how `memory` works. `memory` lets the agent choose specific named collections (`.collection.md` files), because different agents deliberately need access to different data stores with different scopes and lifetimes. Artifact storage has no such per-agent variation: whatever the agent produces goes to the tenant artifact store, governed by the declared retention class.

If a tenant needs to route artifacts from specific agents to a different storage backend (for example, a restricted compliance vault), that is expressed as a tenant-level routing rule, not in the agent file.

#### Relationship to other sections

- **`policies.pii_redaction`**: PII redaction runs *before* artifact storage. If `pii_redaction: true`, the recorded artifact has already been scrubbed. This makes it safer to set `save_input: true` when you need audit records without storing raw personal data.
- **`observability`**: Observability captures structured metrics and traces (latency, token counts, error codes). Artifacts capture the actual content of runs (what was said, what tools returned). They serve different audiences — operations teams use observability; audit and compliance teams use artifacts.
- **`policies.audit_level`**: `audit_level` controls *what the platform logs to its own internal audit system*. `artifacts` controls *what the platform stores in the artifact store accessible to your team*. Both can be set independently.

---

### `policies` — governance rules (recommended)

```yaml
policies:
  pii_redaction: true
  allow_external_http: false
  require_human_approval: false
  blocked_content:
    - "legal_advice"
    - "medical_advice"
  allowed_connectors: ["sharepoint", "google_drive"]
  data_residency: "eu"
  audit_level: "full"            # none | standard | full
```

Policies are machine-enforced governance rules applied by the runtime layer before and after model execution. They are **not** prompt instructions — declaring `pii_redaction: true` does not ask the model to avoid printing PII, it instructs the platform runtime to intercept and scrub PII programmatically. The model never sees the policy declarations.

The runtime must reject or remediate runs that violate declared policies. Unsupported policy keys are hard validation errors — the platform maintains a controlled vocabulary of valid keys.

#### Field reference

| Field | Type | Default | Description |
|---|---|---|---|
| `pii_redaction` | bool | `false` | Scrub PII from inputs and outputs before processing and storage. |
| `allow_external_http` | bool | `false` | Permit tool calls that make outbound HTTP requests to external (non-internal) hosts. |
| `require_human_approval` | bool | `false` | Require explicit human approval before any tool call is executed. |
| `blocked_content` | string[] | `[]` | Categories of output the agent is prohibited from generating. |
| `allowed_connectors` | string[] | `[]` | Named integration connectors the agent may use. All others are denied. |
| `data_residency` | string | tenant default | Geographic region within which all data must remain. |
| `audit_level` | enum | `standard` | Verbosity of the platform's internal audit log for this agent. |

#### `pii_redaction`

When `true`, the platform runtime runs a PII detection pass on both the incoming input and the outgoing output at the boundary of execution — before the model processes the input and before the output is returned or persisted. Detected entities (names, email addresses, phone numbers, account numbers, national identifiers, and similar) are replaced with typed placeholders such as `[PERSON]`, `[EMAIL]`, or `[PHONE]`.

This is enforced at the runtime layer and is independent of any model-level instruction. Setting this to `true` has two practical effects:

- **Input redaction**: the model receives a scrubbed version of the input. The model cannot inadvertently reproduce raw PII from input it never read.
- **Output redaction**: any PII that the model generates — for example, an account number retrieved from a tool and echoed in a response — is scrubbed from the output returned to the caller and from any artifact stored for this run.

PII redaction is closely related to the `artifacts` section: if `pii_redaction: true`, saved artifacts already reflect the redacted values, making it safer to set `artifacts.save_input: true` when audit records are needed without storing raw personal data.

Enabling PII redaction adds latency (typically 10–50 ms). For agents that handle no personal data, leaving this `false` avoids unnecessary overhead.

#### `allow_external_http`

Controls whether the runtime permits tool calls that result in outbound HTTP traffic to hosts outside the internal network (hosts not on the platform's internal allowlist).

When `false` (the default), any tool invocation that would issue an external HTTP request is blocked at the runtime layer before execution. This defense-in-depth control prevents:

- Unintended data exfiltration via model-chosen tool arguments.
- Server-Side Request Forgery (SSRF) scenarios where a malicious prompt tricks the agent into calling an attacker-controlled endpoint.
- Accidental reliance on third-party services without explicit approval.

When `true`, the agent is permitted to call tools that reach external HTTP endpoints. This does not override network-level controls (VPC egress rules, security groups) but removes the AML-layer block. Setting this to `true` should be accompanied by a security review and documented justification.

This policy applies to outbound HTTP from tools called by the agent, not to the agent invocation endpoint itself (which is always internal).

#### `require_human_approval`

When `true`, the runtime pauses execution before every tool call and waits for an explicit approval signal from a human operator before sending the request. This is a coarse, agent-wide gate — it applies to every tool call regardless of the tool.

For finer control — requiring approval only for specific high-risk tools such as `send-email` or `create-ticket` — use `guardrails.tool_calls.require_user_confirmation_for` instead.

`require_human_approval: true` is appropriate for:

- New agents in early production roll-out where confidence in autonomous behavior has not yet been established.
- Agents whose tools have irreversible side effects and where the risk of an incorrect autonomous call is unacceptable.
- Regulatory environments that mandate a human-in-the-loop before any AI-initiated action.

This policy incurs significant latency per tool call, as execution is suspended until a human responds via the approval channel. Do not combine with strict `runtime.timeout_seconds` settings without accounting for this wait time.

#### `blocked_content`

An enumerated list of output content categories the agent is prohibited from generating. The runtime evaluates every response against the declared categories using a platform classifier. If a response matches any listed category, it is refused — the caller receives a refusal message and the response is not delivered.

Values must come from the platform's controlled vocabulary. Common values:

| Value | Blocked content |
|---|---|
| `legal_advice` | Specific legal guidance that could be construed as attorney-client advice |
| `medical_advice` | Specific medical diagnosis or treatment recommendations |
| `financial_advice` | Specific investment or tax recommendations |
| `adult_content` | Sexually explicit material |
| `hate_speech` | Content targeting individuals or groups based on protected characteristics |
| `violence` | Graphic depictions of or instructions for violence |

Blocked content categories are enforced by a platform classifier that runs on the model's output, not by a prompt instruction. Listing a category here does not mean the model will never attempt to generate it — it means the runtime will catch and refuse it if it does.

If the list is empty (`[]`) or the field is omitted, no content-category enforcement is applied beyond default platform-wide filters.

#### `allowed_connectors`

An allowlist of named integration connectors that this agent is permitted to access. A **connector** is a pre-configured integration to an external data service — such as SharePoint, Google Drive, Salesforce, or Jira — provisioned and named at the tenant level.

When this field is present and non-empty, only the listed connector names are accessible. Tool calls that attempt to reach a connector not on the list are blocked at the runtime layer. If this field is omitted or empty, all connector access is denied by default.

Connector names must match identifiers registered in the platform's connector registry. Unresolvable names are hard validation errors at compile time.

This field complements — but does not replace — the `tools` section. A tool that uses a named connector still requires a `ref` in `tools`; `allowed_connectors` provides an additional runtime-layer gate that limits which backing data sources that tool may ultimately reach.

#### `data_residency`

Declares the geographic boundary within which all data associated with this agent's runs must remain: execution, intermediate state, artifact storage, memory reads and writes, and audit logs.

The platform uses this declaration to:

- Route inference requests to model endpoints hosted within the declared region.
- Select a region-compliant artifact store backend at compile time.
- Raise a hard validation error if any referenced resource — model endpoint, artifact store, collection backend — is provisioned in a non-compliant region.

Accepted values follow a region code convention maintained by the platform team. Common values:

| Value | Region |
|---|---|
| `us` | United States (any US region) |
| `eu` | European Union (any EU region) |
| `ap` | Asia-Pacific (any AP region) |
| `us-east-1` | Specific AWS region (US East, N. Virginia) |
| `eu-west-1` | Specific AWS region (EU, Ireland) |

If omitted, the tenant's default residency setting applies. If no tenant default is configured, data residency is unconstrained — a lint warning for any agent operating in a regulated context.

#### `audit_level`

Controls the verbosity of the platform's internal audit log for runs of this agent. The audit log is a platform-managed, immutable record used by security, compliance, and operations teams. It is distinct from the team-accessible artifact store controlled by the `artifacts` section — both can be configured independently.

| Level | What is logged |
|---|---|
| `none` | No audit entries are written. Use only for development or testing agents that handle no real user data. Setting `none` on a production agent is a lint warning. |
| `standard` | Run start, run end, final outcome (success / refused / error), and the names of any tools called. Input content, output content, and tool arguments are not recorded. |
| `full` | Everything in `standard`, plus: tool call arguments and responses, knowledge base retrieval chunks, guardrail evaluation results, model reasoning traces (where available), and retry or fallback events. Required for regulated-industry deployments. |

`full` audit logging produces significantly more data per run and incurs higher storage costs. Use it where compliance requires a detailed evidence trail; use `standard` where operational oversight is sufficient.

Even at `audit_level: "full"`, if `pii_redaction: true` is set, all logged content reflects the already-redacted values — raw PII is never written to the audit log.

---

### `role` — IAM execution role (required)

```yaml
role: "support-agent-role"
```

References the IAM role this agent assumes at runtime. The role defines the ceiling of what the agent may do — specifically, which tools it is permitted to call and with what constraints.

`role` must resolve to a registered `.iam.md` file with `status: active` or `status: deprecated`. A reference to a `disabled` role is a hard validation error.

The agent's `tools` section may declare a subset of the tools permitted by the role. It may not reference tools outside the role's granted set — doing so is a hard validation error. The agent may impose stricter constraints (e.g., add `approval_required: true`) on individual tools, but it cannot loosen them.

Who can *call* this agent is not declared in AML. Caller access is managed at the infrastructure layer — Lambda resource policies, API Gateway authorizers, Cognito, or equivalent — outside of AML files. See the [IAM role definition](04-iam-definition.md) for the full model.

---

### `guardrails` — runtime validation (recommended)

```yaml
guardrails:
  input:
    - ref: "pii-scan"
      mode: "detect"
      on_fail: "redact"
    - ref: "prompt-injection-scan"
      mode: "block"
      on_fail: "refuse"
  tool_calls:
    require_user_confirmation_for:
      - "send-email"
      - "create-ticket"
  output:
    - ref: "schema-validation"
      on_fail: "retry"
    - ref: "unsafe-content-check"
      on_fail: "refuse"
```

#### What guardrails are

Guardrails are runtime intercept points, not prompt instructions. They run in the platform execution layer — before and after model execution, and around every tool call — independently of what the model was told to do. Declaring a guardrail here does not modify the system prompt; it inserts a programmatic check that the model never sees and cannot circumvent.

The distinction matters for reliability. A model can be confused, jailbroken, or simply wrong. A guardrail that detects PII and redacts it will do so regardless of whether the model was instructed to avoid PII. Defense in depth requires both.

#### Guardrail definition files

Every guardrail entry uses `ref`, which resolves to a [guardrail definition file](07-guardrail-definition.md) (`guardrails/<guardrail_id>.guardrail.md`). There is no inline form.

This follows the same pattern as `model`, `tools`, and `knowledge`: the agent file contains only a stable logical name (`pii-scan`), never provider-specific details. The compiler reads the referenced file and injects the full provider configuration — ARN, region, version, credentials, fallback policy — into the compiled payload. Platform-native checks (e.g., `schema-validation`, `jailbreak-check`) also have definition files; the platform team maintains those and agents reference them the same way.

#### `input` guardrails

Input guardrails run on the raw input payload before the model processes it. Each entry is evaluated in order; if any entry's `on_fail` action terminates the run (e.g., `refuse`), later entries in the list are not evaluated.

| Field | Required | Description |
|---|---|---|
| `ref` | yes | Logical name resolving to a `.guardrail.md` file. |
| `mode` | no | `detect` (identify and annotate, do not block) or `block` (prevent processing if triggered). Defaults to `block`. |
| `on_fail` | yes | Action to take when the guardrail triggers. See `on_fail` reference below. |

Common input guardrails:

| Guardrail | Purpose |
|---|---|
| `pii_scan` | Detect or redact personally identifiable information before it reaches the model |
| `prompt_injection_scan` | Detect attempts to override system instructions via user input |
| `jailbreak_check` | Detect adversarial inputs designed to bypass the model's safety training |
| `input_length_check` | Reject inputs exceeding a safe character/token limit |
| `language_check` | Verify the input is in an expected language before processing |

#### `tool_calls` guardrails

Tool-call guardrails intercept the execution of tool calls — after the model has decided to call a tool but before the call is dispatched.

```yaml
tool_calls:
  require_user_confirmation_for:
    - "send-email"
    - "delete-record"
  block_tools_on_input_fail: true    # If an input guardrail triggered, block all tool calls for this run
```

`require_user_confirmation_for` is a list of tool IDs for which the runtime must pause and request explicit human approval before executing the call. This is a more targeted alternative to `policies.require_human_approval` (which applies to every tool call).

Use this for any tool that has irreversible side effects: sending messages, creating or deleting records, calling external APIs with write permissions. Tools that are read-only generally do not need confirmation gates.

`block_tools_on_input_fail` prevents any tool from being called during a run in which an input guardrail was triggered (even if the run was allowed to continue with a `redact` action). Useful when a detected threat in the input — for example, a possible prompt injection — should prevent the agent from taking any external action even if the response is allowed.

#### `output` guardrails

Output guardrails run on the model's response before it is returned to the caller or persisted. They are the last enforcement line and must be considered mandatory for any publicly facing agent.

| Field | Required | Description |
|---|---|---|
| `ref` | yes | Logical name resolving to a `.guardrail.md` file. |
| `mode` | no | `detect` or `block`. Defaults to `block`. |
| `on_fail` | yes | Action to take when the guardrail triggers. |

Common output guardrails:

| Guardrail | Purpose |
|---|---|
| `schema_validation` | Verify the response matches the declared `output.schema` |
| `unsafe_content_check` | Detect harmful, offensive, or policy-violating content |
| `pii_output_scan` | Catch PII the model may have reproduced from tool results or context |
| `hallucination_check` | Flag responses that contradict information retrieved from KBs or tools |
| `citation_check` | Verify required citations are present when `provenance.citations_required: true` |

#### `on_fail` reference

| Value | Behavior |
|---|---|
| `redact` | Remove or mask the offending content and allow the (scrubbed) request or response to proceed. Only valid for guardrails that operate on content (PII, unsafe content); not valid for structural checks like `schema_validation`. |
| `refuse` | Terminate the run and return a platform-standard refusal message to the caller. No response or tool call is executed. Use for security-critical checks (prompt injection, jailbreak, unsafe content). |
| `retry` | Allow the run to retry up to `runtime.retry_policy.max_attempts` times. On exhaustion, falls back to `refuse`. Appropriate for transient quality issues like schema validation failure. |
| `escalate` | Trigger the matching rule in `orchestration.escalation` and route the run to its target. The runtime matches the fired guardrail's `ref` against `escalation` rules with `on: "guardrail"`. If no matching rule is found, the run fails. Use when the request exceeds the agent's scope and needs routing to a specialist or a human. |
| `log_only` | Allow the run to proceed but emit a guardrail event to the audit log. Use only in `mode: "detect"` during roll-out or monitoring phases; never for security controls. |

#### Relationship to `policies`

`policies` and `guardrails` address the same goal — safe, controlled execution — but at different levels:

| | `policies` | `guardrails` |
|---|---|---|
| **Abstraction level** | Declarative intent (what the platform should enforce) | Specific checks (which provider/check to invoke and how) |
| **Granularity** | Agent-wide flags (`pii_redaction: true`) | Per-check configuration with provider-specific parameters |
| **Provider awareness** | None — policies are provider-agnostic | Can reference external provider services (Bedrock, Azure) |
| **Sequencing control** | No ordering | Evaluated in list order; earlier `refuse` stops the chain |

`policies.pii_redaction: true` and an input guardrail `ref: "pii-scan"` can coexist and both run. The policy runs a platform-built redaction pass; the guardrail runs the configured provider check. Use both in depth-in-defense configurations: the policy handles base coverage, the guardrail adds provider-specific precision.

For any tool that writes, deletes, or sends external communications, `tool_calls` guardrails requiring user confirmation are strongly recommended.

---

### `ui` — product presentation (recommended)

```yaml
ui:
  icon: "headset"
  display_name: "Customer Support"
  short_description: "Get help with product and billing questions."
  order: 5
  featured: true
  ui_hints:
    question:
      widget: "textarea"
      label: "Your question"
      placeholder: "What can I help you with?"
    account_id:
      widget: "text"
      label: "Account ID"
      placeholder: "acct-XXXXX"
      helper_text: "Optional. Speeds up lookup if provided."
  localization:
    supported_locales: ["en", "fr", "de", "ja"]
    default_locale: "en"
    localized_ui:
      fr:
        display_name: "Support Client"
        short_description: "Obtenez de l'aide pour vos questions produit et facturation."
        ui_hints:
          question:
            label: "Votre question"
            placeholder: "Comment puis-je vous aider ?"
          account_id:
            label: "Identifiant de compte"
            placeholder: "acct-XXXXX"
            helper_text: "Facultatif. Accélère la recherche si fourni."
```

All `ui` fields control how the agent appears in product surfaces. They must **never** affect runtime behavior. The compiler is permitted to strip the entire `ui` block from the compiled runtime payload.

#### Field reference

| Field | Type | Required | Description |
|---|---|---|---|
| `icon` | string | No | Icon identifier from the platform icon set. Rendered in agent listings, navigation, and cards. Authors must use identifiers from the controlled icon vocabulary; unrecognised values fall back to a platform default. |
| `display_name` | string | Yes | Human-readable name shown in product surfaces (listings, page titles, breadcrumbs). May differ from `meta.name` — `meta.name` is administrative, `display_name` is customer-facing. |
| `short_description` | string | Yes | One sentence shown in catalog listings and agent selection screens. Keep it under 120 characters. |
| `order` | integer | No | Relative display order in lists and menus. Lower numbers appear first. Defaults to the platform default order if omitted. |
| `featured` | boolean | No | When `true`, the platform may surface this agent in promoted or "featured" slots. Defaults to `false`. |

#### `ui_hints` — per-field rendering hints

`ui_hints` maps each input field name (matching a key in `input.schema.properties`) to a set of rendering instructions for the front-end form.

```yaml
ui_hints:
  <field_name>:
    widget: "textarea"      # text | textarea | select | checkbox | date | file | file-multi | hidden
    label: "Your question"  # Human-readable label for the form field
    placeholder: "…"        # Ghost text shown when the field is empty
    helper_text: "…"        # Supplementary hint rendered below the field
    options:                # Required when widget: "select"; list of {value, label} pairs
      - value: "billing"
        label: "Billing issue"
      - value: "technical"
        label: "Technical issue"
    accept: ["image/*", "video/*", ".pdf", ".docx"]  # Optional; "file" and "file-multi" only
    max_size_mb: 10                                    # Optional; "file" and "file-multi" only
```

| Key | Description |
|---|---|
| `widget` | Controls the HTML input type. `text` for single-line, `textarea` for multi-line, `select` for a dropdown, `checkbox` for boolean, `date` for a date picker, `file` for a single-file upload, `file-multi` for a multi-file upload, `hidden` to send a field without showing it. |
| `label` | The visible field label. Falls back to the field name if omitted. |
| `placeholder` | Ghost text inside the field. Keep it instructive, not an example value. |
| `helper_text` | Short guidance text rendered below the field (not inside it). |
| `options` | Required for `widget: "select"`. Each entry must have `value` (sent to the runtime) and `label` (shown to the user). |
| `accept` | Optional. Applies to `file` and `file-multi` only. A list of MIME types (e.g., `image/*`, `video/mp4`) or file extensions (e.g., `.pdf`, `.docx`) the picker should filter to. The runtime enforces the declared types; the front-end may also apply client-side filtering. If omitted, all file types are accepted. |
| `max_size_mb` | Optional. Applies to `file` and `file-multi` only. Maximum size in megabytes for a single uploaded file. The front-end enforces this client-side; the runtime enforces it again at the boundary. Omitting this field means the platform default limit applies. |

`ui_hints` is **purely cosmetic** — the runtime never reads it. A field without a `ui_hint` entry is still rendered (by the front-end's default for its JSON Schema type). `hidden` fields pass their default or injected value through without user interaction; authors must ensure hidden fields do not bypass input validation.

#### `localization` — locale support

```yaml
localization:
  supported_locales: ["en", "fr", "de", "ja"]
  default_locale: "en"
  localized_ui:
    fr:
      display_name: "Support Client"
      short_description: "Obtenez de l'aide pour vos questions produit et facturation."
      ui_hints:
        question:
          label: "Votre question"
          placeholder: "Comment puis-je vous aider ?"
        account_id:
          label: "Identifiant de compte"
          placeholder: "acct-XXXXX"
          helper_text: "Facultatif. Accélère la recherche si fourni."
```

`supported_locales` declares which locales this agent's UI has been prepared for. The platform uses this list to hide the agent in locales where no translation has been provided. `default_locale` is the fallback when the user's locale is not in `supported_locales`.

`localized_ui` provides per-locale overrides for any displayable `ui` field. Three levels of content can be localized:

- **Global description fields** — `display_name` and `short_description`.
- **Per-field hints** — any `ui_hints` entry can be overridden under a `ui_hints` key inside the locale block, supporting `label`, `placeholder`, `helper_text`, and `options[].label`. Only the overridden sub-fields need to be listed.

Anything not present in a locale's block inherits from the `default_locale`. Locale keys must be BCP 47 language tags (e.g., `fr`, `de`, `ja`, `pt-BR`).

Localization applies to **UI labels only**. It does not change the agent's system instructions, output language, or runtime behavior — those are governed by the `# System behavior` section of the Markdown body and by the input payload the caller sends.

---

### `orchestration` — delegation and handoff (optional)

The `orchestration` block governs how this agent drives multi-agent systems — whether it may hand work to other agents or humans, whether it can be called as a tool by other agents, and whether it owns and runs a multi-agent pattern (Graph, Swarm, or Workflow).

This section has two distinct concerns:

1. **Agent-level delegation** — which agents or human queues this individual agent is permitted to route work to.
2. **Pattern ownership** — declaring that this agent defines and drives a multi-agent pattern.

The `orchestration` block is always written from the point of view of the current agent. The agent whose file declares `orchestration.pattern` is the orchestrator — it is the one that builds and runs the pattern. Member agents (nodes in a Graph, peers in a Swarm, task executors in a Workflow) are referenced by `agent_id` but do not repeat the pattern definition in their own files. They only need to exist as registered agents.

Only declare `orchestration` when the agent is expected to drive other agents or a human queue. A purely standalone agent with no multi-agent role should omit this section entirely.

---

#### Agent-level delegation

```yaml
orchestration:
  can_delegate: true
  allowed_delegates: ["refund-agent", "escalation-agent"]
  can_be_invoked_as_tool: false
  escalation:
    - on: "guardrail"
      guardrail: "prompt-injection-scan"   # Matches the ref in guardrails.input or guardrails.output
      target:
        type: "human_queue"
        id: "security-review"
    - on: "guardrail"
      guardrail: "unsafe-content-check"
      target:
        type: "human_queue"
        id: "content-moderation"
    - on: "model_decision"                  # Model explicitly triggers escalation
      target:
        type: "human_queue"
        id: "tier-2-support"
    - on: "node_failure"                    # A downstream graph/swarm/workflow node fails
      node: "billing-specialist"            # Optional: specific node id; omit to match any node
      target:
        type: "agent"
        id: "billing-escalation-agent"
    - on: "node_failure"                    # Catch-all for any other node failure
      target:
        type: "human_queue"
        id: "ops-oncall"
    - on: "timeout"                         # Pattern execution_timeout exceeded
      target:
        type: "human_queue"
        id: "ops-oncall"
```

##### Field reference — delegation

| Field | Type | Default | Description |
|---|---|---|---|
| `can_delegate` | bool | `false` | Permits this agent to hand work to another agent listed in `allowed_delegates`. When `false`, the runtime blocks any delegation attempt regardless of what the model produces. |
| `allowed_delegates` | string[] | `[]` | Explicit whitelist of `agent_id` values this agent may route work to. Must resolve to registered `.agent.md` files. An unresolvable ID is a hard validation error. The runtime must reject any handoff to an ID not on this list. |
| `can_be_invoked_as_tool` | bool | `false` | When `true`, this agent exposes itself as a callable tool so that other orchestrators (Graph nodes, Swarm agents, parent agents) can invoke it. The agent's `input` and `output` schemas serve as the tool's parameter and return type contracts. |
| `escalation` | list | `[]` | Ordered list of trigger-based escalation rules. The runtime evaluates rules top-to-bottom and uses the first matching rule. See below. |

##### `escalation` rules

Each rule has an `on` field that declares when it fires, optional narrowing fields, and a `target` that describes where control goes.

###### `on` — trigger types

| `on` value | Fires when | Narrowing fields |
|---|---|---|
| `guardrail` | A named guardrail fires with `on_fail: "escalate"` in `guardrails.input` or `guardrails.output` | `guardrail`: the `ref` value of the triggering guardrail. Required to match a specific guardrail; omit to match any guardrail escalation. |
| `model_decision` | The agent's model explicitly produces an escalation decision (as governed by the `# Escalation` section of the Markdown body) | — |
| `node_failure` | A downstream node in a Graph, Swarm, or Workflow reaches `FAILED` status | `node`: the `id` of the specific node to match. Optional; omit to match any node failure. |
| `timeout` | The pattern's `execution_timeout` is exceeded | — |

Rules are evaluated in declaration order. The first rule whose `on` type and all narrowing fields match the current event is used. If no rule matches, the run returns a `FAILED` status to the caller with no escalation.

More specific rules (those with narrowing fields like `node` or `guardrail`) must be declared before catch-all rules of the same `on` type, otherwise the catch-all fires first.

###### `target` — escalation destinations

Each rule's `target` has a `type` and an `id`:

| `type` | `id` resolves to | Behavior |
|---|---|---|
| `human_queue` | A named queue in the platform's human-in-the-loop routing registry | Pauses the run and routes it to a human operator. The run resumes (or is closed) when a human responds. |
| `agent` | An `agent_id` registered in `.agent.md` — must have `can_be_invoked_as_tool: true` | Immediately delegates to the target agent and returns its result to the original caller. |

Delegation rights are enforced by the runtime, not by the model. Even if the model's output contains an instruction to delegate to an agent not listed in `allowed_delegates`, the runtime must refuse the handoff and return an error to the caller.

---

#### Multi-agent patterns

When this agent owns and drives a multi-agent architecture, the `orchestration.pattern` block declares and configures that pattern. Three patterns are supported: **Graph**, **Swarm**, and **Workflow**.

> **Key distinction** — the most important difference between the three patterns is *how the execution path is decided*:

> - **Graph**: a developer pre-defines all nodes and edges; an LLM decides which edge to take at each node.
> - **Swarm**: agents autonomously hand off to each other; no developer-defined path exists.
> - **Workflow**: the execution order is fully deterministic, fixed by a dependency graph (DAG); no LLM routing decisions are made.

Graph and Swarm are built-in SDK orchestrators. Workflow is either implemented in code by chaining agent calls or by using the `workflow` tool from `strands-agents-tools`.

---

##### Pattern comparison

| | **Graph** | **Swarm** | **Workflow** |
|---|---|---|---|
| **Core concept** | Structured flowchart where an LLM decides which edge to take | Dynamic team where agents autonomously hand off tasks to each other | Pre-defined DAG executed as a single deterministic action |
| **Structure** | Developer defines all nodes and edges | Developer provides a pool of agents; agents choose the path | Developer defines tasks and their dependencies in code |
| **Execution flow** | Controlled but dynamic | Sequential and autonomous | Deterministic and parallel-where-possible |
| **Cycles allowed** | Yes | Yes | No |
| **State sharing** | Shared `invocation_state` object available to all nodes | Shared context accumulates knowledge from each agent's turn | Task outputs are captured and passed as inputs to dependent tasks |
| **Conversation history** | Full transcript shared across all nodes | Full handoff history shared across agents | Task-specific — each task receives only its direct dependencies' outputs |
| **Behavior control** | User input at each step can influence which path is taken | User's initial prompt sets the goal; swarm runs autonomously | User's prompt may trigger the workflow but cannot alter its structure at runtime |
| **Error handling** | Developer can define explicit error edges to route failures | Agent-driven; relies on handoff limits and timeouts | Systemic — a failing task halts all downstream dependent tasks |
| **Scalability** | Scales with process complexity (branches, conditions) | Scales with number of specialised agents | Scales with repeatable, complex parallel pipelines |

---

##### Graph pattern

A **Graph** is a deterministic directed graph where agents (or nested multi-agent systems) are nodes, connected by edges that carry optional conditions. Execution follows edge dependencies; an LLM decision at each node determines which conditional edge is traversed. Graphs support both DAG and cyclic topologies.

###### When to use Graph

- Processes with conditional branching based on output content (e.g., route to "technical" or "billing" path based on classification).
- Feedback loops where a reviewer node can send work back to a drafter node until a quality bar is met.
- Structured business workflows where every transition must be explicitly modelled.

###### YAML declaration

The current agent is always the first node to execute — it is the implicit entry point of the graph. All other nodes are downstream agents that the current agent dispatches to.

```yaml
orchestration:
  pattern:
    type: "graph"
    execution_timeout: 600            # Total timeout for the whole graph, in seconds
    max_node_executions: 20           # Max total node executions across the entire graph (cycle guard)
    node_timeout: 120                 # Per-node timeout in seconds
    reset_on_revisit: true            # Reset a node's agent state when it is revisited in a cycle
    nodes:
      - id: "billing-specialist"
        agent_ref: "billing-agent"
      - id: "tech-specialist"
        agent_ref: "tech-support-agent"
      - id: "report-writer"
        agent_ref: "reporting-agent"
    edges:
      - from: "$self"
        to: "billing-specialist"
        condition: "result_contains('billing')"   # Optional condition expression
      - from: "$self"
        to: "tech-specialist"
        condition: "result_contains('technical')"
      - from: "billing-specialist"
        to: "report-writer"
      - from: "tech-specialist"
        to: "report-writer"
```

The reserved node identifier `$self` refers to the current agent (the orchestrator). Use it in `edges.from` when the first transition originates from this agent. `$self` must not appear in the `nodes` list — it is always implicitly present.

###### `nodes` field reference

| Sub-field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique node identifier within this graph. Used in `edges`. Lowercase kebab-case. |
| `agent_ref` | string | Yes | `agent_id` of the agent or multi-agent system to execute at this node. Must resolve to a registered `.agent.md` file or a nested pattern declaration. |

###### `edges` field reference

| Sub-field | Type | Required | Description |
|---|---|---|---|
| `from` | string | Yes | Source node `id`. |
| `to` | string | Yes | Target node `id`. |
| `condition` | string | No | Optional predicate that must evaluate to `true` for this edge to be traversed. The runtime evaluates conditions against the `GraphState` object after the source node completes. If omitted, the edge is always traversed (unconditional). |

###### Top-level Graph parameters

The current agent is always the implicit entry point. There is no `entry_point` field.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `execution_timeout` | int (seconds) | 900 | Hard wall-clock limit for the entire graph execution. When reached, the graph halts and returns a `FAILED` status regardless of node progress. |
| `max_node_executions` | int | — | Maximum total number of node executions across the entire graph run. Essential for graphs with cycles — without this, a looping graph can run indefinitely. |
| `node_timeout` | int (seconds) | 300 | Per-node timeout. If any single node exceeds this limit, it is marked `FAILED` and the graph's error handling takes over. |
| `reset_on_revisit` | bool | `false` | When `true`, a node's agent conversation state is cleared each time the node is re-entered (relevant for cyclic graphs with a feedback loop). When `false`, accumulated state is carried across visits. |

###### Common Graph topologies

In all topologies below, `$self` is the current agent (the orchestrator). Only downstream nodes appear in `nodes`.

**Sequential pipeline** — the current agent hands off to a chain; each node feeds the next:

```
$self → analysis → review → report
```

**Parallel fan-out with aggregation** — the current agent fans work to parallel specialists, which converge at an aggregator:

```
$self → worker-1 ┐
$self → worker-2 ├→ aggregator
$self → worker-3 ┘
```

**Branching (conditional routing)** — the current agent classifies and routes to a specialist:

```
$self →(is_billing)→ billing-specialist → billing-report
$self →(is_technical)→ tech-specialist → tech-report
```

**Feedback loop** — a reviewer can send work back for revision until it approves. The current agent starts the draft:

```
$self → reviewer →(approved)→ publisher
        reviewer →(needs_revision)→ $self  (cycle)
```

---

##### Swarm pattern

A **Swarm** is a self-organising team of specialised agents that collaborate through autonomous handoffs. No developer-defined flow exists — agents decide which peer is best suited to handle the next step. Each agent sees the full history of prior contributions and a registry of available peers (declared as tools - we assume that A2A connections are handled by Lambda equivalent functions to enforce security).

###### When to use Swarm

- Multidisciplinary problems that benefit from diverse specialist perspectives (incident response, software design).
- Exploratory or brainstorming tasks where the most useful next step is not known in advance.
- Collaborative synthesis where later agents build explicitly on earlier agents' findings.

###### YAML declaration

The current agent is always the first to execute in the swarm — it is the implicit entry point. The `agents` list contains only the peer agents the current agent may hand off to. All listed agents must have `can_be_invoked_as_tool: true` in their own definitions.

```yaml
orchestration:
  pattern:
    type: "swarm"
    max_handoffs: 20                  # Maximum number of agent-to-agent handoffs
    max_iterations: 20                # Maximum total agent executions across the swarm
    execution_timeout: 900            # Total execution timeout in seconds (15 min default)
    node_timeout: 300                 # Per-agent timeout in seconds (5 min default)
    repetitive_handoff_detection_window: 8   # Size of the sliding window to detect ping-pong
    repetitive_handoff_min_unique_agents: 3  # Minimum unique agents required in that window
    agents:
      - ref: "architect"
      - ref: "coder"
      - ref: "reviewer"
```

###### `agents` field reference

| Sub-field | Type | Required | Description |
|---|---|---|---|
| `ref` | string | Yes | `agent_id` of the agent to include in the swarm pool. Must resolve to a registered `.agent.md` file with `can_be_invoked_as_tool: true`. All listed agents are available for peer handoffs. |

###### Top-level Swarm parameters

The current agent is always the implicit entry point. There is no `entry_point` field.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `max_handoffs` | int | 20 | Maximum number of agent-to-agent handoffs permitted. Once reached, the current agent must produce a final answer without further delegation. Prevents runaway chains. |
| `max_iterations` | int | 20 | Maximum total agent executions across the entire swarm run (including the entry agent). A single agent being revisited counts as a new iteration. |
| `execution_timeout` | int (seconds) | 900 | Hard wall-clock limit for the entire swarm. When reached, the swarm halts and returns a `FAILED` status. |
| `node_timeout` | int (seconds) | 300 | Per-agent timeout. If any individual agent execution exceeds this, it is marked `FAILED`. |
| `repetitive_handoff_detection_window` | int | 0 (disabled) | Size of the recent-handoff sliding window used to detect ping-pong behavior. Set to 0 to disable. When enabled, must be ≥ `repetitive_handoff_min_unique_agents + 1`. |
| `repetitive_handoff_min_unique_agents` | int | 0 (disabled) | Minimum number of unique agents required within the last `repetitive_handoff_detection_window` handoffs. If fewer unique agents are seen, the swarm is terminated with an error. Prevents two agents bouncing work between themselves indefinitely. |

###### How agents coordinate in a Swarm

The current agent receives the original user task and begins the swarm. Each subsequent agent (including the current agent if control returns to it) receives a rich context block containing:

- The original user task.
- The ordered history of which agents have worked on it.
- Accumulated knowledge contributed by previous agents (as key-value pairs).
- The registry of all available peer agents and their descriptions.

Agents invoke the built-in `handoff_to_agent` tool to transfer control. A handoff carries a `message` (the new task or context for the receiving agent) and optionally a structured `context` object with key information to share. Only agents listed in `orchestration.pattern.agents` are valid handoff targets — the runtime rejects any attempt to hand off to an unlisted agent.

The swarm terminates when an agent returns a final answer without calling `handoff_to_agent`, or when a safety limit (`max_handoffs`, `max_iterations`, `execution_timeout`) is reached.

---

##### Workflow pattern

A **Workflow** is a developer-defined task dependency graph (DAG) executed as a single, non-conversational action. Tasks with satisfied dependencies run in parallel automatically. No LLM routing occurs — the execution order is fixed at definition time. The `workflow` tool from `strands-agents-tools` implements this pattern.

###### When to use Workflow

- Repeatable, well-understood pipelines (data extraction → analysis → report).
- Operations where independent steps should run in parallel for efficiency.
- Processes that must be encapsulated as a single, reusable, invokable action.
- Scenarios requiring pause/resume, automatic retry of failed tasks, and detailed step-level audit.

###### YAML declaration

```yaml
orchestration:
  pattern:
    type: "workflow"
    max_workers: 4                         # Max parallel task threads
    retry_failed_tasks: true              # Automatically retry failed tasks once
    tasks:
      - task_id: "data-extraction"
        agent_ref: "data-extractor-agent"
        description: "Extract key financial data from the quarterly report."
        priority: 5                        # Higher number = higher priority when scheduling
        dependencies: []
      - task_id: "trend-analysis"
        agent_ref: "analyst-agent"
        description: "Identify trends comparing against previous quarters."
        priority: 3
        dependencies: ["data-extraction"]
      - task_id: "risk-assessment"
        agent_ref: "risk-agent"
        description: "Assess financial risks from the extracted data."
        priority: 3
        dependencies: ["data-extraction"]
      - task_id: "report-generation"
        agent_ref: "reporting-agent"
        description: "Produce the final quarterly analysis report."
        priority: 1
        dependencies: ["trend-analysis", "risk-assessment"]
```

###### `tasks` field reference

| Sub-field | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | Yes | Unique identifier for this task within the workflow. Used in `dependencies` lists of other tasks. Lowercase kebab-case. |
| `agent_ref` | string | Yes | `agent_id` of the agent that will execute this task. Must resolve to a registered `.agent.md` file. |
| `description` | string | Yes | Plain-language description of what this task must accomplish. Passed to the assigned agent as its input prompt. |
| `priority` | int | 0 | Scheduling hint. When multiple tasks are simultaneously ready (dependencies satisfied), higher-priority tasks are dispatched first. Has no effect on tasks whose dependencies are not yet satisfied. |
| `dependencies` | string[] | `[]` | List of `task_id` values that must complete successfully before this task can be scheduled. An empty list means the task is an entry point and starts immediately. Circular dependencies are a hard validation error. |

###### Top-level Workflow parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `max_workers` | int | platform default | Maximum number of tasks that may execute in parallel. The workflow executor scales thread allocation up to this limit. Higher values increase throughput but also resource consumption. |
| `retry_failed_tasks` | bool | `false` | When `true`, a task that fails is automatically retried once before the workflow propagates the failure downstream. Does not override `runtime.retry_policy` on individual agents — this is a workflow-level retry gate applied before the agent's own retry policy. |

###### Execution model

1. All tasks with no `dependencies` (entry tasks) are scheduled immediately.
2. As tasks complete, the workflow engine checks whether any pending tasks now have all their dependencies satisfied; those are added to the execution queue.
3. Independent ready tasks run in parallel up to `max_workers`.
4. A task receives the original workflow input **plus** the outputs of all its direct dependencies as context.
5. If a task fails and `retry_failed_tasks: true`, it is retried once. A second failure marks the task `FAILED` and halts all downstream tasks that depend on it.
6. The workflow completes when all tasks reach a terminal state (`COMPLETED` or `FAILED`).

###### Workflow tool actions

The executing agent can manage lifecycle through the following actions:

| Action | Description |
|---|---|
| `start` | Begin execution. The tool resolves the dependency graph, schedules entry tasks, and runs parallel execution. |
| `status` | Return the current execution state: overall progress percentage, per-task status, execution times, and error details. |
| `pause` | Suspend execution after the current in-flight tasks complete. State is persisted. |
| `resume` | Resume a paused workflow from where it left off. |
| `list` | List all workflows with their current statuses. |
| `delete` | Remove a workflow definition and its associated state. |


---

### `observability` — tracing, metrics, and logs (recommended)

```yaml
observability:
  tracing:
    enabled: true
    exporter: "otlp"                         # otlp | console | none
    endpoint: "https://otel.internal/v1/traces"
    sampling_rate: 1.0                       # 0.0–1.0; 1.0 = every run, 0.1 = 10%
    include_model_inputs: false              # Include model input messages in spans (PII risk)
    include_model_outputs: true             # Include model output messages in spans
    include_tool_args: "redacted"           # full | redacted | none
    include_tool_results: "redacted"        # full | redacted | none
    include_kb_chunks: false                # Include retrieved KB chunks in spans
    propagate_context: true                 # Propagate W3C Trace Context to tools and sub-agents
    auth:
      type: "bearer"                           # bearer | mtls | none
      credentials_ref: "otel-collector-token" # Reference to a platform secret store entry

  metrics:
    enabled: true
    exporter: "otlp"
    endpoint: "https://otel.internal/v1/metrics"
    export_interval_seconds: 60
    retention_days: 30
    custom_dimensions:
      - "agent_id"
      - "model_id"
      - "environment"
    auth:
      type: "bearer"
      credentials_ref: "otel-collector-token"

  logs:
    enabled: true
    exporter: "otlp"
    endpoint: "https://otel.internal/v1/logs"
    level: "info"                            # debug | info | warn | error
    structured: true                         # Emit JSON-structured log records
    include_trace_context: true              # Correlate log records with active trace/span IDs
    auth:
      type: "bearer"
      credentials_ref: "otel-collector-token"
```

#### Collector authentication

Agent authors **never write secrets inline**. The `auth.credentials_ref` field holds a logical name that the compiler resolves to a platform secret store entry — the same principle used for tool credentials, model provider API keys, and artifact store credentials throughout this spec. The resolved secret (a bearer token, a client certificate, or other material) is injected into the compiled payload at build time and managed by the platform team, not the agent author.

Three authentication modes are supported:

| `auth.type` | How it works | When to use |
|---|---|---|
| `none` | No credential is attached to OTLP requests. Valid only for collectors that are fully internal (same network namespace, no public exposure) and rely on network-level controls (VPC, service mesh policy) for access control. | Internal deployments with network isolation only. |
| `bearer` | The runtime attaches an `Authorization: Bearer <token>` HTTP header to every OTLP request. The token is read from the secret identified by `credentials_ref`. | Most common mode for collectors exposed over HTTPS with token-based auth. |
| `mtls` | The runtime presents a client TLS certificate during the handshake. `credentials_ref` must point to a secret containing both the client certificate and the private key (PEM-encoded). The collector's CA certificate is managed at the tenant level. | High-security environments where mutual authentication is required by policy. |

`credentials_ref` must resolve to a registered entry in the platform secret store. Unresolvable references are hard compile errors. The secret value is never written to the agent definition file, and it is never emitted in logs or audit records. Rotation of the secret is handled at the secret store level — no agent recompile is required unless the `credentials_ref` name itself changes.

If all three signal blocks point to the same collector with the same credentials, they may share the same `credentials_ref` value. The platform recommends a single internal collector endpoint per tenant, scoped by environment, with a single service credential.

#### Field reference — `tracing`

| Field | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `false` | Master switch. When `false`, no spans are emitted regardless of other settings. |
| `exporter` | enum | `otlp` | Export target. `otlp`: push via OTLP protocol to `endpoint`. `console`: write human-readable output to stdout (development only; lint error on `status: active`). `none`: disable export. |
| `endpoint` | string | — | OTLP receiver URL for traces (e.g., an OTel Collector `/v1/traces` path). Required when `exporter: "otlp"`. |
| `sampling_rate` | float | `1.0` | Fraction of runs that produce a trace. `1.0` = every run; `0.1` = one in ten. Applies at the agent runtime layer before the collector. |
| `include_model_inputs` | bool | `false` | Attach full model input message lists to `agent.model_invocation` spans. High PII risk — keep `false` in production unless `pii_redaction: true` is set. |
| `include_model_outputs` | bool | `true` | Attach model response messages to `agent.model_invocation` spans. |
| `include_tool_args` | enum | `redacted` | Level of tool argument detail in `agent.tool_call` spans. `full`: all arguments verbatim. `redacted`: fields marked `sensitive: true` in the tool definition are replaced with `[REDACTED]`. `none`: no arguments recorded. |
| `include_tool_results` | enum | `redacted` | Level of tool response detail in `agent.tool_call` spans. Same enum as `include_tool_args`. |
| `include_kb_chunks` | bool | `false` | Attach retrieved KB chunks as span events on `agent.kb_retrieval` spans. Can produce large spans on high-`max_chunks` retrievals. |
| `propagate_context` | bool | `true` | Inject a W3C `traceparent` header into outbound tool calls and sub-agent invocations to enable distributed tracing across the call chain. |
| `auth.type` | enum | `none` | Authentication mode for OTLP requests. `none` \| `bearer` \| `mtls`. See *Collector authentication* above. |
| `auth.credentials_ref` | string | — | Logical name resolving to a platform secret store entry. Required when `auth.type` is `bearer` or `mtls`. Never write the secret value directly. |

#### Field reference — `metrics`

| Field | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `false` | Master switch. When `false`, no metric data points are emitted. |
| `exporter` | enum | `otlp` | Export target. Same values as `tracing.exporter`. |
| `endpoint` | string | — | OTLP receiver URL for metrics (e.g., `/v1/metrics`). Required when `exporter: "otlp"`. |
| `export_interval_seconds` | int | `60` | How often cumulative metric values are flushed to the exporter. Shorter intervals increase backend write load; longer intervals increase the staleness of dashboards. |
| `retention_days` | int | tenant default | How long metric data is retained in the backend. Enforced by the OTel Collector or backend, not the agent runtime. |
| `custom_dimensions` | string[] | `[]` | List of OTel resource attribute keys to attach to every metric data point from this agent. Values are resolved from the run context at export time (e.g., `"environment"` resolves to `"production"` or `"staging"`). Unresolvable keys are silently dropped. Used to slice metrics by environment, product area, or tenant in dashboards. |
| `auth.type` | enum | `none` | Authentication mode. Same values and semantics as `tracing.auth.type`. |
| `auth.credentials_ref` | string | — | Logical name resolving to a platform secret store entry. Required when `auth.type` is `bearer` or `mtls`. |

#### Field reference — `logs`

| Field | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `false` | Master switch. When `false`, no log records are emitted via the OTel log exporter. |
| `exporter` | enum | `otlp` | Export target. Same values as `tracing.exporter`. |
| `endpoint` | string | — | OTLP receiver URL for logs (e.g., `/v1/logs`). Required when `exporter: "otlp"`. |
| `level` | enum | `info` | Minimum severity. Records below this level are not emitted. `debug` \| `info` \| `warn` \| `error`. |
| `structured` | bool | `true` | Emit JSON-formatted OTel log records instead of plain text lines. Required for field-level querying in log backends. |
| `include_trace_context` | bool | `true` | Attach the active `trace_id` and `span_id` to every log record, enabling correlation with traces in the same OTel backend. |
| `auth.type` | enum | `none` | Authentication mode. Same values and semantics as `tracing.auth.type`. |
| `auth.credentials_ref` | string | — | Logical name resolving to a platform secret store entry. Required when `auth.type` is `bearer` or `mtls`. |

---

All telemetry emitted by this agent is **OpenTelemetry (OTel) compatible**. The platform runtime acts as an OTel instrumentation layer: it creates spans, records metrics, and emits log records using standard OTel data models and exporters. Any OTel-compatible backend — Jaeger, Zipkin, Tempo, Prometheus, OpenSearch, Datadog, Honeycomb, AWS X-Ray, and others — can receive the data via OTLP.

#### The three telemetry primitives

Agent observability requires capturing three distinct kinds of signals. Unlike traditional software, agents add AI-specific telemetry: model invocations, tool decisions, reasoning traces, and KB retrievals must all be observable alongside the usual application-level signals.

---

##### Traces

A **trace** represents one complete agent run, from input receipt to response delivery. It is composed of **spans** — one per meaningful operation the agent performs within that run.

The platform runtime automatically creates the following span types:

| Span type | What it records |
|---|---|
| `agent.run` | Root span for the entire run. Attributes: `agent_id`, `version`, `status` (success / refused / error), total input tokens, total output tokens, total latency. |
| `agent.model_invocation` | One span per model call within the run. Attributes: `model_id`, `temperature`, `reasoning_effort`, `input_token_count`, `output_token_count`, `latency_ms`, finish reason. When `include_model_inputs: true`, the full prompt messages are recorded. When `include_model_outputs: true`, the response message is recorded. |
| `agent.tool_call` | One span per tool invocation. Attributes: `tool_id`, `tool_version`, `latency_ms`, `outcome` (success / error / approval-pending). When `include_tool_args` is set to `full` or `redacted`, the call arguments are captured. When `include_tool_results` is set, the response payload is captured. |
| `agent.kb_retrieval` | One span per knowledge base retrieval. Attributes: `kb_id`, `search_mode`, `chunks_returned`, `latency_ms`. When `include_kb_chunks: true`, the retrieved chunks are stored as span events. |
| `agent.guardrail` | One span per guardrail evaluation. Attributes: `guardrail_id`, `phase` (input / output / tool_call), `triggered` (bool), `action` (redact / refuse / retry / escalate / log_only), `latency_ms`. |

`propagate_context: true` injects a [W3C `traceparent` header](https://www.w3.org/TR/trace-context/) into every outbound tool call and sub-agent invocation. This stitches tool-side and sub-agent spans into the same distributed trace, giving end-to-end visibility across a multi-agent or multi-service call chain.

**`sampling_rate`** controls what fraction of runs produce a trace. `1.0` (full sampling) is the right default for development. In high-volume production, reduce sampling (e.g., `0.1`) to control costs and storage, while using head-based or tail-based sampling at the OTel collector layer to ensure all error and latency-outlier runs are always captured.

---

##### Metrics

Metrics are numeric time-series measurements. The platform emits the following metric families as OTel instruments. All metrics carry attributes for `agent_id`, `version`, and any `custom_dimensions` declared above.

**Agent metrics**

| Metric | Instrument type | Description |
|---|---|---|
| `agent.run.count` | Counter | Total number of runs, by `status` (success / refused / error). |
| `agent.run.duration` | Histogram | End-to-end run latency in milliseconds, from input received to response delivered. |
| `agent.run.loops` | Histogram | Number of agent-loop iterations per run (tool call → model response cycles). |
| `agent.run.input_chars` | Histogram | Size of the raw input payload in characters. |

**Model metrics**

| Metric | Instrument type | Description |
|---|---|---|
| `agent.model.input_tokens` | Counter | Total input tokens consumed across all model invocations. |
| `agent.model.output_tokens` | Counter | Total output tokens generated across all model invocations. |
| `agent.model.latency` | Histogram | Time-to-first-byte and time-to-last-byte for each model invocation. |
| `agent.model.errors` | Counter | Model API errors, by `error_type` (rate_limit / timeout / unavailable / other). |

**Tool metrics**

| Metric | Instrument type | Description |
|---|---|---|
| `agent.tool.invocations` | Counter | Total tool calls, by `tool_id` and `outcome`. |
| `agent.tool.latency` | Histogram | Execution time per tool call by `tool_id`. |
| `agent.tool.errors` | Counter | Tool errors by `tool_id` and `error_type`. |
| `agent.tool.approval_wait` | Histogram | Time spent waiting for human approval, for tools with `approval_required: true`. |

**Guardrail metrics**

| Metric | Instrument type | Description |
|---|---|---|
| `agent.guardrail.triggered` | Counter | Guardrail activations by `guardrail_id`, `phase`, and `action`. |
| `agent.guardrail.latency` | Histogram | Evaluation time per guardrail by `guardrail_id`. |

`custom_dimensions` are OTel resource attributes appended to every metric data point from this agent. Use them to slice metrics by deployment environment, product area, or tenant. Dimensions must be resolvable from the run context at telemetry export time; unresolvable dimension keys are silently dropped at export.

---

##### Logs

Log records are structured, timestamped text events emitted at specific points during a run. With `structured: true`, records are emitted as JSON-formatted OTel log records rather than plain text, enabling machine parsing and field-level querying in any OTel-compatible log backend.

With `include_trace_context: true`, every log record carries `trace_id` and `span_id` attributes from the active OTel span context at emission time. This allows a log line to be correlated directly to the exact model invocation or tool call that produced it, without manual cross-referencing.

The platform emits log records for:

- Run start and end (always, at `info` level).
- Guardrail trigger events (at `warn` level when triggered, `debug` when not triggered).
- Tool call approval requests and outcomes (at `info` level).
- Retry and fallback events (at `warn` level).
- Validation errors and refusals (at `error` level).

The `level` field controls the minimum severity of records emitted. `info` is appropriate for production. `debug` adds span-level lifecycle events and is useful during development. `warn` and `error` reduce log volume but may miss important non-error events; use `warn` only for cost-sensitive, high-throughput agents.

---

#### OTel export architecture

All three signal types share the same exporter architecture. The `exporter: "otlp"` value causes the runtime to push signals using the [OTLP](https://opentelemetry.io/docs/specs/otlp/) protocol over gRPC or HTTP/protobuf to the declared `endpoint`. The endpoint should be an OTel Collector rather than a backend directly — this decouples the agent definition from backend choice and enables fan-out to multiple downstream consumers (monitoring dashboards, tracing UIs, data warehouses, compliance vaults) from a single collection point.

```
Agent Runtime
    │
    │  OTLP (gRPC/HTTP)
    ▼
OTel Collector
    │
    ├──► Tracing backend  (Jaeger / Tempo / X-Ray / Honeycomb / …)
    ├──► Metrics backend  (Prometheus / Datadog / CloudWatch / …)
    └──► Log backend      (OpenSearch / Loki / CloudWatch Logs / …)
```

`endpoint` must point to an internal OTel Collector endpoint. Direct exporting to a third-party backend SaaS endpoint is permitted but not recommended, as it couples the agent definition to a specific vendor and prevents fan-out. The platform team maintains the collector configuration; agent authors only declare the collector endpoint.

When `exporter: "console"` is set, signals are written to stdout in human-readable format for local development. This setting is not permitted on `status: active` agents — it is a hard lint error.

`exporter: "none"` disables export for that signal type. Disabling all three signal types for a production agent is a lint warning.

---

#### Data privacy and security

Observability settings interact directly with `policies.pii_redaction` and `artifacts`:

- When `policies.pii_redaction: true` is set, PII redaction runs **before** any telemetry is recorded. Spans, metrics, and log records never contain raw PII, regardless of `include_model_inputs` or `include_tool_args` settings.
- `include_model_inputs: false` (the default) ensures raw user messages are never written to spans even when `pii_redaction` is `false`. This is the safest default; enable it only when debugging requires it and never in combination with `pii_redaction: false` in production.
- `include_tool_args: "redacted"` replaces known-sensitive argument fields (those annotated as `sensitive: true` in the tool definition) with `[REDACTED]` before the span is exported, while preserving non-sensitive fields for tracing purposes. Use `full` only in isolated development environments.

Telemetry data is subject to the same `data_residency` constraint as all other agent data. The declared OTel Collector endpoint must be hosted within the declared residency region; the compiler validates this at build time.

---

#### Relationship to `artifacts`

`observability` and `artifacts` capture different layers of agent data and serve different audiences:

| | `observability` | `artifacts` |
|---|---|---|
| **What is captured** | Structured signals: spans, metric data points, log records | Actual run content: input payloads, output responses, tool arguments and results |
| **Primary audience** | Platform engineering, SRE, AI engineers | Compliance, legal, security, customer support |
| **Format** | OTel-standardized (OTLP) | Platform-defined storage records |
| **Query surface** | Tracing UIs, metrics dashboards, log query tools | Artifact search API, compliance tooling |
| **Retention driver** | `metrics.retention_days`, OTel backend policy | `artifacts.retention_class` |
| **PII exposure risk** | Low (if defaults respected) | Higher — governed by `save_input` / `pii_redaction` |

Both should be configured for production agents. Observability answers "how did the agent perform?"; artifacts answer "what exactly happened in this specific run?"

Observability settings must respect the `data_residency` and `pii_redaction` policies. Omitting observability entirely for a production agent with side effects is a lint warning.

---

### `evaluation` — quality criteria (optional)

The `evaluation` field is **advisory metadata only**. The platform does not enforce it, gate deployment on it, or trigger alerts from it. Its purpose is to record, alongside the agent definition, what quality criteria the team considers meaningful — so that reviewers, auditors, and tooling can surface them without digging through separate documents.

Two sub-keys are supported:

- **`offline`** — criteria applied against a held-out dataset before deployment (batch evaluation). Typical entries include the dataset name and the metrics the team tracks, such as schema validity rate, escalation precision, or factuality scores.
- **`online`** — criteria monitored against live traffic after deployment. Typical entries include monitor names such as tool error rate or human escalation rate.

Neither dataset names, metric names, nor monitor names are validated by the compiler — there is no registry for them. Record whatever labels are meaningful to your team. Because this field carries no runtime semantics, it may be omitted without consequence.

```yaml
evaluation:
  offline:
    datasets: ["support-gold-v2"]       # Name of the held-out evaluation dataset
    metrics:
      - "schema_valid_rate"             # % of responses matching output.schema
      - "escalation_precision"          # % of escalations that were warranted
      - "factuality"                    # % of factual claims grounded in retrieved KB content
  online:
    monitors:
      - "tool_error_rate"               # % of tool calls that return an error
      - "human_escalation_rate"         # % of runs routed to a human queue
```

---

### `routing` — discovery and dispatch (optional)

```yaml
routing:
  intents: ["support_request", "billing_question", "password_reset"]
  trigger_phrases: ["help", "support", "billing", "reset password"]
  priority: 50
  fallback_agents: ["general-assistant"]
  tool_description: >
    Use this agent for any customer support request related to products,
    billing, or account access. Do not use for general knowledge questions.
```

The `routing` block serves two distinct consumers:

1. **External dispatch systems** — automated routers that decide which agent to invoke for an incoming request. They use `intents`, `trigger_phrases`, and `priority` to score and select agents.
2. **Calling agents (agent-as-tool)** — when `orchestration.can_be_invoked_as_tool: true`, the compiler synthesizes a tool manifest entry for this agent. The `routing` fields directly shape that manifest entry so that calling agents know *when* to invoke this agent, not just *how*.

#### Field reference

| Field | Type | Description |
|---|---|---|
| `intents` | string[] | Canonical intent labels this agent handles. Used by dispatch systems for intent-based routing. When the agent is exposed as a tool, these are appended to the tool description as a machine-readable hint for the calling LLM. |
| `trigger_phrases` | string[] | Representative natural-language phrases that should route to this agent. Used by keyword or embedding-based dispatchers. Also surfaced in the tool manifest as `examples` to guide the calling LLM. |
| `priority` | int (0–100) | Relative priority when multiple agents match the same intent. Higher values win. Default: `50`. Used by dispatch systems only; ignored in tool manifests. |
| `fallback_agents` | string[] | Ordered list of `agent_id` values to try if this agent is unavailable or returns a non-recoverable error. Consumed by dispatch systems only; not surfaced in tool manifests. |
| `tool_description` | string | Explicit description injected into the tool manifest when this agent is compiled as a tool. **Overrides** the default description derived from `meta.description` + `intents`. Use this when the dispatch-oriented `meta.description` is too broad or too user-facing to serve as good tool selection guidance for a calling LLM. Optional; if absent, the compiler uses `meta.description`. |

#### Tool manifest compilation rules

When `orchestration.can_be_invoked_as_tool: true`, the compiler emits a tool manifest entry for this agent. The fields are populated as follows:

| Tool manifest field | Source |
|---|---|
| `name` | `agent_id` |
| `description` | `routing.tool_description` if present, otherwise `meta.description` |
| `intents` | `routing.intents` (appended to description as a structured hint) |
| `examples` | `routing.trigger_phrases` |
| `parameters` | Derived from the agent's `input` schema |
| `returns` | Derived from the agent's `output` schema |

Routing metadata must not silently change policy or runtime behavior. Fields in this block are purely descriptive and operational — they do not affect what the agent does once invoked.

---

### `release` — publishing metadata (optional)

```yaml
release:
  channel: "stable"              # stable | beta | internal
  replaces_version: "0.9.0"
  approvals_required:
    - "product-owner"
    - "ai-safety"
  changelog:
    - version: "1.2.0"
      date: "2026-03-10"
      notes: >-
        Added explicit rule prohibiting speculation about unreleased features
        following a customer complaint. Expanded escalation conditions to
        include repeated billing disputes.
    - version: "1.1.0"
      date: "2026-01-22"
      notes: >-
        Extended system behavior with classification logic (product / billing /
        account). Added three new examples covering edge cases from production review.
    - version: "1.0.0"
      date: "2025-11-15"
      notes: "Initial release."
```

Release metadata is used by publishing systems, not by the model. The whole section is informative and not enforced by the AML.

`changelog` is a list of version entries in reverse chronological order. Each entry has three fields:

| Field | Description |
|---|---|
| `version` | The semantic version this entry describes. Should match a previously published `version` value for this `agent_id`. |
| `date` | ISO 8601 date of publication. |
| `notes` | Free-text description of what changed. Record behavioral changes, prompt tuning decisions, and the reasoning behind rule additions or removals — this history is valuable for auditing agents that handle regulated content. |

The changelog is preserved by the compiler for editorial tooling and stripped from compiled runtime payloads.

---

### `notes` — editorial (optional)

```yaml
notes:
  editor_notes: "Escalation wording still under review with legal."
  migration_notes: "Remove legacy FAQ tool ref in v1.1."
  operational: |
    The account-lookup tool has a known latency spike (up to 8 s) between
    08:00–09:00 UTC due to a scheduled sync job. Retries once on timeout — expected.

    The product-docs KB is refreshed every Tuesday at 02:00 UTC. If a customer
    references an article that does not appear in retrieval, check the KB admin
    panel before filing a bug.

    On-call: #cx-agent-oncall (Slack). Escalate to the platform team if a
    guardrail blocks more than 5% of runs (possible PII classifier false-positive spike).

    Do not test with real account IDs in staging — use acct-test-00001 through
    acct-test-09999.
```

The compiler must preserve `notes` for editorial tooling and strip it from compiled runtime payloads. The runtime must ignore it.

| Field | Description |
|---|---|
| `editor_notes` | In-progress editorial remarks — wording under review, open questions, decisions pending sign-off. |
| `migration_notes` | Instructions for the next version: references to remove, fields to rename, deprecated tools to replace. |
| `operational` | Runbook-like notes for the team operating the agent: known edge cases, dependency quirks, on-call escalation paths, staging constraints, and anything a new team member would need to operate the agent safely. Free-form text; use plain line breaks to separate items. |

---

## Markdown body sections

The Markdown body follows the closing `---` of the YAML front matter. It contains the prose that is compiled into the model's instruction set. The body is not YAML — it is free-form Markdown with section headings that the compiler recognises by name.

The compiler validates that the three required sections are present. Optional sections are included verbatim in the compiled payload when present; the runtime does not strip or reorder them.

| Section | Required | Purpose |
|---|---|---|
| `# Purpose` | Yes | One short paragraph describing what the agent is for |
| `# System behavior` | Yes | Core instruction block for the model |
| `# Rules` | Yes | Non-negotiable rules written for authors and reviewers |
| `# Escalation` | Optional | When to refuse, hand off, or request human review |
| `# Examples` | Optional | Few-shot examples or representative input/output pairs |
| `# Non-goals` | Optional | What the agent explicitly does not do |
| `# Output style notes` | Optional | Human-facing rendering guidance |

---

### `# Purpose`

One short paragraph — typically two to four sentences — that describes what the agent is for, who uses it, and in what context. Write it for the model: it sets the frame of reference for everything that follows.

Keep it factual and specific. Vague purpose statements ("be a helpful assistant") produce vague behavior. A tightly scoped purpose statement helps the model stay on task and self-reject requests that fall outside its role.

**Example — customer support agent:**
```markdown
# Purpose
You are the first-line customer support agent for Acme Cloud Storage. You help customers resolve product issues, understand billing statements, and navigate account settings. You have access to the product knowledge base and the customer's account history via tools. You are not a general-purpose assistant — every response must be grounded in verified product information or account data.
```

**Example — translation agent:**
```markdown
# Purpose
You are a professional translation agent used internally by the legal team to translate contracts and regulatory filings between English, French, German, and Japanese. Translations must be precise and preserve legal terminology. You are not a creative writing or summarisation tool.
```

---

### `# System behavior`

The core instruction block for the model. This is where you describe how the agent should behave: its tone, how it decides when to call tools, how it structures its reasoning, how it handles ambiguous or incomplete input, and what a successful response looks like.

Write this section as direct instructions to the model, using imperative sentences. Be specific about the decision logic you want the model to follow — it cannot infer unstated preferences. Bullet lists and short subsection headings are acceptable when the behavior is multi-faceted.

**Example — customer support agent:**
```markdown
# System behavior
Begin every response by identifying whether the question is about (1) a product feature, (2) a billing matter, or (3) an account operation. This classification determines which tools you use and how you respond.

For product questions: search the product knowledge base first. If the answer is found, cite the relevant article. If not found, say so explicitly and suggest the customer contact support.

For billing questions: always retrieve the customer's account record before answering. Never speculate about charges — only report what the account record shows. If a discrepancy appears, acknowledge it and escalate.

For account operations (password resets, plan changes, cancellations): walk the customer through the documented self-service steps. Do not perform account operations directly — your tools are read-only for account data.

Respond conversationally. Avoid jargon. If a question is ambiguous, ask one focused clarifying question before proceeding — do not make two different assumptions and answer both.
```

**Example — translation agent:**
```markdown
# System behavior
Translate the provided text into the target language specified in the input. Preserve the document's structure, heading levels, numbered lists, and footnotes exactly.

For legal terminology, use the established term in the target language rather than a literal translation. If no established equivalent exists, retain the source-language term in italics and add a bracketed explanatory note.

Do not paraphrase, summarise, or improve the source text. If the source contains an apparent error (grammatical or factual), translate it as written and append a translator's note flagging the issue.

Produce only the translated text. Do not include meta-commentary about the translation process unless a translator's note is warranted.
```

---

### `# Rules`

Non-negotiable constraints that the agent must always respect. Unlike the narrative instructions in `# System behavior`, rules are written as short, enumerable statements — one rule per line — because they must be easy to audit. A reviewer should be able to check each rule by reading a run's artifacts, without needing to interpret intent.

Write rules for human reviewers, not only for the model. Every rule must be specific enough that a reviewer can determine unambiguously whether a given output complies.

Prefer positive statements ("always cite the source") over negative ones ("don't forget to cite") where possible.

**Example — customer support agent:**
```markdown
# Rules
- Every factual claim about a product feature must cite the knowledge base article it came from.
- Never answer a billing question without first retrieving the customer's account record via the account-lookup tool.
- Do not give legal advice, financial advice, or medical advice under any circumstances.
- Do not speculate about pricing changes, unreleased features, or internal roadmap plans.
- Do not apologise more than once per conversation. Repeated apologies do not help the customer.
- If a customer provides an account ID, verify it exists before using it in any response.
- If a tool call fails, tell the customer what you were attempting to retrieve and why you could not complete the request. Do not silently omit information.
```

**Example — translation agent:**
```markdown
# Rules
- Translate only the text explicitly provided. Do not add context, explanations, or elaborations.
- Preserve all formatting: headings, bullet lists, numbered lists, bold, italic, footnote markers.
- If the source language cannot be detected with confidence, return an error — do not guess.
- Do not translate proper nouns, brand names, or defined terms unless a glossary entry exists.
- Translator's notes must be enclosed in square brackets and prefixed with "Translator's note:".
- Never store or reference the content of one translation when processing a separate request.
```

---

### `# Escalation`

Describes the conditions under which the agent should route the conversation to a different agent or a human queue, and what information it should pass along. This section must be consistent with the `orchestration.escalation` rules declared in the YAML — the prose here is the behavioral instruction the model follows to decide *when* to escalate; the YAML rules are the runtime mechanism that executes the handoff.

Include: the conditions that trigger escalation, what the agent should say to the user before handing off, and what context should be passed to the receiving party.

**Example — customer support agent:**
```markdown
# Escalation
Escalate to the **tier-2 support queue** when any of the following is true:
- The customer explicitly asks to speak with a human agent.
- The account record shows two or more unresolved billing disputes in the past 90 days.
- A tool call returns an authorization error or a rate-limit error on the second retry.
- The customer's request requires modifying account data (changing payment method, cancelling a plan, issuing a refund).

Before escalating, tell the customer: "I'm going to connect you with a specialist who can help with this directly. Please hold — they will have the full context of our conversation."

Pass the following to the receiving agent: the customer's account ID, a one-sentence summary of the issue, and the names of any tools that were called.

Do **not** escalate for general frustration, impatient tone, or repeated questions — attempt to resolve these with patience and clarity first.
```

---

### `# Examples`

Few-shot examples that demonstrate the expected input/output pattern. The model uses these to calibrate tone, format, and decision logic — especially in edge cases. Include at least one positive example (a well-handled request) and, when relevant, a negative example (a request the agent should decline or escalate).

Each example must faithfully represent a case the agent will encounter in production. Fabricated or overly simple examples are worse than none — they can mislead the model about what the real distribution looks like.

**Example — customer support agent:**
```markdown
# Examples

**Example 1 — product question with KB citation**

User: "Does the Pro plan include version history?"

Agent: "Yes, the Pro plan includes 180 days of version history for all file types. You can access previous versions from the file's context menu under *Version History*. (Source: [Help article #1042 — Version history on the Pro plan])"

---

**Example 2 — billing question requiring account lookup**

User: "I was charged $49 last Tuesday but I'm on the $29 plan."

Agent: [calls account-lookup tool]

"I can see your account is on the Pro Monthly plan at $29/month. The $49 charge on April 8 was for an add-on storage expansion of 500 GB that was activated on April 7. If you did not authorise this, I can escalate this to our billing team for a review and potential refund."

---

**Example 3 — out-of-scope request (decline)**

User: "Can you help me draft a letter threatening legal action against Acme?"

Agent: "That's outside what I can help with — I'm here for product, billing, and account questions. For legal matters, please consult a qualified attorney. Is there anything else I can help you with today?"
```

---

### `# Non-goals`

An explicit list of what the agent does *not* do. Non-goals prevent scope drift over time and set correct expectations for users and callers. They also help the model self-reject off-topic requests confidently, without hedging.

Write non-goals as plain, assertive statements. They should be specific enough to draw a clear boundary, not so broad that they describe the entire space of things the agent does not do.

**Example — customer support agent:**
```markdown
# Non-goals
This agent does not:
- Process refunds, credits, or billing adjustments — these require tier-2 handling.
- Modify account settings, upgrade or downgrade plans, or cancel subscriptions.
- Provide legal, financial, or medical advice.
- Answer general knowledge questions unrelated to Acme products.
- Access or discuss other customers' account data.
- Make commitments about future product features or pricing changes.
```

---

### `# Output style notes`

Human-facing guidance for how the agent formats and presents its responses. This covers tone, response length, use of markdown, list formatting, and any surface-specific constraints (e.g., a chat widget that does not render markdown should receive plain text).

This section is consumed by the model, not the compiler. Keep it concise — the model should be able to follow it without extensive interpretation.

**Example — customer support agent:**
```markdown
# Output style notes
- Write in a warm, professional tone. Use plain language — avoid technical jargon unless the customer used it first.
- Keep responses under 150 words unless a step-by-step explanation is genuinely required.
- Use numbered lists for sequential steps (e.g., how to reset a password). Use bullet lists for parallel options.
- Use bold text sparingly — only for key terms or critical warnings.
- Do not use markdown headings (`#`, `##`) inside responses. The chat widget renders these as large text, which looks out of place in a conversation.
- End responses with a clear next step or a question that moves the conversation forward. Do not end with "Let me know if you have any other questions" — it is filler.
```

---

## Validation rules

### Hard validation failures (block publication)

- Missing any required top-level field.
- Unknown or unsupported `spec_version`.
- Invalid `agent_id` format, or attempted change of `agent_id` for an existing lineage.
- Invalid semantic version format or attempt to re-publish an existing `agent_id` + `version` pair.
- Invalid `status` value.
- Invalid JSON Schema in `input.schema` or `output.schema`.
- A `tools.ref` that does not resolve to a registered `.tool.md`.
- A `knowledge.ref` that does not resolve to a registered `.kb.md`.
- An agent-level override that would loosen a registry-level security constraint.
- A tool or KB with `status: disabled` referenced by an `active` agent.
- Unknown guardrail IDs, policy keys, or connector identifiers.
- Prohibited runtime values: disallowed models, temperature outside the platform-allowed range.
- Missing required Markdown body sections: `# Purpose`, `# System behavior`, `# Rules`.
- Orchestration delegates or tool confirmation references that point to undeclared tools or agents.

### Recommended lint rules (warn but allow publication)

- Description is too long (over 200 characters) or too vague.
- No input examples for an externally facing agent.
- Output schema is too loose (unrestricted `additionalProperties`).
- `knowledge` section absent for an agent that makes mutable factual claims.
- Observability omitted for a production agent with side effects.
- Temperature above 0.7 for a business-critical deterministic workflow.
- Shared memory writeback enabled without `approval_required: true`.
- A tool or KB with `status: deprecated` referenced without a migration note in `notes`.

---

## Minimal complete example

```markdown
---
spec_version: "1.2"
agent_id: "faq-agent"
version: "1.0.0"
status: "active"

meta:
  name: "FAQ Agent"
  category: "support"
  description: "Answer common product questions from the help center."
  owner: "cx-team"

runtime:
  model: "claude-4-haiku"
  temperature: 0.2
  timeout_seconds: 30
  max_input_chars: 5000
  max_output_tokens: 800

input:
  schema:
    type: object
    properties:
      question: { type: string }
    required: ["question"]

output:
  schema:
    type: object
    properties:
      answer: { type: string }
      confidence: { type: string, enum: ["high", "medium", "low"] }
    required: ["answer", "confidence"]

tools:
  - ref: "search-product-kb"
  tool_choice: "auto"

knowledge:
  - ref: "product-docs"

memory:
  mode: "session"
  writeback:
    enabled: false

artifacts:
  enabled: true
  save_input: false
  save_output: true
  retention_class: "standard"

policies:
  pii_redaction: true
  allow_external_http: false
  blocked_content: ["legal_advice", "medical_advice"]
  audit_level: "standard"

guardrails:
  input:
    - id: "pii_scan"
      mode: "detect"
      on_fail: "redact"
  output:
    - id: "schema_validation"
      on_fail: "retry"

ui:
  display_name: "FAQ Agent"
  short_description: "Quick answers to common product questions."
---

# Purpose
Answer common product questions quickly and accurately using the official help center documentation.

# System behavior
You are a helpful product support assistant for SME users.
Answer the user's question using the product documentation.
If the answer is not found in the documentation, say so clearly and suggest they contact support.
Keep answers concise — two to four sentences unless detail is genuinely needed.

# Rules
- Only answer questions that can be grounded in the product documentation.
- Do not give legal, financial, or medical advice.
- Do not speculate about unreleased features.
- If the question is ambiguous, ask one clarifying question.

# Non-goals
This agent does not handle billing disputes, refund requests, or account changes.
```
