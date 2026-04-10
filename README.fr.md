# Hash Web Token (HWT)

[documentation officielle](https://hwtprotocol.com/hwt-protocol) [issues](https://github.com/hwt-protocol/hwt-protocol/issues) [discussions](https://github.com/hwt-protocol/hwt-protocol/discussions)

Un protocole de token sans état, natif au réseau, pour la délégation inter-domaines.

Les tokens sont signés par une origine émettrice (issuer), vérifiables par toute partie pouvant atteindre le domaine de l'émetteur, et portent le contexte d'autorisation littéralement ou par référence.

**Statut :** Brouillon v0.7.

---

## Ce que HWT fait

- Définit un format de token compact, séparé par des points, avec expiration structurelle
- Fournit une canonicalisation déterministe des entrées signées pour que tout vérificateur (verifier) puisse reconstruire et vérifier la signature
- Spécifie la découverte de clé (key discovery) via JWKS sur `/.well-known/hwt-keys.json`
- Définit la découverte de métadonnées d'origine sur `/.well-known/hwt.json`
- Fournit une chaîne de délégation (delegation chain) structurée (`del`) avec des règles normatives de construction de chaîne
- Définit un registre de codecs (`j` / JSON est la référence ; codecs communautaires dans CONVENTIONS.md)
- Suit le cache HTTP standard partout — aucune infrastructure de cache spéciale requise

## Ce que HWT ne couvre pas

- **État du token et révocation (revocation).** Le fait qu'un token ait été invalidé après émission n'est pas une propriété de la chaîne d'octets signée. Les applications nécessitant une invalidation immédiate maintiennent leur propre dépôt d'état. La courte durée de vie des tokens est le mécanisme principal de limitation de l'exposition.
- **Juridiction et souveraineté des données.** Aucun champ de juridiction au niveau du protocole. Les applications intègrent le contexte de juridiction dans `authz` via un schéma privé.
- **Émission de tokens.** HWT définit la vérification, pas l'émission. WebAuthn/FIDO2 est un complément naturel pour la partie émission.
- **Contenu des schémas.** La signification des rôles RBAC, le contenu des schémas privés — négociés entre émetteurs et applications consommatrices.
- **Évaluation des autorisations.** La question de savoir si les valeurs `authz` d'un token vérifié autorisent une action spécifique relève de la décision de l'application consommatrice. OPA et Casbin sont des compléments naturels.
- **Systèmes déconnectés.** L'absence d'accès réseau impose une architecture de confiance différente.
- **Payloads volumineux.** HWT est optimisé pour les tokens de taille petite à moyenne dans les contraintes de taille d'en-tête HTTP.
- **Authentification.** HWT décrit le contrat transmis après authentification. La manière dont l'identité est établie avant l'émission du token est hors périmètre.
- **Découverte d'agents.** La façon dont un agent localise un autre agent par capacité est une primitive distincte.
- **Protocole de communication entre agents.** HWT définit des credentials, pas des protocoles de session ou de messagerie.
- **Liaison de session.** DPoP (RFC 9449) est un mécanisme externe compatible pour les déploiements proof-of-possession.

---

## Format du token

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

Six champs séparés par des points. Le point est le séparateur de champs et NE DOIT PAS apparaître dans `key-id`, `format` ou le nom de niveau supérieur.

`format` déclare le codec. `j` (JSON) est le seul codec requis. Les codecs communautaires supplémentaires sont documentés dans CONVENTIONS.md.

L'expiration est structurelle — un token dépassant sa valeur `expires` est inconditionnellement invalide, quelle que soit la validité de la signature.

---

## Exemple de chaîne de délégation

Trois sauts : un utilisateur auprès d'un fournisseur d'identité → un agent d'orchestration → un service worker → la cible API finale.

**Token racine** (détenu par l'utilisateur, émis par `auth.example.com`) :

```json
{
  "iss": "https://auth.example.com",
  "sub": "user:4503599627370495",
  "tid": "root-a1b2",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] }
}
```

**Token intermédiaire** (émis par `agent.example.com` pour le worker) :

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

**Token dérivé final** (émis par `worker.example.com` pour l'API cible) :

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

L'API cible vérifie la signature du token final contre les clés publiées de `worker.example.com`, confirme que la chaîne `del` est structurellement intacte (elle est couverte par la signature externe), et peut retracer l'ensemble de la lignée d'autorisation jusqu'à l'utilisateur d'origine. L'autorisation n'a pas été élevée à aucun saut — `roles: ["editor"]` tout au long de la chaîne.

---

## Découverte de clé

```
GET https://{issuer-domain}/.well-known/hwt-keys.json
```

Format JWKS (RFC 7517). `alg` par clé est obligatoire. Plusieurs clés simultanées supportées pour la rotation.

## Métadonnées d'origine

```
GET https://{issuer-domain}/.well-known/hwt.json
```

Déclare l'URL de l'émetteur, la version du protocole, les schémas authz supportés, la configuration de l'audience, la limite de profondeur de délégation et les URL des points de terminaison.

---

## Modèle de confiance

L'ancre de confiance est le domaine de l'émetteur, pas une autorité centrale. La vérification dépend de la résolution correcte du DNS et de la connexion TLS au point de terminaison well-known de l'émetteur. La posture de production recommandée est un allowlist d'émetteurs pré-enregistrés — les vérificateurs rejettent les tokens d'émetteurs absents de la liste, convertissant la découverte de clé en une recherche dans un cache local sans dépendance DNS à l'exécution.

---

## Documentation

- [SPEC.md](SPEC.md) — spécification normative
- [CONTRIBUTING.md](CONTRIBUTING.md) — comment proposer des modifications
- [CONVENTIONS.md](CONVENTIONS.md) — modèles d'utilisation communautaires, exemples de codecs, recettes RBAC, guides de délégation, gestion des juridictions
- [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) implémentations de référence et communautaires

## Implémentations

**Bibliothèque de référence**
- [hwtr-js](../hwtr-js/) — implémentation de référence JavaScript

**Démos**
- [hwt-demo](../hwt-demo/) — démos JavaScript fonctionnelles incluant ce qui suit (déploiements Deno et Cloudflare Workers, etc.).
- [demo-agent-chain.js](../hwt-demo/demo-agent-chain.js) — chaîne de délégation d'agents IA
- [demo-del-verify.js](../hwt-demo/demo-del-verify.js) — vérification complète de chaîne del[] avec détection de lien révoqué
- [demo-multiparty.js](../hwt-demo/demo-multiparty.js) — vérification bilatérale multi-parties
- [demo-federation.js](../hwt-demo/demo-federation.js) — passerelle fédérée, émetteurs indépendants multiples
- [demo-mesh.js](../hwt-demo/demo-mesh.js) — maillage de microservices sans service mesh
- [demo-partner-api.js](../hwt-demo/demo-partner-api.js) — accès API partenaire inter-organisations
- [demo-edge.js](../hwt-demo/demo-edge.js) — vérification en périphérie et hors ligne

---

## Licence

Licence Apache 2.0

