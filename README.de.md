# Hash Web Token (HWT)

[Offizielle Dokumentation](https://hwtprotocol.com/hwt-protocol) [Issues](https://github.com/hwt-protocol/hwt-protocol/issues) [Diskussionen](https://github.com/hwt-protocol/hwt-protocol/discussions)

Ein zustandsloses, netzwerknatives Token-Protokoll für domänenübergreifende Delegation.

Tokens werden von einer ausstellenden Origin signiert, sind von jeder Partei verifizierbar, die die Domain des Issuers (Aussteller) erreichen kann, und tragen Autorisierungskontext entweder direkt oder als Referenz.

**Status:** Draft v0.7.

---

## Was HWT tut

- Definiert ein kompaktes, durch Punkte getrenntes Token-Format mit struktureller Ablaufzeit
- Bietet deterministische kanonische Signatureingabe, sodass jeder Verifier (Prüfer) die Signatur rekonstruieren und prüfen kann
- Spezifiziert Key Discovery (Schlüsselermittlung) über JWKS unter `/.well-known/hwt-keys.json`
- Definiert Origin-Metadaten-Discovery unter `/.well-known/hwt.json`
- Bietet eine strukturierte Delegation Chain (Delegierungskette) (`del`) mit normativen Kettenaufbauregeln
- Definiert eine Codec-Registry (`j` / JSON ist die Baseline; Community-Codecs in CONVENTIONS.md)
- Verwendet durchgängig Standard-HTTP-Caching – keine spezielle Cache-Infrastruktur erforderlich

## Was HWT nicht adressiert

- **Token-Zustand und Revocation (Widerruf).** Ob ein Token nach der Ausstellung ungültig gemacht wurde, ist keine Eigenschaft des signierten Byte-Strings. Anwendungen, die eine sofortige Invalidierung benötigen, pflegen ihren eigenen Zustandsspeicher. Kurze Token-Lebensdauern sind der primäre Mechanismus zur Risikobegrenzung.
- **Zuständigkeit und Datensouveränität.** Kein Jurisdiktionsfeld auf Protokollebene. Anwendungen betten Jurisdiktionskontext über ein privates Schema in `authz` ein.
- **Token-Ausstellung.** HWT definiert Verifizierung, nicht Ausstellung. WebAuthn/FIDO2 ist eine natürliche Ergänzung für die Ausstellungsseite.
- **Schema-Inhalt.** Was RBAC-Rollen bedeuten, was private Schemata enthalten – wird zwischen Issuern und verbrauchenden Anwendungen ausgehandelt.
- **Autorisierungsauswertung.** Ob die `authz`-Werte eines verifizierten Tokens eine bestimmte Aktion autorisieren, entscheidet die verbrauchende Anwendung. OPA und Casbin sind natürliche Ergänzungen.
- **Air-Gapped-Systeme.** Kein Netzwerkzugang erfordert eine andere Vertrauensarchitektur.
- **Große payloads.** HWT ist für kleine bis mittlere Tokens innerhalb der HTTP-Header-Größenbeschränkungen optimiert.
- **Authentifizierung.** HWT beschreibt den nach der Authentifizierung übergebenen Vertrag. Wie Identität vor der Token-Ausstellung etabliert wird, liegt außerhalb des Geltungsbereichs.
- **Agenten-Discovery.** Wie ein Agent einen anderen Agenten anhand von Fähigkeiten findet, ist ein separates Primitive.
- **Agenten-zu-Agenten-Kommunikationsprotokoll.** HWT definiert Credentials, keine Session- oder Messaging-Protokolle.
- **Session-Bindung.** DPoP (RFC 9449) ist ein kompatibler externer Mechanismus für proof-of-possession-Deployments.

---

## Token-Format

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

Sechs durch Punkte getrennte Felder. Der Punkt ist der Feldtrenner und DARF NICHT in `key-id`, `format` oder im Top-Level-Namen erscheinen.

`format` deklariert den Codec. `j` (JSON) ist der einzige erforderliche Codec. Weitere Community-Codecs sind in CONVENTIONS.md dokumentiert.

Ablaufzeit ist strukturell – ein Token, der seinen `expires`-Wert überschritten hat, ist bedingungslos ungültig, unabhängig von der Signaturvalidität.

---

## Beispiel einer Delegation Chain

Drei Hops: ein Benutzer bei einem Identity-Provider → ein Orchestrierungsagent → ein Worker-Service → das finale API-Ziel.

**Root-Token** (gehalten vom Benutzer, ausgestellt von `auth.example.com`):

```json
{
  "iss": "https://auth.example.com",
  "sub": "user:4503599627370495",
  "tid": "root-a1b2",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] }
}
```

**Zwischentoken** (ausgestellt von `agent.example.com` für den Worker):

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

**Finaler abgeleiteter Token** (ausgestellt von `worker.example.com` für die Ziel-API):

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

Die Ziel-API verifiziert die Signatur des finalen Tokens gegen die veröffentlichten Schlüssel von `worker.example.com`, bestätigt, dass die `del`-Kette strukturell intakt ist (sie ist durch die äußere Signatur abgedeckt), und kann die vollständige Autorisierungsherkunft bis zum ursprünglichen Benutzer zurückverfolgen. Die Autorisierung wurde bei keinem Hop eskaliert – durchgehend `roles: ["editor"]`.

---

## Key Discovery

```
GET https://{issuer-domain}/.well-known/hwt-keys.json
```

JWKS-Format (RFC 7517). `alg` pro Schlüssel ist obligatorisch. Mehrere gleichzeitige Schlüssel für Rotation unterstützt.

## Origin-Metadaten

```
GET https://{issuer-domain}/.well-known/hwt.json
```

Deklariert die Issuer-URL, Protokollversion, unterstützte authz-Schemata, Audience-Konfiguration, Delegationstiefengrenze und Endpunkt-URLs.

---

## Vertrauensmodell

Der Vertrauensanker ist die Domain des Issuers, keine zentrale Autorität. Die Verifizierung setzt voraus, dass DNS korrekt auflöst und TLS zum well-known-Endpunkt des Issuers verbindet. Die empfohlene Produktionskonfiguration ist eine vorab registrierte Issuer-allowlist – Verifier lehnen Tokens von Issuern ab, die nicht in der Liste stehen, und wandeln Key Discovery in eine lokale Cache-Abfrage ohne Laufzeit-DNS-Abhängigkeit um.

---

## Dokumentation

- [SPEC.md](SPEC.md) — normative Spezifikation
- [CONTRIBUTING.md](CONTRIBUTING.md) — wie Änderungen vorgeschlagen werden
- [CONVENTIONS.md](CONVENTIONS.md) — Community-Nutzungsmuster, Codec-Beispiele, RBAC-Rezepte, Delegation-Walkthroughs, Jurisdiktionsbehandlung
- [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) Referenz- und Community-Implementierungen

## Implementierungen

**Referenzbibliothek**
- [hwtr-js](../hwtr-js/) — JavaScript-Referenzimplementierung

**Demos**
- [hwt-demo](../hwt-demo/) — JavaScript-Demo-Implementierungen, einschließlich der folgenden (Deno- und Cloudflare-Workers-Deployments usw.).
- [demo-agent-chain.js](../hwt-demo/demo-agent-chain.js) — KI-Agenten-Delegation Chain
- [demo-del-verify.js](../hwt-demo/demo-del-verify.js) — Vollständige del[]-Kettenverifizierung mit Erkennung widerrufener Links
- [demo-multiparty.js](../hwt-demo/demo-multiparty.js) — Bilaterale Verifizierung mehrerer Parteien
- [demo-federation.js](../hwt-demo/demo-federation.js) — Föderiertes Gateway, mehrere unabhängige Issuer
- [demo-mesh.js](../hwt-demo/demo-mesh.js) — Microservice-Mesh ohne Service-Mesh
- [demo-partner-api.js](../hwt-demo/demo-partner-api.js) — Partnerübergreifender API-Zugriff
- [demo-edge.js](../hwt-demo/demo-edge.js) — Edge- und Offline-Verifizierung

---

## Lizenz

Apache License 2.0

