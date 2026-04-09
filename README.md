# Hash Web Token (HWT)

[canonical documentation](https://hwtprotocol.com/hwt-protocol) [issues](https://github.com/hwt-protocol/hwt-protocol/issues) [discussions](https://github.com/hwt-protocol/hwt-protocol/discussions)

A stateless, network-native token protocol for cross-domain delegation.

Tokens are signed by an issuing origin, verifiable by any party that can reach the issuer's domain, and carry authorization context either literally or by reference.

**Status:** Draft v0.7.

---

## What HWT does

- Defines a compact, dot-separated token format with structural expiry
- Provides deterministic signed input canonicalization so any verifier can reconstruct and check the signature
- Specifies key discovery via JWKS at `/.well-known/hwt-keys.json`
- Defines origin metadata discovery at `/.well-known/hwt.json`
- Provides a structured delegation chain (`del`) with normative chain construction rules
- Defines a codec registry (`j` / JSON is the baseline; community codecs in CONVENTIONS.md)
- Follows standard HTTP caching throughout — no special cache infrastructure needed

## What HWT does not address

- **Token state and revocation.** Whether a token has been invalidated after issuance is not a property of the signed byte string. Applications requiring immediate invalidation maintain their own state store. Short token lifetimes are the primary exposure-bounding mechanism.
- **Jurisdiction and data sovereignty.** No jurisdiction field at the protocol level. Applications embed jurisdiction context in `authz` via a private schema.
- **Token issuance.** HWT defines verification, not issuance. WebAuthn/FIDO2 is a natural complement for the issuance side.
- **Schema content.** What RBAC roles mean, what private schemas contain — negotiated between issuers and consuming applications.
- **Authorization evaluation.** Whether a verified token's `authz` values authorize a specific action is the consuming application's decision. OPA and Casbin are natural complements.
- **Air-gapped systems.** No network access means a different trust architecture is required.
- **Large payloads.** HWT is optimized for small to medium tokens within HTTP header size constraints.
- **Authentication.** HWT describes the contract passed after authentication. How identity is established before token issuance is out of scope.
- **Agent discovery.** How an agent locates another agent by capability is a separate primitive.
- **Agent-to-agent communication protocol.** HWT defines credentials, not session or messaging protocols.
- **Session binding.** DPoP (RFC 9449) is a compatible external mechanism for proof-of-possession deployments.

---

## Token format

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

Six dot-separated fields. The dot is the field separator and MUST NOT appear in `key-id`, `format`, or top-level name.

`format` declares the codec. `j` (JSON) is the only required codec. Additional community codecs are documented in CONVENTIONS.md.

Expiry is structural — a token past its `expires` value is unconditionally invalid regardless of signature validity.

---

## Delegation chain example

Three hops: a user at an identity provider → an orchestration agent → a worker service → the final API target.

**Root token** (held by the user, issued by `auth.example.com`):

```json
{
  "iss": "https://auth.example.com",
  "sub": "user:4503599627370495",
  "tid": "root-a1b2",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] }
}
```

**Intermediate token** (issued by `agent.example.com` for the worker):

```json
{
  "iss": "https://agent.example.com",
  "sub": "svc:orchestrator",
  "aud": "https://worker.example.com",
  "tid": "mid-c3d4",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] },
  "del": [
    { "iss": "https://auth.example.com", "sub": "user:4503599627370495", "tid": "root-a1b2" }
  ]
}
```

**Final derived token** (issued by `worker.example.com` for the target API):

```json
{
  "iss": "https://worker.example.com",
  "sub": "svc:worker",
  "aud": "https://api.target.example.com",
  "tid": "final-e5f6",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] },
  "del": [
    { "iss": "https://auth.example.com", "sub": "user:4503599627370495", "tid": "root-a1b2" },
    { "iss": "https://agent.example.com", "sub": "svc:orchestrator", "tid": "mid-c3d4" }
  ]
}
```

The target API verifies the final token's signature against `worker.example.com`'s published keys, confirms the `del` chain is structurally intact (it is covered by the outer signature), and can trace the full authorization lineage back to the original user. Authorization was not escalated at any hop — `roles: ["editor"]` throughout.

---

## Key discovery

```
GET https://{issuer-domain}/.well-known/hwt-keys.json
```

JWKS format (RFC 7517). `alg` per key is mandatory. Multiple concurrent keys supported for rotation.

## Origin metadata

```
GET https://{issuer-domain}/.well-known/hwt.json
```

Declares the issuer URL, protocol version, supported authz schemas, audience configuration, delegation depth limit, and endpoint URLs.

---

## Trust model

The trust anchor is the issuer's domain, not a central authority. Verification depends on DNS resolving correctly and TLS connecting to the issuer's well-known endpoint. The recommended production posture is a pre-registered issuer allowlist — verifiers reject tokens from issuers not in the list, converting key discovery into a local cache lookup with no runtime DNS dependency.

---

## Documentation

- [SPEC.md](SPEC.md) — normative specification
- [CONTRIBUTING.md](CONTRIBUTING.md) — how to propose changes
- [CONVENTIONS.md](CONVENTIONS.md) — community usage patterns, codec examples, RBAC recipes, delegation walkthroughs, jurisdiction handling
- [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) reference and community implementations

## Implementations

**Reference library**
- [hwtr-js](../hwtr-js) — JavaScript reference implementation

**Demos**
- [hwt-demo](../hwt-demo) — JavaScript working demos including the following (Deno and Cloudflare Workers deployments, etc).
- [demo-agent-chain.js](../hwt-demo/demo-agent-chain.js) — AI agent delegation chain
- [demo-del-verify.js](../hwt-demo/demo-del-verify.js) — Full del[] chain verification with revoked link detection
- [demo-multiparty.js](../hwt-demo/demo-multiparty.js) — Multi-party bilateral verification
- [demo-federation.js](../hwt-demo/demo-federation.js) — Federated gateway, multiple independent issuers
- [demo-mesh.js](../hwt-demo/demo-mesh.js) — Microservice mesh without a service mesh
- [demo-partner-api.js](../hwt-demo/demo-partner-api.js) — Cross-organization partner API access
- [demo-edge.js](../hwt-demo/demo-edge.js) — Edge and offline verification

---

## License

Apache License 2.0

