# Hash Web Token (HWT)

[正式文件](https://hwtprotocol.com/hwt-protocol) [議題](https://github.com/hwt-protocol/hwt-protocol/issues) [討論](https://github.com/hwt-protocol/hwt-protocol/discussions)

無狀態的網路原生 token 協定，用於跨網域委派。

Token 由發行端來源簽署，任何能連接到 issuer（發行方）網域的一方均可驗證，並以明文或引用方式攜帶授權上下文。

**狀態：** 草案 v0.7。

---

## HWT 的功能

- 定義緊湊的點分隔 token 格式，具備結構性到期時間
- 提供確定性的已簽署輸入正規化，使任何 verifier（驗證方）均可重建並驗證簽章
- 規定透過 `/.well-known/hwt-keys.json` 的 JWKS 進行 key discovery（金鑰探索）
- 定義 `/.well-known/hwt.json` 的來源元資料探索
- 提供結構化的 delegation chain（委派鏈）（`del`），並附有規範性鏈路構建規則
- 定義 codec 登錄（`j` / JSON 為基準；社群 codec 記錄於 CONVENTIONS.md）
- 全程遵循標準 HTTP 快取 — 無需特殊快取基礎設施

## HWT 不涵蓋的事項

- **Token 狀態與撤銷（revocation）。** token 在發行後是否已被作廢並非已簽署位元組串的屬性。需要立即作廢的應用程式應自行維護狀態儲存。短暫的 token 有效期是限制暴露風險的主要機制。
- **司法管轄與資料主權。** 協定層面沒有管轄權欄位。應用程式透過私有 schema 在 `authz` 中嵌入管轄上下文。
- **Token 發行。** HWT 定義驗證，不定義發行。WebAuthn/FIDO2 是發行端的自然補充。
- **Schema 內容。** RBAC 角色的含義、私有 schema 的內容 — 由 issuer 與使用端應用程式協商決定。
- **授權評估。** 已驗證 token 的 `authz` 值是否授權特定操作由使用端應用程式決定。OPA 與 Casbin 是自然的補充工具。
- **隔離網路系統。** 無網路存取意味著需要不同的信任架構。
- **大型 payload。** HWT 針對 HTTP 標頭大小限制內的中小型 token 進行最佳化。
- **身份驗證。** HWT 描述身份驗證後傳遞的契約。在 token 發行前如何確立身份不在範圍之內。
- **代理人探索。** 代理人如何依能力定位另一個代理人是獨立的基元。
- **代理人間通訊協定。** HWT 定義憑證，而非 session 或訊息傳遞協定。
- **Session 綁定。** DPoP（RFC 9449）是 proof-of-possession 部署的相容外部機制。

---

## Token 格式

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

六個以點分隔的欄位。點是欄位分隔符，不得出現在 `key-id`、`format` 或頂層名稱中。

`format` 宣告 codec。`j`（JSON）是唯一必要的 codec。額外的社群 codec 記錄於 CONVENTIONS.md。

到期時間是結構性的 — 超過 `expires` 值的 token 無條件無效，無論簽章是否有效。

---

## 委派鏈範例

三跳：身份提供者的使用者 → 編排代理人 → 工作服務 → 最終 API 目標。

**根 token**（由使用者持有，由 `auth.example.com` 發行）：

```json
{
  "iss": "https://auth.example.com",
  "sub": "user:4503599627370495",
  "tid": "root-a1b2",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] }
}
```

**中間 token**（由 `agent.example.com` 為工作服務發行）：

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

**最終衍生 token**（由 `worker.example.com` 為目標 API 發行）：

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

目標 API 以 `worker.example.com` 公開的金鑰驗證最終 token 的簽章，確認 `del` 鏈在結構上完整（由外層簽章涵蓋），並可將完整授權溯源追溯至原始使用者。授權在任何跳轉中均未提升 — 全程保持 `roles: ["editor"]`。

---

## 金鑰探索

```
GET https://{issuer-domain}/.well-known/hwt-keys.json
```

JWKS 格式（RFC 7517）。每個金鑰的 `alg` 為必填。支援多個並行金鑰以進行輪換。

## 來源元資料

```
GET https://{issuer-domain}/.well-known/hwt.json
```

宣告 issuer URL、協定版本、支援的 authz schema、受眾設定、委派深度限制及端點 URL。

---

## 信任模型

信任錨點是 issuer 的網域，而非中央機構。驗證依賴 DNS 正確解析與 TLS 連接至 issuer 的 well-known 端點。建議的正式環境做法是預先註冊 issuer allowlist — verifier 拒絕不在清單中的 issuer 所發行的 token，將 key discovery 轉換為本地快取查詢，無需執行時 DNS 依賴。

---

## 文件

- [SPEC.md](SPEC.md) — 規範性規格書
- [CONTRIBUTING.md](CONTRIBUTING.md) — 如何提交變更提案
- [CONVENTIONS.md](CONVENTIONS.md) — 社群使用模式、codec 範例、RBAC 做法、委派鏈說明、管轄處理
- [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) 參考實作與社群實作

## 實作

**參考程式庫**
- [hwtr-js](../hwtr-js) — JavaScript 參考實作

**示例**
- [hwt-demo](../hwt-demo) — JavaScript 可執行示例，包含以下內容（Deno 與 Cloudflare Workers 部署等）。
- [demo-agent-chain.js](../hwt-demo/demo-agent-chain.js) — AI 代理人委派鏈
- [demo-del-verify.js](../hwt-demo/demo-del-verify.js) — 完整 del[] 鏈路驗證與已撤銷連結偵測
- [demo-multiparty.js](../hwt-demo/demo-multiparty.js) — 多方聯合授權
- [demo-federation.js](../hwt-demo/demo-federation.js) — 自發性跨網域聯盟，多個獨立 issuer
- [demo-mesh.js](../hwt-demo/demo-mesh.js) — 無服務網格的微服務網格
- [demo-partner-api.js](../hwt-demo/demo-partner-api.js) — 跨組織夥伴 API 存取
- [demo-edge.js](../hwt-demo/demo-edge.js) — 邊緣與離線驗證

---

## 授權條款

Apache License 2.0

