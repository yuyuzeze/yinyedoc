# 第１０章 PDFファイル化方式

本章では、新たなシステム（以下「本システム」）でのPDFファイル化方式について整理する。
既存システムの資産（帳票テンプレートファイルおよびレイアウト設計）を最大限に活用し、開発工数を削減しつつ信頼性を確保するため、以下のライブラリを採用し最新バージョンへアップグレードする。

---

## 10.1. 採用ライブラリ一覧

- ExcelCreator 12.0（Advance Software）

選定理由：ExcelファイルをテンプレートとしてExcel / PDF両方の出力が容易であること。旧システムのExcel資産を流用して、セルへのデータ切替ロジックを最小限の修正で実装できること。

---

## 10.2. 移行方針

- ExcelCreator 12.0（Advance Software）

現行システムの旧バージョン（ExcelCreator v10）は.NET 10非対応かつ2021年6月販売終了済みのため、v12への新規ライセンス購入が必要となる。
v10.0からv12.0に移行する。ExcelテンプレートファイルはそのままV12で継承可能。セルへのデータバインドロジックは最小限の修正で流用可。

---

## 10.3. 実装方式

### 10.3.1. フロントエンド（Angular）実装方針

AngularフロントエンドはHTTP GETリクエストをresponseType: 'blob'で送信し、レスポンスをBlobオブジェクトとして受け取る。ExcelCreatorいずれの出力方式においても、フロントエンド側の実装は共通とする。ダウンロードロジックは共通ユーティリティ（FileDownloadUtil）として実装し、全帳票画面から再利用できるようにする。

### 10.3.2. バックエンド実装方針

**表10-1　出力方式の基本方針**

| 項目 | 方針 |
|---|---|
| テンプレート方式 | 書式・罫線・タイトルを設定済みの .xlsx をテンプレートとして使用する。 |
| データ流し込み | 名前定義セルに SetValue でスカラー値を設定する。明細行は範囲名を指定して SetList で一括展開する。 |
| 出力先 | HTTP レスポンスとして MemoryStream 経由でブラウザに直接ダウンロードさせる。長期保存が必要な帳票のみ Azure Blob Storage（reports/）に追加保存する。 |
| テンプレート管理 | GovSystem.Api/ExcelTemplates/ 配下にサブシステム別に格納する。 |

**表10-2　処理フロー（ExcelCreator）**

| 順番 | 処理 | 内容 |
|---|---|---|
| 1 | フロントエンド | Angularの「帳票出力」ボタン押下により、検索条件DTOをbodyに含めたPOSTリクエストを送信する。レスポンスはblob形式で受信する。 |
| 2 | データ取得 | ControllerがDTOを受け取り、Service経由でRepository → EF Core → Azure SQLからデータを取得する。 |
| 3 | テンプレート読込 | IExcelExportService.ExportAsyncにテンプレート名とfillActionを渡す。ExcelCreator v12がテンプレート.xlsxを読み込む。 |
| 4 | データ設定 | fillAction内でSetValue（固定セルへのスカラー値設定）およびSetList（明細行の範囲名一括展開）を実行する。 |
| 5 | バイト列返却 | MemoryStreamにSaveし、byte[]としてControllerに返却する。 |
| 6 | レスポンス | ControllerがFileレスポンスを返す。日本語ファイル名はUri.EscapeDataString()でエンコードする。ブラウザが.pdfとしてダウンロードする。 |

### 10.3.3. 帳票テンプレート移行方針

旧システムの帳票資産は下記の方針で新システムに移行する。

**表10-3　資産移行方針**

| 資産種別 | 旧システムの場所 | 新システムへの移行方針 |
|---|---|---|
| Excelテンプレート（.xlsx） | Report/ | Infrastructure/Reports/ExcelCreator/Templates/{Subsystem}/ 配下にそのままコピーする。v12との互換性が維持されているため、テンプレートファイル自体の修正は原則不要。 |
| 帳票生成ロジック（.aspx.vb） | Report/{Subsystem}/ ○○.aspx.vb | ExcelCreatorServiceImpl.cs に統合する。旧システムのコードパターンを参考に、サービスメソッドとして再実装する。 |

---

## 10.4. ExcelCreator ライブラリ使用上の主要設定項目

ExcelCreatorの主要なメソッドおよびプロパティを以下に示す。詳細はAdvance Software ExcelCreator公式ドキュメントを参照。

**表10-4　ExcelCreator 使用上必要な主要設定項目と代表的なオプション設定　※詳細は公式ドキュメント参照**

| 部品名（メソッド / プロパティ） | 説明 |
|---|---|
| `Creator.OpenBook(fileName, templateName)` | 既存のExcelテンプレートファイル（.xlsx）をオーバーレイオープンする。templateNameにGovSystem.Api/ExcelTemplates/{Subsystem}/配下のテンプレートパスを指定する。※必須 |
| `Creator.SetValue(cellName, value)` | 名前定義セルにスカラー値を設定する。固定セル（作成日・タイトル等）のデータ流し込みに使用する。 |
| `Creator.SetList(rangeName, dataTable)` | 範囲名を指定して明細行のデータを一括展開する。DataTableをそのまま渡すことができる。 |
| `Creator.DefaultFontName` | デフォルトフォント名を設定する。日本語帳票では「メイリオ」または「ＭＳ ゴシック」を指定する。 |
| `Creator.SaveToStream(stream)` | 生成したExcelファイルをMemoryStreamに保存する。ControllerへはMemoryStream.ToArray()でbyte[]として返却する。Excel出力時に使用する。 |
| `Creator.SavePdf(stream)` | 生成したExcelファイルをPDF形式でMemoryStreamに保存する。ExcelCreatorのPDF変換機能を使用するためサーバーへのMicrosoft Excelインストールは不要。PDF出力時に使用する。 |
| `Creator.CloseBook()` | Excelファイルを閉じて出力を確定する。OpenBook後は必ず呼び出すこと。※必須 |
| `Creator.SetOpenPassword(password)` | 作成するPDF / Excelを開くためのパスワードを設定する。 |

> ※ 必須項目以外のオプション設定は、外部設定ファイル（appsettings.json）から読み出す方式にする。
> ※ `SavePdf()` はExcelCreatorのPDF変換機能を使用するため、サーバーへのMicrosoft Excelインストールは不要。

---

## 10.5. プロジェクト構造

帳票出力機能は業務ロジック（CRUD）と責務を分離するため、各層に帳票専用のフォルダ・クラスを設ける。

```
GovSystem/
├── src/
│   ├── GovSystem.Api/
│   │   ├── Controllers/
│   │   │   ├── Hosho/              ← 業務Controller（既存。帳票は含めない）
│   │   │   └── Report/             ★ 帳票専用Controller（新設）
│   │   │       ├── Hosho/
│   │   │       │   └── HoshoReportController.cs
│   │   │       ├── Satei/
│   │   │       ├── Keiri/
│   │   │       └── Suishin/
│   │   └── ExcelTemplates/         ★ Excelテンプレート格納（新設）
│   │       ├── Hosho/
│   │       │   └── hosho1030.xlsx  ← 旧テンプレート流用
│   │       ├── Keiri/
│   │       │   └── keiri1010.xlsx  ← 旧テンプレート流用
│   │       └── Suishin/
│   │
│   ├── GovSystem.Application/
│   │   └── Services/Report/        ★ 帳票サービスIF（新設）
│   │       ├── IExcelExportService.cs
│   │       └── IReportDataService.cs
│   │
│   └── GovSystem.Infrastructure/
│       └── Reports/ExcelCreator/   ★ 帳票実装（新設）
│           ├── Templates/
│           │   ├── Hosho/
│           │   ├── Keiri/
│           │   └── Suishin/
│           └── ExcelCreatorServiceImpl.cs
```

帳票専用エンドポイントは業務APIのURIと区別するため、`/api/report/` プレフィックスを使用する。

| No | URI | 出力形式 | ライブラリ |
|---|---|---|---|
| 1 | `POST /api/report/{subsystem}/{screen}/export?format=excel` | Excel（.xlsx） | ExcelCreator |
| 2 | `POST /api/report/{subsystem}/{screen}/export?format=pdf` | PDF（Excel変換） | ExcelCreator |

> ※ フロントエンドからは検索条件DTOをbodyに含めたPOSTリクエストで送信する。レスポンスはblob形式で受信してブラウザにダウンロードさせる。
