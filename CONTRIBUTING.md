# Contributing to AML

Thank you for your interest in contributing to the Agent Modeling Language. AML is an open specification — contributions from platform engineers, product owners, compliance teams, and AI practitioners are all welcome.

---

## Table of contents

- [Ways to contribute](#ways-to-contribute)
- [Development setup](#development-setup)
- [Submitting changes](#submitting-changes)
- [Writing guidelines](#writing-guidelines)
- [Spec authoring rules](#spec-authoring-rules)
- [Code of conduct](#code-of-conduct)

---

## Ways to contribute

| Type | Description |
|---|---|
| **Bug report** | Found an inconsistency in the spec, a broken example, or a documentation error? Open an issue. |
| **Feature proposal** | Want to propose a new field, file kind, or extension mechanism? Open a discussion issue first. |
| **Example file** | Add a real-world `.agent.md`, `.tool.md`, or other AML file to the `docs/` examples section. |
| **Spec clarification** | Improve wording in an existing spec page without changing its semantics. |
| **New spec section** | Propose and write a new section — this requires an issue and discussion before a PR. |
| **JSON Schema** | Help maintain or extend the machine-readable JSON Schema for AML file kinds. |

---

## Development setup

AML documentation is built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

**Prerequisites**

- Python 3.10+
- `pip`

**Install dependencies**

```bash
pip install mkdocs-material mkdocs-with-pdf
```

**Serve locally**

```bash
mkdocs serve
```

The site will be available at `http://127.0.0.1:8000`. Changes to files in `docs/` hot-reload automatically.

**Build the static site**

```bash
mkdocs build
```

**Build with PDF export** (requires `ENABLE_PDF_EXPORT=1`)

```bash
ENABLE_PDF_EXPORT=1 mkdocs build
```

---

## Submitting changes

1. **Fork** the repository and create a branch from `main`.

   ```bash
   git checkout -b feat/my-contribution
   ```

2. Make your changes. Follow the [writing guidelines](#writing-guidelines) below.

3. Run `mkdocs serve` and verify your changes render correctly.

4. Commit with a clear message:

   ```
   docs: add example for content-moderation guardrail
   spec: clarify tool output schema validation rules
   fix: correct broken YAML in send-email.tool.md
   ```

   Prefix conventions: `spec:`, `docs:`, `fix:`, `example:`, `schema:`, `chore:`.

5. Open a pull request against `main`. Fill in the PR template:
   - What changed and why
   - Whether this changes spec semantics (breaking vs. non-breaking)
   - Which section of the spec is affected

6. At least one maintainer review is required before merge.

---

## Writing guidelines

- Write for a mixed audience: product owners, governance teams, and engineers should all be able to follow the text.
- Prefer short sentences and active voice.
- Use admonition blocks (`!!! note`, `!!! warning`) for important callouts — see the [MkDocs Material docs](https://squidfunk.github.io/mkdocs-material/reference/admonitions/).
- All YAML examples must be valid and, where possible, self-contained.
- Every new field introduced in a spec page must have: a type, whether it is required or optional, a default value (if any), and a short example.

---

## Spec authoring rules

These rules apply specifically to changes that modify the AML specification (files `00-` through `08-` in `docs/`).

- **Do not remove fields** that are already documented as `required` without a major version bump and a migration note.
- **Do not rename fields** without deprecating the old name first.
- All new fields must include a `since:` annotation (e.g., `since: v1.3`) in their description table row.
- Breaking changes require updating the version in `docs/00-introduction.md` and adding a `CHANGELOG.md` entry.
- If a change affects JSON Schema validity, update `docs/08-json-schema.md` accordingly.

---

## Code of conduct

This project follows the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/) Code of Conduct. By participating, you agree to uphold it. Please report unacceptable behavior to the maintainers via a private GitHub message or by email listed in the repository profile.
