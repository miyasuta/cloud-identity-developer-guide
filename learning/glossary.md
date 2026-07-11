# IAS / AMS 用語集 — 意味・設定箇所・使う場面

> XSUAA から CIS（SAP Cloud Identity Services）へ移る際に出てくる用語を、**意味／どこで設定するか／どういう場合に使うか** の3点で引けるようにした逆引き用語集です。各用語は、詳しく扱っている章へリンクしています。
>
> **表の読み方**
> - **設定箇所**: その用語が「登場・設定される場所」（ファイル・管理コンソール・コードなど）。「—」は設定対象ではなく概念・自動生成されるもの。
> - **使う場面**: 実務で意識するタイミング。CAP を主眼にしています。

---

## 1. サービス・基盤

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **CIS**（SAP Cloud Identity Services） | 認証・認可・プロビジョニングを束ねるサービス群の総称。公開 docs では **SCI** 表記 | — | CIS 全体を指すとき（[README](README.md)） |
| **IAS**（Identity Authentication Service） | 認証・OIDC トークン発行を担う。XSUAA の「トークン発行」部分の後継 | `identity` サービスインスタンス | ユーザー認証・トークン発行（[02](02-authentication.md)） |
| **AMS**（Authorization Management Service） | 認可（ポリシーベース判定）を担う。XSUAA の scope/role 認可の後継 | `identity` の `authorization.enabled: true` で有効化 | 認可ポリシーの評価（[03](03-authorization.md)） |
| **IPS**（Identity Provisioning Service） | ユーザー／グループを人事・他 IdP と同期 | IPS のソース／ターゲット transformation | ユーザー同期・ポリシー割当の自動化（[05 §4](05-role-assignment-admin.md#4-ipsidentity-provisioningの位置づけ--ポリシー割当も自動化できる)） |
| **Identity Directory** | CIS の永続化レイヤー。全アプリの割当を横断的に集約する中央ハブ | 管理コンソール／SCIM2 API | 複数アプリの権限を横断して割り当てるとき（[05 §3](05-role-assignment-admin.md#複数アプリを横断する割当先--identity-directory-が中央ハブ)） |

---

## 2. 認証 — トークンと資格情報

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **`identity` サービス** | IAS/AMS を提供する BTP サービス。XSUAA の `xsuaa` サービスに相当 | `mta.yaml` の `resources`（`service: identity`） | アプリを IAS にバインドするとき（[04 §2](04-configuration-artifacts.md#2-サービス定義の成果物--xsuaa-リソース--identity-リソース)） |
| **X.509 証明書 / mTLS** | client secret に代わる資格情報。相互 TLS で本人性を証明 | サービスバインディング（証明書はプラットフォームが発行） | アプリ⇔IAS、App-to-App の認証（[02 §4](02-authentication.md#4-資格情報の変化client-secret--x509-証明書mtls)） |
| **`cnf.x5t#S256`** | トークンに埋め込まれる証明書サムプリント。所有証明（proof of possession） | トークンに自動付与 | mTLS でトークンの持ち主を検証するとき（[02](02-authentication.md)） |
| **proof token 検証** | `@sap/xssec` が行う所有証明・x5t 検証。`cert` ルートで既定有効 | `cds.requires.auth.config.validation` | 既定で有効。無効化したいときだけ設定（[06 参考](06-libraries-implementation.md)） |
| **`iss` / `ias_iss`** | トークン発行者（IAS テナントの URL） | トークンに自動付与 | トークン検証・信頼確認 |
| **`app_tid`** | アプリのテナント識別子（マルチテナント） | トークンに自動付与 | テナント判別 |
| **`azp`** | authorized party（トークンを受け取るクライアント） | トークンに自動付与 | 呼び出し元の識別 |
| **`aud`** | audience（トークンの宛先アプリ） | トークンに自動付与 | 宛先検証（App-to-App では Provider） |
| **scope クレーム（不在）** | XSUAA と違い IAS トークンには **scope（認可情報）が入らない** | — | 「認可はトークンに載らない」を理解する起点（[01](01-overview-paradigm-shift.md)） |
| **App Router（approuter）** | 対話ログインフローを担う BTP コンポーネント | `cds add approuter`／`app/router` | ユーザーの IAS ログイン時（[06](06-libraries-implementation.md)） |

---

## 3. 認可モデル — DCL

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **DCL**（Data Control Language） | 認可ポリシーを書く専用言語 | `*.dcl` ファイル（`dclRoot` 配下） | 認可を定義するとき（[03 §2](03-authorization.md#2-dcl-入門--認可をコードで書く)） |
| **DCN**（Data Control Notation） | DCL をコンパイルした機械可読の中間形式 | `gen/dcn`（ビルド生成物） | ローカルテスト・実行時評価（自動生成） |
| **`dclRoot`** | DCL ファイルを置くルートフォルダ | Node `ams/dcl`／Java `srv/src/main/resources/ams` | DCL の配置場所（[04 §3](04-configuration-artifacts.md#3-認可定義の成果物--xs-securityjson-の中身--dcl-ファイル群)） |
| **`schema.dcl`** | 属性（AMS attribute）の型付き宣言。旧 `xs-security.json` の `attributes` | `dclRoot` 直下 | ABAC/インスタンスベースの属性を宣言するとき（[03 §2](03-authorization.md#属性条件とスキーマ)） |
| **`POLICY`** | 認可ポリシー。旧 role-template に相当 | base policy の `*.dcl` | ロール／権限を定義するとき（[03 §2](03-authorization.md#2-dcl-入門--認可をコードで書く)） |
| **アクション × リソース** | `GRANT <action> ON <resource>` の判定単位。旧 scope に相当 | `*.dcl` の `POLICY` 内 | 非 CAP で権限を定義するとき（[03 §2](03-authorization.md)） |
| **`WHERE` / `RESTRICT`** | 属性条件（ABAC）。旧 role-template の attribute に相当 | `POLICY` 内の条件式 | カテゴリ・ジャンル等で絞るとき（[03 §2](03-authorization.md#属性条件とスキーマ)） |
| **`IS NOT RESTRICTED` / `IS RESTRICTED`** | 管理者が絞り込んで「よい／なければ不可」を示すガードレール | base policy 内 | 開発者が管理者の裁量枠を決めるとき（[03 §2](03-authorization.md#base-policy-と実行時ポリシー開発者-vs-管理者)） |
| **`ASSIGN ROLE`** | cds ロールを割り当てる CAP 専用の糖衣構文（＝ `GRANT <role> ON $SCOPES`） | base policy の `*.dcl`（CAP は自動生成） | CAP で「そのロールを誰に割り当てるか」を定義（[03 §2](03-authorization.md#cap-の場合はロールベース)） |
| **`$SCOPES`** | cds ロールを表す特別なリソース。CAP では通常唯一の AMS リソース | DCL 内（自動） | CAP のロールモデルを理解する（[03 §2](03-authorization.md#cap-の場合はロールベース)） |
| **`@ams.attributes`** | cds モデル要素を DCL の属性へマッピングする CAP アノテーション | cds モデル | CAP でインスタンスベース認可を書くとき（[03 §3](03-authorization.md#3-インスタンスベース行レベル認可-)） |
| **`$user.<attr>`** | ユーザープロファイル属性を条件に使う参照 | `schema.dcl` 宣言 ＋ ポリシー条件 | 単一ポリシーでユーザーごとに絞る ABAC（[動的認可](dynamic-authorization-user-attributes.md)） |

---

## 4. 認可ポリシーの種類

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **base policy** | 開発者が DCL ソースに書く土台のポリシー。全テナントに移送される | `*.dcl`（deployer で配布） | 通常の認可定義（[03 §2](03-authorization.md#ポリシーの移送--base-policy-はコードとして移送custom-policy-はテナントに残る)） |
| **custom policy** | 管理者が管理コンソールで base から派生（`USE ... RESTRICT`）。そのテナントのみ | IAS 管理コンソール | テナント固有調整・緊急対応（本流ではない。[05 §2](05-role-assignment-admin.md#2-割当の手順実際の画面遷移)） |
| **`DEFAULT POLICY`** | 割当なしで全テナントに自動適用される base policy | `*.dcl` | 全ユーザー共通の既定権限を配りたいとき（[03 §2](03-authorization.md#ポリシーの移送--base-policy-はコードとして移送custom-policy-はテナントに残る)） |
| **`INTERNAL POLICY`** | API 権限グループで呼ばれた相手に一律適用。管理者に見えず割当不要 | Provider の `internal` DCL パッケージ | **主体伝播で外部アプリの権限を絞るときのみ。自社内 CAP 連携ではまず使わない**（[06 §5](06-libraries-implementation.md#5-app-to-app技術通信の実装--ias_apis-からロール自動付与)） |
| **`USE ... RESTRICT`** | 既存ポリシーを絞り込んで派生ポリシーを作る構文 | base／custom policy | 行レベル絞り込み・管理者の実行時調整（[03 §3](03-authorization.md#3-インスタンスベース行レベル認可-)） |
| **Combine Authorization Policies** | 同一アプリ内の複数ポリシーを1つに合成する管理機能 | IAS 管理コンソール | 同一アプリ内で割当をまとめたいとき（アプリ跨ぎ不可。[05 §3](05-role-assignment-admin.md#3-グループの役割の違い-)） |

---

## 5. 認可の配布・評価

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **Authorization Bundle** | AMS が base＋実行時ポリシーを中央コンパイルした配布物 | —（AMS が生成） | 実行時評価の仕組みを理解する（[03 §4](03-authorization.md#4-判定はどこでいつ起きるか--実行時-pdp-と-authorization-bundle)） |
| **PDP**（Policy Decision Point） | アプリのプロセス内でポリシーを評価する判定器 | クライアントライブラリ内（自動） | 「判定はアプリ内・外部呼び出しなし」を理解する（[03 §4](03-authorization.md#4-判定はどこでいつ起きるか--実行時-pdp-と-authorization-bundle)） |
| **AMS Policies Deployer App** | DCL を AMS インスタンスへアップロードする独立デプロイ成果物 | `gen/policies`（CAP は自動生成） | DCL を dev→test→prod へ移送するとき（[04 §4](04-configuration-artifacts.md#4-ビルドデプロイの成果物--cds-add-ams-が生成するもの)） |
| **`autoDeployDcl`** | 起動時に DCL を自動アップロードする Node.js オプション（既定無効） | `cds` 設定（cds-Plugin） | ハイブリッドテスト時（[04 §4](04-configuration-artifacts.md#4-ビルドデプロイの成果物--cds-add-ams-が生成するもの)） |

---

## 6. 割当・管理

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **認可ポリシー（割当単位）** | CIS での割当対象。XSUAA のロールコレクションに代わる | IAS 管理コンソール（Authorization Policies タブ） | ユーザーに権限を割り当てるとき（[05 §1](05-role-assignment-admin.md#1-割当という行為の対比)） |
| **User Group（自動生成グループ）** | ポリシー作成時に同名で自動生成されるグループ。追加＝割当 | IAS 管理コンソール（Users & Authorizations → User Groups） | 割当の実体を理解する（[05 §3](05-role-assignment-admin.md#3-グループの役割の違い-)） |
| **Role Collection（ロールコレクション）** | 【XSUAA】複数アプリの role-template を束ねる器。CIS に同等物なし | 【対比用】BTP コックピット | XSUAA との対比（[05 §1](05-role-assignment-admin.md#1-割当という行為の対比)） |
| **Assignments ペイン** | ポリシーにユーザーを追加する管理コンソール UI（＝グループ操作） | IAS 管理コンソール | 手動でユーザーを割り当てるとき（[05 §2](05-role-assignment-admin.md#2-割当の手順実際の画面遷移)） |
| **SCIM2 / `/Groups` API** | 業界標準のプロビジョニング API。HR/IGA が標準対応 | Identity Directory の SCIM2 エンドポイント | 既存 IGA ツールで一括割当するとき（[05 §3](05-role-assignment-admin.md#複数アプリを横断する割当先--identity-directory-が中央ハブ)） |
| **`assignGroup` / `condition`** | IPS transformation で属性条件によりポリシー用グループへ自動割当 | IPS ターゲット transformation | 属性ベースの割当自動化（粗いポリシー向け。[05 §4](05-role-assignment-admin.md#4-ipsidentity-provisioningの位置づけ--ポリシー割当も自動化できる)） |

---

## 7. App-to-App・技術通信

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **App-to-App Integration** | アプリ間通信を IAS の Provider/Consumer モデルで認可する仕組み | 管理コンソール／`identity` 設定 | サービス間連携（[05 §5](05-role-assignment-admin.md#5-app-to-app-どのアプリがどの-api-を消費してよいかも管理者が決める)、[app2app](app2app-and-certificate-operations.md)） |
| **API 権限グループ（`provided-apis`）** | Provider が公開する API の名前ラベル。`ias_apis` に載る文字列。**ポリシーではない** | `identity` の `config.provided-apis`（`mta.yaml`）／Trust タブ | Provider が API を公開するとき（[app2app](app2app-and-certificate-operations.md)） |
| **`ias_apis`（クレーム）** | 呼び出し元が消費を許可された API 権限グループ名の配列 | トークンに自動付与（provided-apis ＋ dependency 承認の結果） | 消費可能な API を実行時に判定（[02 §5](02-authentication.md#5-app-to-app-の認証方式技術通信--主体伝播)） |
| **dependency** | Consumer 側が「この API を消費する」と登録する宣言 | Consumer の Trust タブ Dependencies | 他アプリの API を呼ぶ側の設定（[app2app](app2app-and-certificate-operations.md)） |
| **technical user（技術ユーザー）** | システム名義の呼び出し。CAP は `ias_apis` から同名 cds ロールを自動付与 | —（トークンのフローで決まる） | サービス連携・バッチ（[06 §5](06-libraries-implementation.md#5-app-to-app技術通信の実装--ias_apis-からロール自動付与)） |
| **principal propagation（主体伝播）** | エンドユーザーの identity を転送する呼び出し | Destination `OAuth2JWTBearer` 等 | ユーザーの代理で他アプリを呼ぶとき（[app2app](app2app-and-certificate-operations.md)） |
| **`principal-propagation`（All APIs）** | 上限を設けず、ユーザー本来のポリシーをそのまま適用する特別グループ | `provided-apis` に任意で公開 | 主体伝播で権限を絞らないとき（[06 §5](06-libraries-implementation.md#internal-policy-とは--cap-では基本的に使わない)） |
| **Destination（`OAuth2ClientCredentials` / `OAuth2JWTBearer`）** | 技術通信＝前者、主体伝播＝後者。宛先ごとの接続設定 | BTP Destination サービス | 呼び出し先への接続を定義するとき（[app2app](app2app-and-certificate-operations.md)） |

---

## 8. ライブラリ・CAP 連携

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **`@sap/xssec`** | トークン検証ライブラリ。IAS 対応は v4 系 | `package.json` | 認証（CAP は自動連携。[06 §1](06-libraries-implementation.md#1-全体像--どのライブラリが何に置き換わるか)） |
| **`@sap/ams`** | Node.js の AMS クライアント／CAP プラグイン | `package.json`（`cds add ams` が追加） | CAP/Node の認可（[06 §3](06-libraries-implementation.md#3-cap-の場合実はほとんど変わらない-)） |
| **`@sap/ams-dev`** | ローカルで DCL→DCN をコンパイルする開発用モジュール | `package.json`（devDependencies） | ローカル開発・テスト（[06](06-libraries-implementation.md)） |
| **`cap-ams-support` / `jakarta-ams`** | Java（CAP）の AMS 連携・クライアント | `pom.xml`（`cds add ams` が追加） | CAP Java の認可（[06 早見表](06-libraries-implementation.md#ライブラリ早見表)） |
| **`ams-core` / `spring-boot-ams`** | 非 CAP Java の AMS クライアント | `pom.xml` | 素の Java で認可を書くとき（[06 §7](06-libraries-implementation.md#7-参考非-capplain-nodejs--java--goの場合)） |
| **認証 `kind`（`ias`）** | CAP の認証戦略。XSUAA は `xsuaa` | `package.json` の `cds.requires.auth` | IAS 認証に切り替えるとき（[06 §3](06-libraries-implementation.md#では開発者は実際に何を変えるのか)） |
| **`ias-auth`（バインド kind）** | ハイブリッドテストで `cds bind` が付与するバインド種別 | `.cdsrc-private.json`（自動） | ローカルから IAS インスタンスに接続するとき（[06](06-libraries-implementation.md)） |
| **`cds add ams`** | AMS＋IAS プラグイン導入・`mta.yaml` 更新をまとめて行う CLI | プロジェクトルートで実行 | CAP に AMS を導入するとき（[04 §4](04-configuration-artifacts.md#4-ビルドデプロイの成果物--cds-add-ams-が生成するもの)） |
| **`cds build --for ams`** | cds アノテーションから DCL を生成するビルド | ビルド時（Maven は plugin） | DCL を自動生成するとき（[06 §3](06-libraries-implementation.md#では開発者は実際に何を変えるのか)） |
| **`AuthorizationsProvider` / `SciAuthorizationsProvider`** | リクエストの認可集合を組み立てる（Node: `IdentityServiceAuthProvider`） | コード（非 CAP／マッパー登録時） | 非 CAP、または App-to-App マッパー登録（[06 §5](06-libraries-implementation.md#5-app-to-app技術通信の実装--ias_apis-からロール自動付与)） |
| **`HybridAuthProvider`（Java: `HybridAuthorizationsProvider`）** | XSUAA scope を AMS ポリシーへマッピングする移行用プロバイダ | コード（`ScopeMapper` を渡す） | XSUAA→AMS 移行期に両トークンを受けるとき（[06 §6](06-libraries-implementation.md#6-移行を助ける仕組み--xsuaa-と-ams-の橋渡し)） |
| **`checkPrivilege()` / `Decision` / `isGranted()`** | 【非 CAP】action×resource を判定する API と結果 | コード | 非 CAP でプログラム的に判定（[06 §7](06-libraries-implementation.md#7-参考非-capplain-nodejs--java--goの場合)） |
| **`getPotentialResources/Actions/Privileges`** | 付与されうる権限を列挙（UI 事前チェック用。条件は無視） | コード | メニュー・ボタンの出し分け（[06 §7](06-libraries-implementation.md#7-参考非-capplain-nodejs--java--goの場合)） |
| **`req.user.is('role')`** | cds ロール判定 API。XSUAA/IAS で書き方は同じ | ハンドラのコード | プログラム的にロールを確認（[06 §2](06-libraries-implementation.md#2-xsuaa-時代のおさらい)） |

---

## 9. 証明書・運用

| 用語 | 意味 | 設定箇所 | 使う場面 |
|---|---|---|---|
| **`X509_GENERATED`** | SAP 管理の自動発行証明書（`validity` 指定可） | `identity` バインディング設定 | 証明書ローテーションを自動化したいとき（[app2app](app2app-and-certificate-operations.md)） |
| **Automatic Service Key Renewal** | CF でサービスキーを自動更新する仕組み（`${timestamp}`） | `mta.yaml` の `service-keys` | CF で証明書を自動切替するとき（[app2app](app2app-and-certificate-operations.md)） |
| **`credentialsRotationPolicy`** | Kyma でバインディング資格情報を自動ローテーション | Kyma の ServiceBinding | Kyma で証明書を自動更新するとき（[app2app](app2app-and-certificate-operations.md)） |

---

## 関連ページ

- 各用語の背景は本ガイドの [README](README.md) と 01〜07 章を参照してください。
- 実装寄りの詳細は公開ドキュメント（[Getting Started](../docs/Authorization/GettingStarted.md) / [Technical Communication](../docs/Authorization/TechnicalCommunication.md) / [Authorization Checks](../docs/Authorization/AuthorizationChecks.md)）にあります。
