# Hash Web Token (HWT)

[공식 문서](https://hwtprotocol.com/hwt-protocol) [이슈](https://github.com/hwt-protocol/hwt-protocol/issues) [토론](https://github.com/hwt-protocol/hwt-protocol/discussions)

도메인 간 위임(cross-domain delegation)을 위한 상태 비저장(stateless), 네트워크 네이티브 token 프로토콜입니다.

Token은 발급자(issuer) 오리진이 서명하고, 발급자의 도메인에 접근할 수 있는 누구나 검증할 수 있으며, 인가 컨텍스트를 직접 또는 참조 형식으로 담습니다.

**상태:** 초안 v0.7.

---

## HWT가 하는 것

- 구조적 만료를 포함한 간결한 점(`.`) 구분 token 형식 정의
- 검증자(verifier)가 서명 입력을 재구성하고 확인할 수 있도록 결정론적인 서명 입력 정규화 제공
- `/.well-known/hwt-keys.json`의 JWKS를 통한 키 발견(key discovery) 명세
- `/.well-known/hwt.json`의 오리진 메타데이터 발견 정의
- 규범적 체인 구성 규칙을 포함한 구조화된 위임 체인(delegation chain, `del`) 정의
- codec 레지스트리 정의(`j`/JSON이 기준이며, 커뮤니티 codec은 CONVENTIONS.md에 기술)
- 표준 HTTP 캐싱을 전반적으로 따름 — 별도의 캐시 인프라 불필요

## HWT가 다루지 않는 것

- **Token 상태 및 폐기(revocation).** Token이 발급 후 무효화되었는지는 서명된 바이트 문자열의 속성이 아닙니다. 즉각적인 무효화가 필요한 애플리케이션은 자체 상태 저장소를 유지합니다. 짧은 token 유효 기간이 노출 범위를 제한하는 기본 메커니즘입니다.
- **관할권 및 데이터 주권.** 프로토콜 수준에서 관할권 필드를 정의하지 않습니다. 애플리케이션은 비공개 스키마를 통해 `authz`에 관할권 컨텍스트를 포함합니다.
- **Token 발급.** HWT는 검증을 정의하며, 발급은 정의하지 않습니다. WebAuthn/FIDO2가 발급 측의 자연스러운 보완재입니다.
- **스키마 내용.** RBAC 역할의 의미, 비공개 스키마의 내용 — 이는 발급자와 소비 애플리케이션 간에 협의합니다.
- **인가 평가.** 검증된 token의 `authz` 값이 특정 행위를 허가하는지 여부는 소비 애플리케이션의 결정입니다. OPA와 Casbin이 자연스러운 보완재입니다.
- **에어갭 시스템.** 네트워크 접근이 없는 경우 다른 신뢰 아키텍처가 필요합니다.
- **대용량 payload.** HWT는 HTTP 헤더 크기 제약 내의 소~중형 token에 최적화되어 있습니다.
- **인증.** HWT는 인증 이후 전달되는 계약을 기술합니다. Token 발급 전에 신원을 확립하는 방법은 범위 밖입니다.
- **에이전트 발견.** 에이전트가 다른 에이전트를 기능으로 찾는 방법은 별도의 기본 요소입니다.
- **에이전트 간 통신 프로토콜.** HWT는 자격증명을 정의하며, 세션 또는 메시징 프로토콜은 정의하지 않습니다.
- **세션 바인딩.** DPoP(RFC 9449)는 proof-of-possession 배포와 호환되는 외부 메커니즘입니다.

---

## Token 형식

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

점으로 구분된 여섯 개의 필드. 점은 필드 구분자이며 `key-id`, `format`, 최상위 이름에는 포함되어서는 안 됩니다.

`format`은 codec을 선언합니다. `j`(JSON)는 유일한 필수 codec입니다. 추가 커뮤니티 codec은 CONVENTIONS.md에 기술되어 있습니다.

만료는 구조적입니다 — `expires` 값을 지난 token은 서명 유효성에 관계없이 무조건 무효입니다.

---

## 위임 체인 예시

세 단계: 신원 공급자의 사용자 → 오케스트레이션 에이전트 → 워커 서비스 → 최종 API 대상.

**루트 token** (`auth.example.com`이 발급, 사용자 보유):

```json
{
  "iss": "https://auth.example.com",
  "sub": "user:4503599627370495",
  "tid": "root-a1b2",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] }
}
```

**중간 token** (`agent.example.com`이 워커를 위해 발급):

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

**최종 파생 token** (`worker.example.com`이 대상 API를 위해 발급):

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

대상 API는 `worker.example.com`의 공개 키로 최종 token의 서명을 검증하고, del 체인이 구조적으로 무결한지 확인하며(외부 서명으로 보호됨), 전체 인가 출처를 원래 사용자까지 추적할 수 있습니다. 어느 단계에서도 인가는 상승되지 않았습니다 — 전 구간에 걸쳐 `roles: ["editor"]`.

---

## 키 발견

```
GET https://{issuer-domain}/.well-known/hwt-keys.json
```

JWKS 형식(RFC 7517). 키별 `alg` 지정이 필수입니다. 교체를 위한 여러 동시 키를 지원합니다.

## 오리진 메타데이터

```
GET https://{issuer-domain}/.well-known/hwt.json
```

발급자 URL, 프로토콜 버전, 지원하는 authz 스키마, 오디언스 구성, 위임 깊이 한도, 엔드포인트 URL을 선언합니다.

---

## 신뢰 모델

신뢰 앵커는 중앙 기관이 아닌 발급자의 도메인입니다. 검증은 DNS가 올바르게 해석되고 TLS가 발급자의 well-known 엔드포인트에 연결되는 것에 의존합니다. 권장되는 프로덕션 방식은 사전 등록된 발급자 allowlist입니다 — 검증자는 목록에 없는 발급자의 token을 거부하여, 키 발견을 런타임 DNS 의존성 없이 로컬 캐시 조회로 전환합니다.

---

## 문서

- [SPEC.md](SPEC.md) — 규범적 명세
- [CONTRIBUTING.md](CONTRIBUTING.md) — 변경 제안 방법
- [CONVENTIONS.md](CONVENTIONS.md) — 커뮤니티 사용 패턴, codec 예시, RBAC 레시피, 위임 안내, 관할권 처리
- [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) 참조 및 커뮤니티 구현체

## 구현체

**참조 라이브러리**
- [hwtr-js](../hwtr-js/) — JavaScript 참조 구현체

**데모**
- [hwt-demo](../hwt-demo/) — 다음 항목을 포함한 JavaScript 동작 데모(Deno 및 Cloudflare Workers 배포 등)
- [demo-agent-chain.js](../hwt-demo/demo-agent-chain.js) — AI 에이전트 위임 체인
- [demo-del-verify.js](../hwt-demo/demo-del-verify.js) — del[] 체인 전체 검증 및 폐기된 링크 감지
- [demo-multiparty.js](../hwt-demo/demo-multiparty.js) — 다자간 양방향 검증
- [demo-federation.js](../hwt-demo/demo-federation.js) — 연합 게이트웨이, 여러 독립 발급자
- [demo-mesh.js](../hwt-demo/demo-mesh.js) — 서비스 메시 없는 마이크로서비스 메시
- [demo-partner-api.js](../hwt-demo/demo-partner-api.js) — 조직 간 파트너 API 접근
- [demo-edge.js](../hwt-demo/demo-edge.js) — 엣지 및 오프라인 검증

---

## 라이선스

Apache License 2.0

