# Changelog

All notable changes to the AML specification are documented here.

This project follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.2.1] - 2026-05-01

> Introduced machine-readable JSON Schemas for all AML artifact kinds, alongside a reorganized `docs/examples/` tree and a broad documentation refresh across all core spec pages.

### Added
- **JSON Schemas for AML artifacts**: introduced machine-readable schemas in `schemas/` for agents, tools, knowledge bases, IAM roles, models, collections, and guardrails.
- **Expanded documentation examples**: added a full `docs/examples/` tree with concrete examples for each AML artifact kind.

### Changed
- **Documentation refresh across core specs**: revised `00-introduction.md` through `08-json-schema.md` to align terminology, structure, and authoring guidance.
- **MkDocs configuration update**: refreshed `mkdocs.yml` to reflect the updated documentation organization and examples.

### Planned
- Quickstart guide for authoring a first agent end-to-end

---

## [1.2.0] — 2026-04-16

### Added
- **Memory Collection definition** (`06-collection-definition.md`): new `collection.md` file kind for long-term memory stores. Defines `store_type`, `namespace`, `ttl_days`, `embedding_model`, and retention policy fields.
- **Guardrail definition** (`07-guardrail-definition.md`): new `guardrail.md` file kind for pre/post-response safety and compliance filters. Supports AWS Bedrock Guardrails and custom filter chains.
- **Model definition** (`05-model-definition.md`): new `model.md` file kind to decouple model configuration from agent files. Agents now reference a model by ID rather than embedding parameters inline.
- **`08-json-schema.md`**: describes how inline JSON Schema is expressed in YAML within AML tool and agent files.
- Example files for all new file kinds: `customer-preferences.collection.md`, `bedrock-pii-scan.guardrail.md`, `claude-4-sonnet.model.md`, `gpt-4o.model.md`.

### Changed
- Agent definition updated to reference `model`, `memory`, and `guardrails` by ID instead of inline blocks.
- `tools[].allowed_actions` field renamed to `tools[].permissions` for consistency with IAM definition language.

### Fixed
- Corrected `tool.output.schema` example in `02-tool-definition.md` (was missing required `type` field at root level).

---

## [1.1.0] — 2026-01-14

### Added
- **IAM Policy definition** (`04-iam-definition.md`): new `iam.md` file kind that defines the permission ceiling for an agent's tool calls. Agents reference an IAM role by ID; the compiler enforces that tool references are within the role's allowed actions.
- **Knowledge Base definition** (`03-kb-definition.md`): new `kb.md` file kind for vector store configuration and retrieval policy.
- Example files: `brand-guidelines.kb.md`, `legal-policies.kb.md`.

### Changed
- Agent definition `knowledge_bases` field changed from inline configuration to a list of `kb.md` file references.

---

## [1.0.0] — 2026-04-13

### Added
- Initial specification release.
- **Agent definition** (`01-agent-definition.md`): persona, behavioral instructions, model settings, input/output contracts, tool references.
- **Tool definition** (`02-tool-definition.md`): tool contract, authentication requirements, invocation policy, input/output JSON Schema.
- Example files: `support-agent.agent.md`, `translator.agent.md`, `send-email.tool.md`, `glossary-lookup.tool.md`.
- MkDocs Material site with dark/light mode, search, and PDF export.

---

[Unreleased]: https://github.com/alirion-io/aml/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/alirion-io/aml/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/alirion-io/aml/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/alirion-io/aml/releases/tag/v1.0.0
