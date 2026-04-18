# HWT Community Conventions

Community reference definitions for authorization schemas and jurisdiction vocabulary. These are not normative — they carry no protocol compliance weight. Implementations are not required to use them.

Their value is shared vocabulary: two services that have never coordinated can exchange tokens carrying `RBAC/1.0.2` or `GDPR/2.0/DE` and understand what the other means. Using a definition from this document signals intent to conform to that definition's structure. What the consuming application does with the claim is its own responsibility.

**Carrying a claim is not compliance.** A token with `GDPR/2.0/DE` does not make the issuing or consuming application GDPR-compliant. A token with `RBAC/1.0.2` does not make authorization correct. These are structural definitions and shared vocabulary, not certifications.

Changes to this document go through a PR with a short description of what is being added or changed and why. No formal review period is required.

---

## Authorization Schemas

An authorization schema defines the structure of an `authz` object in an HWT token. The `scheme` field identifies which definition applies. All other fields are schema-defined.

Private schemas are the expected default — use an origin-relative or absolute URL as the `scheme` value and define the structure however your application needs. The schemas below exist to support cross-domain interoperability between parties who want a shared baseline without coordinating a private schema.

When extrapolating to a new schema, the pattern is:

- Pick a short, unambiguous identifier with a version suffix (`Name/major.minor.patch`)
- Define every field the schema introduces — name, type, whether it is required, and what it means
- Provide a minimal example
- Document what array `authz` means for this schema when combined with others

---

### `RBAC/1.0.2` — Role-Based Access Control

Role-based access control with an optional capability list. Suitable for systems where access is governed by named roles, optionally refined by explicit capability grants.

**Fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `scheme` | string | Yes | Must be `"RBAC/1.0.2"` |
| `roles` | string[] | Yes | One or more role names. Non-empty array. Role semantics are application-defined. |
| `capabilities` | string[] | No | Explicit capability grants. Used to refine what a role permits in a specific token without changing the role definition globally. |

Role names and capability strings are opaque to the protocol. Their meaning is negotiated between the issuer and the consuming application. Common conventions: lowercase with underscores (`edit_posts`), namespaced with a colon (`posts:edit`). Pick one form and be consistent.

**Minimal example:**

```json
"authz": {
  "scheme": "RBAC/1.0.2",
  "roles": ["editor"]
}
```

**With capabilities:**

```json
"authz": {
  "scheme": "RBAC/1.0.2",
  "roles": ["editor"],
  "capabilities": ["edit_posts", "upload_files", "publish_posts"]
}
```

**Multi-role:**

```json
"authz": {
  "scheme": "RBAC/1.0.2",
  "roles": ["editor", "billing_admin"]
}
```

**In array `authz` alongside a private schema:**

```json
"authz": [
  { "scheme": "RBAC/1.0.2", "roles": ["editor"] },
  { "scheme": "/schemas/partner-access/v2", "clearance": "confidential" }
]
```

When used in array `authz`, `RBAC/1.0.2` is evaluated independently. The consuming application decides whether all schemes must pass (`all`) or any one is sufficient (`any`), as declared in the issuer's `hwt.json` via `authz_evaluation`.

**Version notes:** `capabilities` added in this version. `roles` non-empty requirement clarified before initial publication. Prior draft versions were not separately released.

---

### Defining a private schema

If `RBAC/1.0.2` doesn't fit your model, define your own. Use an origin-relative path as the `scheme` value:

```json
"authz": {
  "scheme": "/schemas/my-policy/v1",
  "clearance": "confidential",
  "regions": ["EU", "US-WEST"]
}
```

The URL form is the unambiguous signal to any verifier that this is a private schema. No registration is required. Document your schema's fields wherever your team keeps API documentation. If your schema becomes useful to others outside your organization and you want a short identifier, open a PR to add it here.

---

## Jurisdiction Vocabulary

The `authz` field is the place to carry jurisdiction context when needed — as a private schema field, or by adopting the vocabulary below. These identifiers give teams a shared shorthand for common regulatory frameworks so that jurisdiction claims are legible across implementations without out-of-band coordination.

**This vocabulary does not define compliance.** What a regulatory framework requires of your application — data residency, retention limits, consent flows, breach notification — is entirely your responsibility. These identifiers name a framework; they do not describe or enforce what it demands.

### Identifier format

```
Regulation/Version/Country[,Country]
```

- Regulation: identifier from the table below, or a private string (URL form recommended for private identifiers)
- Version: major.minor of this vocabulary entry (not the regulation's legal version)
- Country: ISO 3166-1 alpha-2 codes, comma-separated when multiple apply

Multiple independent declarations are semicolon-separated:

```
GDPR/2.0/DE,FR;CCPA/1.0/US
```

### Carrying jurisdiction in a token

Jurisdiction context belongs in `authz` as a private schema field or alongside other `authz` entries. Example using a private schema for jurisdiction alongside RBAC:

```json
"authz": [
  { "scheme": "RBAC/1.0.2", "roles": ["editor"] },
  { "scheme": "/schemas/jurisdiction/v1", "jur": "GDPR/2.0/DE,FR" }
]
```

Or embedded directly in a private authz schema when jurisdiction is always present:

```json
"authz": {
  "scheme": "/schemas/data-access/v3",
  "datasets": ["analytics"],
  "jur": "HIPAA/1.0/US"
}
```

How jurisdiction is represented in the token is the issuer's decision. These identifiers are vocabulary — use them in whatever structure your application defines.

### Standard identifiers

| Identifier | Full name | Applicable regions | Vocabulary version |
|---|---|---|---|
| `GDPR` | EU General Data Protection Regulation | EU/EEA member states, UK | `2.0` |
| `CCPA` | California Consumer Privacy Act | `US` (California) | `1.0` |
| `HIPAA` | US Health Insurance Portability and Accountability Act | `US` | `1.0` |
| `PIPEDA` | Canada Personal Information Protection and Electronic Documents Act | `CA` | `1.0` |
| `LGPD` | Brazil Lei Geral de Proteção de Dados | `BR` | `1.0` |
| `PDPA` | Thailand Personal Data Protection Act | `TH` | `1.0` |
| `APPI` | Japan Act on the Protection of Personal Information | `JP` | `1.0` |

**Examples:**

```
GDPR/2.0/DE
GDPR/2.0/DE,FR,NL
HIPAA/1.0/US
CCPA/1.0/US
GDPR/2.0/GB,IE;HIPAA/1.0/US
LGPD/1.0/BR
```

### Private jurisdiction identifiers

For internal or sector-specific frameworks not listed here, use an absolute or origin-relative URL as the identifier — the same convention as private `authz` schemas:

```
/compliance/internal-data-policy/v2/US
https://compliance.example.com/frameworks/sector-specific/v1/US,CA
```

To add a community identifier, open a PR with the full name, applicable regions, and a brief description.

---

## Token Exchange Scope Conventions

The `scope` field in token exchange requests ([SPEC](SPEC.md) §8.2) restricts the derived token's `authz` to a subset of the subject token's `authz`. Scope semantics are schema-defined ([SPEC](SPEC.md) §8) and deferred to community conventions.

This section is a placeholder. Scope convention definitions belong here when the community is ready to propose them. Until then, issuers and consumers that use `scope` in token exchange define its semantics privately and document them alongside their `authz` schema definitions.

---

## Extrapolating new schemas

Both sections above follow the same pattern. When defining something new — whether an authorization schema or a jurisdiction-carrying convention — the useful questions are:

**What is the minimum structure that makes this legible to a verifier that has never seen it before?** Every field should earn its place. Prefer flat structures; nested objects make consuming application code more complex without usually adding clarity.

**What does it mean when combined with other schemas in array `authz`?** If your schema is only valid alone, say that. If it's designed to compose, describe how.

**What does absence mean?** If a field is optional, what should the consuming application infer when it's missing? Document the default explicitly rather than leaving it to interpretation.

**What is not your schema's job?** Evaluation logic, conflict resolution between schemas, and compliance enforcement are the consuming application's concern. The schema defines structure; it does not define policy.
