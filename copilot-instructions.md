# JAFFIC 林業信用基金基幹システム — GitHub Copilot Instructions

## プロジェクト概要

林業信用基金基幹システムの刷新プロジェクト。
旧システム（ASP.NET Web Forms + VB.NET）を Angular 18 + .NET 10 に移行する。
対象ユーザー：政府内部職員 約40名。画面数：約100画面。

---

## 技術スタック

### フロントエンド
- **フレームワーク**: Angular 18（Standalone Components）
- **認証**: @azure/msal-angular 3.x（Entra ID SSO）
- **通知**: ngx-toastr（nrm→緑 / wrn→黄 / err→赤）
- **フォーム**: Reactive Forms
- **配置**: IIS Root `/`（Same-Origin、CORS不要）

### バックエンド
- **フレームワーク**: ASP.NET Core .NET 10
- **認証**: Microsoft.Identity.Web 2.x（Entra ID JWT検証）
- **ORM**: Entity Framework Core 10 + Repository/UnitOfWork パターン
- **バリデーション**: FluentValidation 11.x
- **ログ**: Serilog（ファイル出力のみ）
- **配置**: IIS Sub-App `/api`（Kestrel in-process）

### インフラ
- Azure VM（Windows Server 2022）
- Azure SQL Database
- Azure Blob Storage（Managed Identity）
- Azure Key Vault
- Microsoft Entra ID

---

## URI命名規則

```
/api/<subsystem>/<screen>/<method>

例:
/api/hosho/hosho1010/search
/api/keiri/keiri1010/jikoToroku
/api/kyotsu/kyotsu0010/getCodeMaster
```

- `subsystem`: 小文字ローマ字（hosho / satei / suishin / kashi / kitaku / keiri / kyotsu）
- `screen`: サブシステム略称 + 4桁連番（例: hosho1010）
- `method`: lowerCamelCase英語（Controller ActionMethod名と1:1対応）

---

## APIレスポンス構造

すべてのAPIは `ApiResponse<T>` を返す。

```csharp
public class ApiResponse<T>
{
    public T? Result { get; set; }
    public MessageContainer Messages { get; set; } = new();
    public int StatusCode { get; set; }
}

public class MessageContainer
{
    public List<string> NrmList { get; set; } = new(); // 正常メッセージ（緑トースト）
    public List<string> WrnList { get; set; } = new(); // 警告メッセージ（黄トースト）
    public List<string> ErrList { get; set; } = new(); // エラーメッセージ（赤トースト）
}
```

JSON例:
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

---

## ServiceResult\<T\> パターン

Service層では `ServiceResult<T>` を生成し、Controller で `ApiResponse<T>` に変換する。

```csharp
// Service層
public ServiceResult<HoshoDto> GetHosho(string hoshoNo)
{
    // 正常
    return ServiceResult<HoshoDto>.Success(dto, "処理が完了しました。");
    // 警告
    return ServiceResult<HoshoDto>.Warning(dto, "期限が近づいています。");
    // エラー
    return ServiceResult<HoshoDto>.Error("対象データが存在しません。");
}

// Controller層
[HttpGet("search")]
public IActionResult Search([FromQuery] string hoshoNo)
{
    var result = _hoshoService.GetHosho(hoshoNo);
    return FromServiceResult(result);
}
```

---

## Middleware Pipeline（処理順序）

```
❶ ExceptionHandlerMiddleware  → 全例外キャッチ → errList[ESYS50001]
❷ Serilog RequestLogging      → ファイルログ記録
❸ Authentication              → Entra ID JWT検証
❹ Authorization               → ロール・ポリシー確認
❺ MapControllers              → Controllerへルーティング
```

---

## 認証フロー

```
① MsalGuard → 未認証検知 → Entra ID ログイン画面
② Access Token取得 → sessionStorageに保存
③ MsalInterceptor → 全HTTPリクエストに Authorization: Bearer <JWT> 付与
④ Microsoft.Identity.Web → JWT検証
⑤ UserRole テーブル（業務DB）でロール確認
⑥ RoleGuard → メニューレベルアクセス制御
⑦ 未認可 → 403表示のみ（アカウントロックなし）
```

---

## バリデーション責任分担

| 層 | 担当 |
|---|------|
| **Angular（フロントエンド）** | 必須 / 文字列長 / 形式（メール・数値・日付）/ 範囲 / 相関チェック |
| **.NET 10（バックエンド）** | マスタ存在確認 / 重複チェック / 業務ルール違反 / FluentValidation |

---

## サブシステム構成

| サブシステム | URI | Controller | Angular feature | 旧フォルダ |
|------------|-----|-----------|----------------|----------|
| HOSHO（保証） | `/hosho` | `Controllers/Hosho/` | `features/hosho/` | `hoshou/shinsa/` |
| SATEI（査定） | `/satei` | `Controllers/Satei/` | `features/satei/` | `hoshou/kanri/` |
| SUISHIN（推進） | `/suishin` | `Controllers/Suishin/` | `features/suishin/` | 推進フォルダ |
| KEIRI（経理） | `/keiri` | `Controllers/Keiri/` | `features/keiri/` | `keiri/` |
| KYOTSU（共通） | `/kyotsu` | `Controllers/Kyotsu/` | `features/kyotsu/` | `kyoutsu/` |
| KASHI（貸付）★新規 | `/kashi` | `Controllers/Kashi/` | `features/kashi/` | — |
| KITAKU（寄託）★新規 | `/kitaku` | `Controllers/Kitaku/` | `features/kitaku/` | — |
| AUTH（認証） | `/auth` | `Controllers/Auth/` | — | — |

---

## フォルダ構造

### Angular
```
src/app/
├── core/
│   ├── interceptors/        # auth / error / loading
│   ├── guards/              # auth.guard / role.guard
│   ├── services/            # auth / current-user / notification
│   └── models/              # api-response / pagination
├── shared/
└── features/
    ├── hosho/
    ├── satei/
    ├── suishin/
    ├── kashi/
    ├── kitaku/
    ├── keiri/
    └── kyotsu/
```

### .NET 10
```
src/
├── GovSystem.API/
│   ├── Controllers/
│   │   ├── Hosho/
│   │   ├── Satei/
│   │   ├── Suishin/
│   │   ├── Kashi/
│   │   ├── Kitaku/
│   │   ├── Keiri/
│   │   ├── Kyotsu/
│   │   └── Auth/
│   └── BackgroundServices/
├── GovSystem.Application/
│   ├── Services/
│   └── DTOs/
├── GovSystem.Domain/
│   └── Entities/
└── GovSystem.Infrastructure/
    ├── Data/
    │   ├── AppDbContext.cs
    │   └── Repositories/
    └── External/
```

---

## 命名規則

| 種別 | 規則 | 例 |
|------|------|---|
| Controller | PascalCase + `Controller` | `Hosho1010Controller` |
| Service | PascalCase + `Service` | `HoshoShinsakuService` |
| DTO | PascalCase + `Dto` | `HoshoShinsaDto` |
| Entity | PascalCase | `HoshoShinsa` |
| DBテーブル | snake_case | `hosho_shinsa` |
| DBカラム | snake_case | `hosho_no` |
| Angular Component | kebab-case | `hosho-shinsa-list` |
| Angular Service | camelCase + `Service` | `hoshoShinsaService` |
| URI method | lowerCamelCase | `jikoToroku` |

---

## 辞書ファイル一覧

詳細仕様は以下を参照してください。Copilot Chatで使用する際は `#file:` で指定してください。

```
#file:docs/dictionary/01_terms.md          # 業務用語集（日本語↔英語↔DB物理名）
#file:docs/dictionary/02_old_new_mapping.md # 旧システム↔新システム対照表
#file:docs/dictionary/03_db/HOSHO_tables.md # HOSHOテーブル定義
#file:docs/dictionary/03_db/SATEI_tables.md # SATEIテーブル定義
#file:docs/dictionary/02_subsystems/HOSHO.md
#file:docs/dictionary/02_subsystems/SATEI.md
#file:docs/dictionary/02_subsystems/KEIRI.md
#file:docs/dictionary/02_subsystems/KYOTSU.md
```

---

## Copilot使用例

```
// Controller生成
#file:docs/dictionary/02_subsystems/HOSHO.md を参考に
HOSHO1010（保証審査入力）のControllerを作成してください。
URI: /api/hosho/hosho1010/search

// Angular Component生成
#file:docs/dictionary/01_terms.md を参考に
HoshoShinsa の一覧画面コンポーネントを作成してください。
ApiResponse<T>を使用し、ngx-toastrで通知してください。

// DB Entityの生成
#file:docs/dictionary/03_db/HOSHO_tables.md を参考に
hosho_shinsaテーブルのEntityクラスを生成してください。
```
