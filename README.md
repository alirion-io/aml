# Agent Modeling Language (AML)

AML is a portable, human-readable format for describing cloud AI agents and their dependencies — tools, knowledge bases, memory collections, IAM policies, model settings, and guardrails — in plain Markdown and YAML.

The central design commitment is that **a non-developer product owner should be able to read an agent file, understand what the agent does, approve or reject its behavior, and request changes — without asking an engineer to explain it**.

## Why AML?

Existing agent definition formats are tightly coupled to specific tools (Anthropic Claude, GitHub Copilot, Cursor, etc.). AML takes a different approach:

- **Framework-agnostic** — decouples agent intent from the runtime that executes it
- **Human-readable** — authored in Markdown + YAML, reviewable by product, legal, and compliance teams
- **Governance-first** — capabilities, tool access, and guardrails are explicit, versionable, and auditable
- **Deployment-ready** — AML files are compiled, validated, and deployed through a platform pipeline

## The seven file kinds

| Kind | Extension | Purpose |
|---|---|---|
| Agent | `.agent.md` | Persona, instructions, model, tools, memory, guardrails |
| Tool | `.tool.md` | Tool contract, auth requirements, invocation policy |
| Knowledge Base | `.kb.md` | Vector store configuration and retrieval policy |
| IAM Policy | `.iam.md` | Permission ceiling for what tools an agent may call |
| Model | `.model.md` | Model ID, provider, parameter defaults |
| Memory Collection | `.collection.md` | Long-term memory store configuration |
| Guardrail | `.guardrail.md` | Pre/post-response safety and compliance filters |

## Documentation

Full specification and examples are published at **[https://alirion-io.github.io/aml/](https://alirion-io.github.io/aml/)**.

| Section | Description |
|---|---|
| [Introduction](https://alirion-io.github.io/aml/00-introduction/) | What AML is, where it sits in the agentic stack, and the seven file kinds |
| [Agent Definition](https://alirion-io.github.io/aml/01-agent-definition/) | Full specification for `.agent.md` files |
| [Tool Definition](https://alirion-io.github.io/aml/02-tool-definition/) | Full specification for `.tool.md` files |
| [Knowledge Base](https://alirion-io.github.io/aml/03-kb-definition/) | Full specification for `.kb.md` files |
| [IAM Policy](https://alirion-io.github.io/aml/04-iam-definition/) | Full specification for `.iam.md` files |
| [Model Definition](https://alirion-io.github.io/aml/05-model-definition/) | Full specification for `.model.md` files |
| [Memory Collection](https://alirion-io.github.io/aml/06-collection-definition/) | Full specification for `.collection.md` files |
| [Guardrail Definition](https://alirion-io.github.io/aml/07-guardrail-definition/) | Full specification for `.guardrail.md` files |
| [JSON Schema](https://alirion-io.github.io/aml/08-json-schema/) | Machine-readable JSON Schema for all file kinds |

## Repository structure

```
docs/              # Specification source (MkDocs)
  *.agent.md       # Example agent files
  *.tool.md        # Example tool files
  *.kb.md          # Example knowledge base files
  *.model.md       # Example model files
  *.collection.md  # Example memory collection files
  *.guardrail.md   # Example guardrail files
mkdocs.yml         # Documentation site configuration
```

## Current version

**AML v1.2**

## License

See [LICENSE](LICENSE).
