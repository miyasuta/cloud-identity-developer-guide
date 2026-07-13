# App-to-App 連携の設定と証明書運用

> このファイルは **App-to-App 連携の設定・運用面**を扱います。
> — dependency（依存関係）の登録、Destination の設定、**証明書のライフサイクル／ローテーション運用**。
>
> 認証方式そのもの（トークン取得フローや `ias_apis` クレームの意味）は [02. 認証の違い](02-authentication.md) を参照してください。
>
> **`questions.md` の質問への回答:**
> - App-to-App の dependency の考え方（Provider が API を公開 → Consumer が dependency を登録 → `ias_apis` に載る）
> - Destination／アプリ側の具体設定と、証明書更新の運用（自動化できるか、Cloud Foundry での操作）

---

## 1. 登場人物と関係

XSUAA では、アプリ間連携は主に「Destination ＋ OAuth2（SAML Bearer / client credentials）＋ XSUAA の scope」で表現していました。CIS ではこれが **IAS の「API 権限グループ（API permission group）」と「dependency（依存関係）」** に置き換わります。

- **Provider（提供側）アプリ**: 自分の API を **API 権限グループ**として公開する。
- **Consumer（利用側）アプリ**: Provider の API 権限グループに対する **dependency** を登録する。
- **テナント管理者**: 「どの Consumer が、どの Provider の、どの API 権限グループを消費してよいか」を承認する（＝ dependency の作成）。**許可を出すのはアプリではなく管理者**。
- 実行時、Consumer が取得する **IAS トークンの `ias_apis` クレーム**に、許可された API 権限グループ名が入る。

```mermaid
flowchart LR
    subgraph P["Provider アプリ"]
        API["API権限グループ<br/>例: sales-orders"]
    end
    subgraph C["Consumer アプリ"]
        DEP["dependency<br/>例: name=orderDep<br/>→ sales-orders を指す"]
    end
    ADMIN([テナント管理者]) -->|承認して dependency 作成| DEP
    DEP -.参照.-> API
    C -->|"① resource=urn:...:name:orderDep でトークン要求"| IAS[IAS]
    IAS -->|"② token（aud=Provider, ias_apis=['sales-orders']）"| C
    C -->|"③ token を付けて API 呼び出し"| P
    P -->|"④ ias_apis を検証して認可"| P
    style API fill:#d5f9e0,stroke:#27ae60,color:#1a1a1a
    style DEP fill:#d5e8f9,stroke:#2980b9,color:#1a1a1a
    style IAS fill:#f9e6d5,stroke:#e67e22,color:#1a1a1a
```

> トークン取得のフロー（技術通信＝client credentials／主体伝播＝JWT bearer・token exchange）と `ias_apis` の詳細は [02 §5](02-authentication.md) を参照。

---

## 2. 設定手順：API 公開と dependency 登録

**Provider 側：API を公開する**
- 管理コンソール: `Applications and Resources > Applications > [Provider] > Trust タブ > Application APIs > Provided APIs` に API 権限グループ名を追加（URN 準拠・最大50文字・最大50個）。「Allow all APIs for principal propagation」を選ぶと `ias_apis` に `principal-propagation` が入る。
- または `identity` サービスのパラメータで宣言的に定義:

```json
{ "provided-apis": [ { "name": "sales-orders", "description": "Access sales orders" } ] }
```

**Consumer 側：dependency を登録する**
- 管理コンソール: `Applications and Resources > Applications > [Consumer] > Trust タブ > Application APIs > Dependencies` で、一意の dependency 名を付け、Provider アプリと API を選択（最大400個）。

> SAP も「Destination を作る感覚に近い」と案内しています。dependency 名はシナリオガイド等で管理者に伝えるのが定石です。

> **⚠️ ただし：Consumer 側の dependency は宣言的（mta.yaml）に書けない**
> Provider 側の API 公開は上記のとおり `identity` サービスの `provided-apis` パラメータで**宣言的に定義できます**（mta.yaml に載る）。しかし **Consumer 側の dependency 登録（＝管理者による承認）は、この管理コンソール操作でしか行えず、mta.yaml では表現できません**。
> これは XSUAA からの見落としやすい影響です。XSUAA では **Destination を mta.yaml に宣言的に定義**すれば App-to-App の配線がデプロイ記述子だけで完結し、環境を作り直してもそのまま再現できました。CIS では Destination（§3）を宣言できても、**dependency という手動の承認ステップがデプロイ記述子の外に残る**ため、IaC / CI-CD による完全な再現デプロイが一段難しくなります（この追加コストは [07 §4](07-summary-migration-cheatsheet.md#4-デメリット追加コストの正直な棚卸し) にも記載）。

---

## 3. Destination の設定

Destination の組み方には **2 通り**あります。**誰の資格情報でトークンを取るか**が本質的な分岐点です。CAP で IAS 保護アプリを呼ぶなら §3.1（資格情報を Destination に持たせない）が推奨、汎用・非 CAP や従来型は §3.2 です。

### 3.1 推奨（CAP）: 資格情報を持たない Destination 🔑

CAP（Java／Cloud SDK）で別の IAS アプリを呼ぶ場合、Destination には **client id も secret も証明書も書きません**。認証タイプを `NoAuthentication` にし、`cloudsdk.ias-dependency-name` で IAS の dependency を指すだけです。

```
# BTP Destination（プロパティ表記）
Name=app2app
Type=HTTP
URL=https://<Provider アプリ URL>
ProxyType=Internet
Authentication=NoAuthentication
cloudsdk.ias-dependency-name=<dependency 名>
```

実行時、Cloud SDK は **Consumer アプリにバインドされた自分の `identity` インスタンスの資格情報**を使ってトークンを取得します。つまり——**本人性は「自分のバインディング」**、**呼ぶ相手の指定は「dependency」**、という二段構えです（[02 §5](02-authentication.md)）。

- 前提: 両アプリが同じ IAS テナントを信頼し、IAS 側で dependency が登録済み（§2）。
- ユーザー文脈の伝播（named user / technical user）は、CAP の remote service 側 `onBehalfOf`（`currentUser` / `systemUser` / `systemUserProvider`）で制御します。
- **この方式の最大の利点**: 資格情報が Destination に一切載らないため、**証明書 / secret のローテーション時に Destination を触る必要がありません**（§4.4）。

> さらに、Consumer と Provider が **同じ `identity` インスタンスを共有**する構成なら、**Destination すら不要**です。CAP の remote service に `binding: name: <shared-identity>` を直接指定できます（CAP Java: "Binding to a Service with Shared Identity"、[共有 IAS 構成](shared-ias-app-central-dcl.md)）。

### 3.2 従来型: 資格情報を Destination に持たせる（汎用 / 非 CAP）

CAP プラグインを使わない場合や、汎用の Destination サービス経由で組む場合は、認証タイプに応じて **資格情報を Destination に設定**します。

- **`OAuth2ClientCredentials`** — 技術通信（技術ユーザー）。ログインユーザーの文脈を**伝播しない**システム間連携。
- **`OAuth2JWTBearer`** — 主体伝播。ログインユーザーの文脈を**維持**（別トークン種別からは token exchange 系）。

> `OAuth2JWTBearer`（IAS）は **跨サブアカウント／跨リージョン**でもそのまま使えます。信頼は IAS テナントの dependency に集約されるため、XSUAA の `OAuth2SAMLBearerAssertion` で必要だった **サブアカウント間の SAML 信頼設定は不要**です（変化点の詳細は [02 §5](02-authentication.md)）。

以下は主体伝播（`OAuth2JWTBearer`）の例です。技術通信なら `Authentication` を `OAuth2ClientCredentials` に変えます（`tokenService.body.resource` の dependency 指定は同じ）。

```json
{
  "Name": "app2app",
  "Type": "HTTP",
  "URL": "https://<Provider アプリ URL>",
  "ProxyType": "Internet",
  "Authentication": "OAuth2JWTBearer",
  "clientId": "<Consumer の client id>",
  "clientSecret": "<Consumer の client secret>",
  "tokenServiceURL": "https://<CIS テナント URL>/oauth2/token",
  "tokenServiceURLType": "Dedicated",
  "tokenService.body.resource": "urn:sap:identity:application:provider:name:<dependency名>"
}
```

ここで **トークン取得の認証（`clientSecret`）を mTLS（証明書）に置き換えられます**。ただしこの方式は **資格情報のコピーを Destination が抱える**ため、ローテーション時に **Destination 側の更新が必要**になります（§4.4）。この「トークン取得に使う証明書 / secret」の出所とローテーションが、次章の主題です。

---

## 4. 証明書のライフサイクルと運用 🔑

> **`questions.md` の質問への回答:**
> 「Destination 経由で App-to-App 連携している場合、証明書の更新が定期的に必要になるのか？」

**結論: X.509 証明書には有効期限が必ずあるため「更新（ローテーション）」自体は避けられません。ただし "手動で定期更新が必要か" は、証明書の出所（プロビジョニング方式）によって変わります。**

XSUAA の client secret は「一度発行したら（明示的にローテーションしない限り）長く使える」運用が可能でした。CIS の証明書ベースでは **有効期限が本質的に存在** するのが最大の意識の変化です。

### 4.1 証明書の出所と更新方式（credential-type）

`identity` サービスのバインディングは、`credential-type` によって証明書の扱いが変わります。

| credential-type | 証明書の在り処 | 更新（ローテーション） | 手動作業 |
|---|---|---|---|
| `SECRET` | （証明書ではなく client secret） | secret のローテーション | — |
| **`X509_GENERATED`** | **バインディングに含まれる**（SAP が生成） | `validity`（日数）で期限指定。**デフォルト 30 日・最長 30 日**（`validity-type` は `DAYS` のみ、範囲 1–30）。**サービスキー/バインディングのローテーション**で更新（CF は再デプロイ/リバインド、Kyma は自動。→ 4.2） | ランタイム依存（日々の手作業は不要にできる） |
| `X509_PROVIDED` | バインディングに**含まれない**（アプリが提供） | 自前の証明書管理に依存 | 自前で更新が必要 |

> **アプリ側はランタイムの証明書差し替えに対応済み**
> `X509_PROVIDED` や、ローテーションで `X509_GENERATED` の証明書が切り替わる場合、証明書は実行中に変わります。AMS クライアントライブラリはこれを想定しており、Node.js は `identityService.setCertificateAndKey(cert, key)`、CAP プラグインは `amsCapPluginRuntime.credentials.certificate/key` の更新で、**実行中に新しい証明書へ差し替え**できます（[AuthorizationBundle](../docs/Authorization/AuthorizationBundle.md) の Certificate Configuration 節）。

**推奨パターン（手動更新ほぼ不要）**: アプリを `identity` に **`credential-type: X509_GENERATED` でバインド**し、セキュリティライブラリ（`@sap/xssec` / SAP Cloud SDK / AMS）が **そのバインディング証明書で直接トークンフローを実行**します（Destination に証明書を手で載せない）。バインディングパラメータは、このリポジトリの [DeployDCL](../docs/Authorization/DeployDCL.md) と同じ書き方です:

```yaml [mta.yaml（バインディング側）]
modules:
  - name: my-consumer-srv
    requires:
      - name: my-consumer-identity   # identity サービスインスタンス
        parameters:
          config:
            credential-type: X509_GENERATED
            validity: 30          # 証明書の有効日数（範囲 1–30、既定 30）
            validity-type: DAYS   # DAYS のみ対応
```

> **出典（`validity` の値）**: SAP 公式 [Reference Information for the Identity Service of SAP BTP](https://help.sap.com/docs/identity-authentication/identity-authentication/reference-information-for-identity-service-of-sap-btp)（[GitHub ソース](https://github.com/SAP-docs/btp-cloud-identity-services/blob/main/docs/Integrating-the-Service/reference-information-for-the-identity-service-of-sap-btp-9379444.md)）。`identity` サービスの `X509_GENERATED` は **`validity` 既定 30・範囲 1–30 DAYS**、`validity-type` は `DAYS` のみ、`key-length` は 2048（既定）/4096。**「7日」「最長1年」は誤り**——7日は別サービス（XSUAA）のデフォルト、1年は identity・XSUAA いずれの公式リファレンスにも無い。

### 4.1.1 binding と service-key ではローテーションのされ方が違う（重要）

有効期限は**バインディング単位でもサービスキー単位でもなく「生成される証明書（クレデンシャル）単位」**です。binding と service-key は**それぞれ独立した証明書＝独立した有効期限**を持ち、`credential-type` / `validity` は **config を書いた場所のクレデンシャル**に効きます。**CAP のデフォルトは `credential-type` を binding 側**（`modules[].requires[].parameters.config`）に置くので、効くのは **その binding の証明書**です（capire もこの書き方）。

ローテーションの起こり方が両者で決定的に違います:

| | **binding**（CAP デフォルト） | **service-key**（`resources[].service-keys`） |
|---|---|---|
| 更新のトリガ | **アプリの再デプロイ時に CF が binding を再作成 → IAS が新証明書を発行**（設定が未変更でも） | deploy では自動再作成されない。**名前で永続**するので回すには工夫が要る |
| CF での回し方 | **再デプロイ（＝移送）するだけ**。`service-keys` も `${timestamp}` も不要 | サービスキー名に **`${timestamp}`** を使い、再デプロイ毎に新キー＝新証明書（旧キー自動削除） |
| いつ使う | 直バインドの CAP アプリ | 外部 Consumer・サブアカウント跨ぎなど、**バインドできない相手に資格情報を渡す**場合 |

> **要点（＝「デフォルト構成から変えたくない」への回答）**: **CAP デフォルトの binding のままでよい**。証明書は**アプリを再デプロイ（cTMS 移送）するたびに自動で更新**されます。`${timestamp}` サービスキーへ作り替える必要はありません（あれは service-key を消費する構成向け）。

> ⚠️ **gotcha（IAS の User Management / SCIM API を使う場合のみ）**: binding 再作成で証明書がローテートされると、IAS が「**API Access for application users**」を自動的に無効化し、手動で再有効化するまで User Management REST API が止まる、という報告があります（[SAP Community](https://community.sap.com/t5/human-capital-management-q-a/ias-service-binding-auto-rotates-x-509/qaq-p/14268538)）。**通常のトークン認証／AMS だけなら無関係**です。

> ⚠️ **要環境確認**: 「**未変更の再デプロイでも binding が再作成され新証明書が出る**」は SAP Community の報告ベースで、一次リファレンスの明文ではありません。本番前に自環境で **再デプロイ前後の証明書（serial / notBefore）が変わること**を一度確認してください。`app-identifier` を付けると**ローテーション後も subject（DN）が安定**し、信頼設定が壊れません（capire も「rotation に必要」と明記）。

### 4.2 ランタイム別のローテーション方法

証明書の**ローテーション方法はランタイムで異なります**（質問への直接回答）。

- **Cloud Foundry（classic）**: **完全無停止の自動更新はありません。再デプロイ or リバインドが必要**です。**直バインドの CAP アプリなら、再デプロイ（＝移送）するだけで binding が再作成され新証明書に更新**されます（4.1.1）。**最長 30 日**なので長い期限で頻度を下げる余地は小さく、**月次以内のローテーションが前提**——通常リリース／定期移送に吸収させます（4.2.2）。`${timestamp}` サービスキー方式（下記）は、**binding ではなく service-key を消費する構成**（外部 Consumer 等）向けの選択肢です。手動更新（再デプロイ / unbind→rebind）の使い分けは 4.2.1 を参照。
- **Kyma**: SAP BTP サービスオペレーターの **`credentialsRotationPolicy`** により、**バックグラウンドで自動ローテーション**（再デプロイ不要）。
- いずれの場合も、ライブラリは実行時の証明書差し替えに対応（4.1）なので、新しい証明書がバインディングに届けば**再起動なしで追従**します。

CF のサービスキー自動更新（`${timestamp}`）の記述例 — **これは binding ではなく service-key を消費する構成向け**（外部 Consumer・跨ぎ等）。直バインドの CAP なら不要で、再デプロイだけで回ります（4.1.1）:

```yaml [mta.yaml（CF: サービスキー自動更新／service-key 消費時）]
resources:
  - name: my-consumer-identity
    type: org.cloudfoundry.managed-service
    parameters:
      service: identity
      service-plan: application
      service-keys:
        - name: consumer-key-${timestamp}   # 再デプロイ毎に新キー=新証明書、旧キーは自動削除
          config:
            credential-type: X509_GENERATED
            validity: 30          # 範囲 1–30（既定 30）。1年などは不可
            validity-type: DAYS
  # アプリ側は env-var-name で固定エイリアス参照にする（キー名が毎回変わるため）
```

```mermaid
flowchart TB
    subgraph AUTO["自動化できる（推奨）"]
        direction TB
        A1["Cloud Foundry<br/>直バインド: 再デプロイで binding 再作成<br/>（service-key 消費時のみ ${timestamp}）"]
        A2["Kyma<br/>credentialsRotationPolicy<br/>（バックグラウンド自動）"]
    end
    subgraph MANUAL["手動更新が必要"]
        direction TB
        M1["Destination に<br/>独自証明書を手動アップロード<br/>→ 期限前に差し替え＆Destination更新"]
    end
    AUTO -->|運用負荷: 低| OK([期限切れ事故を防ぎやすい])
    MANUAL -->|運用負荷: 高| CARE([期限管理を忘れると失敗])

    style AUTO fill:#d5f9e0,stroke:#27ae60,color:#1a1a1a
    style MANUAL fill:#f9e6d5,stroke:#e67e22,color:#1a1a1a
```

### 4.2.1 再デプロイ（MTA）と unbind→rebind、どちらで更新するか

証明書を切り替える（＝新しいバインディング証明書に更新する）手段は主に 2 つあります。**どちらが良いかは「デプロイ成果物とパイプラインを持っているか」で変わります**——一方的に再デプロイが上ではありません。

| | MTA 再デプロイ（binding 再作成） | 手動 unbind → rebind |
|---|---|---|
| **前提（必要なもの）** | **デプロイ可能な成果物（`mta.yaml`/`mtar`）＋デプロイ権限** | **`cf` CLI アクセスのみ（ソース・パイプライン不要）** |
| 方式 | 宣言的（`mta.yaml` に記述） | 命令的（`cf` CLI の手作業） |
| 自動化 | CI/CD で定期実行できる | 都度手動・忘れやすい |
| 証明書の切替 | 再デプロイで **binding 再作成 → 新証明書**（旧は置き換え。service-key 消費時は `${timestamp}` で旧キー自動削除） | unbind を先にすると**資格情報が一時的に消える（ギャップ）** |
| 再現性 / 監査 | ソース管理・パイプラインに残る | 記録が残りにくい |
| 向く場面 | **CI/CD 管理下の定常ローテーション** | **ソースを持たない運用者**の対応、**緊急**（鍵漏洩の疑い）、MTA 管理外の単発 |

- **ソース／パイプラインを持つチーム**なら、定常ローテーションは **MTA 再デプロイが既定**。宣言的で、直バインドなら再デプロイだけで binding が再作成され新証明書に更新、CI/CD に載せられ監査も効く。
- **ソースや mtar を持たない運用者**（本番を `cf` で見るだけの立場、緊急対応など）にとっては、**unbind→rebind の方が現実的**。再デプロイは成果物とデプロイ権限がないと実行できないが、unbind→rebind は `cf` アクセスだけで完結する——ここが再デプロイにない強み。
- unbind→rebind を使うときは「**新しいバインドを先に作ってから古いバインドを外す**」順にしてギャップ（認証断）を避ける。unbind を先にすると、その間アプリは有効な資格情報を持たない。
- ⚠️ **MTA 管理下のリソースを手で unbind/rebind すると、次の `cf deploy` がデスクリプタ定義に合わせて元に戻す（ドリフト）**ことがある。MTA で管理しているなら手動更新はその場しのぎと割り切り、恒久策は通常の再デプロイ（＝移送）に寄せるのが安全。
- どちらでも、CF では新しい資格情報を読ませるためにアプリの **restage / 再起動**が要る（ライブラリの実行時差し替え（4.1）を使わない限り）。

### 4.2.2 定期ローテーションを cTMS 移送と衝突させずに回す

**採用方針**: CF では **再デプロイ方式**を基本にする（**CAP デフォルトの直バインドのまま、構成変更なし**）。UPS 経由で資格情報を配布し定期更新する案も検討したが、①UPS は配布層であって証明書の発生源にならず結局ローテーション・ジョブの自作が要る、②秘密鍵のコピーが増える、③自動検出されず資格情報ロケーションの手当てが要る、④restage 前提、と割に合わないため見送り。ZTIS（`X509_ATTESTED`、自動ローテーション）は最も楽だが、**環境での利用可否が要確認**（公開ドキュメント・Kyma モジュールは存在するが CF/顧客利用の可否は landscape 依存）なので、確実な既定路線は再デプロイ方式とする。

**衝突回避の原則**: 本番へのデプロイ権限を **cTMS 一本に保つ**。ローテーションもその同じ管を通し、GitHub Actions 等から**直接 `cf deploy` する第2経路を作らない**。

前提となる事実 — **直バインドは再デプロイのたびに CF が binding を再作成し、IAS が（設定未変更でも）新証明書を発行**する（[SAP Community](https://community.sap.com/t5/human-capital-management-q-a/ias-service-binding-auto-rotates-x-509/qaq-p/14268538) 報告、4.1.1）。ゆえに「変更が無くても再移送すればローテーションになる」。（service-key 消費構成の場合は、`${timestamp}` が **mtar に焼き込まれずデプロイ時に解決**される性質で同じ効果を得る＝[service-keys 公式](https://github.com/SAP-docs/btp-cloud-platform/blob/main/docs/30-development/service-keys-32297f1.md)。）

推奨パターン:

1. **トリガは cTMS 移送に向ける**（直 `cf deploy` にしない）。スケジューラがやるのは cTMS 移送のキックで、出口は常に cTMS ＝書き手は1つ。
2. **無操作フォールバックにする**: 一律に定期移送するのではなく「**本番に直近 ~25 日 移送が無ければ、現行の本番承認版を再移送する**」ゲート付きにする。通常リリースが月内にあればフォールバックは発火せず、**通常移送と重ならない**。cTMS はノード単位で移送をシリアライズするため、仮に重なってもクロバーせずキューされる。
3. **再移送するのは現行の本番承認版**（main の先端や任意スナップショットではない）。コード変更を持ち込まない。
4. **安全網**: `validity: 30`（最大）で窓を最大化／発火は ~20〜25 日の余裕を持たせる（移送1回のスリップ＝失効＝障害）／**証明書・バインディングの経過日数を監視しアラート**（失効忘れが証明書運用最大の事故）。

| | 内容 |
|---|---|
| 既定方針 | 再デプロイ方式を cTMS 経由で（CAP デフォルトの直バインドのまま、構成変更なし） |
| 発火条件 | 直近 ~25 日 本番移送が無い時だけ（無操作フォールバック） |
| 移送対象 | 現行の本番承認版（変更を持ち込まない） |
| 窓 / 余裕 | `validity: 30`、発火は 20〜25 日目安 |
| 監視 | 証明書/バインディング経過日数のアラート必須 |

> **上級策（任意）**: 回転する identity 資格情報リソースだけを**別 MTA / 別移送ノード**に切り出すと、ローテーション移送がアプリのコードに触れず blast radius を最小化できる。構成は一段複雑になるので、まずは上記 1–4 で足りることが多い。

> ⚠️ **要環境確認**: cTMS で「**同一バージョンの再移送**」を再実行できるか（インポートキューの再処理・API 駆動での再トリガ可否）は、パイプライン / cTMS 設定に依存する。定期ローテーションを組む前に自環境で一度確認すること。

### 4.3 手動更新が残るケース

- **Destination に独自の証明書（キーストア）を自分でアップロード**してトークン取得の mTLS に使う場合。→ 有効期限前に、更新した証明書をアップロードして Destination を手動更新する必要があります。SAP 公式も「有効期限前に、更新した証明書をアップロードして Destination を手動更新する（Rotate certificates before expiry by uploading the updated destination certificate）」と明記しています。
- なお BTP Destination サービスには「デフォルトクライアント証明書」を自動生成・自動更新する仕組みもあり、これを使えば手動更新を避けられます。

> **要点**: 「証明書だから毎回手動で入れ替えが必要」ではありません。**アプリを `identity` に `X509_GENERATED` でバインドし、CF は再デプロイ（直バインドは binding 再作成で自動更新／service-key 消費時は `${timestamp}`）、Kyma は `credentialsRotationPolicy`** に任せれば、日々の手作業は不要にできます。手動運用が残るのは「Destination に独自証明書を手で載せた」ケースです。

### 4.4 Destination 側のローテーション操作 🔑

§4.1〜4.3 は **Consumer アプリが IAS からトークンを取るための資格情報（＝バインディング側）**の更新でした。では、コックピットの **Destination 側**には何が必要か——これは **資格情報を Destination に複製しているか**で決まります。§3 の 2 方式がそのまま効いてきます。

`OAuth2ClientCredentials` / `OAuth2JWTBearer` は、トークン取得のために **client id ＋ secret（または証明書）を Destination 設定内にコピーとして保持**します。このコピーは**バインディングのローテーションに自動追従しません**。ここが分岐点です。

| Destination の持ち方 | ローテーション時の Destination 操作 |
|---|---|
| **① 資格情報を持たない**（§3.1：`NoAuthentication` ＋ `cloudsdk.ias-dependency-name`、または共有 identity バインディング） | **不要**。ライブラリが実行時に新しいバインディング資格情報を読む。Destination には資格情報が無いので触らなくてよい |
| **② client secret をインライン**（§3.2 の `clientSecret`） | **必要**。コックピット **Connectivity → Destinations → Edit** で `Client Secret`（変わっていれば `Client ID` も）を新しい値に貼り替え → **Save**。自動化するなら **Destination service REST API** / **MTA の destination-content（`init_data`）** / **Terraform** |
| **③ 証明書キーストアをアップロード**（§3.2 で clientSecret を mTLS 化） | **必要**。Edit で更新キーストアを **再アップロード → Save**（§4.3）。または Destination service の **デフォルトクライアント証明書**（自動更新）を使う |

> **⚠️ 証明書/secret のローテーション（§4.2）と ② の相性に注意**: 再デプロイのたびにバインディングが作り直されて証明書/secret が変わる運用にすると、**それを固定値で持つ静的 Destination は回すたびに壊れます**。両立させたいなら Destination 更新を同じ CI/CD パイプラインで自動化するか、そもそも **① を選んで二重管理を消す**のが定石です。

**結論**: 「Destination に資格情報のコピーを持たせた瞬間、回すものが 2 つ（バインディング＋Destination）になり、両者は同期しない」。これを避けられるのが §3.1 の推奨構成であり、**①なら §4.2 のバインディング更新だけで、Destination は無操作**で済みます。

---

## 関連

- [02. 認証の違い](02-authentication.md) — 認証方式（トークン発行者・フロー・`ias_apis`・mTLS）
- [Technical Communication](../docs/Authorization/TechnicalCommunication.md) — `ias_apis` を AMS 内部ポリシーに変換する認可側の詳細
