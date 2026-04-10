# Hash Web Token (HWT)

[documentación canónica](https://hwtprotocol.com/hwt-protocol) [issues](https://github.com/hwt-protocol/hwt-protocol/issues) [discussions](https://github.com/hwt-protocol/hwt-protocol/discussions)

Un protocolo de token sin estado y nativo de red para la delegación entre dominios.

Los tokens son firmados por un origen emisor, verificables por cualquier parte que pueda alcanzar el dominio del issuer (emisor), y llevan contexto de autorización ya sea de forma literal o por referencia.

**Estado:** Borrador v0.7.

---

## Qué hace HWT

- Define un formato de token compacto, separado por puntos, con expiración estructural
- Proporciona canonicalización determinista de la entrada firmada para que cualquier verifier (verificador) pueda reconstruir y comprobar la firma
- Especifica key discovery (descubrimiento de claves) via JWKS en `/.well-known/hwt-keys.json`
- Define el descubrimiento de metadatos del origen en `/.well-known/hwt.json`
- Proporciona una delegation chain (cadena de delegación) estructurada (`del`) con reglas normativas de construcción de cadena
- Define un registro de codecs (`j` / JSON es la base; codecs de la comunidad en CONVENTIONS.md)
- Sigue el caché HTTP estándar en todo momento — sin infraestructura de caché especial

## Lo que HWT no contempla

- **Estado del token y revocation (revocación).** Si un token ha sido invalidado después de su emisión no es una propiedad de la cadena de bytes firmada. Las aplicaciones que requieren invalidación inmediata mantienen su propio almacén de estado. Los tokens de corta duración son el mecanismo principal para acotar la exposición.
- **Jurisdicción y soberanía de datos.** Sin campo de jurisdicción a nivel de protocolo. Las aplicaciones incorporan contexto de jurisdicción en `authz` mediante un esquema privado.
- **Emisión de tokens.** HWT define la verificación, no la emisión. WebAuthn/FIDO2 es un complemento natural para la fase de autenticación previa a la emisión.
- **Contenido del esquema.** Qué significan los roles RBAC, qué contienen los esquemas privados — se negocia entre issuers y las aplicaciones consumidoras.
- **Evaluación de autorización.** Si los valores `authz` de un token verificado autorizan una acción específica es decisión de la aplicación consumidora. OPA y Casbin son complementos naturales.
- **Sistemas sin acceso a la red (air-gapped).** Sin acceso a la red se requiere una arquitectura de confianza diferente.
- **Payloads grandes.** HWT está optimizado para tokens pequeños y medianos dentro de los límites de tamaño de los encabezados HTTP.
- **Autenticación.** HWT describe el contrato que se pasa después de la autenticación. Cómo se establece la identidad antes de la emisión del token está fuera del alcance.
- **Descubrimiento de agentes.** Cómo un agente localiza a otro agente por capacidad es un primitivo separado.
- **Protocolo de comunicación entre agentes.** HWT define credenciales, no protocolos de sesión o mensajería.
- **Vinculación de sesión.** DPoP (RFC 9449) es un mecanismo externo compatible para despliegues de proof-of-possession (prueba de posesión).

---

## Formato del token

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

Seis campos separados por puntos. El punto es el separador de campos y NO DEBE aparecer en `key-id`, `format` ni en el nombre de nivel superior.

`format` declara el codec. `j` (JSON) es el único codec requerido. Codecs adicionales de la comunidad están documentados en CONVENTIONS.md.

La expiración es estructural — un token pasada su fecha `expires` es incondicionalmente inválido independientemente de la validez de la firma.

---

## Ejemplo de delegation chain

Tres saltos: un usuario en un proveedor de identidad → un agente de orquestación → un servicio worker → el destino final de la API.

**Token raíz** (en poder del usuario, emitido por `auth.example.com`):

```json
{
  "iss": "https://auth.example.com",
  "sub": "user:4503599627370495",
  "tid": "root-a1b2",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] }
}
```

**Token intermedio** (emitido por `agent.example.com` para el worker):

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

**Token derivado final** (emitido por `worker.example.com` para la API de destino):

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

La API de destino verifica la firma del token final contra las claves publicadas de `worker.example.com`, confirma que la cadena `del` está estructuralmente íntegra (está cubierta por la firma externa), y puede rastrear el linaje de autorización completo hasta el usuario original. La autorización no fue escalada en ningún salto — `roles: ["editor"]` en todo momento.

---

## Key discovery

```
GET https://{issuer-domain}/.well-known/hwt-keys.json
```

Formato JWKS (RFC 7517). `alg` por clave es obligatorio. Se admiten múltiples claves concurrentes para rotación.

## Metadatos del origen

```
GET https://{issuer-domain}/.well-known/hwt.json
```

Declara la URL del issuer, la versión del protocolo, los esquemas authz soportados, la configuración de audiencia, el límite de profundidad de delegación y las URLs de los endpoints.

---

## Modelo de confianza

El ancla de confianza es el dominio del issuer, no una autoridad central. La verificación depende de que el DNS resuelva correctamente y TLS se conecte al endpoint well-known del issuer. La postura de producción recomendada es un allowlist de issuers pre-registrados — los verifiers rechazan tokens de issuers que no están en la lista, convirtiendo el key discovery en una búsqueda de caché local sin dependencia de DNS en tiempo de ejecución.

---

## Documentación

- [SPEC.md](SPEC.md) — especificación normativa
- [CONTRIBUTING.md](CONTRIBUTING.md) — cómo proponer cambios
- [CONVENTIONS.md](CONVENTIONS.md) — patrones de uso de la comunidad, ejemplos de codecs, recetas RBAC, guías de delegation chain, manejo de jurisdicción
- [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) — implementaciones de referencia y de la comunidad

## Implementaciones

**Biblioteca de referencia**
- [hwtr-js](../hwtr-js/) — implementación de referencia en JavaScript

**Demostraciones**
- [hwt-demo](../hwt-demo/) — demostraciones funcionales en JavaScript incluyendo lo siguiente (despliegues en Deno y Cloudflare Workers, etc.).
- [demo-agent-chain.js](../hwt-demo/demo-agent-chain.js) — Cadena de delegación de agentes de IA
- [demo-del-verify.js](../hwt-demo/demo-del-verify.js) — Verificación completa de cadena del[] con detección de enlaces revocados
- [demo-multiparty.js](../hwt-demo/demo-multiparty.js) — Verificación bilateral entre múltiples partes
- [demo-federation.js](../hwt-demo/demo-federation.js) — Pasarela federada con múltiples issuers independientes
- [demo-mesh.js](../hwt-demo/demo-mesh.js) — Malla de microservicios sin una malla de servicios
- [demo-partner-api.js](../hwt-demo/demo-partner-api.js) — Acceso a API de socios entre organizaciones
- [demo-edge.js](../hwt-demo/demo-edge.js) — Verificación en el borde y sin conexión

---

## Licencia

Apache License 2.0


