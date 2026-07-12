# 認証クレデンシャルの選択と、ローカル / 実 IAS でのテスト

> **このノートの要点**
> このガイドの [02 章](02-authentication.md)や[証明書運用ノート](app2app-and-certificate-operations.md)では、CIS の資格情報を **X.509 証明書 / mTLS** を主役に説明しました。しかし「**CIS では証明書認証しかできないのか？**」というと、そうではありません。
> - `identity` サービスは **client secret（`credential-type: SECRET`）も選べます**。secret を使えば **証明書の有効期限・ローテーションを一切考えなくてよくなります**。ただし**推奨方式ではなく**、**マルチテナントアプリでは X.509 が必須**です（§1）。
> - 日々のテストは、そもそも IAS を叩かず **認証をモックする**のが定石です（§2）。実 IAS に対して .http / Postman でトークンを取得する方法（secret / mTLS）も示します（§3）。

---

## 1. クレデンシャル方式の選択 — 本当に証明書だけなのか

`identity` サービスのバインディングは `credential-type` で資格情報の種類を選べます。**証明書一択ではありません。**

| credential-type | 資格情報の在り処 | 有効期限 / ローテーション | 使える条件 |
|---|---|---|---|
| **`SECRET`** | client id ＋ **client secret** | 有効期限による強制失効はない（＝**証明書ローテーション不要**） | **シングルテナントアプリのみ** |
| **`X509_GENERATED`** | バインディングに含まれる証明書（SAP 生成） | 有効期限あり。要ローテーション（CF は再デプロイ、Kyma は自動。[証明書運用ノート](app2app-and-certificate-operations.md#4-証明書のライフサイクルと運用-)） | シングル／マルチ両方 |
| **`X509_PROVIDED`** | アプリが自前で提供する証明書 | 自前管理（要ローテーション） | シングル／マルチ両方 |

**正確な線引き**（SAP Help で確認）:

- **マルチテナント（SaaS）アプリ**: **X.509 が必須**。secret は使えません。
- **シングルテナントアプリ**: **secret でも X.509 でもよい**（SAP Help: *"For single-tenant applications, you can use either client credentials or X.509"*）。
- AMS への mTLS 設定でも、資格情報の指定は `"credential-types": ["binding-secret", "x509"]` のように書け、**binding-secret がデフォルト、x509 は追加で有効化**という位置づけです。

### secret を使う場合のトレードオフ

> **✅ secret のメリット**
> - **証明書の有効期限・ローテーションを考えなくてよい**。X509 で必要だった「CF の `${timestamp}` 再デプロイ」「Kyma の `credentialsRotationPolicy`」といった運用がまるごと不要になります。
> - Postman / CI / ローカルからのトークン取得が単純（証明書をクライアントに仕込む必要がない → §3）。
>
> **⚠️ ただし推奨方式ではない**
> - SAP のベストプラクティスは **X.509 / mTLS**。共有シークレットは漏洩リスクがあり、より強い本人性を得られる証明書が推奨されます（[02 §資格情報](02-authentication.md)の方向性）。
> - secret も本来は定期ローテーションが推奨されます（漏洩対策）。「有効期限による強制失効がない」だけで、**放置してよいわけではありません**。
> - そして繰り返しですが、**マルチテナント SaaS では選択肢になりません**。

**まとめると**——「**動く・運用が楽**」なのは secret、「**推奨・多くの場合必須**」なのは X.509 です。開発／テスト用途や単一テナントで割り切るなら secret、マルチテナント SaaS や本番のセキュリティ強化では X.509、という使い分けになります。証明書ローテーションを避けたいという理由だけで本番を secret にするのは、**セキュリティ推奨との引き換え**である点を意識してください。

---

## 2. ローカルテスト — 認証はモックする（IAS を叩かない）

日々の開発・テストで実 IAS のトークンを取る必要は、基本ありません。CAP の公開ガイドは明確に **「authentication はモックし、authorization はモックするな」** と述べています（[Testing.md](../docs/Authorization/Testing.md#mock-authentication-not-authorization)）。認証だけモックすれば、**AMS のポリシー評価コードは本番と同じもの**が走るため、テストが信頼できます。

認証 `kind` を `mocked` にし、mock ユーザーへポリシーを直接割り当てます。

```jsonc
// package.json（cds.requires.auth）
"auth": {
  "[development]": {
    "kind": "mocked",
    "users": {
      "alice": { "policies": ["shopping.CreateOrders", "shopping.DeleteOrders"] },
      "bob":   { "policies": ["local.OrderAccessory"] }
    }
  }
}
```

`cds watch` に対しては **basic 認証**で叩くだけです（mock ユーザーのパスワードは空でよい）。

```http
### alice で取得（Authorization: Basic は base64("alice:") = "YWxpY2U6"）
GET http://localhost:4004/odata/v4/CatalogService/Books
Authorization: Basic YWxpY2U6
```

- 多くの REST クライアント（Postman / VS Code REST Client）は Basic 認証ヘルパーで「ユーザー名 `alice`・パスワード空」を指定すれば同じことができます。
- ポリシー割当を変えたいテストごとに `users` の `policies` を差し替えるだけ。**IAS も証明書も不要**で、これがローカルテストの主戦場です。

---

## 3. 実 IAS に対してトークンを取得する（Postman / .http）

デプロイ済みアプリを**実トークン**で叩きたい場合。まず **バインディングから接続情報を取得**します（`cf env <app>` またはサービスキー、Kyma は Secret）。必要なのは `url`・`clientid` と、`clientsecret`（secret 方式）または `certificate`／`key`（X.509 方式）です。

### 3.1 client secret 方式（`credential-type: SECRET`）

技術ユーザートークンを client credentials で取得します。

```http
### @name ias
POST https://<tenant>.accounts.ondemand.com/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id={{clientid}}&client_secret={{clientsecret}}

### 取得したトークンでアプリを呼ぶ
GET https://<app-url>/odata/v4/CatalogService/Books
Authorization: Bearer {{ias.response.body.$.access_token}}
```

- これは **技術ユーザートークン**（`azp = sub`）で、実エンドユーザーの文脈は乗りません。
- **ユーザー文脈が必要**なら、`grant_type=password`（ROPC。テナントで許可されている場合）か、Postman の **OAuth2 authorization code** フローを使います。

### 3.2 X.509 / mTLS 方式（`credential-type: X509_*`）

トークン取得自体を mTLS で行うため、**クライアント証明書を提示**します（secret は不要）。トークンエンドポイントは証明書用 URL をバインディングから使ってください（ホスト名は環境依存）。

- **curl**:
  ```sh
  curl --cert cert.pem --key key.pem \
    -X POST https://<tenant>.<cert-host>/oauth2/token \
    -d grant_type=client_credentials -d client_id=<clientid>
  ```
- **Postman**: Settings → **Certificates** で当該 IAS ホストにクライアント証明書（cert / key）を登録 → 通常どおりリクエスト。
- **VS Code REST Client（.http）**: `settings.json` の `rest-client.certificates` にホスト別で `cert`／`key` を設定。

> **⚠️ 注意: トークンが取れても認可は別**
> 実 IAS トークンを取得できても、**そのユーザー／テナントへのポリシー割当が AMS バンドルに載っていなければ、認可チェックは通りません**（[03 §4](03-authorization.md#4-判定はどこでいつ起きるか--実行時-pdp-と-authorization-bundle)）。認可まで含めて検証したいなら「実テナントに割当済み」の環境か、次の §4 ハイブリッドテストを使ってください。ロジックだけ確認したいなら §2 のモックが速くて確実です。

---

## 4. ハイブリッドテスト — 本番の割当そのままでローカル実行

本番のポリシー・割当をそのまま使ってローカルで動かしたい場合は、CAP の [hybrid testing](https://cap.cloud.sap/docs/advanced/hybrid-testing) を使います。ローカルのアプリを実 `identity` インスタンスにバインドし、**本番ランドスケープの AMS バンドル（ポリシー＋割当）を引いて**評価できます（[Testing.md](../docs/Authorization/Testing.md) の "CAP Hybrid Testing"）。

---

## 使い分けの早見

| やりたいこと | 方法 | クレデンシャル |
|---|---|---|
| 認可ロジックを素早く検証 | §2 mocked 認証 | 不要（basic 認証で mock ユーザー） |
| デプロイ済みアプリを実トークンで叩く | §3 IAS トークン取得 | secret（単純）or 証明書（mTLS） |
| 本番の割当込みで検証 | §4 ハイブリッドテスト | 実 `identity` バインディング |
| 本番運用の資格情報 | §1 | **X.509 推奨**（マルチテナントは必須）。単一テナントは secret も可 |

---

## 関連

- [02. 認証の違い](02-authentication.md) — 認証方式・トークン発行者・mTLS
- [App-to-App 連携の設定と証明書運用](app2app-and-certificate-operations.md) — 証明書のライフサイクル／ローテーション
- [06. ライブラリ・実装の違い](06-libraries-implementation.md) — CAP の認証 `kind`・AMS 連携
- [Testing AMS Integration（公開 docs）](../docs/Authorization/Testing.md) — mocked 認証・ローカルテスト・ハイブリッドテスト
