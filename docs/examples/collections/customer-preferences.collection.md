---
spec_version: "1.2"
collection_id: "customer-preferences"
version: "1.0.0"
status: "active"

meta:
  name: "Customer Preferences"
  description: >
    Stores and retrieves per-user preferences learned across sessions,
    such as preferred language, communication style, and product interests.
    Used by support and personalization agents to tailor responses without
    requiring users to repeat their preferences on each interaction.
  owner: "cx-platform-team"
  tags: ["preferences", "customer", "personalization"]
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
    - type: "semanticMemoryStrategy"
      name: "FactExtractor"
      namespace: "/facts/{actorId}"
      retrieval:
        top_k: 5
        relevance_score: 0.6

writeback:
  enabled: true
  strategy: "user-preference"
---

# Purpose

Provides persistent user preference memory for support and personalization agents. Preferences and facts about the user are stored per actor (user identity) and retrieved automatically at run start. Agents can immediately tailor tone, language, and product recommendations without requiring the user to re-state context at the beginning of each session.

# Content coverage

Includes explicitly stated preferences ("I prefer email over phone"), implicitly inferred preferences (language choice, formality level), and factual self-disclosures (account type, product in use). Does not store conversation transcripts — use a `session-state-cache` collection for that.

# Notes

This collection contains PII (actor identifiers mapped to preference data) and is subject to GDPR right-to-erasure requirements. The platform's memory service must support per-actor data deletion triggered by a data subject request.

Access rights are controlled by the IAM role assigned to each agent that references this collection. `approval_required: true` is set because this is a project-scoped collection shared across all sessions for a given user — a human approval step prevents a misbehaving or adversarially prompted agent from silently overwriting learned preferences with incorrect data.
