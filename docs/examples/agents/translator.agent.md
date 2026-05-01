# Example — Translator Agent

---

```markdown
---
spec_version: "1.2"
agent_id: "translator"
version: "1.1.0"
status: "active"

meta:
  name: "Translator"
  category: "language"
  description: "Translate business text between supported languages while preserving meaning, tone, and terminology."
  owner: "product-ai"
  tags: ["translation", "language", "business", "localization"]

runtime:
  model: "claude-4-sonnet"
  fallback_model: "claude-4-haiku"
  reasoning_effort: "medium"
  temperature: 0.2
  timeout_seconds: 60
  max_input_chars: 20000
  max_output_tokens: 2000
  max_turns: 4
  max_tool_calls: 6
  retry_policy:
    max_attempts: 2
    backoff_seconds: 2

interface:
  input:
    schema:
      type: object
      properties:
        source_text:
          type: string
          description: "The text to translate."
        source_language:
          type: string
          description: "Optional. Detected automatically if omitted."
        target_language:
          type: string
          description: "The target language (e.g. 'French', 'de', 'ja')."
        tone:
          type: string
          enum: ["formal", "neutral", "casual"]
          description: "Preferred tone for the output."
      required: ["source_text", "target_language"]
    examples:
      - source_text: "Please find attached the revised invoice."
        target_language: "French"
        tone: "formal"
      - source_text: "Hey, just checking in on that order."
        target_language: "German"
        tone: "casual"
  output:
    schema:
      type: object
      properties:
        translated_text:
          type: string
          description: "The translated output."
        detected_language:
          type: string
          description: "Detected source language when source_language was omitted."
        notes:
          type: array
          items: { type: string }
          description: "Optional notes about ambiguity, cultural nuance, or terminology choices."
      required: ["translated_text"]
    render:
      format: "markdown"
    provenance:
      citations_required: false

tools:
  tool_choice: "auto"
  refs:
    - ref: "glossary-lookup"
    - ref: "terminology-memory"

knowledge:
  refs:
    - ref: "terminology-kb"
      required: false
    - ref: "brand-guidelines"
      required: false
  retrieval:
    search_mode: "hybrid"
    trigger: "auto"
    max_chunks: 5
    max_tokens: 1800
    grounding_required: true
    citations_required: false
    use_when:
      - "source text includes specialized business or product terminology"
      - "brand voice consistency is required"
    avoid_when:
      - "request is simple conversational text with no domain terminology"

memory:
  mode: "project"

state:
  session_history: "ephemeral"
  working_state: "session"
  scratchpad_visible_to_model: true
  max_state_tokens: 1200

artifacts:
  enabled: true
  save_input: false
  save_output: true
  save_intermediate: false
  retention_class: "standard"

policies:
  pii_redaction: true
  allow_external_http: false
  require_human_approval: false
  blocked_content:
    - "legal_advice"
    - "medical_advice"
  audit_level: "standard"

guardrails:
  input:
    - ref: "pii_scan"
      mode: "detect"
      on_fail: "redact"
  output:
    - ref: "schema_validation"
      on_fail: "retry"

ui:
  icon: "languages"
  display_name: "Translator"
  short_description: "Translate business text between languages."
  order: 10
  featured: true
  ui_hints:
    source_text:
      widget: "textarea"
      placeholder: "Paste the text to translate…"
    target_language:
      widget: "select"
    tone:
      widget: "radio"
  localization:
    supported_locales: ["en", "fr", "de", "ja", "es"]
    default_locale: "en"
    localized_ui:
      fr:
        display_name: "Traducteur"
        short_description: "Traduisez vos textes professionnels entre les langues."

routing:
  intents: ["translate_text", "localize_email", "convert_language"]
  trigger_phrases: ["translate", "convert to French", "in German", "localize"]
  priority: 50
  fallback_agents: ["general-assistant"]

release:
  channel: "stable"
  rollout_percent: 100
  replaces_version: "1.0.0"
  approvals_required:
    - "product-owner"

observability:
  tracing:
    enabled: true
    log_inputs: false
    log_outputs: true
    log_tool_args: "redacted"
  metrics:
    enabled: true
    retention_days: 30

evaluation:
  offline:
    datasets: ["translator-gold-v1"]
    metrics:
      - "schema_valid_rate"
      - "terminology_accuracy"
      - "bleu_score"
    thresholds:
      schema_valid_rate: ">=0.995"
      terminology_accuracy: ">=0.92"
  online:
    monitors:
      - "tool_error_rate"
    thresholds:
      tool_error_rate: "<=0.02"

testing:
  smoke_tests:
    - id: "basic_translation_en_to_fr"
    - id: "formal_tone_preserved"
    - id: "pii_redacted_in_output"
    - id: "legal_question_refused"
  compile_assertions:
    - "required_sections_present"
    - "all_tool_ids_registered"
    - "all_kb_ids_registered"

notes:
  editor_notes: "Tone handling for Japanese informal register still under review."
---

# Purpose
Translate business text accurately between supported languages while preserving meaning,
structure, formatting, and approved terminology. The agent is aimed at SME users who need
professional-quality translations for business communications, documents, and product content.

# System behavior
You are a professional business translator serving SME users.
Translate the provided text into the requested target language.
Preserve intent, structure, names, dates, numerical values, and formatting.
Use natural business language — do not translate word-for-word when idiomatic phrasing is clearer.
If the source language is not specified, detect it automatically.
When translating technical or product terminology, look it up using the glossary tool
and apply approved translations consistently.
Apply brand voice guidance when translating content intended for external communications.

# Rules
- Do not invent or add content that is not present in the source text.
- Preserve proper nouns unless they have a clearly established translated form.
- Keep bullet lists, heading structure, and paragraph breaks intact.
- Flag ambiguous or culturally sensitive phrasing in the `notes` field of the output.
- If the requested target language is not supported, say so and list supported languages.
- Do not provide legal certifications or sworn translation services.

# Escalation
Escalate to a human reviewer when:
- The source text contains legal language requiring certified translation.
- The source text appears corrupted, truncated, or structurally inconsistent.
- The user explicitly requests sworn or notarised translation.

# Examples

## Example 1 — formal business email
Input: English email, target French, tone formal
Output: Formal French translation. Flag any ambiguous idioms in notes.

## Example 2 — product UI string
Input: Short English UI label, target Japanese, no tone specified
Output: Concise Japanese translation using product terminology from glossary.
Include a note if the approved terminology differs from the natural translation.

# Non-goals
- This agent does not certify or legally validate translations.
- This agent does not perform audio or video transcription.
- This agent does not rewrite or edit the source text before translating.
```
