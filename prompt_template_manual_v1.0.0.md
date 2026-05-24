# 旧システム移行 プロンプトテンプレート手册

> **用途**: ASP.NET Web Forms + VB.NET → Angular 18 + .NET 10 移行時の
> コード解析・設計書生成・コード生成に使うプロンプトの書き方テンプレート集
>
> **使い方**:
> 1. 目的のテンプレートをコピー
> 2. `{{  }}` の中を自分の情報に置き換え
> 3. Claudeに送信

---

## ■ なぜテンプレートが必要か

悪いプロンプト例:
> 「このコードをAngularに変換して」

→ Claude が何を出力すべきか判断できず、不完全な結果になる

良いプロンプトの4要素:
```
① 役割   : Claudeに「誰として振る舞うか」を伝える
② 背景   : プロジェクトの技術スタック・制約を伝える
③ 入力   : 渡すコードや情報の形式を説明する
④ 出力   : 欲しい結果の形式・項目を正確に指定する
```

---

## ═══════════════════════════════════════════
## TEMPLATE 01 ── コード解析（万能・最初に使う）
## ═══════════════════════════════════════════

```
【役割】
あなたはASP.NET Web Forms + VB.NETシステムの移行専門アーキテクトです。

【プロジェクト背景】
移行元: ASP.NET Web Forms + VB.NET
移行先: Angular 18 + .NET 10 Web API + EF Core + Azure SQL Database
画面分類: HOSHO（保証）/ SATEI（査定）/ KEIRI（経理）/ KYOTSU（共通）
APIパターン: POST /api/{subsystem}/{screenId}/{method}
レスポンス: ApiResponse<T> { result, messages{nrmList/wrnList/errList}, statusCode }

【入力】
以下は旧システムの {{ 画面名 }}（{{ ファイル名 }}.aspx / .vb）のコードです。
コードがスクリーンショットの場合、読み取れない部分は[判読不能]と記載してください。

--- コードここから ---
{{ ここにコードまたはスクリーンショットの説明を貼り付け }}
--- コードここまで ---

【出力形式】
以下の項目をすべて抽出し、Markdown表形式で出力してください。省略禁止。

## 1. 画面基本情報
| 項目 | 内容 |
|---|---|
| 画面名（日本語） | |
| ファイル名 | |
| Namespace / クラス名 | |
| 画面種別 | 一覧 / 入力 / 詳細 / 確認 のどれか |
| 画面モード | Enum等による分岐があれば記載 |
| 対応サブシステム | HOSHO / SATEI / KEIRI / KYOTSU |
| 推奨画面ID | XXXXX0000 形式 |

## 2. 表示項目（全列）
| No | 日本語ラベル | 旧フィールド名 | データ型 | 表示形式 | 備考 |
|---|---|---|---|---|---|

## 3. 入力・検索条件（全項目）
| No | 日本語ラベル | 旧フィールド名 | 型 | バリデーション | 備考 |
|---|---|---|---|---|---|

## 4. 使用テーブル
| テーブル名 | 役割 | JOIN条件 |
|---|---|---|

## 5. SQL概要
（文字列結合されたSQLを整形して1つのSQL文として出力）

## 6. 業務ロジック
（Page_Load / ボタンclick / その他イベントごとに箇条書き）

## 7. Session変数一覧
| Session名 | 用途 | 新システムでの代替手段 |
|---|---|---|

## 8. 画面遷移
| 方向 | 画面名 | 渡すパラメータ |
|---|---|---|
| 呼び出し元 | | |
| 遷移先 | | |

## 9. 特殊ロジック・注意点
（和暦変換 / フラグ変換 / 列表示切替 / 外部連携 など）
```

---

## ═══════════════════════════════════════════
## TEMPLATE 02 ── 設計書（式様書）生成
## ═══════════════════════════════════════════

```
【役割】
あなたは政府系システム移行プロジェクトの設計担当者です。

【プロジェクト背景】
{{ TEMPLATE 01 の出力結果をここに貼り付け }}

【指示】
上記の解析結果をもとに、以下の形式で画面詳細設計書を日本語Markdownで作成してください。

【出力形式・必須セクション】

# 画面詳細設計書 — {{ 画面名 }}（{{ 画面ID }}）

## 1. 基本情報
（表形式: 画面ID/画面名/サブシステム/APIパス/画面種別/旧ファイル名）

## 2. 画面レイアウト概要
（テキストAAアート: ヘッダー/検索エリア/一覧エリア/ボタンエリア/ページング）

## 3. 表示項目定義
（表: No/項目名/画面ラベル/新DTO名/型/表示形式/必須/備考）

## 4. 検索・入力条件定義
（表: No/ラベル/旧名/新名/型/必須/バリデーションルール/エラーメッセージ）

## 5. ボタン定義
（表: ボタン名/ラベル/処理内容/遷移先）

## 6. 業務処理フロー
（ロード時 / 検索時 / 選択/保存時 をステップ番号付き箇条書き）

## 7. エラー・警告メッセージ定義
（表: 条件/メッセージ内容/種別(errList/wrnList/nrmList)/メッセージコード）

## 8. 使用テーブル・SQL設計
（表: テーブル名/論理名/用途/結合条件）
（旧SQL → 新EF Core LINQの変換例）

## 9. Session変数 → 新システム対応表
（表: 旧Session名/用途/新対応方法）

## 10. 特殊ロジック対応方針
（各特殊ロジックについて旧実装と新実装を対比）

## 11. 改訂履歴
（表: バージョン/日付/担当/内容）

【制約】
- 項目の省略禁止
- 不明な項目は「要確認」と記載
- メッセージコードは EXXX40001 / WXXX30001 / IXXX20001 形式
```

---

## ═══════════════════════════════════════════
## TEMPLATE 03 ── Angular コード生成
## ═══════════════════════════════════════════

```
【役割】
あなたはAngular 18 + TypeScriptの実装担当者です。

【技術スタック（変更禁止）】
- Angular 18 Standalone Components
- Angular Material 3（mat-table / mat-paginator / mat-form-field）
- Reactive Forms（FormGroup / FormControl / Validators）
- HttpClient + MsalInterceptor（Bearerトークン自動付与）
- ngx-toastr（nrmList→success / wrnList→warning / errList→error）
- ApiResponse<T> = { result: T, messages: {nrmList, wrnList, errList}, statusCode }
- PagedResult<T> = { items: T[], totalCount, pageNumber, pageSize, totalPages }
- apiUrl = '/api'（固定、環境変数不要）

【設計情報】
{{ TEMPLATE 02 の設計書をここに貼り付け、または要点を箇条書き }}

【指示】
以下のファイルをすべて生成してください。各ファイルの先頭にファイルパスをコメントで記載。

【生成ファイル①】{{ screenId }}.model.ts
- 表示項目に対応するListItemインターフェース
- 検索条件に対応するSearchRequestインターフェース
- 選択時ペイロードインターフェース
- 画面モードのユニオン型

【生成ファイル②】{{ screenId }}.service.ts
- @Injectable({ providedIn: 'root' })
- search()メソッド: POST /api/{subsystem}/{screenId}/search
- その他必要なAPIメソッド

【生成ファイル③】{{ screenId }}.component.ts
- Standalone Component
- FormGroup定義（全検索条件）
- MatTableDataSource + displayedColumns（モード別）
- MatPaginator連携
- onSearch() / onSelect() / onPageChange()
- NotificationService.handleMessages()でトースト処理

【生成ファイル④】{{ screenId }}.component.html
- mat-card > 検索フォーム > mat-table > mat-paginator 構成
- 全表示カラムをmat-column-defで定義
- 検索条件を全てmat-form-fieldで定義
- ページングをmat-paginatorで実装

【生成ファイル⑤】{{ screenId }}.component.scss
- 最小限のスタイル（テーブル縞模様 / ボタン配置）

【制約】
- any型使用禁止
- console.log使用禁止
- コメントは日本語で記載
- エラーハンドリングはerror.interceptor.tsに委譲（コンポーネント内でcatchError不要）
```

---

## ═══════════════════════════════════════════
## TEMPLATE 04 ── .NET 10 バックエンドコード生成
## ═══════════════════════════════════════════

```
【役割】
あなたは.NET 10 + Clean Architectureの実装担当者です。

【技術スタック（変更禁止）】
- ASP.NET Core 10 Web API
- Clean Architecture（API / Application / Domain / Infrastructure）
- EF Core 10（Code First, Managed Identity接続）
- Repository<T> + UnitOfWork パターン
- FluentValidation 11.x（業務チェック専用）
- AutoMapper 13.x（Entity ↔ DTO）
- ServiceResult<T> → BaseApiController.FromServiceResult() → ApiResponse<T>
- Serilog（ファイル出力のみ、DBログなし）
- [Authorize] 必須（Microsoft.Identity.Web / Entra ID JWT）

【ServiceResult パターン】
ServiceResult<T> { Data, NrmList, WrnList, ErrList }
BaseApiController.FromServiceResult() で ApiResponse<T> に自動変換

【設計情報】
{{ TEMPLATE 02 の設計書をここに貼り付け、または要点を箇条書き }}

【指示】
以下のファイルをすべて生成してください。各ファイルの先頭にファイルパスをコメントで記載。

【生成ファイル①】Domain/Entities/{{ EntityName }}.cs
- テーブル定義に対応するエンティティクラス
- データアノテーション不使用（Fluent APIで設定）

【生成ファイル②】Application/DTOs/{{ ScreenName }}Dto.cs
- {{ ScreenName }}ListItemDto（表示項目全列）
- {{ ScreenName }}SearchRequestDto : PagedRequestDto（検索条件全項目）
- 必要に応じてその他DTO

【生成ファイル③】Application/Mappings/{{ ScreenName }}Profile.cs
- AutoMapper Profile
- Entity → ListItemDto のマッピング定義
- 型変換・計算フィールドのマッピングも含む

【生成ファイル④】Application/Validators/{{ ScreenName }}SearchValidator.cs
- FluentValidation AbstractValidator<{{ ScreenName }}SearchRequestDto>
- 設計書7のエラー条件をすべて実装
- メッセージコードを含むエラーメッセージ

【生成ファイル⑤】Application/Services/I{{ ScreenName }}Service.cs + {{ ScreenName }}Service.cs
- インターフェース定義
- SearchAsync()実装（EF Core LINQ + Skip/Take ページング）
- 旧SQLの全WHERE条件をLINQで再現
- ServiceResult<PagedResult<{{ ScreenName }}ListItemDto>> を返す

【生成ファイル⑥】API/Controllers/{{ Subsystem }}/{{ ScreenName }}Controller.cs
- [Authorize] 付与
- [ApiController] / [Route("api/{subsystem}/{screenId}")]
- [HttpPost("search")] SearchAsync
- BaseApiController.FromServiceResult() で返却
- ILogger<T> によるSerilogログ出力

【制約】
- 文字列SQL直書き禁止（EF Core LINQを使用）
- Session不使用（JWT Claimsを使用）
- ServiceResult以外のレスポンス形式禁止
- メソッドに日本語コメント必須
```

---

## ═══════════════════════════════════════════
## TEMPLATE 05 ── DB設計書生成
## ═══════════════════════════════════════════

```
【役割】
あなたはデータベース設計担当者です。

【前提】
- Azure SQL Database
- EF Core 10 Code First（Fluent API）
- 旧システム: SQL Server + ADO.NET SqlHelper

【解析情報】
{{ TEMPLATE 01 の出力結果をここに貼り付け }}

【指示】
以下の形式でDB設計書を生成してください。

## 1. 使用テーブル一覧
（表: テーブル物理名 / 論理名 / 種別(マスタ/トランザクション) / 説明）

## 2. テーブル定義（メインテーブルのみ詳細化）

### {{ テーブル名 }}
（表: No / カラム物理名 / カラム論理名 / 型 / サイズ / NOT NULL / PK / FK / 説明）

## 3. インデックス設計（推奨）
（表: テーブル / カラム / 種別 / 理由）

## 4. EF Core Fluent API 設定例
（OnModelCreating でのテーブル設定コード）

## 5. 旧SQL → EF Core LINQ 変換

### 旧SQL（整形版）
```sql
（旧システムのSQL文字列結合を整形したSQL）
```

### 新LINQ（EF Core 10）
```csharp
（LINQクエリ。Skip/Takeによるページング含む）
```

## 6. Migration コマンド
```bash
dotnet ef migrations add {{ MigrationName }} \
  --project GovSystem.Infrastructure \
  --startup-project GovSystem.API
dotnet ef database update \
  --project GovSystem.Infrastructure \
  --startup-project GovSystem.API
```
```

---

## ═══════════════════════════════════════════
## TEMPLATE 06 ── 移行チェックリスト生成
## ═══════════════════════════════════════════

```
【役割】
あなたは移行プロジェクトのQA担当者です。

【対象画面】
{{ 画面名 }}（{{ 画面ID }}）

【設計・実装情報】
{{ TEMPLATE 02〜05 の出力を簡潔にまとめて貼り付け }}

【指示】
この画面の移行チェックリストを以下のカテゴリで生成してください。
各項目は [ ] チェックボックス形式で、具体的な確認内容を記載してください。

## 設計完了チェック
## DB・SQL変換チェック
## Angular実装チェック
## .NET実装チェック
## 特殊ロジック対応チェック
## テスト確認チェック
```

---

## ═══════════════════════════════════════════
## TEMPLATE 07 ── スクリーンショット解析専用
## ═══════════════════════════════════════════

> コードファイルがなく、画像だけある場合に使用

```
【役割】
あなたはASP.NET Web Forms + VB.NET コードの解析専門家です。

【入力】
添付画像はVisual Studioで開いた旧システムのコードスクリーンショットです。
複数枚ある場合は画像1〜Nの順に連続したコードです。

【読み取りルール】
- 読み取れない文字: [判読不能] と記載
- 画像の切れ目: [続き] と記載
- コメント行（' で始まる行）: 業務の意図として活用する
- SQL文字列結合（& vbCrLf で連結）: 最終的な1つのSQL文として整形して出力
- Session("xxx"): Session変数として必ずリストアップ

【出力】
TEMPLATE 01 と同じ出力形式（8セクション）で出力してください。
```

---

## ═══════════════════════════════════════════
## ■ 効果的なプロンプトのコツ（重要）
## ═══════════════════════════════════════════

### コツ1: 出力形式を「表」で指定する
```
❌ 悪い例: 「テーブル情報を教えて」
✅ 良い例: 「以下の表形式で出力: | テーブル名 | 役割 | JOIN条件 |」
```
→ 表形式にすることで出力が安定し、次のプロンプトで使い回しやすくなる

### コツ2: 「省略禁止」を明示する
```
「全項目を出力してください。省略禁止。不明な場合は"要確認"と記載。」
```
→ Claudeは長い出力を「...など」と省略しがちなので、明示が必要

### コツ3: 前のステップの出力を次のプロンプトに貼り付ける
```
TEMPLATE 01（解析） → 出力をコピー
→ TEMPLATE 02（設計書）の「解析情報」に貼り付けて送信
→ 出力をコピー
→ TEMPLATE 03（Angularコード）に貼り付けて送信
```
→ 各ステップが前のステップの情報を引き継ぐため精度が上がる

### コツ4: 技術スタックを毎回明記する
```
「Angular 18 + .NET 10 + EF Core + Azure SQL Database。
 Session不使用（JWT）。文字列SQL直書き禁止（LINQ使用）。」
```
→ これがないとClaudeが古いパターン（Session / ADO.NET）で生成してしまう

### コツ5: 複数ファイルは「ファイル①②③」と番号で指定する
```
「以下のファイルをすべて生成: ①model.ts ②service.ts ③component.ts ...」
```
→ 番号を振ることで抜け漏れなく全ファイルが生成される

### コツ6: 不確かな部分は「要確認」タグで管理する
```
「仕様が不明な箇所には【要確認: 理由】とコメントを入れてください」
```
→ 後から顧客に確認すべき点が自動的にマークされる

---

## ■ よくある失敗パターン

| 失敗 | 原因 | 対処法 |
|---|---|---|
| Claudeが旧いパターンで生成する | 技術スタックを指定していない | 毎回「.NET 10 / Angular 18」と明記 |
| 項目が省略される | 「全部出して」と言っていない | 「省略禁止」と明記 |
| 次のステップで情報が変わる | 前の出力を引き継いでいない | 前の出力を貼り付けて渡す |
| SQLが不完全 | 複数画像の情報が分断される | 「画像1〜N連続」と明記 |
| エラーハンドリングが違う | APIレスポンス形式を指定していない | ApiResponse<T>の構造を毎回明記 |

---

## ■ テンプレート使用フロー（まとめ）

```
STEP1: 旧コードを入手
  └── ファイルあり → そのままTEMPLATE 01に貼り付け
  └── 画像のみ   → TEMPLATE 07 → 出力をTEMPLATE 01に渡す
        ↓
STEP2: TEMPLATE 01 → 解析結果を保存
        ↓
STEP3: TEMPLATE 02 → 設計書（式様書）生成
        ↓（並行して実施可能）
STEP4a: TEMPLATE 03 → Angular コード生成
STEP4b: TEMPLATE 04 → .NET 10 コード生成
STEP4c: TEMPLATE 05 → DB設計書生成
        ↓
STEP5: TEMPLATE 06 → チェックリスト生成
        ↓
STEP6: レビュー → 差分修正 → 次の画面へ
```

---

*バージョン: 1.0.0*
*対象移行: ASP.NET Web Forms + VB.NET → Angular 18 + .NET 10*
*更新日: 2026年3月*
