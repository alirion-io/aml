# Example — Brand Guidelines Knowledge Base


---

```markdown
---
spec_version: "1.2"
kb_id: "brand-guidelines"
version: "3.0.0"
status: "active"

meta:
  name: "Brand Guidelines"
  description: >
    Brand and style guidance covering tone of voice, naming conventions, capitalization
    rules, communication style, and visual language descriptions. Used by translation,
    marketing, and content agents to ensure all generated output is consistent with
    company brand standards.
  owner: "brand-team"
  tags: ["brand", "style", "tone", "voice", "naming"]
  last_updated: "2026-04-15"

source:
  type: "document_set"
  uri: "kb://brand-guidelines-v3"
  freshness_window_days: 90
  index_refresh: "every-7d"

scope:
  domains:
    - "brand_voice"
    - "style"
    - "tone"
    - "naming"
    - "capitalization"
  include:
    - "tone of voice guidance (formal, friendly, technical registers)"
    - "naming and capitalization conventions for products, features, and teams"
    - "preferred and forbidden vocabulary"
    - "punctuation and formatting style rules"
    - "communication style for different audience types"
  exclude:
    - "visual design specifications (colors, fonts, spacing)"
    - "product troubleshooting or technical documentation"
    - "legal interpretation or compliance requirements"
    - "pricing or commercial terms"
  languages: ["en", "fr", "de", "ja", "es"]
  version_coverage: "current"

classification:
  trust_level: "authoritative"
  data_classification:
    contains_pii: false
    compliance: []
    confidentiality: "internal"

retrieval_defaults:
  search_mode: "semantic"
  max_chunks: 4
  max_tokens: 1200
  result_granularity: "section"
  reranking: false
  citation_preference: "optional"
  deduplicate: true
  min_score: 0.60

retrieval:
  use_when:
    - "translation or content must follow brand tone of voice"
    - "naming or capitalization conventions are relevant to the output"
    - "preferred or forbidden vocabulary may apply"
    - "content is intended for external customer communications"
  avoid_when:
    - "output is purely technical or developer-facing with no brand language"
    - "question is purely factual with no communication style dimension"
    - "content is an internal draft not intended for publication"

freshness:
  window_days: 90
  on_stale: "warn"
  last_verified: "2026-04-15"
---

# Purpose
The authoritative brand guidelines knowledge base. All agents generating external-facing
communications should retrieve from this source when brand voice, naming, or style
consistency is relevant.

# Content coverage
- Part 1: Tone of voice — formal, friendly, and technical registers with examples.
- Part 2: Naming conventions — product names, feature names, team names, capitalization rules.
- Part 3: Vocabulary — preferred terms, forbidden terms, and alternatives.
- Part 4: Communication style — audience-specific guidance for customers, developers, and partners.

# Notes
Updated quarterly by the brand team. Contact brand@company.com or #brand-guidelines
in Slack for correction requests. The current version is v3 (January 2025).
Legacy v2 guidelines are archived and no longer authoritative.
```
