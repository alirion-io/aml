# Example — Legal Policies Knowledge Base


---

```markdown
---
spec_version: "1.2"
kb_id: "legal-policies"
version: "1.1.0"
status: "active"

meta:
  name: "Legal Policies"
  description: >
    Internal summaries of company legal policies, terms of service, privacy policy,
    acceptable use policy, and data processing agreements. Used by compliance and
    support agents to provide factual, policy-grounded answers to legal policy questions.
    Agents must not interpret or apply this content as legal advice.
  owner: "legal-team"
  tags: ["legal", "compliance", "policy", "privacy", "terms"]
  last_updated: "2026-04-15"

source:
  type: "document_set"
  uri: "kb://legal-policies-internal"
  freshness_window_days: 30
  index_refresh: "every-24h"

scope:
  domains:
    - "legal_policy"
    - "privacy"
    - "compliance"
    - "terms_of_service"
    - "data_processing"
  include:
    - "terms of service summaries"
    - "privacy policy summaries"
    - "acceptable use policy"
    - "data processing agreement summaries"
    - "GDPR and CCPA compliance FAQs"
    - "data retention policy summaries"
  exclude:
    - "legal opinions or interpretations"
    - "case-specific legal advice"
    - "draft or unapproved policy documents"
    - "internal legal strategy documents"
    - "litigation materials"
  languages: ["en"]
  version_coverage: "current"

classification:
  trust_level: "authoritative"
  data_classification:
    contains_pii: false
    compliance: ["GDPR-Art30", "CCPA"]
    confidentiality: "confidential"

retrieval_defaults:
  search_mode: "hybrid"
  max_chunks: 3
  max_tokens: 1200
  result_granularity: "section"
  reranking: true
  citation_preference: "required"
  deduplicate: true
  min_score: 0.70

retrieval:
  use_when:
    - "question asks about the company's terms of service or acceptable use policy"
    - "question relates to privacy policy, data collection, or data deletion rights"
    - "question involves GDPR or CCPA rights (access, deletion, portability)"
    - "question asks whether a specific activity is permitted under the AUP"
  avoid_when:
    - "question requires legal interpretation or case-specific legal advice"
    - "question is outside company policy scope"
    - "the specific policy question is not covered in the current document set"

freshness:
  window_days: 30
  on_stale: "refuse"
  last_verified: "2026-04-15"
---

# Purpose
Provides support and compliance agents with factual, policy-grounded answers to
questions about company legal policies. Content is sourced from approved policy
documents and must be treated as authoritative factual summaries, not legal advice.

# Content coverage
Terms of Service, Privacy Policy, Data Processing Agreement, GDPR/CCPA FAQ,
and Data Retention Policy.

# Notes
This is a restricted KB. Access is controlled at the IAM role level — only roles that explicitly grant access to this KB may reference it.
Contact legal@company.com for access requests or policy corrections.
Agents using this KB must include a disclaimer that responses are policy summaries,
not legal advice.
```
