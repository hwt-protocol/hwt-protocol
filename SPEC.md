# Hash Web Token (HWT) Protocol
## Draft Specification v0.7

**Repository:** https://github.com/hwt-protocol/

---

## Abstract

Hash Web Token (HWT) is a stateless, network-native token protocol. Tokens are signed by an issuing origin, verifiable by any party that can reach the issuer's domain, and carry authorization context either literally or by reference. The protocol defines token structure, signed input canonicalization, key discovery, origin metadata, delegation chain construction, and a codec registry.

HWT is a stateless protocol. Token state — including revocation, session management, and invalidation — is explicitly outside the protocol's scope and the application's responsibility. The same applies to jurisdiction enforcement, authorization evaluation, and schema semantics. Narrowing scope to what the signed byte string can structurally guarantee is the design intent. See Section 13.

Token issuance is explicitly out of scope. HWT defines the contract that passes between parties after a principal has been authenticated by whatever mechanism the issuing service uses.

**Trust model summary.** Verification depends on DNS resolving correctly and TLS connecting to the issuer's domain. The trust anchor is the integrity of the initial key fetch, not a central identity provider. Verifiers that restrict accepted issuers to a pre-configured allowlist operate under a stronger security model than those accepting tokens from previously-unknown origins. See Section 11.1.

This specification is intended to be stable. The token format, signed input canonicalization, and verification algorithm are not expected to change. Extensions — authorization schemas, codecs, endpoint declarations — are community concerns and do not require spec revision.

---

## 1. Design Principles

**Stateless verification.** Any verifier with access to the issuer's public keys can verify any token without contacting the issuing service. Verification is a local operation after an initial cache warm. The initial cache warm requires DNS resolution and a TLS connection to the issuer's well-known endpoint. The trust anchor for all subsequent local verification is the integrity of that initial fetch.

**Token state is the application's concern.** HWT defines what a signed token structurally guarantees: that the payload was signed by the issuer, that it has not expired, and that the delegation chain has not been tampered with. Everything beyond that — whether a token has been revoked, whether a session is still active, whether a user's permissions have changed — is state that lives outside the signed byte string. HWT intentionally does not address it. Applications that need immediate invalidation maintain their own state store and check it in the application layer. Short token lifetimes are the primary mechanism for bounding exposure.

**Origin sovereignty.** Each issuing domain is a first-class authority. There are no federation agreements, no central identity provider, no registration requirement. Any domain that publishes valid well-known endpoints is a valid issuer.

**Protocol as meta-standard.** HWT defines how authorization schemas are referenced and how delegation chains are constructed. It does not define what schemas contain, what jurisdictions require, or how applications evaluate claims. Those are negotiated between issuers and consuming applications.

**Network-native, cache-optimized.** The protocol assumes network availability. Intermittent offline operation is handled by standard HTTP caching. No special offline mode exists. Expired tokens do not work — this is correct behavior, not a failure.

**Compact by design.** HWT tokens are optimized to fit comfortably within HTTP header size limits (practical limit ~8KB for most implementations; tokens SHOULD stay well under this ceiling). Large payloads are out of scope.

**Explicit scope boundaries.** Narrower protocol commitments produce more precise security claims, more predictable library behavior, and fewer assumptions an integrator can accidentally rely on. When the boundary is explicit, the application knows exactly what it is responsible for. See Section 13.

---

## 2. Token Format

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

| Field | Description |
|---|---|
| `hwt` | Literal prefix identifying the token type |
| `signature` | Base64url-encoded asymmetric signature over the canonical signed input (see Section 2.1) |
| `key-id` | Identifier of the signing key, matched against the issuer's JWKS |
| `expires-unix-seconds` | Expiry as a UNIX timestamp in seconds. Structural — cannot be omitted or overridden by a payload field. |
| `format` | Codec identifier. `j` (JSON) is the baseline. See Section 5. |
| `payload` | Base64url-encoded token data in the declared codec |

Fields are dot-separated. Tokens with a past `expires` value are unconditionally invalid regardless of signature validity.

**The dot is the field separator. It MUST NOT appear in any value that is encoded literally into the token string.** This applies to `key-id`, `format` (codec identifier), and all top-level payload key names. A dot in any of these fields will break parsing — there is no escaping mechanism. This constraint applies once, here, because violating it produces tokens that cannot be parsed and the error will not always be immediately obvious. Codec identifiers, `kid` values, and payload key names must all be dot-free.

**Signature algorithms.** For cross-domain verification, HWT requires asymmetric public key signatures — ECDSA (P-256 or P-384), ECDSA P-521, and Ed25519. P-521 is conforming-optional: implementations are not required to support it, and issuers using P-521 SHOULD verify that key generation and export succeed in their target environment before deploying (JWKS `alg: "ES512"`; support varies by runtime). HMAC is valid for single-party deployments where the issuer and verifier share a secret and cross-domain public-key verification is not required. HMAC tokens are not conforming for the cross-domain protocol but are a legitimate use of the token format in private contexts.

The signing algorithm is declared per-key in the JWKS document (`alg` field, per RFC 7517). Verifiers MUST use the algorithm declared for the matched `key-id`. Tokens signed with an algorithm that conflicts with the key's declared `alg` MUST be rejected.

### 2.1 Signed Input Canonicalization

The signature is computed over a canonical string constructed as follows:

```
signedInput = expires + "." + format + "." + base64url(encode(payload))
            [ + "." + base64url(encode(hiddenData)) ]   // when hiddenData is provided
```

Where:
- `expires` is the raw UNIX timestamp string as it appears in field 4 of the token
- `format` is the codec identifier string as it appears in field 5
- `base64url(encode(payload))` is the encoded payload as it appears in field 6 — the same bytes
- `hiddenData` is optional caller-supplied associated data that is signed but not embedded in the token wire format; it MUST be supplied identically at both signing and verification time, and MUST be passed through the same codec encode path used at signing time

**On hiddenData.** The signed input MAY include caller-supplied associated data not embedded in the token. What this data contains and how it is coordinated between signer and verifier is outside the protocol. Its effect is that a token cannot be verified without supplying the correct hidden data, creating an additional verification dependency that the issuer and verifier negotiate between themselves.

**Key properties:**
- `key-id` and the `hwt` prefix are NOT part of the signed input. For asymmetric algorithms, substituting a different `key-id` causes signature verification to fail against that key's public material — the signed input need not include `kid` because it is structurally self-defeating to replace it. For HMAC deployments, `kid` selection is the caller's responsibility.
- All fields in the signed input are dot-separated, matching the token's field separator.
- The encoded payload bytes are identical in the signed input and the token wire format — there is no re-encoding step during verification.

Verifiers MUST reconstruct `signedInput` from the token fields before verification.

---

## 3. Payload Structure

The payload is a key-value structure encoded in the declared codec. The following keys are defined by the protocol. All other keys are application-defined and ignored by the protocol layer.

Top-level payload key names MUST NOT contain a dot (`.`). See Section 2.

### 3.1 Required Keys

| Key | Description |
|---|---|
| `iss` | Issuer. The origin URL of the issuing domain (e.g., `https://domain.com`). Used to locate well-known endpoints. |
| `sub` | Subject. See Section 3.4. |
| `authz` | Authorization schema reference and optional instance values. See Section 4. |

### 3.2 Optional Keys

| Key | Description |
|---|---|
| `aud` | Audience. Intended recipient service identifier. String or array of strings. Array form requires `aud_array_permitted: true` in issuer's `hwt.json` — verifiers MUST reject array `aud` if the issuer has not declared this. When present, verifiers MUST reject tokens where no `aud` value matches their own identifier. When absent, any verifier may accept the token. Required when issuer declares `aud_required: true` in `hwt.json`. |
| `tid` | Token ID. Opaque string identifier for this token instance. No protocol-level normative behavior is attached. Available for application use: logging, audit trails, application-level state management, or any other purpose the issuer and consuming application agree on. Generation method is issuer-defined. |
| `iat` | Issued-at timestamp in UNIX seconds. Informational; expiry is structural in the token format. |
| `del` | Delegation chain. Ordered array of Provenance Records. See Section 3.5. |

### 3.3 Reserved Keys

`meta` is reserved for future use. Implementations MUST NOT use it for any purpose.

### 3.4 Subject (`sub`)

`sub` is an opaque string identifying the entity the token represents. It MUST be a string — numeric values MUST be serialized as strings (e.g., `"sub": "4503599627370495"`, not `"sub": 4503599627370495`). This avoids large-integer parsing inconsistencies across languages and runtimes.

`sub` is the externally-visible, privacy-preserving identifier the issuer presents to consuming applications. Internal database keys, linkage identifiers, or other privileged references to the same user SHOULD NOT appear in tokens presented to external services. Issuers that use rotatable alias identifiers or anonymous IDs for privacy compliance should use those values here.

`sub` format is issuer-defined. Recommended convention: namespaced string.

```
"sub": "user:4503599627370495"
"sub": "svc:payments"
"sub": "user@domain.com"
```

The consuming application and issuer are responsible for agreeing on `sub` semantics. The protocol treats it as opaque.

### 3.5 Delegation Chain (`del`)

`del` is an ordered array of **Provenance Records** (see Section 3.6) tracing the authorization lineage from the root principal to the most recent delegator. The token's own `sub` is the final delegate — it is not repeated in `del`.

**Order:** earliest (root) principal first, most recent delegator last.

```json
"del": [
  { "iss": "https://auth.example.com", "sub": "user:4503599627370495", "tid": "t-a1b2" },
  { "iss": "https://agent-a.example.com", "sub": "svc:agent-a", "tid": "t-c3d4" }
]
```

**What the signature covers.** The signature on the outer token covers the entire `del` array. The chain cannot be tampered with after issuance — this is the security guarantee the protocol provides for delegation. What the signature does not cover: whether any link in the chain has been subsequently invalidated by the issuing application. Token state is outside the protocol; applications requiring immediate invalidation of delegation chains maintain their own state and check it in the application layer.

**`tid` in `del` entries.** Including `tid` in Provenance Records is RECOMMENDED for auditability. It enables the consuming application to trace each chain link to a specific token instance at its issuer — useful for logging, debugging, and application-level state lookups. It is not required by the protocol.

**Depth limit.** The protocol default depth limit is 10. Issuers MAY declare a limit via `max_delegation_depth` in `hwt.json`, but this value can only decrease the applicable limit, not increase it. Verifiers MUST enforce a local maximum depth that issuer declaration cannot override; the recommended local maximum is 10. Tokens with a `del` array exceeding the applicable limit MUST be rejected. Verifiers SHOULD detect and reject cyclic chains (where any `iss`+`sub` pair appears more than once in the `del` array plus the outer token's own `iss`+`sub`). The depth limit is a DoS protection that bounds the number of network fetches a verifier makes per verification.

### 3.6 Provenance Record

A **Provenance Record** is a structured object identifying a principal by issuer, subject, and optionally token instance. It is defined here as a named type for use wherever the protocol requires cross-domain traceable attribution.

```
{ "iss": <HTTPS origin>, "sub": <subject string>, "tid": <token ID string, optional> }
```

| Field | Required | Description |
|---|---|---|
| `iss` | Yes | HTTPS origin of the issuing domain. MUST resolve to valid HWT well-known endpoints. |
| `sub` | Yes | Subject identifier at the referenced issuer. Treated as opaque by the protocol. |
| `tid` | No | Token instance ID. Recommended for auditability when available. |

`del` is the first use of this type. Future fields that require traceable cross-domain attribution SHOULD use this structure rather than defining ad-hoc alternatives.

---

## 4. Authorization (`authz`)

The `authz` key carries the schema reference and, optionally, the instance values that apply to this token. The protocol defines the structure of this field. Evaluation of the authorization claim against application requirements is the consuming application's responsibility and is outside the protocol's scope.

The `authz` key accepts three forms.

### 4.1 String form — scheme reference only

When authorization instance values live in the well-known metadata or are resolved entirely by the consuming application:

```json
"authz": "RBAC/1.0.2"
```

Bare names without versions (e.g., `"RBAC"`) are NOT valid. Versions are expected for all standard registry references.

### 4.2 Object form — scheme and instance values together

When the token carries both the schema reference and the concrete authorization values. The object MUST contain a `scheme` key. All other keys are schema-defined.

```json
"authz": {
  "scheme": "RBAC/1.0.2",
  "roles": ["editor", "contributor"]
}
```

Example — private org schema with origin-relative reference:

```json
"authz": {
  "scheme": "/schemas/internal-policy/v3",
  "roles": ["analyst"],
  "groups": ["finance", "read-only"]
}
```

For RBAC-specific examples including capability grants and multi-role patterns, see CONVENTIONS.md.

### 4.3 Array form — multiple schemes

When a token must satisfy more than one authorization framework simultaneously:

```json
"authz": [
  { "scheme": "RBAC/1.0.2", "roles": ["editor"] },
  { "scheme": "/schemas/partner-policy/v2", "clearance": "confidential" }
]
```

Array evaluation semantics are declared in `hwt.json` via `authz_evaluation`. Default when not declared: `"all"`. Evaluation is the consuming application's responsibility.

### 4.4 Schema reference resolution

| Value form | Resolution |
|---|---|
| `"RBAC/1.0.2"` | Community convention identifier — see CONVENTIONS.md |
| `"/path/to/schema"` | Origin-relative: prepend `iss` value |
| `"https://..."` | Absolute URL — issuer's responsibility. Content-addressed URLs strongly recommended. |

**Private schemas.** The URL form is the unambiguous signal that a schema is private. No special prefix or registration is required. Private schemas appear in the issuer's `hwt.json` `authz_schemas` array for discovery. In practice, most deployments will use private schemas — community convention identifiers exist for cross-domain interoperability, not to enumerate every possible authorization model.

**Security note.** The trust boundary for `authz` references is the token signature. A verifier trusting the issuer's key trusts the issuer's schema references.

---

## 5. Codec Registry

The `format` field in the token structure declares the codec used to encode the payload. Codec identifiers are short ASCII strings: one letter followed by one to nineteen alphanumeric characters (total length: 2–20 characters). Identifiers MUST NOT contain a dot (`.`).

`j` (JSON, RFC 8259) is the only normative codec. All conforming implementations MUST support `j`. It is the interoperability baseline. Implementations that encounter an unsupported codec identifier MUST reject the token — they MUST NOT attempt to parse the payload as JSON.

Additional codecs may be defined by the community. The identifier format above applies to all community-defined codecs. Proprietary codec identifiers SHOULD begin with an uppercase letter; versioned codecs MAY use a trailing number (e.g., `MyCodec2`). Examples of community codecs and their implementation details are in CONVENTIONS.md.

---

## 6. Key Discovery

Issuers MUST publish public keys at:

```
https://{iss-domain}/.well-known/hwt-keys.json
```

The document follows the JWKS (JSON Web Key Set) format [RFC 7517]. Each key entry MUST include:
- `kid` — matches `key-id` in tokens signed with this key. MUST NOT contain a dot (`.`).
- `alg` — the signing algorithm (`ES256`, `ES384`, `ES512`, `EdDSA`). `ES512` (P-521) support varies by environment; see Section 2.
- `use: "sig"` — key use declaration

A single `hwt-keys.json` document MAY contain any number of keys. Issuers MAY operate multiple keys simultaneously — for key rotation overlap, purpose-specific signing, or algorithm diversity. The `kid` in each token selects the applicable key. `kid` namespacing is the issuer's responsibility; the protocol treats `kid` values as opaque strings.

**Example `hwt-keys.json` with multiple concurrent keys:**

```json
{
  "keys": [
    {
      "kty": "EC",
      "kid": "key-2024-01",
      "use": "sig",
      "alg": "ES256",
      "crv": "P-256",
      "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
      "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0"
    },
    {
      "kty": "OKP",
      "kid": "key-2025-01",
      "use": "sig",
      "alg": "EdDSA",
      "crv": "Ed25519",
      "x": "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo"
    }
  ]
}
```

**Key loading patterns.** Implementations SHOULD pre-load and cache keys for trusted issuers at application startup rather than fetching on first token encounter. Pre-loading converts the first-verification latency into a startup cost and enables immediate rejection of unknown issuers. Two patterns are common:

*Pre-registered issuers (recommended for production):*
```
at startup:
  for each trusted issuer:
    fetch /.well-known/hwt-keys.json
    parse and import keys into local key registry
    index by iss::kid

at verification:
  look up iss::kid in local registry
  if not found: reject (unknown issuer or unknown key)
  verify signature locally — no network call
```

*Unknown-issuer accept (higher trust, higher risk):*
```
at verification:
  extract iss from token payload
  look up iss::kid in cache
  if not found: fetch /.well-known/hwt-keys.json from iss origin
               import keys, cache, retry lookup
  verify signature
```

The unknown-issuer pattern requires trusting DNS and TLS at key fetch time for every previously-unseen issuer. See Sections 11.1 and 11.2.

**Key rotation procedure:**
1. Publish the new key in `hwt-keys.json` alongside the existing key
2. Wait at minimum the current `max-age` of `hwt-keys.json` before using the new key — this allows verifier caches to warm
3. Begin signing new tokens with the new key
4. Retire the old key only after all tokens signed with it have expired

Verifiers MUST:
- Cache `hwt-keys.json` per HTTP `Cache-Control` headers
- Re-fetch immediately (bypassing cache) when encountering an unknown `kid`
- Scope `kid` lookup to the `iss` origin's JWKS — a `kid` resolved from one issuer's JWKS MUST NOT be used to verify tokens from a different issuer
- Reject tokens with a `kid` that cannot be resolved after a forced re-fetch
- Rate-limit forced re-fetches per origin (recommended: no more than one per 60 seconds) to prevent `kid`-flood abuse

---

## 7. Origin Metadata

Issuers SHOULD publish origin metadata at:

```
https://{iss-domain}/.well-known/hwt.json
```

When `hwt.json` is absent or unreachable, verifiers apply the documented defaults 
for all fields: `authz_evaluation: "all"`, `aud_required: false`, 
`aud_array_permitted: false`, `max_delegation_depth: 10`. Verification proceeds 
normally. Consuming applications that depend on issuer-declared schema enumeration 
or endpoint discovery require the document to be present; the protocol does not.

**Example document:**

```json
{
  "issuer": "https://domain.com",
  "authz_schemas": [
    "RBAC/1.0.2",
    "/schemas/internal-rbac/v3"
  ],
  "authz_evaluation": "all",
  "aud_required": false,
  "aud_array_permitted": false,
  "max_delegation_depth": 10,
  "endpoints": {
    "token_exchange": "https://domain.com/hwt/token-exchange"
  }
}
```

| Field | Required | Description |
|---|---|---|
| `issuer` | Yes | Must match `iss` in tokens exactly |
| `authz_schemas` | Yes | Schemas this issuer may use. Informational — consuming applications use this to discover what schemas to expect. |
| `authz_evaluation` | No | `"all"` or `"any"` for array `authz`. Default: `"all"` |
| `aud_required` | No | If `true`, verifiers MUST reject tokens lacking `aud`. Default: `false` |
| `aud_array_permitted` | No | If `true`, tokens from this issuer MAY carry `aud` as an array. Verifiers MUST reject array `aud` when this is `false` or absent. Default: `false` |
| `max_delegation_depth` | No | Maximum permitted `del` array length declared by this issuer. This value can only decrease the verifier's effective maximum — it cannot increase the verifier's locally configured cap. Default when absent: protocol default of 10. |
| `endpoints` | No | Grouped service endpoint declarations. See Section 7.1. |

### 7.1 Endpoint Declarations

The `endpoints` object groups service endpoint URLs for this issuer. All values MUST be HTTPS URLs.

| Key | Description |
|---|---|
| `token_exchange` | URL of the token exchange endpoint. See Section 8. When present, this issuer supports delegated token issuance. |

Verifiers MUST ignore unrecognized keys in the `endpoints` object.

The `endpoints` object is optional. Issuers that declare no endpoints operate in pure stateless-verification mode: tokens are verified locally against cached keys with no network calls beyond the initial key fetch.

---

## 8. Token Exchange

Token exchange is the mechanism by which a delegating agent obtains a new token scoped to a downstream recipient, carrying a `del` chain that traces the full authorization lineage. This is the protocol-level operation that populates the `del` field.

### 8.1 Normative Chain Construction Rules

These rules are normative regardless of transport mechanism.

**Verification before construction.** The entity constructing the derived token MUST verify that both the subject token and the actor token are currently valid — signatures valid, not expired — before appending a Provenance Record and issuing the derived token. This verification event is the moment of trust. Once the record is appended and the outer token is signed, the chain is frozen.

**Who may append.** A Provenance Record may only be appended by an entity that holds a valid, unexpired token for the subject being recorded. An entity cannot insert itself into a chain it did not participate in.

**Authorization attenuation.** The derived token's `authz` MUST be equal to or a strict subset of the subject token's `authz` for each schema present. The constructing entity MUST NOT issue a derived token claiming permissions not present in the subject token. When `scope` is provided, it restricts the derived token's `authz` further. When absent, the derived token inherits the subject token's `authz` unchanged. You can only delegate what you have, never more.

**Derived token requirements.** A token carrying a `del` chain MUST:
- Have `iss` set to the issuer constructing the derived token
- Have `sub` set to the actor token's `sub`
- Have `aud` set to the requested audience when specified
- Have `del` set to: the subject token's existing `del` array (if any) followed by a new Provenance Record for the subject token's `iss`, `sub`, and optionally `tid`
- Have `expires` set to no later than the subject token's expiry
- SHOULD include `tid` for auditability

**Depth check.** The constructing entity MUST confirm that the resulting `del` array would not exceed the applicable `max_delegation_depth` before issuing. If it would exceed the limit, the exchange MUST be rejected rather than issuing a token that downstream verifiers will reject.

### 8.2 Reference HTTP Transport

*This section is informational. Implementations MAY use a different transport. The chain construction rules in Section 8.1 are normative regardless of transport.*

Agents POST to the issuer's `endpoints.token_exchange` URL with a JSON body:

```json
{
  "subject_token": "<hwt-token being delegated>",
  "subject_token_type": "hwt",
  "actor_token": "<hwt-token of the requesting agent>",
  "actor_token_type": "hwt",
  "audience": "https://target-service.example.com",
  "scope": "optional-scope-restriction"
}
```

| Field | Required | Description |
|---|---|---|
| `subject_token` | Yes | The token being delegated. |
| `subject_token_type` | Yes | MUST be `"hwt"`. |
| `actor_token` | Yes | The token of the agent requesting the exchange. |
| `actor_token_type` | Yes | MUST be `"hwt"`. |
| `audience` | Yes | The intended recipient of the derived token. |
| `scope` | No | Optional restriction on the derived token's authorization. Scope semantics are schema-defined. |

On success (HTTP 200):

```json
{
  "token": "<new-hwt-token>",
  "token_type": "hwt",
  "expires_in": 3600
}
```

Common error responses:

| HTTP Status | Meaning |
|---|---|
| 400 | Malformed request or missing required fields |
| 401 | `actor_token` invalid or expired |
| 403 | Exchange not permitted — subject token does not authorize this delegation, or resulting `del` chain would exceed `max_delegation_depth` |
| 422 | `subject_token` invalid or expired |
| 503 | Exchange service temporarily unavailable |

---

## 9. Caching Model

HWT well-known documents use standard HTTP caching [RFC 9111].

**Normal operation:**
- Verifiers cache `hwt-keys.json` and `hwt.json` per `Cache-Control: max-age` headers
- Conditional revalidation via `ETag` is supported

**Forced invalidation:**
- Unknown `kid` triggers immediate re-fetch of `hwt-keys.json`, bypassing cache, subject to rate limiting
- This is the only mandatory cache bypass defined by the protocol

---

## 10. Standard Registry

There is no standard registry defined by this protocol. Any registry of 
`authz` schema identifiers, codec identifiers, or related vocabulary is a community 
artifact, separate from this specification. CONVENTIONS.md contains a proposed set 
of community conventions including example schema identifiers (e.g., `RBAC/1.0.2`) 
and codec identifiers — these are starting points for interoperability, not 
normative protocol elements.

Private schemas and codecs use URL form (origin-relative or absolute) and require 
no registration anywhere. The URL form is the unambiguous signal that a value is 
private and self-describing.

Versioned identifiers (e.g., `RBAC/1.0.2`) used by community convention carry an 
implicit immutability expectation: a published identifier should not be redefined. 
That expectation lives in the community, not in this spec.

---

## 11. Security Considerations

### 11.1 DNS and Transport Security

HWT verification depends on DNS resolving correctly and TLS connecting to the issuer's domain. The trust anchor is the issuer's domain, not a central authority.

**The concrete DNS attack.** An attacker who controls DNS for an issuer's domain can serve fabricated well-known endpoints with their own key material and issue tokens that verify correctly against those keys. Signature verification succeeds — the trust failure is DNS, not cryptography. This attack is not unique to HWT; it applies to any domain-based trust model including TLS certificate issuance itself.

**Mitigations in approximate order of effectiveness:**

1. Pre-register trusted issuer origins at the verifier and reject tokens from unknown issuers. This eliminates the unknown-origin attack surface entirely — a fabricated origin cannot be registered without out-of-band action.
2. DNSSEC validation at the verifier (RFC 4033). Cryptographically authenticated DNS responses prevent zone substitution attacks.
3. DANE (RFC 6698). Binds TLS certificates to DNS records, closing the gap between DNSSEC and TLS certificate trust.
4. Certificate Transparency monitoring for issuer domains.
5. Short token lifetimes — limits the replay window if compromise is detected after the fact.

Verifiers that accept tokens from previously-unknown issuers MUST understand they are trusting the DNS and TLS path to that issuer at the moment of first key fetch. This is a deliberate operational choice, not a protocol default. The recommended posture for production deployments is pre-registered issuers.

- Issuers serving production workloads SHOULD enable DNSSEC
- Verifiers SHOULD perform DNSSEC validation when resolver support is available
- All well-known endpoints and all declared `endpoints` values MUST be served over HTTPS. HTTP is not permitted anywhere in the protocol.
- TLS certificate validation is mandatory

### 11.2 SSRF via `iss` Field

In the unknown-issuer flow, the verifier constructs a URL from the `iss` value in the token payload and fetches from it. This is a Server-Side Request Forgery risk: an attacker can craft a token with `iss` pointing to an internal network endpoint, a cloud metadata service (e.g., `169.254.169.254`), or any other host reachable from the verifier but not intended as a public issuer.

Applications implementing unknown-issuer key fetch MUST restrict fetch targets. At minimum: block RFC 1918 addresses, link-local addresses (169.254.0.0/16, fe80::/10), loopback, and unresolvable private hostnames. The pre-registered issuer posture eliminates this risk entirely, since key fetch targets are determined at configuration time, not by token content.

### 11.3 Replay Attacks

Tokens are bearer credentials. A captured valid token can be replayed until expiry. Mitigations: short token expiry limits the replay window; `aud` binding prevents cross-service replay; `tid` enables application-level replay detection when the application maintains a seen-token store. For deployments where token theft is a primary threat model, DPoP (Demonstrating Proof-of-Possession, RFC 9449) is a compatible extension mechanism. DPoP binds tokens to an ephemeral key pair held by the legitimate client — a stolen token cannot be used without the corresponding private key. No HWT format changes are required; DPoP operates as an additional HTTP header alongside the HWT bearer token.

### 11.4 Confused Deputy

When `aud` is absent, any verifier may accept the token. Services that issue tokens for specific recipients SHOULD declare `aud_required: true` and include `aud` in all issued tokens.

### 11.5 Key Compromise

If a signing key is compromised:
1. Immediately remove the compromised key from `hwt-keys.json`
2. Issue a new signing key
3. Re-issue tokens for active sessions

All tokens signed under the compromised key become unverifiable once the key is removed from `hwt-keys.json`. This is the structural revocation mechanism for key compromise — no revocation list required.

### 11.6 Algorithm Downgrade

Verifiers MUST use the algorithm declared in the JWKS `alg` field. Failure to verify with the declared algorithm means the token is invalid — not a signal to try alternatives.

### 11.7 `kid` Flood

Presenting tokens with unknown `kid` values forces verifier re-fetches. Rate-limit forced re-fetches per origin (recommended: one per 60 seconds maximum).

### 11.8 Delegation Chain Depth

Unbounded `del` arrays create opportunities for DoS through deep chain traversal. The depth limit bounds this. Verifiers MUST enforce the limit as the first step of delegation chain verification, before any structural processing of chain entries. The verifier's local depth cap cannot be increased by issuer declaration.

### 11.9 Cross-Protocol Signature Confusion

A theoretical risk exists if an application uses the same key material across HWT and another protocol that constructs an identical canonical signed form. Preventing this is the application's responsibility: do not share private keys across protocol contexts. The `hwt` literal in field 1 of the token wire format is the protocol discriminator at the parsing layer; key material should be scoped to match.

---

## 12. Verification Algorithm

A conforming verifier MUST execute the following steps in order. Failure at any step means the token is invalid.

**Issuer allowlist check (RECOMMENDED pre-condition).** Before step 1, implementations SHOULD restrict accepted issuers to a pre-configured allowlist. Tokens from issuers not in the allowlist SHOULD be rejected immediately. This eliminates the unknown-origin attack surface described in Sections 11.1 and 11.2 and converts verification into a fully local operation after initial key loading. The algorithm below handles the unknown-issuer case for deployments that require it; implementers should understand the DNS trust dependency that case entails.

Steps 1–7 are cryptographic verification. Steps 8–9 are structural authorization validation. Step 10 is the handoff to the consuming application. Steps 11–13 are delegation chain verification, required when `del` is present.

A cryptographically valid token may fail structural authorization validation — these are distinct outcomes. HTTP response guidance: cryptographic failure → 401; structural authorization failure → 403; issuer unreachable → 503.

**Cryptographic verification:**

1. Split on `.` — confirm exactly six fields; confirm field 1 is literal `hwt`
2. Check `expires` (field 4) against current time — reject if past. Implementations MAY apply a clock skew tolerance not to exceed 5 minutes.
3. Decode `payload` (field 6) using the codec declared in `format` (field 5) — reject if codec is unsupported or decode fails
4. Extract `iss` from decoded payload — reject if absent or not a valid HTTPS URL
5. Fetch `/.well-known/hwt-keys.json` from the `iss` origin (from cache if valid). For pre-registered issuers, this is a local cache lookup with no network call. For unknown-issuer deployments, see Section 11.2 for SSRF requirements before fetching.
6. Locate the key where `kid` matches field 3, scoped to the `iss` origin's JWKS. If not found: re-fetch bypassing cache, subject to rate limit. If still not found: reject.
7. Verify the signature (field 2) over the canonical signed input (Section 2.1) using the `alg` declared for the matched key — reject if invalid

**Structural authorization validation:**

8. Fetch `/.well-known/hwt.json` from the `iss` origin (from cache if valid), 
   if available. When absent or unreachable, apply field defaults as specified 
   in Section 7 and continue.
9. If `aud` present as string: confirm it matches the verifier's own canonical identifier — reject if not. If `aud` present as array: reject if `aud_array_permitted` is not `true` in `hwt.json`; otherwise confirm at least one value matches — reject if none match. If `aud_required: true` in `hwt.json`: reject if `aud` absent.

**Application handoff:**

10. Return the verified, decoded payload to the consuming application. Evaluation of `authz` values against application requirements is the consuming application's responsibility. Application-level state checks — including any token invalidation the application tracks via `tid` or other means — occur here.

**Delegation chain verification** (required when `del` is present):

11. Confirm `del` array length does not exceed the applicable limit: the most restrictive of the verifier's own local cap (MUST be enforced; recommended maximum: 10), any `max_delegation_depth` declared in the issuer's `hwt.json` (can only decrease the effective limit, never increase it), and the protocol default of 10 when neither is configured. This check MUST occur before any further processing of chain entries. Verifiers SHOULD check for cycles (duplicate `iss`+`sub` pairs across the chain and outer token) at this step.
12. For each entry in `del` (in order, root first): confirm `iss` is a valid HTTPS URL and `sub` is present. The outer token's signature guarantees the chain contents have not been tampered with after issuance; there is no per-entry cryptographic re-verification of original delegation tokens. Application-level state checks for individual chain links are the consuming application's responsibility.
13. Delegation chain is structurally valid. Continue.

---

## 13. Out of Scope

The following are explicitly outside this protocol. Being explicit about these boundaries is intentional: narrower commitments produce more precise security claims, more predictable library behavior, and clearer integrator responsibility.

- **Token state and revocation.** Whether a token has been invalidated after issuance is not a property of the signed byte string. HWT defines what a token structurally guarantees; it does not define session lifecycle. Applications requiring immediate invalidation maintain their own state store and check it in the application layer. Short token lifetimes are the primary mechanism for bounding exposure window. This is a deliberate scope boundary, not a gap.
- **Jurisdiction and data sovereignty.** Outside the protocol scope. Applications that need jurisdiction context embed it in `authz` via a private schema or carry it through other application-defined payload fields. See CONVENTIONS.md for patterns.
- **Token issuance.** How a principal obtains a token is between the principal and the issuing service. HWT defines verification, not issuance. A natural complement for the issuance side is WebAuthn/FIDO2 (W3C), which is also domain-sovereign and requires no central provider.
- **Schema content.** What RBAC roles mean, what private schemas contain — negotiated between issuers and consuming applications.
- **Authorization evaluation.** Whether a verified token's `authz` values authorize a specific action is the consuming application's decision. Policy evaluation engines such as OPA (Open Policy Agent) or Casbin are natural complements.
- **Air-gapped systems.** Environments without network access require a different trust architecture.
- **Large payloads.** HWT is optimized for small to medium tokens within HTTP header size constraints.
- **Authentication.** HWT describes the contract passed after authentication. How identity is established before token issuance is out of scope.
- **Agent discovery.** How an agent locates another agent by capability is a separate primitive. Service meshes (Istio, Linkerd) handle this within a cluster; W3C DIDs are a compatible complement for cross-domain agent identity.
- **Agent-to-agent communication protocol.** HWT defines credentials, not session or messaging protocols.
- **Session binding.** Binding tokens to a specific client session is outside the core protocol. DPoP (RFC 9449) is a compatible external mechanism; see Section 11.3.

---

## Appendix A: Non-Normative Deployment Conventions

*This appendix is informational. Nothing here is required for protocol conformance.*

### A.1 Token Lifetime by Context

Token expiry is the primary security control for bounding token exposure. Shorter lifetimes reduce replay risk without any external infrastructure dependency. Applications requiring finer-grained invalidation implement that in the application layer.

| Context | Suggested range | Reasoning |
|---|---|---|
| Financial API | 5–15 minutes | Tight exposure window; re-authentication acceptable |
| General API | 1–4 hours | Balanced availability and exposure |
| Internal service | Up to 24 hours | High availability tolerance |
| Long-lived agent delegation | Up to 7 days | Explicit operational choice; requires application-layer invalidation strategy |

### A.2 `tid` Usage

`tid` is an opaque string available for application use. See CONVENTIONS.md for usage patterns including replay detection, audit trail construction, and application-level state management.

### A.3 Audience Binding

Services that accept tokens from multiple issuers should declare a stable, canonical identifier as their expected audience value and configure verifiers to enforce `aud` matching. The verifier's canonical identifier for `aud` matching is its publicly-reachable HTTPS origin URL, configured explicitly at deployment time — not derived from request context. Issuers should set `aud_required: true` in `hwt.json` for any deployment where audience ambiguity is a concern.

### A.4 Key Management Cadence

Key rotation on a regular schedule (quarterly is common) reduces the exposure window of any individual key without requiring a security incident to trigger rotation. Overlap new and old keys for at least the `hwt-keys.json` cache TTL before switching active signing to the new key.

### A.5 `authz_schemas` Discovery

Publishing `authz_schemas` in `hwt.json` enables consuming applications to enumerate expected schemas before processing tokens. Applications may use this to fail fast when receiving an unknown schema rather than encountering the unknown value inside a token.

### A.6 Well-Known Endpoint Distribution

For production deployments, serving well-known endpoints through a geographically distributed CDN with long `max-age` values reduces verifier latency and improves resilience to regional issuer outages. The protocol's cache-first design makes this a natural fit.

### A.7 Issuer Allowlist Patterns

**Static configuration** — trusted issuers declared in application config at deploy time. Keys loaded at startup. No runtime key fetch for known issuers. Appropriate for closed systems with a known, stable set of issuers.

**Dynamic registration with allowlist** — new issuers can be added through an administrative API, triggering a key pre-load at registration time. Unknown-origin tokens are still rejected. Appropriate for platforms that onboard new issuers at runtime but want to maintain the security properties of pre-registration.

Both patterns are compatible with the verification algorithm in Section 12 and eliminate the SSRF risk described in Section 11.2. The unknown-issuer cold-fetch path exists for permissive deployments; it is not the recommended default.

---

## Appendix B: Open Items

| Item | Status | Notes |
|---|---|---|
| IANA well-known URI registration | Pending | Administrative. `hwt.json` and `hwt-keys.json` pending IANA registration under RFC 8615. |
| Token exchange scope semantics | Deferred | `scope` field defined structurally in Section 8; content semantics are schema-defined. Community conventions may address standard scope schemas in CONVENTIONS.md. |
| DPoP binding | Compatible extension | RFC 9449 DPoP is compatible for proof-of-possession deployments. No spec changes required; library support TBD. |

---

## Appendix C: Full Payload Examples

**Blog editor, single RBAC scheme:**

```json
{
  "iss": "https://myblog.com",
  "sub": "user:4503599627370495",
  "aud": "https://api.myblog.com",
  "tid": "a1b2c3d4e5f6",
  "iat": 1743900000,
  "authz": {
    "scheme": "RBAC/1.0.2",
    "roles": ["editor"],
    "capabilities": ["edit_posts", "upload_files", "publish_posts"]
  }
}
```

**Service account, multi-scheme authz:**

```json
{
  "iss": "https://platform.example.com",
  "sub": "svc:data-pipeline",
  "tid": "svc-tok-9a8b",
  "authz": [
    { "scheme": "RBAC/1.0.2", "roles": ["service"] },
    { "scheme": "/schemas/data-access/v2", "datasets": ["analytics", "reporting"] }
  ]
}
```

**Delegated agent token — two-hop chain:**

```json
{
  "iss": "https://agent-b.example.com",
  "sub": "svc:agent-b",
  "aud": "https://api.target-service.com",
  "tid": "derived-tok-7c8d",
  "iat": 1743900000,
  "authz": {
    "scheme": "RBAC/1.0.2",
    "roles": ["editor"]
  },
  "del": [
    {
      "iss": "https://auth.example.com",
      "sub": "user:4503599627370495",
      "tid": "root-tok-a1b2"
    },
    {
      "iss": "https://agent-a.example.com",
      "sub": "svc:agent-a",
      "tid": "mid-tok-c3d4"
    }
  ]
}
```

**Broad-portability token, no `aud`, alias subject:**

```json
{
  "iss": "https://auth.example.com",
  "sub": "user@example.com",
  "iat": 1743900000,
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["member"] }
}
```

---

## Appendix D: Relationship to Existing Standards

| Standard | Relationship |
|---|---|
| JWT (RFC 7519) | Key naming conventions (`iss`, `sub`, `aud`, `iat`) preserved where semantically equivalent. `jti` renamed `tid`. Token format, signed input canonicalization, and verification model are distinct. |
| JWKS (RFC 7517) | HWT uses JWKS format for key publication. `alg` per key is mandatory. Multiple concurrent keys in a single document are explicitly supported. |
| OIDC Discovery | `hwt.json` serves a similar discovery function without assuming a centralized identity provider. Endpoint declarations grouped under `endpoints` object. |
| OAuth 2.0 Token Exchange (RFC 8693) | Section 8 token exchange is informed by RFC 8693 request/response structure adapted to HWT token types. The HTTP transport shape is informational; the chain construction rules in Section 8.1 are normative. |
| HTTP Caching (RFC 9111) | HWT caching follows standard HTTP semantics throughout. |
| DNSSEC (RFC 4033) | Recommended for production issuers. See Section 11.1. |
| DANE (RFC 6698) | Binds TLS certificates to DNS records; compatible with HWT deployment. Recommended alongside DNSSEC for production issuers. |
| DPoP (RFC 9449) | Proof-of-possession extension. Compatible with HWT — no format changes required. Recommended for deployments where token theft is a primary threat. See Section 11.3 and Appendix B. |
| RFC 8615 | Well-known URI registration. `hwt.json` and `hwt-keys.json` pending IANA registration. |
| WebAuthn / FIDO2 (W3C) | Natural complement for the token issuance side. Also domain-sovereign; requires no central provider. Token issuance is out of scope for HWT; WebAuthn addresses the authentication step that precedes it. |
| OPA / Casbin | Policy evaluation engines that consume verified HWT claims as input. Authorization evaluation is out of scope for HWT. |
| W3C DIDs | Not used by the protocol. HWT `sub` is issuer-scoped. For cross-issuer portable agent identity, DID Documents are a compatible separate mechanism. |

---

*HWT Protocol Draft v0.7 — https://github.com/hwt-protocol/*
*Licensed under the Apache License, Version 2.0*
*Copyright 2026 Jim Montgomery and HWT Contributors*  
*SPDX-License-Identifier: Apache-2.0*

