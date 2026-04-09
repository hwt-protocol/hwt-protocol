# Hash Web Token (HWT)

[正規ドキュメント](https://hwtprotocol.com/hwt-protocol) [issues](https://github.com/hwt-protocol/hwt-protocol/issues) [discussions](https://github.com/hwt-protocol/hwt-protocol/discussions)

クロスドメイン委任のための、ステートレスなネットワークネイティブトークンプロトコル。

トークンは発行元オリジンによって署名され、issuer（発行者）のドメインに到達可能な任意のパーティが検証でき、認可コンテキストをリテラルまたは参照形式で保持する。

**ステータス：** Draft v0.7。

---

## HWTができること

- コンパクトなドット区切りのトークンフォーマットを定義し、構造的な有効期限を持つ
- 任意のverifier（検証者）が署名を再構築・確認できるよう、決定論的な署名入力の正規化を提供する
- `/.well-known/hwt-keys.json`のJWKS経由でkey discovery（鍵探索）を規定する
- `/.well-known/hwt.json`でオリジンメタデータのdiscoveryを定義する
- 規範的なチェーン構築ルールを持つstructured delegation chain（`del`）を提供する
- codecレジストリを定義する（`j` / JSONがベースライン；コミュニティcodecはCONVENTIONS.mdに記載）
- 標準的なHTTPキャッシュに全面的に準拠 — 特別なキャッシュインフラ不要

## HWTがカバーしない領域

- **トークンの状態とrevocation（失効）。** 発行後にトークンが無効化されたかどうかは、署名済みバイト列の属性ではない。即時無効化が必要なアプリケーションは独自の状態ストアを管理する。トークンのライフタイムを短くすることが、露出リスクを抑える主要なメカニズムである。
- **管轄とデータ主権。** プロトコルレベルに管轄フィールドはない。アプリケーションはプライベートスキーマを通じて`authz`に管轄コンテキストを埋め込む。
- **トークン発行。** HWTは検証を定義するものであり、発行は定義しない。WebAuthn/FIDO2は発行側の自然な補完手段である。
- **スキーマコンテンツ。** RBACロールの意味やプライベートスキーマの内容は、issuerと利用側アプリケーション間で取り決める。
- **認可評価。** 検証済みトークンの`authz`値が特定のアクションを許可するかどうかは、利用側アプリケーションの判断である。OPAやCasbinは自然な補完ツールとなる。
- **エアギャップ環境。** ネットワークアクセスがない場合は、異なるトラスト・アーキテクチャが必要となる。
- **大きなpayload。** HWTはHTTPヘッダーサイズ制約内の小〜中規模トークン向けに最適化されている。
- **認証。** HWTは認証後に受け渡されるコントラクトを記述する。トークン発行前にアイデンティティがどのように確立されるかはスコープ外である。
- **エージェントdiscovery。** エージェントが別のエージェントをケイパビリティによって特定する方法は、別のプリミティブである。
- **エージェント間通信プロトコル。** HWTはcredentialを定義するものであり、セッションやメッセージングプロトコルは定義しない。
- **セッションバインディング。** DPoP（RFC 9449）はproof-of-possessionデプロイメントに対応した互換性のある外部メカニズムである。

---

## トークンフォーマット

```
hwt.signature.key-id.expires-unix-seconds.format.payload
```

6つのドット区切りフィールド。ドットはフィールドセパレーターであり、`key-id`、`format`、またはトップレベル名に含まれてはならない。

`format`はcodecを宣言する。`j`（JSON）が唯一の必須codecである。追加のコミュニティcodecはCONVENTIONS.mdに記載されている。

有効期限は構造的なものである — `expires`値を過ぎたトークンは、署名の有効性にかかわらず無条件に無効である。

---

## Delegation chainの例

3ホップ：アイデンティティプロバイダーのユーザー → オーケストレーションエージェント → ワーカーサービス → 最終APIターゲット。

**ルートトークン**（ユーザーが保持、`auth.example.com`が発行）：

```json
{
  "iss": "https://auth.example.com",
  "sub": "user:4503599627370495",
  "tid": "root-a1b2",
  "authz": { "scheme": "RBAC/1.0.2", "roles": ["editor"] }
}
```

**中間トークン**（`agent.example.com`がワーカー向けに発行）：

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

**最終派生トークン**（`worker.example.com`がターゲットAPI向けに発行）：

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

ターゲットAPIは最終トークンの署名を`worker.example.com`の公開鍵と照合して検証し、`del`チェーンが構造的に完全であること（外側の署名によってカバーされている）を確認し、元のユーザーまで完全な認可の系譜を辿ることができる。いずれのホップでも認可は昇格していない — 全体を通じて`roles: ["editor"]`のまま。

---

## Key discovery

```
GET https://{issuer-domain}/.well-known/hwt-keys.json
```

JWKSフォーマット（RFC 7517）。鍵ごとの`alg`は必須。ローテーションに備えた複数の同時使用鍵をサポート。

## オリジンメタデータ

```
GET https://{issuer-domain}/.well-known/hwt.json
```

issuer URL、プロトコルバージョン、サポートするauthzスキーマ、audienceの設定、委任深度の上限、エンドポイントURLを宣言する。

---

## トラストモデル

トラストアンカーはissuerのドメインであり、中央機関ではない。検証はDNSが正しく解決し、issuerのwell-knownエンドポイントへTLS接続できることに依存する。推奨される本番環境の構成は、issuerの事前登録許可リストである — リストにないissuerからのトークンをverifierが拒否することで、key discoveryをランタイムのDNS依存のないローカルキャッシュ参照に変換する。

---

## ドキュメント

- [SPEC.md](SPEC.md) — 規範的仕様
- [CONTRIBUTING.md](CONTRIBUTING.md) — 変更提案の方法
- [CONVENTIONS.md](CONVENTIONS.md) — コミュニティ使用パターン、codecの例、RBACレシピ、委任のウォークスルー、管轄の取り扱い
- [IMPLEMENTATIONS.md](./IMPLEMENTATIONS.md) — リファレンスおよびコミュニティ実装

## 実装

**リファレンスライブラリ**
- [hwtr-js](../hwtr-js) — JavaScriptリファレンス実装

**デモ**
- [hwt-demo](../hwt-demo) — JavaScriptの動作デモ（以下を含む）（Deno・Cloudflare Workersデプロイなど）
- [demo-agent-chain.js](../hwt-demo/demo-agent-chain.js) — AIエージェントのdelegation chain
- [demo-del-verify.js](../hwt-demo/demo-del-verify.js) — 失効リンク検出を含む完全なdel[]チェーン検証
- [demo-multiparty.js](../hwt-demo/demo-multiparty.js) — マルチパーティの双方向検証
- [demo-federation.js](../hwt-demo/demo-federation.js) — フェデレーテッドゲートウェイ、複数の独立したissuer
- [demo-mesh.js](../hwt-demo/demo-mesh.js) — サービスメッシュなしのマイクロサービスメッシュ
- [demo-partner-api.js](../hwt-demo/demo-partner-api.js) — 組織間パートナーAPIアクセス
- [demo-edge.js](../hwt-demo/demo-edge.js) — エッジとオフライン検証

---

## ライセンス

Apache License 2.0

---

