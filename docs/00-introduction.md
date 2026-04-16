# Introduction

> **Current version**: 1.2

> **Audience**: Product owners, non-developer agent authors, platform architects, AI governance teams

---

## What is Agent Modeling Language?

Agent Modeling Language (AML) is a file-based format for describing AI agents and their dependencies — tools, knowledge bases, memory collections, guardrails, and orchestration rules — in plain text that both humans and machines can read.

An author writes one Markdown file per agent. A platform compiler reads that file, validates it against the spec, resolves all external references, and produces an immutable compiled payload that the runtime executes. Authors never write code; the runtime never executes raw authored text.

The central design commitment is that **a non-developer product owner should be able to read an agent file, understand what the agent does, approve or reject its behavior, and request changes — without asking an engineer to explain it**.

---

## Why a markup-based format?

Most production agent frameworks today require engineers to write Python, TypeScript, or JSON configuration. This creates a hard dependency: every time a business stakeholder wants to change an agent's persona, restrict a tool, or update escalation rules, an engineer must be involved.

AML breaks that dependency. The authoring surface is text — specifically YAML front matter for structured machine-readable fields and Markdown prose for behavioral instructions. These are formats that product managers, legal reviewers, compliance officers, and domain experts already use every day. The platform handles compilation, validation, and deployment.

This is not a new idea in software. Infrastructure-as-code tools like Terraform and Kubernetes manifests proved that declarative text files, version-controlled in Git and compiled by automated pipelines, are a more reliable and auditable way to manage infrastructure than manual configuration through UIs. AML applies the same principle to AI agent behavior.

---

## The agentic stack and where AML sits

A production agentic system has several layers. AML operates at the **definition layer** — the layer that describes intent and capability — sitting above runtime infrastructure and below the model itself.

```
┌─────────────────────────────────────────────────┐
│  Business logic & user interface                │  Product, UX
├─────────────────────────────────────────────────┤
│  Agent definition layer  ◄── AML lives here     │  AML files (.agent.md, .tool.md, .kb.md, .collection.md, .iam.md, .model.md, .guardrail.md)
├─────────────────────────────────────────────────┤
│  Compiler & registry                            │  Validates, resolves refs, emits compiled JSON
├─────────────────────────────────────────────────┤
│  Runtime orchestration                          │  Executes compiled payloads; calls model + tools
├─────────────────────────────────────────────────┤
│  Model inference                                │  LLM provider (OpenAI, Anthropic, Google, etc.)
├─────────────────────────────────────────────────┤
│  Tool & knowledge infrastructure                │  APIs, vector stores, databases, queues
└─────────────────────────────────────────────────┘
```

AML intentionally does not prescribe what happens below the compiler. The runtime, model provider, vector store, and tool infrastructure are platform choices. AML describes the intent and constraints; the platform decides how to fulfil them.

---

## Seven file kinds

AML v1.2 defines seven distinct file kinds that together describe a complete agentic capability:

**Agent definition** (`agents/<id>.agent.md`)
Describes one agent: its purpose, behavioral instructions, model settings, input and output contracts, memory policy, guardrails, governance rules, and references to the tools and knowledge bases it is permitted to use. The agent references an IAM role by ID, which defines the ceiling of what tools it may call. This is the file a product owner authors and approves.

**Tool definition** (`tools/<id>.tool.md`)
Describes one registered tool: what it does, how to call it, its authentication requirements, invocation policy, and guidance for when the agent should and should not use it. Tool files are authored by engineers and approved by the platform team. Agent files reference tools by ID.

**Knowledge base definition** (`knowledge/<id>.kb.md`)
Describes one approved knowledge source: its content scope, storage type, retrieval defaults, freshness policy, and trust level. KB files are authored by knowledge owners and approved by the platform team. Agent files reference knowledge bases by ID.

**IAM role definition** (`iam/<id>.iam.md`)
Describes a named execution role that an agent assumes at runtime — what tools the agent is permitted to call and with what constraints. This maps directly to the cloud execution role concept (e.g., an AWS Lambda execution role). IAM files are the single authoritative ceiling for an agent's tool access. Who can *call* an agent is not declared in AML; that is a cloud infrastructure concern managed outside these files. IAM role files are authored and owned by the platform security team.

**Memory collection definition** (`collections/<id>.collection.md`)
Describes one registered memory collection: its storage backend (e.g., Amazon Bedrock AgentCore Memory, Valkey/Redis, or a custom endpoint), lifetime scope (session, project, or tenant), retrieval configuration, and writeback policy. Collection files are authored and maintained by platform engineers. Agent files reference collections by ID in their `memory.read_collections` and `memory.write_collections` fields. The compiler resolves the reference and injects the backend configuration — including credentials references and namespace patterns — into the compiled payload. This means agent authors never write connection strings or function ARNs; they only declare which named collection the agent may use.

**Model definition** (`models/<id>.model.md`)
Maps a stable, human-readable model identifier (e.g., `claude-4-sonnet`) to a specific provider, provider-level model name, and connection configuration. Agent files reference models by their `model_id`; the compiler resolves the reference and injects the correct provider configuration into the compiled payload. Model files are authored and maintained by the platform ML team. This indirection decouples agent authors from provider-specific naming conventions and lets the platform team update model versions or swap providers without touching any agent definition.

**Guardrail definition** (`guardrails/<id>.guardrail.md`)
Describes one named runtime check: its provider (e.g., AWS Bedrock Guardrails, Azure AI Content Safety, or a platform-native check), invocation details, failure policy, and fallback behavior. Agent files reference guardrails by ID in their `guardrails.input`, `guardrails.output`, and `guardrails.tool_calls` sections. The compiler resolves the reference and injects the full provider configuration — including credentials references, region, version pinning, and fallback guardrail — into the compiled payload. This means agent authors never write provider ARNs, version numbers, or credentials; they only declare which named guardrail the agent uses. Guardrail definition files are authored and maintained by the AI safety team. See the [Guardrail Definition specification](07-guardrail-definition.md) for the full field reference.

This separation and being platform/framework agnostic are the core innovation of AML. 

---

## Key design principles

**Human-authored, machine-enforced.** Authors write text; the platform enforces policy. An agent's behavior is shaped by both its prose instructions (which guide the model) and its structured YAML fields (which are compiled into hard runtime constraints). Prompt text alone is never a sufficient security or compliance control.

**Registry-first.** Tools and knowledge bases that are used by more than one agent live in the shared registry and are referenced by ID. Inline definitions are not permitted to avoid having discrepancies between agents.

**Least-privilege references.** An agent can reference a shared tool but restrict it further — requiring approval, limiting call counts, or narrowing scope. Agents cannot loosen constraints set in the registry.

**Explicit capabilities.** An agent may only use the tools and knowledge sources explicitly declared in its definition. The runtime must reject tool calls or knowledge lookups that are not in the compiled payload. There is no implicit ambient capability.

**Policy outside the model.** Guardrails, PII handling, content blocking, and approval requirements are machine-enforced at the runtime layer. They are declared in YAML, not in prompt text. A prompt instruction like "never reveal PII" is editorial guidance, not an enforceable control.

**Structured outputs by default.** Every production agent should declare a typed output schema. The runtime validates every response against it. An output that fails schema validation is a run error, not a silent malformation.

**Operational readiness as a first-class concern.** Observability, evaluation, release, and testing sections are part of the spec, not afterthoughts. An agent definition that lacks tracing, evaluation criteria, and a release channel is incomplete.

---

## Alignment with existing frameworks

AML does not invent new concepts. It synthesises the best patterns from established agent frameworks into a human-editable, version-controllable, compiler-validated format. The table below maps AML concepts to their counterparts in the major frameworks.

| AML concept | MCP | OpenAI Agents SDK | LangChain | Google ADK | AutoGen | AWS Bedrock | Azure AI Foundry |
|---|---|---|---|---|---|---|---|
| Tool registry (`.tool.md`) | MCP tool server | Tool registered at assistant/org level | ToolKit | Tool declaration | Tool registration | Action group | Tool component |
| Knowledge base registry (`.kb.md`) | MCP resource | File search, vector store | Retriever | Data store | Memory protocol | Knowledge base | Grounding data source |
| Memory collection (`.collection.md`) | — | — | Memory store config | Session service config | Memory protocol | AgentCore Memory resource | — |
| Agent definition (`.agent.md`) | — (MCP is tool-side) | Assistant configuration | Chain / Agent config | Agent definition | Agent config | Agent blueprint | AI project config |
| Model definition (`.model.md`) | — | Model object / provider config | LLM config | Model config | Model client config | Bedrock model access + foundation model ID | AI model deployment |
| Compiler + registry validation | Capability negotiation | Schema validation | Runnable validation | Agent graph build | — | CloudFormation / CDK | ARM / Bicep template |
| Guardrails as machine-enforced policy | Security boundary enforcement | Input / output guardrails | Middleware callbacks | Safety checks | — | Bedrock Guardrails | Content filters |
| Compiled immutable payload | Server manifest | Published assistant snapshot | Compiled chain | Deployed agent | — | Deployed agent version | Deployed AI project |
| `spec_version` + `version` | Protocol version negotiation | API versioning | — | — | — | Agent version | — |

### What AML improves over each framework

**Over MCP**: MCP defines the tool-side protocol excellently but does not specify how the agent side should declare which tools it uses, under what policy, or with what behavioral instructions. AML fills that gap.

**Over OpenAI Agents SDK**: The SDK requires Python and is code-first. Non-developer stakeholders cannot read, approve, or edit agent configurations. AML makes the definition layer accessible to the full team.

**Over LangChain**: LangChain chains are powerful but agent configurations are often scattered across constructor arguments, environment variables, and prompt templates in multiple files. AML consolidates everything into one canonical file per agent.

**Over Google ADK**: ADK provides excellent session and state separation, and AML borrows those patterns directly. AML adds the governance, release, evaluation, and observability sections that ADK leaves to the platform.

**Over AutoGen**: AutoGen treats memory and tools as explicit runtime protocols, which AML reflects. AML adds the authoring-time governance layer — approval workflows, release channels, compliance policies — that AutoGen does not specify.

**Over AWS Bedrock Agents**: Bedrock's action groups and knowledge bases are configured through the console or CloudFormation, requiring engineering or DevOps involvement for every change. AML enables product owners to author and approve agent behavior in text, with the platform handling deployment.

**Over Azure AI Foundry**: Foundry provides good project-level resource management. AML adds the behavioral layer — prose instructions, rules, escalation paths, evaluation criteria — in a format that non-engineers can read and approve.

---

## The repository structure

A team using AML typically maintains a Git repository with this structure:

```
agents/
  translator.agent.md
  support-agent.agent.md
  refund-agent.agent.md

tools/
  glossary-lookup.tool.md
  search-product-kb.tool.md
  send-email.tool.md
  create-ticket.tool.md

knowledge/
  brand-guidelines.kb.md
  terminology-kb.kb.md
  product-docs.kb.md
  legal-policies.kb.md

collections/
  customer-preferences.collection.md
  session-state-cache.collection.md

iam/
  support-agent-role.iam.md
  translator-agent-role.iam.md

models/
  claude-4-sonnet.model.md
  gpt-4o.model.md

guardrails/
  pii-scan.guardrail.md
  prompt-injection-scan.guardrail.md
  unsafe-content-check.guardrail.md
  schema-validation.guardrail.md
```

Each file is independently versioned, reviewable via pull request, and approvable by the relevant stakeholder. A product owner approves agent files. An engineer approves tool files. A knowledge owner approves KB files. A platform engineer approves collection files. A security team member approves IAM files. A ML team member approves model files. An AI safety team member approves guardrail files. The CI pipeline validates every file on every commit and produces a dependency report showing which agents would be affected by a proposed change to a shared tool, KB, collection, model, or guardrail.

---

## The compilation and publishing pipeline

Authors never run the runtime directly. The lifecycle is:

1. Author edits a `.agent.md`, `.tool.md`, `.kb.md`, `.collection.md`, `.model.md`, `.iam.md`, or `.guardrail.md` file.
2. CI runs schema validation and lint checks on the changed file.
3. For tool, KB, collection, IAM, model, or guardrail changes, CI runs cross-impact analysis: which agents reference this resource?
4. The compiler resolves all `ref:` references, merges registry definitions with agent-level overrides, and emits a compiled JSON payload.
5. An approver reviews the compiled output and the impact report.
6. The platform publishes an immutable versioned payload, sealed with a spec hash.
7. The runtime pulls only published, compiled, version-pinned payloads. It never executes raw Markdown.

Production runs always reference a specific `agent_id` and `version`. There are no mutable "latest" references in production.

---

## Versioning philosophy

AML uses two version numbers that must not be confused.

The `spec_version` field identifies the AML format version — currently `"1.2"`. It is controlled by the platform team. The runtime rejects any payload compiled from an unknown spec version.

The `version` field on each file is the semantic version of that specific definition. It follows standard semver conventions: patch for wording fixes and examples, minor for backward-compatible behavior changes, major for contract-breaking changes to input/output schemas or tool access. Every published change must increment the version; the platform must reject a published payload whose version matches an already-published version for that ID.

---

## A note on what AML is not

AML is a **definition** format, not an **execution** format. It does not describe algorithms, control flow, or code. It describes intent, constraints, capabilities, and policies. The runtime decides how to fulfill the definition — which model to call, how to schedule tool invocations, how to manage session state. AML gives the runtime enough information to make those decisions safely and consistently.

AML is also not a **conversation** format. It does not record conversations, store user data, or log model outputs. Those concerns belong to the runtime's observability and storage layers. AML only describes the configuration of the agent — what it is allowed to do, and how it should behave.
