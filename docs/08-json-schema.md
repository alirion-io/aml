# JSON Schema in YAML

> **Audience**: Authors of agent, tool, and knowledge base definition files


---

## Overview

Several fields across AML definition files — `input.schema`, `output.schema`, `parameters`, and `response_schema` — accept a **JSON Schema** value written directly in YAML. This page is the single reference for how to write those schemas.

AML supports a **practical subset of JSON Schema draft-07**. Features reserved for `$ref`, `$id`, `definitions`, and recursive schemas are not supported in AML schemas. All other keywords described here are valid.

---

## Scalar types

```yaml
# String
field_name:
  type: string
  description: "A short label."

# Integer (no decimal part)
retry_count:
  type: integer
  description: "Number of retry attempts."

# Number (integer or decimal)
confidence_score:
  type: number
  description: "Model confidence, 0.0–1.0."

# Boolean
is_verified:
  type: boolean
  description: "True if the account is verified."
```

---

## Objects

The top-level schema of any `input`, `output`, or `parameters` block must have `type: object`.

```yaml
type: object
properties:
  user_id:
    type: string
    description: "Stable user identifier."
  email:
    type: string
    description: "User's email address."
  age:
    type: integer
    description: "Age in years."
required: ["user_id", "email"]
additionalProperties: false
```

| Keyword | Purpose |
|---|---|
| `properties` | Named field definitions |
| `required` | List of property names that must be present |
| `additionalProperties` | `false` rejects unknown fields; `true` (default) allows them |

---

## Arrays

```yaml
tags:
  type: array
  items:
    type: string
  description: "List of tag strings."
  minItems: 1
  maxItems: 20

# Array of objects
attachments:
  type: array
  items:
    type: object
    properties:
      filename:
        type: string
      size_bytes:
        type: integer
    required: ["filename"]
  description: "File attachments."
```

---

## String constraints

```yaml
username:
  type: string
  minLength: 3
  maxLength: 64
  pattern: "^[a-z0-9_-]+$"
  description: "Lowercase alphanumeric username."

created_at:
  type: string
  format: "date-time"
  description: "ISO 8601 timestamp."
```

**Supported `format` values:**

| Value | Meaning |
|---|---|
| `date` | `YYYY-MM-DD` |
| `date-time` | `YYYY-MM-DDTHH:MM:SSZ` (ISO 8601) |
| `time` | `HH:MM:SS` |
| `email` | Valid email address |
| `uri` | Absolute URI |
| `uuid` | UUID v4 |
| `ipv4` / `ipv6` | IP address |

---

## Number constraints

```yaml
temperature:
  type: number
  minimum: 0.0
  maximum: 1.0
  description: "Model temperature."

page_size:
  type: integer
  minimum: 1
  maximum: 100
  default: 10
  description: "Pagination page size."
```

---

## Enumerations

```yaml
status:
  type: string
  enum: ["pending", "active", "resolved", "closed"]
  description: "Current ticket status."

priority:
  type: integer
  enum: [1, 2, 3]
  description: "Priority level: 1 = high, 3 = low."
```

---

## Default values

```yaml
language:
  type: string
  default: "en"
  description: "BCP 47 language tag for the response."
```

`default` is informational — the runtime does not inject defaults automatically. Use it to communicate expected behavior to the calling agent.

---

## Nullable fields

To allow a field to be either a value or `null`, use an array of types:

```yaml
resolved_at:
  type: ["string", "null"]
  format: "date-time"
  description: "Timestamp when the ticket was resolved, or null if still open."
```

---

## Files

JSON Schema has no native `file` type, and the compiler cannot apply special handling (size enforcement, MIME validation, virus scanning, audit logging) unless it can reliably identify a field as a file. To make that possible, AML requires a **canonical attachment object shape** — a fixed set of property names that the compiler recognises as a file field regardless of the parent field name.

### Canonical attachment object

Every file field, whether the content is inline or passed by reference, must use this object shape:

| Property | Type | Required | Purpose |
|---|---|---|---|
| `transfer` | `string` enum `inline` \| `ref` | Yes | Tells the compiler which delivery pattern is used |
| `filename` | `string` | Yes | Original file name including extension |
| `mime_type` | `string` | Yes | MIME type; use `enum` or `const` to restrict accepted formats |
| `max_size_bytes` | `integer` | Yes | Maximum permitted file size in bytes; enforced by the tool backend |
| `content` | `string` | When `transfer: inline` | Base64-encoded file bytes (`contentEncoding: base64`) |
| `url` | `string` (`format: uri`) | When `transfer: ref` | URI of the pre-stored file |

The compiler detects a file field by the presence of the `transfer` property with a value of `inline` or `ref`. Any object carrying that property is treated as an attachment and is subject to the platform's file-handling rules.

### Inline content (`transfer: inline`)

Embed file bytes directly in the payload. Best for small files (under 1 MB).

```yaml
attachment:
  type: object
  properties:
    transfer:
      type: string
      const: "inline"
    filename:
      type: string
      description: "Original file name including extension."
    mime_type:
      type: string
      enum: ["image/png", "image/jpeg", "image/webp"]
      description: "MIME type of the file."
    max_size_bytes:
      type: integer
      const: 524288        # 512 KB
      description: "Maximum accepted file size. Files larger than this are rejected."
    content:
      type: string
      contentEncoding: base64
      contentMediaType: "application/octet-stream"
      description: "Base64-encoded file content."
  required: ["transfer", "filename", "mime_type", "max_size_bytes", "content"]
  additionalProperties: false
  description: "A single inline image attachment. Maximum 512 KB."
```

### File reference (`transfer: ref`)

Pass a URI to a file already stored in an accessible location (object storage, CDN, presigned URL). Best for large files or when the file already exists in the platform.

```yaml
document:
  type: object
  properties:
    transfer:
      type: string
      const: "ref"
    filename:
      type: string
      description: "Original file name including extension."
    mime_type:
      type: string
      enum:
        - "application/pdf"
        - "text/plain"
        - "text/csv"
      description: "MIME type of the file at the given URL."
    max_size_bytes:
      type: integer
      const: 10485760      # 10 MB
      description: "Maximum accepted file size. Files larger than this are rejected by the tool backend."
    url:
      type: string
      format: "uri"
      description: >
        Presigned URL of the uploaded document. Must be accessible by the tool's
        service account and must not expire before the request is processed.
  required: ["transfer", "filename", "mime_type", "max_size_bytes", "url"]
  additionalProperties: false
  description: "A reference to an externally stored document. Maximum 10 MB."
```

### Multiple attachments

To accept a list of files, wrap the attachment object in an array:

```yaml
attachments:
  type: array
  items:
    type: object
    properties:
      transfer:
        type: string
        const: "ref"
      filename: { type: string }
      mime_type:
        type: string
        enum: ["application/pdf", "image/png", "image/jpeg"]
      max_size_bytes:
        type: integer
        const: 5242880     # 5 MB
      url:
        type: string
        format: "uri"
    required: ["transfer", "filename", "mime_type", "max_size_bytes", "url"]
    additionalProperties: false
  minItems: 1
  maxItems: 5
  description: "Up to 5 file attachments, each a maximum of 5 MB."
```

### Choosing a delivery pattern

| | `transfer: inline` | `transfer: ref` |
|---|---|---|
| File size | Small (< 1 MB) | Any |
| File already stored | No | Yes |
| Audit / redaction | Content is logged unless `log_args: redacted` | Only the URL is logged |
| Latency | Adds payload weight | Minimal |

For tools with `data_classification.input_pii_risk: true`, prefer `transfer: ref` and set `log_args: "redacted"` so file content is never written to logs.

---

## Combining schemas

Use these keywords sparingly. Prefer flat `object` schemas with `required` and `additionalProperties: false` where possible.

### `oneOf` — exactly one must match

```yaml
contact:
  oneOf:
    - type: object
      properties:
        email: { type: string }
      required: ["email"]
    - type: object
      properties:
        phone: { type: string }
      required: ["phone"]
  description: "Either an email or a phone number, not both."
```

### `anyOf` — one or more must match

```yaml
identifier:
  anyOf:
    - type: string
      format: "uuid"
    - type: string
      pattern: "^EXT-[0-9]{6}$"
  description: "A UUID or a legacy EXT-NNNNNN identifier."
```

### `allOf` — all must match

```yaml
named_entity:
  allOf:
    - type: object
      properties:
        id: { type: string }
      required: ["id"]
    - type: object
      properties:
        name: { type: string }
      required: ["name"]
```

---

## Nested objects

```yaml
type: object
properties:
  address:
    type: object
    description: "Postal address."
    properties:
      street:
        type: string
      city:
        type: string
      country:
        type: string
        description: "ISO 3166-1 alpha-2 country code."
    required: ["street", "city", "country"]
    additionalProperties: false
required: ["address"]
```

---

## Writing good descriptions

Descriptions in `input` and `output` schemas are read by the model at runtime. Write them from the **model's perspective** — describe what the field means in the context of the agent's task, not what the field is structurally.

| Weak | Strong |
|---|---|
| `"The account ID field."` | `"Customer account identifier from the CRM. Provide this whenever the user has authenticated; omit if the session is anonymous."` |
| `"Boolean flag."` | `"Set to true if the agent escalated this case to a human agent."` |

---

## Complete example

```yaml
type: object
properties:
  question:
    type: string
    minLength: 1
    maxLength: 2000
    description: "The customer's question or request, in their own words."
  account_id:
    type: ["string", "null"]
    pattern: "^acct-[0-9]{5}$"
    description: "Customer account identifier. Null if the user is unauthenticated."
  language:
    type: string
    format: "language-tag"
    default: "en"
    description: "BCP 47 language tag of the user's preferred language."
  attachments:
    type: array
    items:
      type: object
      properties:
        filename: { type: string }
        mime_type: { type: string }
        size_bytes: { type: integer, minimum: 0 }
      required: ["filename", "mime_type"]
    maxItems: 5
    description: "Files uploaded by the user alongside the question."
required: ["question"]
additionalProperties: false
```
