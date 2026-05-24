# 政府内部システム移行プロジェクト — プロジェクトコンテキスト定義書 v7

> **本文書の目的**: プロジェクトの業務概要・技術スタック・アーキテクチャ・API設計方針を一元管理する。
> コード生成・設計レビュー・ドキュメント作成の際は本文書を最優先の前提情報として参照すること。
> **更新履歴**: v6（JAFFIC クラウド基盤準拠 + Azure Files/SFTP + セキュリティ・監視・NW 拡充）→ **v7（第2段階PaaS廃止確定 + .NET 10移行 + 認証再設計 + ログ・レポート・外部IF詳細確定 + 設計書MD一式追加）**

---

## 1. プロジェクト概要

| 項目 | 内容 |
|------|------|
| **プロジェクト名** | 林業信用基金基幹システム（JAFFIC 林業信用基金基幹システム） |
| **上位基盤** | **JAFFIC クラウド（Microsoft Azure 上に構築された政府共通クラウド基盤）** |
| **目的** | 既存 ASP.NET Web Forms + VB.NET システムを Angular 18 + **.NET 10** に全面刷新 |
| **対象業務** | 保証業務 / 査定業務 / 推進業務（出資者管理）/ 経理業務 / 共通 + **貸付業務（新規）/ 寄託業務（新規）** |
| **ユーザー** | 信用基金林業部門・信用基金経理課（約40名、政府内部職員） |
| **画面数** | 約100画面 |
| **言語** | 日本語のみ（多言語対応なし） |
| **認証** | Microsoft Entra ID による SSO（JAFFIC 既存テナントを継続利用・MFA 対応） |
| **連携システム** | 格付判定システム（外部）/ 財務会計システム（BASE-One）/ 融資機関システム / 債権回収会社 |
| **OCR** | AnyForm OCR（ハンモック社）— オンプレミス独立稼働 |

### JAFFIC クラウド環境における位置付け

```
JAFFIC クラウド環境（Microsoft Azure）
├── 農業（業務アプリ + 個別ミドル + DB論理）   ← 農業事業者
├── ★林業（業務アプリ + 個別ミドル + DB論理） ← 本プロジェクト（林業事業者）
├── 漁業（業務アプリ + 個別ミドル + DB論理）   ← 漁業事業者
├── 文書（業務アプリ + 個別ミドル + DB論理）   ← 文書事業者
├── 財務（業務アプリ + 個別ミドル + DB論理）   ← 財務事業者
└── 基幹LAN（資産管理 + プロキシ）             ← 基幹LAN事業者
    │
    ├─── DB(物理) / サーバ / ストレージ ────── JAFFIC クラウド事業者が共通管理
    ├─── NW / 監視 / ログ / バックアップ / セキュリティ（共通基盤）
    └─── FW/WAF / 外部DNS / 統合アカウント / プロキシ
```

### デプロイ方式（★v7変更: 第2段階PaaS廃止確定）

| フェーズ | 構成 | フロントエンド | バックエンド | DB |
|---------|------|--------------|------------|-----|
| **確定（唯一）** | **VM + IIS** | IIS Root `/`（Angular 静的ファイル） | IIS Sub-App `/api`（Kestrel in-process） | Azure SQL Database |

> **⚠️ 重要（v7確定）**: 第2段階（Azure Static Web Apps + App Service）は**廃止・削除**。
> VM + IIS が最終デプロイ方式。PaaS 移行の記述はすべて無効。
> Azure SQL Managed Instance → **Azure SQL Database** に変更（JAFFIC クラウド基盤の実態に合わせる）。

---

## 2. 画面構成とサブシステム設計方針

### 2.1 設計方針：旧システム画面軸に準拠

旧システム（ASP.NET Web Forms + VB.NET）はメニュータブ「保証 / 査定 / 推進 / 経理 / 共通」の5分類で構成されており、新システムもこの画面軸を引き継ぐ。

**2軸の関係**：

| 軸 | 内容 | 用途 |
|---|---|---|
| **画面軸**（メニュータブ） | 保証 / 査定 / 推進 / 経理 / 共通 | URIサブシステム・Angular feature・Controller フォルダの分類基準 |
| **業務フロー軸** | 債務保証_平時 / 債務保証_有事 / 貸付 / 寄託 | 設計書・業務概略図での分類（APIの内部ロジック設計に使用） |

> **重要**: URIのサブシステム名は「画面軸」に従う。業務フロー軸（平時/有事等）はAPI内部の設計分類として参照する。

### 2.2 サブシステム構成（7 + 1）

| # | サブシステム | URI | Angularタブ | 旧フォルダ対応 | 由来 |
|---|------------|-----|------------|--------------|------|
| 1 | **HOSHO（保証）** | `/hosho` | 保証タブ | `hoshou/shinsa/` | 旧システム引継ぎ |
| 2 | **SATEI（査定）** | `/satei` | 査定タブ | `hoshou/kanri/` | 旧システム引継ぎ |
| 3 | **SUISHIN（推進）** | `/suishin` | 推進タブ | 推進フォルダ | 旧システム引継ぎ |
| 4 | **KEIRI（経理）** | `/keiri` | 経理タブ | `keiri/` | 旧システム引継ぎ |
| 5 | **KYOTSU（共通）** | `/kyotsu` | 共通タブ | `kyoutsu/` | 旧システム引継ぎ |
| 6 | **KASHI（貸付）** ★新規 | `/kashi` | 推進タブ内 | 該当なし | **今回新規追加** |
| 7 | **KITAKU（寄託）** ★新規 | `/kitaku` | 推進タブ内 | 該当なし | **今回新規追加** |
| 8 | **AUTH（認証）** | `/auth` | 全業務共通 | — | 新規 |

> KASHI / KITAKU は推進タブ内のサブメニューとして配置する。
> URI メソッド名は **lowerCamelCase 英語**（例: `jikoToroku`）とし、Controller ActionMethod 名と 1:1 対応させる。

### 2.3 各サブシステムの画面・機能概要

#### HOSHO — 保証タブ（旧システム引継ぎ）

| 画面区分 | 主な画面・機能 | API 番号帯 |
|---------|-------------|----------|
| 保証審査 | 保証審査申込受付・保証変更入力・更新延長入力・被保証者別保証状況 | HOSHO10xx |
| 格付・審査 | 随時格付・格付判定（外部連携 Polly）・保証条件計算 | HOSHO20xx |
| 保証引受 | 保証書登録・出資金申請・貸付実行報告書 | HOSHO30xx |
| 期中管理 | 債還状況入力・債還状況管理 | HOSHO40xx |
| 管理業務 | 月次処理・保証料計算・統計資料作成・ブルーフリスト印刷・未収保証料送金依頼 | HOSHO50xx〜60xx |

#### SATEI — 査定タブ（旧システム引継ぎ）

| 画面区分 | 主な画面・機能 | API 番号帯 |
|---------|-------------|----------|
| 事故・代弁 | 事故登録・事故報告起案・代弁請求起案・利息損害金計算・償還入力 | SATEI10xx〜20xx |
| 求償権管理 | 求償権管理台帳（一覧表示・台帳編集）・求償権償却 | SATEI30xx〜40xx |
| 管理業務 | 月次処理・保証人情報・自己査定結果取込・ブルーフリスト印刷 | SATEI50xx〜60xx |

#### SUISHIN — 推進タブ

| 画面区分 | 主な画面・機能 | API 番号帯 |
|---------|-------------|----------|
| 出資者管理（旧） | 出資者一覧・出資者マスタ・出資登録（譲渡/受入/払戻）・月次処理 | SUISHIN10xx〜20xx |
| 貸付業務（新規★） | 受付・審査・貸付引受・契約変更・期中管理・償還業務・統計資料 | KASHI10xx〜70xx |
| 寄託業務（新規★） | 受付・審査・貸付引受・契約変更・期中管理・償還業務（貸付と同構成） | KITAKU10xx〜70xx |

#### KEIRI — 経理タブ

| 画面区分 | 主な画面・機能 | API 番号帯 |
|---------|-------------|----------|
| 連携データ | 連携データ検索（保証料・回収金・入金実績）・未収金入力 | KEIRI10xx |
| 集計・出力 | 前受収益計算書出力 | KEIRI20xx |

#### KYOTSU — 共通タブ

| 画面区分 | 主な画面・機能 | API 番号帯 |
|---------|-------------|----------|
| 横断参照 | 出資者別利用状況・コードマスタ取得・共通機能 | KYOTSU00xx |

---

## 3. 技術スタック（確定版）

### 3.1 フロントエンド

| 項目 | 採用技術 | バージョン | 備考 |
|------|---------|---------|------|
| フレームワーク | Angular | **18.2.x**（将来 Angular 20 推奨） | VPN内部システムのためリスク低。`ng update` で随時升级 |
| 言語 | TypeScript | 5.4.x | |
| UI ライブラリ | Angular Material + CDK | 18.2.x | Material 3 デザイン |
| 状態管理 | Angular Signals | — | RxJS と併用 |
| HTTP 通信 | HttpClient + HttpInterceptor | — | MsalInterceptor / ErrorInterceptor / LoadingInterceptor |
| ルーティング | @angular/router | 18.2.x | Lazy Loading（サブシステム単位） |
| フォーム | Reactive Forms | — | ReactiveFormsModule |
| 認証 | @azure/msal-angular + @azure/msal-browser | 3.x | Entra ID SSO。Authorization Code Flow + PKCE |
| 通知 | ngx-toastr | 19.x | nrmList→緑 / wrnList→黄 / errList→赤 |
| 多言語 | なし | — | 日本語固定 |
| 配置 | IIS Root `/` | — | VM 内。Same-Origin で CORS 不要 |

**主要 Interceptor**:

| ファイル | 役割 |
|---------|------|
| `auth.interceptor.ts` (MsalInterceptor) | Entra ID Access Token を全リクエストに自動付与 |
| `error.interceptor.ts` | 401→MSAL再ログイン / 403→forbidden画面表示のみ（アカウントロックなし） / 500→エラートースト |
| `loading.interceptor.ts` | リクエスト中のプログレスバー自動表示 |

### 3.2 バックエンド（★v7変更: .NET 10 に移行）

| 項目 | 採用技術 | バージョン | 備考 |
|------|---------|---------|------|
| フレームワーク | ASP.NET Core | **.NET 10** | v6の.NET 8から移行確定 |
| 言語 | C# | 13 | |
| API 形式 | REST API（Controller ベース） | — | Swagger / OpenAPI（Swashbuckle） |
| 認証 | Microsoft.Identity.Web | 2.x | Entra ID JWT 検証 |
| バリデーション | FluentValidation | 11.x | フロントエンド: 入力検証（必須/長さ/形式/範囲/相関）/ バックエンド: 業務チェック（マスタ存在/重複/業務ルール） |
| マッピング | AutoMapper | 13.x | Entity ↔ DTO |
| ログ | Serilog | 8.x | ファイル出力のみ（★DB出力廃止 → 後述） |
| 外部 API 耐障害 | Polly | 8.x | リトライ3回 + サーキットブレーカー |
| メール送信 | MailKit | — | SMTP Relay + Basic 認証。認証情報は Azure Key Vault 管理 |
| 配置 | IIS Sub-App `/api` | — | VM 内。Kestrel in-process |

**Middleware Pipeline（処理順序）**:

```
❶ ExceptionHandlerMiddleware  → 全例外キャッチ → errList[ESYS50001] + Serilog出力
❷ Serilog RequestLogging      → 全リクエスト/レスポンスをファイルに記録
❸ Authentication              → Microsoft.Identity.Web が Entra ID JWT を検証
❹ Authorization               → [Authorize(Roles=...)] / [Authorize(Policy=...)]
❺ MapControllers              → Controller にルーティング
```

### 3.3 認証・認可（★v7詳細確定）

| 項目 | 方式 | 備考 |
|------|------|------|
| 認証方式 | MSAL-based SSO（Entra ID） | v6の二要素メール認証から変更 |
| ロール管理 | UserRole テーブル（業務DB内） | Entra ID App Registration には格納しない |
| ロール制御 | メニューレベル RoleGuard | ログイン後のアクセス制御 |
| 未認可アクセス | 403 画面表示のみ | アカウントロックなし |
| UserRole テーブル構造 | `user_oid` / `role_cd` / `valid_flg` / `upn` / `updated_at` / `updated_by` | 業務DB（GovSystemDB）に配置 |
| MFA | Entra ID テナント設定に依存 | アプリ側実装なし |

### 3.4 データアクセス

| 項目 | 採用技術 | バージョン | 備考 |
|------|---------|---------|------|
| ORM | Entity Framework Core | 10.0.x | Code First + Fluent API + Migration |
| パターン | Repository\<T\> + UnitOfWork | — | `IRepository<T>` / `IUnitOfWork` |
| 補助 | Dapper | 2.1.x | 複雑クエリ・集計処理用 |
| 接続方式 | DefaultAzureCredential（Managed Identity） | — | .NET API（IIS Sub-App）→ Azure SQL Database |
| 設定 | appsettings.json | — | 接続文字列管理 |
| BackgroundService | 同一プロセス・同一 AppDbContext | — | CsvFileWatcherService 等 |

### 3.5 ログ出力（★v7変更: DB出力廃止）

| 項目 | 方式 | 備考 |
|------|------|------|
| ログ基盤 | Serilog | ファイル出力のみ |
| **廃止** | ~~GovSystemLogDB（Azure SQL MI）~~ | ~~Serilog.Sinks.MSSqlServer~~ → **削除** |
| **廃止** | ~~Serilog.Sinks.MSSqlServer~~ | NuGet パッケージも削除 |
| Entra ID 認証ログ | Entra ID 診断設定 → Log Analytics | アプリ側実装なし |
| OS・インフラログ | AMA（Azure Monitor Agent）→ Log Analytics Workspace | DCR 適用手順は JAFFIC クラウドベンダー要確認 |
| ファイルローテーション | 1日単位・30日保持 | `logs/govlog-yyyyMMdd.txt` |

### 3.6 帳票出力（★v7詳細確定）

| 帳票種別 | ライブラリ | バージョン | 備考 |
|---------|---------|---------|------|
| PDF 帳票 | **ActiveReports v20**（MESCIUS） | v20 | 旧: ActiveReports v14.3（.NET Framework）。商用ライセンス要新規取得 |
| Excel 帳票（テンプレート） | **ExcelCreator v12**（アドバンスソフトウェア社） | v12 | 旧: ExcelCreator v10。`.xlsx` テンプレートベース生成。商用ライセンス要新規取得 |
| Excel（アドホック/リスト出力） | **ClosedXML** | 0.102.x | MIT ライセンス。テンプレート不要の一覧出力に使用 |
| CSV 出力 | **StreamWriter + StringBuilder** | — | NuGet ライブラリ不使用 |

**レポート配置**:
```
Controllers/Report/          ← 帳票 Controller
Services/Report/             ← 帳票 Service インターフェース
Infrastructure/Reports/
  ActiveReports/             ← ActiveReports v20 実装
  ExcelCreator/              ← ExcelCreator v12 実装
```

> ActiveReports は **コードベース Section Reports**（`.vb` ファイルに `InitializeComponent()` でレイアウト定義）。`.rpx` ファイルは不要。
> MESCIUS File Converter ツールで VB.NET の名前空間を自動更新しつつ移行。

### 3.7 ファイル管理・外部連携（★v7詳細確定）

| 区分 | 方式 | 備考 |
|------|------|------|
| Webアップロード | Angular → .NET API → Azure Blob Storage | Managed Identity（APIから）。ブラウザへの SAS トークン直接発行なし |
| AnyForm OCR → Blob | AzCopy（SAS トークン） | オンプレミス機は Managed Identity 不可。SAS トークンで認証 |
| 財務会計システム（新→財務） | HOSYO.csv / KAISYU.csv を Blob 経由で送信 | IF-EXT-002 |
| 財務会計システム（財務→新） | NYUKIN.csv を Blob 経由で受信 | IF-EXT-002 |
| ダウンロード（PDF/Excel/CSV） | MemoryStream → FileContentResult → フロントエンド直接 | 一時ファイル（reports/ コンテナ）は使用しない |
| SFTP（外部バッチ） | VM 上 OpenSSH Server（Port 22） | NSG で接続元 IP 限定 |

> **Azure Blob SFTP 機能**: JAFFIC 環境での利用可否は未確認（要ベンダー確認）。
> **BASE-One SFTP 対応**: 財務会計システム側の SFTP 対応可否も要確認。

### 3.8 JAFFIC クラウド基盤（Azure インフラ）

#### コンピューティング・DB・ストレージ

| サービス | 用途 | 備考 |
|---------|------|------|
| Azure VM（B2ms） | IIS + .NET 10 AP サーバー | Windows Server 2022 |
| **Azure SQL Database** | 業務DB（GovSystemDB） | ★v7変更: SQL Managed Instance → SQL Database |
| Azure Blob Storage | アプリ用ファイル管理・CSV取込・帳票一時出力 | コンテナ: `uploads/` `csv-import/` `csv-processed/` `backups/` |
| SFTP（VM 上 OpenSSH） | 外部システムとのバッチファイル交換 | NSG で接続元 IP 限定 |

#### ネットワーク・セキュリティ（JAFFIC 共通基盤）

| サービス | 機能 | 用途 |
|---------|------|------|
| Azure Virtual Network | ネットワーク | VNet + サブネット分離 |
| NSG | 通信制御 | サブネット毎の IP/ポート/プロトコル制御 |
| VPN Gateway | クラウド接続 | 基幹LAN ↔ Azure IPsec-VPN 接続 |
| Azure Firewall | ファイアウォール | 外部/内部ネットワーク間の通信監視・保護 |
| Azure DDoS Protection | DDoS 対策 | — |
| Entra ID（P1） | ユーザ管理 | JAFFIC 既存 Entra ID を継続使用。SSO・MFA |
| Azure RBAC | アクセス権管理 | 最小権限の原則 |
| Azure Key Vault | シークレット管理 | SMTP 認証情報・証明書等 |
| Defender for Server | ウイルス対策 | リアルタイム脅威検知 |
| Azure Bastion | 踏み台 | VM への安全なリモートアクセス |

#### 監視・ログ（JAFFIC 共通基盤）

| サービス | 機能 | 用途 |
|---------|------|------|
| Azure Monitor | 監視 | VM メトリクス・アラート |
| Log Analytics Workspace | ログ管理 | AMA 経由で OS/インフラログ収集。Entra ID 認証ログ |
| AMA（Azure Monitor Agent） | ログ収集 | DCR 適用手順は JAFFIC ベンダー確認中 |
| Azure Backup | バックアップ | VM スナップショット |

---

## 4. アーキテクチャ

### 4.1 確定デプロイ構成（VM + IIS）

```
Azure VM (Windows Server 2022 / B2ms)
└── IIS 10
    └── Website: https://system.gov.local:443
        ├── / (Root)        → Angular 18 静的ファイル
        │   Pool_Frontend (No Managed Code) / URL Rewrite（SPA対応）
        └── /api (Sub-App)  → .NET 10 Web API
            Pool_API (No Managed Code) / AspNetCoreModuleV2 → Kestrel
└── OpenSSH Server (SFTP — Port 22)  → 外部ファイル交換用
```

- Same-Origin のため CORS 設定不要 / `apiUrl = '/api'`
- **第2段階（PaaS）は廃止。この構成が最終形。**

### 4.2 コンポーネント図（EF Core 配置）

```
基盤共通部品
└── DBアクセス
    ├── EF Core 本体（🔵）
    ├── AppDbContext（🟢）
    └── Repository / UnitOfWork（🟡）
```

### 4.3 認証フロー（MSAL SSO）

```
① MsalGuard が未認証を検知 → Entra ID ログイン画面にリダイレクト
② ユーザー認証（MFA対応） → Access Token / ID Token 取得 → sessionStorage に保存
③ MsalInterceptor が全 HTTP リクエストに Authorization: Bearer <JWT> を自動付与
④ Microsoft.Identity.Web が JWT を検証（署名・有効期限・aud クレーム）
⑤ UserRole テーブル（業務DB）でロール確認 → RoleGuard でメニュー制御
⑥ 未認可アクセス → 403 画面表示のみ（アカウントロックなし）
```

### 4.4 通信フロー

```
Angular Component
  → MsalInterceptor（Access Token 自動付与）
  → HttpClient: POST /api/{subsystem}/{screen}/{method}
  → IIS → Kestrel → .NET 10 Middleware Pipeline
      → Controller → FluentValidation → Service（ServiceResult<T>生成）
      → AutoMapper → Repository<T> → EF Core → Azure SQL Database
  → ApiResponse<T> { result, messages{nrmList/wrnList/errList}, statusCode }
  → NotificationService: nrmList→緑トースト / wrnList→黄 / errList→赤
```

### 4.5 ネットワーク構成（JAFFIC クラウド準拠）

```
基幹LAN → VPN Gateway (IPsec) → VNet (10.0.0.0/16)
  ├── Web Subnet (10.0.1.0/24)
  │   └── NSG → VM (IIS + Angular + .NET 10 + SFTP)
  ├── DB Subnet
  │   └── NSG → Azure SQL Database
  ├── Storage Subnet
  │   └── Azure Blob Storage (VNet Service Endpoint)
  └── Management Subnet
      └── Azure Bastion（踏み台）
```

### 4.6 AnyForm OCR → CSV 取込フロー

```
紙帳票 → スキャン → AnyForm OCR（ハンモック社・オンプレミス独立稼働）
  → デザイナーで帳票テンプレート設計（初回のみ）
  → リーダーで OCR 文字認識 → ベリファイヤーで人手確認・修正
  → CSV エクスポート（指定フォルダに出力）
  → AzCopy（SAS トークン）で Blob Storage (csv-import/) にアップロード
  → .NET 10 BackgroundService が定期検知
  → CsvHelper でパース → バリデーション → EF Core → DB 一括投入
  → 処理済み CSV を csv-processed/ に移動 → Serilog ファイルログ記録
```

---

## 5. API 設計方針

### 5.1 URI 命名規則

```
/api/<subsystem>/<screen>/<method>

subsystem : サブシステム略称（小文字ローマ字）
screen    : サブシステム略称（小文字）+ 4桁連番  例: hosho1010
method    : lowerCamelCase 英語（例: jikoToroku）
            Controller ActionMethod 名と 1:1 対応

例: /api/hosho/hosho1010/search
例: /api/keiri/keiri1010/jikoToroku
例（ローカル）: proxy.conf.json で /api → localhost:5001 に転送
```

### 5.2 サブシステムとController・featureの対応

| サブシステム | URI | Controller 配置 | Angular feature | 由来 |
|------------|-----|----------------|----------------|------|
| HOSHO（保証） | `/hosho` | `Controllers/Hosho/` | `features/hosho/` | 旧: `hoshou/shinsa/` |
| SATEI（査定） | `/satei` | `Controllers/Satei/` | `features/satei/` | 旧: `kanri/` |
| SUISHIN（推進） | `/suishin` | `Controllers/Suishin/` | `features/suishin/` | 旧: 推進フォルダ |
| KEIRI（経理） | `/keiri` | `Controllers/Keiri/` | `features/keiri/` | 旧: `keiri/` |
| KYOTSU（共通） | `/kyotsu` | `Controllers/Kyotsu/` | `features/kyotsu/` | 旧: `kyoutsu/` |
| KASHI（貸付）★新規 | `/kashi` | `Controllers/Kashi/` | `features/kashi/` | 新規追加（推進タブ内） |
| KITAKU（寄託）★新規 | `/kitaku` | `Controllers/Kitaku/` | `features/kitaku/` | 新規追加（推進タブ内） |
| AUTH（認証） | `/auth` | `Controllers/Auth/` | — | 新規 |

### 5.3 API ID 命名規則

`<サブシステム大文字><4桁画面番号><メソッド名>`

### 5.4 統一レスポンス構造

`ApiResponse<T>` + `MessageContainer` (nrmList / wrnList / errList)

```json
{
  "result": {},
  "messages": {
    "nrmList": ["処理が完了しました。"],
    "wrnList": [],
    "errList": []
  },
  "statusCode": 200
}
```

### 5.5 メッセージコード体系

`I{SYS}2xxxx` / `W{SYS}3xxxx` / `E{SYS}4xxxx` / `ESYS5xxxx`

### 5.6 ServiceResult\<T\> パターン

Service 層で `ServiceResult<T>` を生成 → `BaseApiController.FromServiceResult()` で `ApiResponse<T>` に変換。

### 5.7 HTTP ステータスと Angular 処理

| コード | Angular 処理 |
|--------|------------|
| 200 / 201 / 204 | `NotificationService.handleMessages()` で nrm/wrn/err 振り分け |
| 400 | errList をトースト表示 |
| 401 | MSAL 再ログイン |
| 403 | forbidden 画面表示のみ |
| 500 | エラートースト |

### 5.8 バリデーション責任分担（★v7確定）

| 層 | 担当 |
|---|------|
| **フロントエンド（Angular）** | 入力検証: 必須 / 文字列長 / 形式（メール・数値）/ 範囲 / 相関チェック |
| **バックエンド（.NET 10）** | 業務チェック: マスタ存在確認 / 重複チェック / 業務ルール違反 / FluentValidation |

---

## 6. プロジェクト構成（フォルダ構造）

### 6.1 Angular 18 フロントエンド

```
GovSystem.Web/
├── src/
│   ├── app/
│   │   ├── core/
│   │   │   ├── interceptors/
│   │   │   │   ├── auth.interceptor.ts      # MsalInterceptor
│   │   │   │   ├── error.interceptor.ts
│   │   │   │   └── loading.interceptor.ts
│   │   │   ├── guards/
│   │   │   │   ├── auth.guard.ts            # MsalGuard
│   │   │   │   └── role.guard.ts            # UserRole テーブルベース
│   │   │   ├── services/
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── current-user.service.ts
│   │   │   │   └── notification.service.ts
│   │   │   └── models/
│   │   │       ├── api-response.model.ts
│   │   │       └── pagination.model.ts
│   │   ├── shared/
│   │   └── features/
│   │       ├── hosho/
│   │       ├── satei/
│   │       ├── suishin/
│   │       ├── kashi/
│   │       ├── kitaku/
│   │       ├── keiri/
│   │       └── kyotsu/
│   └── environments/
│       └── environment.ts    # apiUrl = '/api'（固定）
├── proxy.conf.json
└── angular.json
```

### 6.2 .NET 10 バックエンド

```
GovSystem.API/
├── src/
│   ├── GovSystem.API/
│   │   ├── Controllers/
│   │   │   ├── Hosho/
│   │   │   ├── Satei/
│   │   │   ├── Suishin/
│   │   │   ├── Kashi/
│   │   │   ├── Kitaku/
│   │   │   ├── Keiri/
│   │   │   ├── Kyotsu/
│   │   │   ├── Auth/
│   │   │   └── Report/          # 帳票 Controller
│   │   ├── BackgroundServices/
│   │   │   └── CsvFileWatcherService.cs
│   │   └── Program.cs
│   ├── GovSystem.Application/
│   │   ├── Services/
│   │   │   └── Report/          # 帳票 Service インターフェース
│   │   └── DTOs/
│   ├── GovSystem.Domain/
│   │   └── Entities/
│   └── GovSystem.Infrastructure/
│       ├── Data/
│       │   ├── AppDbContext.cs
│       │   └── Repositories/
│       ├── Reports/
│       │   ├── ActiveReports/   # PDF 帳票実装
│       │   └── ExcelCreator/    # Excel 帳票実装
│       └── External/
└── GovSystem.sln
```

---

## 7. NuGet パッケージ（.NET 10）

### 必須

| パッケージ | バージョン | 用途 |
|-----------|---------|------|
| Microsoft.EntityFrameworkCore | 10.0.x | ORM |
| Microsoft.EntityFrameworkCore.SqlServer | 10.0.x | SQL Database プロバイダー |
| Microsoft.Identity.Web | 2.x | Entra ID SSO 認証 |
| Swashbuckle.AspNetCore | 6.5.x | Swagger / OpenAPI |
| AutoMapper | 13.x | Entity ↔ DTO マッピング |
| FluentValidation.AspNetCore | 11.x | 入力バリデーション |
| Serilog.AspNetCore | 8.x | ログ |
| Serilog.Sinks.File | 5.x | ファイルログ出力 |
| ~~Serilog.Sinks.MSSqlServer~~ | ~~6.x~~ | **★v7削除（DB出力廃止）** |
| Azure.Storage.Blobs | 12.x | Blob Storage 操作 |
| Azure.Security.KeyVault.Secrets | 4.x | Key Vault |
| Azure.Identity | 1.x | Managed Identity 認証 |
| CsvHelper | 31.x | CSV パース |
| Polly | 8.x | 外部 API リトライ・CB |
| MailKit | 最新 | SMTP メール送信 |

### 推奨・商用

| パッケージ | バージョン | 用途 |
|-----------|---------|------|
| ActiveReports（MESCIUS） | **v20** | PDF 帳票出力（商用ライセンス） |
| ExcelCreator（アドバンスソフトウェア社） | **v12** | Excel テンプレート帳票（商用ライセンス） |
| ClosedXML | 0.102.x | Excel アドホック出力（MIT） |
| Dapper | 2.1.x | 複雑クエリ用 |

---

## 8. npm パッケージ（Angular 18）

| パッケージ | バージョン | 用途 |
|-----------|---------|------|
| @angular/* | 18.2.x | Angular コア一式 |
| @azure/msal-angular | 3.x | Entra ID SSO（Angular 統合） |
| @azure/msal-browser | 3.x | MSAL ブラウザコア |
| rxjs | 7.8.x | リアクティブプログラミング |
| typescript | 5.4.x | TypeScript コンパイラ |
| ngx-toastr | 19.x | トースト通知（nrm/wrn/err 色分け） |
| date-fns | 3.x | 日付操作 |
| file-saver | 2.0.x | ファイルダウンロード |

---

## 9. 開発環境

```bash
# バックエンド（Visual Studio 2022）
# GovSystem.sln → F5 → https://localhost:5001/swagger

# フロントエンド
cd D:\Projects\GovSystem.Web
ng serve --proxy-config proxy.conf.json
# → http://localhost:4200  /api → localhost:5001 に自動転送
```

```json
// proxy.conf.json
{ "/api": { "target": "https://localhost:5001", "secure": false, "changeOrigin": true } }
```

**テスト環境**: 個人 Azure テスト環境 + ローカル Windows Server（会社PC ネットワーク制限のため個人環境で実施）

---

## 10. 外部システム連携（★v7詳細確定）

| 連携先 | IF番号 | プロトコル | 方向 | 実装 | 該当サブシステム |
|--------|--------|---------|------|------|--------------|
| 格付判定システム | IF-EXT-001 | HTTPS REST（VNet-to-VNet VPN + Private Endpoint + Entra ID OAuth2.0 Client Credentials） | 双方向 | HttpClient + Polly | HOSHO（審査） |
| 財務会計システム（BASE-One） | IF-EXT-002 | Azure Blob SFTP（★要確認） | 双方向 | HOSYO.csv/KAISYU.csv 送信 / NYUKIN.csv 受信 | HOSHO・SATEI・KASHI・KITAKU・KEIRI |
| AnyForm OCR → CSV 取込 | — | AzCopy + SAS Token → Blob | 受信 | CsvHelper → EF Core | SATEI（自己査定）・全業務 |
| SSO 認証 | — | OIDC / OAuth 2.0 | 認証 | Entra ID + MSAL | 全業務 |
| メール送信 | — | SMTP Relay | 送信 | MailKit + Azure Key Vault | 全業務 |

---

## 11. 未確定事項（要顧客・ベンダー確認）

| No. | 項目 | 現在の想定 | 確認ポイント |
|-----|------|-----------|------------|
| 1 | DCR 適用手順・AMA 導入 | JAFFIC ベンダー対応 | DCR フロー・AMA インストール状況・Log Analytics Workspace 割り当て・KQL アクセス権・NW ルーティング |
| 2 | Azure Blob SFTP 機能 | JAFFIC 環境で利用可能想定 | JAFFIC 環境での利用可否確認 |
| 3 | BASE-One SFTP 対応 | 財務会計システム側対応想定 | BASE-One の SFTP 対応可否 |
| 4 | AnyForm OCR 機能詳細 | 要ハンモック社確認 | CSVファイル名タイムスタンプ対応 / Verifier 改修 / ヘッダー・明細混在CSV / 自動帳票分類 / PDF同時出力・ファイル名紐付け |
| 5 | Log Analytics KQL 監査 | KQL で十分と想定 | 専用DB監査ログ要否 |
| 6 | 格付判定システム連携仕様 | REST API（Polly リトライ） | エンドポイント・認証方式・レスポンス形式 |
| 7 | 貸付・寄託の画面詳細 | 業務概略図（C）ベースで設計 | 画面仕様・入力項目・業務ルール確定 |
| 8 | Entra ID 設定詳細 | JAFFIC 既存テナント | テナント設定・App Role 定義・ユーザー/グループ構成 |
| 9 | JAFFIC Firewall ルール | Azure Firewall 使用 | 通信許可ルールの申請・調整 |

---

## 12. 技術選定理由

| 選定 | 理由 |
|------|------|
| Angular 18（not React/Vue） | 100画面の政府系システムに統一規範が最適。VPN内部システムで脱保リスク低 |
| **.NET 10**（v6の.NET 8から移行） | 最新 LTS。長期サポート期間の確保 |
| Azure SQL Database（not MI） | JAFFIC クラウド基盤の実態に合わせて変更 |
| Entra ID SSO（MSAL） | JAFFIC 既存テナント継続。全システム SSO。MFA 対応。二要素メール認証より堅牢 |
| IIS 子アプリ（VM + IIS 固定） | JAFFIC の VM 方案に一致。VPN 内部で CDN 不要。Same-Origin で CORS 不要。PaaS 廃止 |
| Serilog ファイルのみ | GovSystemLogDB 廃止。Entra ID ログは診断設定、OS ログは AMA で Log Analytics に集約 |
| ActiveReports v20 + ExcelCreator v12 | 旧システム（v14.3 + v10）から移行。コードベース Section Reports で MESCIUS Converter を活用 |
| AzCopy + SAS Token（OCR→Blob） | オンプレミス機は Managed Identity 不可のため SAS Token で認証 |
| ClosedXML（アドホック出力） | MIT ライセンス。テンプレート不要の一覧・CSV 出力に使用 |
| MailKit（SMTP Relay） | M365 モダン認証への将来移行パスを確保しつつ現行 SMTP Relay で実装 |
| ServiceResult\<T\> パターン | Service 層で nrmList/wrnList を生成し Controller で ApiResponse に変換。業務ロジックと HTTP 応答を分離 |
| UserRole テーブル（業務DB内） | Entra ID App Registration ではなく業務DBでロール管理。柔軟な権限変更が可能 |

---

*本文書バージョン: **v7***
*更新内容: v6 → v7*
- *第2段階 PaaS（SWA + App Service）廃止確定 → VM + IIS が最終形*
- *.NET 8 → .NET 10 移行確定*
- *認証方式: 二要素メール認証 → MSAL-based SSO + UserRole テーブル（業務DB）に再設計*
- *ログ: GovSystemLogDB・Serilog.Sinks.MSSqlServer 廃止 → Serilog ファイル出力のみ*
- *帳票: ActiveReports v20（MESCIUS）+ ExcelCreator v12（アドバンスソフトウェア社）確定*
- *外部IF: IF-EXT-001（格付け）・IF-EXT-002（財務会計 SFTP）詳細追加*
- *ファイル管理: AzCopy + SAS Token（OCR機）・MemoryStream直接返却（ダウンロード）確定*
- *バリデーション責任分担確定（フロント: 入力検証 / バック: 業務チェック）*
- *プロジェクト名: 林業信用基金基幹システム（JAFFIC 林業信用基金基幹システム）に更新*
- *未確定事項: DCR/AMA・Blob SFTP・BASE-One SFTP・AnyForm OCR 詳細機能を追加*

*サブシステム: HOSHO / SATEI / SUISHIN / KEIRI / KYOTSU / KASHI★ / KITAKU★ / AUTH*
