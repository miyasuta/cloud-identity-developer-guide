# XSUAA 経験者のための SAP Cloud Identity Services (CIS) 認証・認可 学習ガイド

> XSUAA ベースの開発を理解している人が、**SAP Cloud Identity Services（以下 CIS）** で「何が・なぜ変わったのか」を、図と対応表で素早く掴むための学習ノートです。

## このガイドの狙い

XSUAA から CIS への移行は、単なるサービスの置き換えではなく **設計思想（パラダイム）の転換** を伴います。本ガイドは細かい手順書ではなく、まず **考え方の違い** を腹落ちさせることを目的にしています。

> **略称について**: 正式名称は `SAP Cloud Identity Services`。略称は SAP でも1つに固定されておらず、**CIS** と **SCI** の両方が使われます。本ガイドでは **CIS** に統一します。ただし、このリポジトリの公開ドキュメント（`docs/`）は **SCI** と表記しているため、そちらを読むときは「SCI = CIS」と読み替えてください。

このガイドを読み終えると、次の 4 つを図と表だけで説明できるようになります。

1. **認証と認可の「分離」** — なぜ CIS では役割が分かれたのか
2. **認可がトークンから外れた** — scope 入り JWT がなくなった意味
3. **静的 RBAC → ポリシーベース認可** — `xs-security.json` の scope から DCL ポリシー／実行時評価へ
4. **client secret → 証明書（mTLS）** — 資格情報と信頼の変化

## CIS を構成する 3 つのサービス

XSUAA が「認証委譲・トークン発行・認可（scope）」を **一手に担う中央サーバー** だったのに対し、CIS では役割が 3 サービスに分かれます。

| サービス | 略称 | 役割 | XSUAA 時代の対応 |
|---|---|---|---|
| Identity Authentication Service | **IAS** | 認証・OIDC トークン発行 | XSUAA が担っていた「トークン発行」部分 |
| Authorization Management Service | **AMS** | 認可（ポリシーベース判定） | XSUAA の scope / role による認可部分 |
| Identity Provisioning Service | **IPS** | ユーザー／グループのプロビジョニング | SCIM 連携・ユーザー同期 |

> 本ガイドではサービス全体を **CIS**、認可部分を **AMS** と表記します（前述のとおり、公開 docs では CIS を **SCI** と表記）。

## 読む順序

| # | ページ | 内容 |
|---|---|---|
| — | **README（このページ）** | 全体像・狙い・読む順 |
| 01 | [パラダイムシフト](01-overview-paradigm-shift.md) | 認証と認可の「分離」— 最重要コンセプト |
| 02 | [認証の違い](02-authentication.md) | IAS = OIDC 発行者、mTLS / 証明書、トークン構造、App-to-App の認証方式（フローと `ias_apis`） |
| 03 | [認可の違い](03-authorization.md) | scope/RBAC → DCL ポリシー・実行時 PDP・インスタンスベース認可 |
| 04 | [設定成果物の違い](04-configuration-artifacts.md) | `xs-security.json` → `identity` + `authorization.enabled` + DCL |
| 05 | [ロール割当・管理の違い](05-role-assignment-admin.md) | BTP コックピット（ロールコレクション）→ IAS 管理コンソール（ポリシー割当）、IPS の位置づけ、App-to-App の権限決定者 |
| 06 | ライブラリ・実装の違い *(作成予定)* | `@sap/xssec` scope 判定 → AMS クライアントライブラリ、CAP 連携 |
| 07 | まとめ・移行チートシート *(作成予定)* | XSUAA → CIS 概念対応表・移行観点 |

> 本ガイドは段階的に作成しています。まずは **01 から** お読みください。

### 補足トピック（設定・運用）

| ページ | 内容 |
|---|---|
| [App-to-App 連携の設定と証明書運用](app2app-and-certificate-operations.md) | dependency 登録・Destination 設定・証明書のローテーション運用（02 の認証方式に対する設定/運用面） |
| [共有 IAS アプリと集中 DCL（複数 CAP 構成）](shared-ias-app-central-dcl.md) | 複数の CAP マイクロサービスが 1 つの IAS/AMS アプリ（`identity` インスタンス）を共有する場合の、集中 DCL（schema＋base policy）と各 CAP 側の設定・ビルド（03 の DCL/ポリシーに対する構成面） |
| [ユーザー属性による動的な認可（CAP ＋ AMS）](dynamic-authorization-user-attributes.md) | `$user.division` などユーザー属性を条件に使い、単一ポリシーでロール爆発を回避する ABAC パターン。IAS 属性設定・`schema.dcl` の `$user` 宣言・`getInput` マッピング・ポリシー2方式（03 §3 のインスタンスベース認可の発展） |

## 関連する公開ドキュメント

より実装寄りの詳細は、このリポジトリの公開ガイドを参照してください。

- [Getting Started（AMS の導入）](../docs/Authorization/GettingStarted.md)
- [Technical Communication（技術通信・主体伝播）](../docs/Authorization/TechnicalCommunication.md)
