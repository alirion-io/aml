# Example — Glossary Lookup Tool

---

```markdown
---
spec_version: "1.2"
tool_id: "glossary-lookup"
version: "1.2.0"
status: "active"

meta:
  name: "Glossary Lookup"
  description: >
    Look up approved business terminology, domain-specific vocabulary, and preferred
    translations for known source and target language pairs. Use when the source text
    contains specialized business language, product terminology, or repeated terms
    that must remain consistent. Do not use for general grammar correction or
    style rewriting unrelated to terminology.
  owner: "linguistics-team"
  tags: ["terminology", "glossary", "translation", "retrieval"]
  last_updated: "2026-04-15"

type: "retrieval"

parameters:
  type: object
  properties:
    query:
      type: string
      description: "Term, phrase, or short expression to look up."
    source_language:
      type: string
      description: "Optional. ISO 639-1 language code of the source term (e.g. 'en', 'de')."
    target_language:
      type: string
      description: "Optional. ISO 639-1 language code of the desired translation."
    domain:
      type: string
      description: "Optional. Domain hint to narrow the lookup (e.g. 'finance', 'product', 'legal')."
  required: ["query"]

transport:
  type: "rest-api"
  base_url: "https://glossary.internal.example.com/v1"
  endpoint: "GET /lookup"
  timeout_ms: 6000
  retry_policy:
    max_attempts: 2
    on_status: [503, 504]
  auth:
    scheme: "service-account"
    token_source: "secrets:prod/agent/glossary-token"

use_guidance:
  use_when:
    - "source text includes technical, regulated, branded, or domain-specific terminology"
    - "terminology consistency across a document or project is important"
    - "a term may have multiple valid translations and the approved one should be used"
  avoid_when:
    - "the request is simple conversational translation with no specialized terms"
    - "the term is a common everyday word with an obvious translation"
    - "the query is a full sentence rather than a term or phrase"
  side_effects: "None. Read-only."

response_schema:
  type: object
  properties:
    entries:
      type: array
      description: "Matching glossary entries."
      items:
        type: object
        properties:
          term:
            type: string
            description: "The source term as stored in the glossary."
          translation:
            type: string
            description: "Approved translation of the term."
          domain:
            type: string
            description: "Domain the term belongs to, e.g. 'finance', 'product'."
          approved:
            type: boolean
            description: "True if this is an approved entry; false if under review."
          notes:
            type: string
            description: "Optional usage notes from the linguistics team."
          source:
            type: string
            description: "Origin of the entry, e.g. the glossary version or contributing team."
        required: ["term", "translation", "approved"]
    total:
      type: integer
      description: "Total number of matching entries."
  required: ["entries", "total"]

error_codes:
  400:
    meaning: "Missing or malformed query parameter"
    agent_action: "Fix the query. Do not retry."
  404:
    meaning: "No matching entries found"
    agent_action: "Proceed with best-effort translation. Note the missing glossary entry in output notes."
  429:
    meaning: "Rate limited"
    agent_action: "Wait 5 seconds, retry once."
  503:
    meaning: "Glossary service unavailable"
    agent_action: "Retry with backoff. If unavailable after 2 attempts, proceed without glossary lookup and note it."
---

# Purpose
Provides agents with access to the company's curated terminology glossary for consistent
use of approved business and product terminology across translations and communications.

# Notes
- The glossary is maintained by the linguistics team. Submit term requests via #linguistics.
- `approved: true` entries must be used; `approved: false` entries are candidates under review.
- A 404 response means the term is not yet in the glossary — proceed with best judgment
  and flag the gap in the output notes so it can be added.
- Changelog:
  - 1.2.0 — Added `domain` filter parameter and `source` field in response.
  - 1.1.0 — Added `approved` flag to distinguish approved from candidate entries.
  - 1.0.0 — Initial release.
```
