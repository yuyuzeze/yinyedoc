# JAFFIC 林業信用基金基幹システム — GitHub Copilot Instructions

## プロジェクト概要

林業信用基金基幹システムの刷新プロジェクト。
旧システム（ASP.NET Web Forms + VB.NET）を Angular 20 + .NET 10 に移行する。
対象ユーザー：政府内部職員 約40名。画面数：約100画面。

- フロントエンドリポジトリ: `linyefrontend`（Angular 20 — ルート直下に `src/` 等）
- バックエンドリポジトリ: `linyebackend`（ソリューション: `Nogyosuisan.sln` — ルート直下に `src/` 等）

---

## 技術スタック

### フロントエンド（linyefrontend）
- **フレームワーク**: Angular 20（Standalone Components）
- **認証**: @azure/msal-angular 3.x + @azure/msal-browser 3.x（Entra ID SSO）
- **通知**: ngx-toastr 19.x（nrmList→緑 / wrnList→黄 / errList→赤）
- **フォーム**: Reactive Forms
- **スタイル**: SCSS
- **開発**: `ng serve`（:4200）→ `proxy.conf.json` で `/api` を `:5001` に転送

### バックエンド（linyebackend）
- **フレームワーク**: ASP.NET Core .NET 10
- **ソリューション**: `Nogyosuisan.sln`（プロジェクト: Api / Application / Domain / Infrastructure）
- **認証**: Microsoft.Identity.Web 2.x（Entra ID JWT検証）
- **ORM**: EF Core + Repository / UnitOfWork パターン
- **JSON**: Newtonsoft.Json（`AddNewtonsoftJson()`）
- **ログ**: Serilog（ファイル出力のみ — `C:\Logs\app\app-.log`）
- **開発**: `dotnet run`（:5001）

---

## フォルダ構造

### Angular 20（linyefrontend）
```
src/
├── app/
│   ├── core/
│   │   ├── auth/
│   │   │   ├── msal-config.ts            # MSALInstanceFactory / MSALGuardConfigFactory
│   │   │   ├── msal-bootstrap.ts         # msalBootstrapFactory (APP_INITIALIZER)
│   │   │   ├── api-access.service.ts     # ensureAccessToken()
│   │   │   └── auth-error.interceptor.ts # @deprecated → error.interceptor を使用
│   │   ├── guards/
│   │   │   ├── auth.guard.ts             # environment.auth.enabled ? MsalGuard : true
│   │   │   └── role.guard.ts             # UserRole テーブルベース
│   │   ├── interceptors/
│   │   │   ├── error.interceptor.ts      # 401/403/500 + errList トースト
│   │   │   └── loading.interceptor.ts    # LoadingService.show/hide
│   │   ├── logging/
│   │   │   ├── logging.interceptor.ts    # リクエストログ送信
│   │   │   ├── logging.service.ts
│   │   │   ├── global-error.handler.ts   # ErrorHandler 実装
│   │   │   └── app-message-ids.ts        # AppMessageIds 定数
│   │   ├── messages/
│   │   │   ├── message-catalog-loader.service.ts  # APP_INITIALIZER でカタログ読込
│   │   │   ├── message-format.util.ts    # resolveMessage(code, params, fallback)
│   │   │   └── msgx-message-ids.ts       # MsgxMessageIds 定数
│   │   ├── models/
│   │   │   └── api-response.model.ts     # ApiResponse<T> / MessageContainer / ApiMessageItem
│   │   ├── services/
│   │   │   ├── auth-context.service.ts   # ensureLoaded()（認証OFF時 /api/auth/me）
│   │   │   ├── notification.service.ts   # handleMessages() / showError() / showSuccess()
│   │   │   └── loading.service.ts
│   │   └── utils/
│   │       └── api-response.util.ts      # unwrapApiResponse<T>()
│   ├── shared/
│   └── features/
│       ├── hosho/
│       ├── satei/
│       ├── suishin/
│       ├── kashi/
│       ├── kitaku/
│       ├── keiri/
│       └── kyotsu/
└── environments/
    ├── environment.ts           # production (auth.enabled=true)
    └── environment.development.ts
```

### .NET 10（linyebackend）
```
src/
├── Api/
│   ├── Controllers/
│   │   ├── Hosho/
│   │   ├── Satei/
│   │   ├── Suishin/
│   │   ├── Kashi/
│   │   ├── Kitaku/
│   │   ├── Keiri/
│   │   ├── Kyotsu/        # DemoItemsController はここ
│   │   ├── Auth/
│   │   └── Report/
│   ├── Middleware/
│   │   ├── ExceptionHandlerMiddleware.cs
│   │   ├── RequestLogContextMiddleware.cs
│   │   └── RoleMiddleware.cs
│   ├── Models/
│   │   ├── ApiResponse.cs       # Result / Messages / StatusDetailMessage / StatusCode
│   │   ├── ServiceResult.cs     # IsSuccess / Data / StatusCode / Messages / StatusDetailMessage
│   │   ├── MessageContainer.cs  # NrmList / WrnList / ErrList (List<ApiMessageItem>)
│   │   └── ApiMessageItem.cs    # Code / Message / Params
│   ├── BackgroundServices/
│   │   └── CsvFileWatcherService.cs
│   └── Program.cs
├── Application/
│   ├── Services/
│   └── DTOs/
├── Domain/
│   └── Entities/
└── Infrastructure/
    ├── Data/
    │   ├── AppDbContext.cs
    │   └── Repositories/
    ├── Reports/
    │   ├── ActiveReports/     # PDF 帳票
    │   └── ExcelCreator/      # Excel 帳票
    └── External/
```

---

## Middleware Pipeline（処理順序）

```csharp
app.UseMiddleware<ExceptionHandlerMiddleware>();   // 全例外キャッチ → errList
app.UseSerilogRequestLogging(...);                 // HTTPリクエストログ
app.UseSwagger(); app.UseSwaggerUI();              // 開発時のみ
if (IsDevelopment) app.UseCors();                 // localhost:4200 のみ
app.UseRouting();
app.UseAuthentication();                           // JWT検証
app.UseMiddleware<RequestLogContextMiddleware>();   // ログコンテキスト（UPN等）付与
app.UseMiddleware<RoleMiddleware>();               // UserRole テーブルでロール確認
app.UseAuthorization();
app.MapGet("/health", ...);
app.MapControllers();
```

---

## モデル定義

### ApiMessageItem（C# / TypeScript 共通）
```csharp
// C#
public class ApiMessageItem
{
    public string Code { get; set; }     // 例: "EKYOTSU40401"
    public string Message { get; set; }  // テンプレートまたは整形済み本文
    public string[]? Params { get; set; }
}
```
```typescript
// TypeScript
export interface ApiMessageItem {
  code: string;     // 例: "MSGXE0001"
  message: string;
  params?: (string | number)[];
}
```

### ApiResponse\<T\>
```csharp
// C#
public class ApiResponse<T>
{
    public T? Result { get; set; }
    public MessageContainer Messages { get; set; } = new();
    public string? StatusDetailMessage { get; set; }
    public int StatusCode { get; set; }
}
```
```json
// JSON
{
  "result": {},
  "messages": {
    "nrmList": [{ "code": "IHOSHO20001", "message": "処理が完了しました。" }],
    "wrnList": [],
    "errList": []
  },
  "statusDetailMessage": null,
  "statusCode": 200
}
```

### ServiceResult\<T\>（Service層 → Controller層変換）
```csharp
// 正常
ServiceResult<T>.Success(data, StatusCodes.Status200OK, new ApiMessageItem("IHOSHO20001", "完了"));

// 警告付き正常
ServiceResult<T>.OkWithWarning(data, new ApiMessageItem("WHOSHO30001", "期限が近い"), nrmMsg);

// エラー
ServiceResult<T>.Failure(StatusCodes.Status400BadRequest, new ApiMessageItem("EHOSHO40001", "入力エラー"));

// 404
ServiceResult<T>.NotFound("対象データが存在しません。", "EKYOTSU40401");
```

---

## BaseApiController パターン

```csharp
[ApiController]
[Route("api/[controller]")]  // または [Route("api/hosho/hosho1010")]
public class Hosho1010Controller : BaseApiController
{
    private readonly IHosho1010Service _service;
    public Hosho1010Controller(IHosho1010Service service) => _service = service;

    [HttpGet("search")]
    public async Task<ActionResult<ApiResponse<HoshoSearchResultDto>>> Search(
        [FromQuery] HoshoSearchQuery query,
        CancellationToken cancellationToken) =>
        FromServiceResult(await _service.SearchAsync(query, cancellationToken));

    [HttpPost("jikoToroku")]
    public async Task<ActionResult<ApiResponse<HoshoDto>>> JikoToroku(
        [FromBody] HoshoCreateDto dto,
        CancellationToken cancellationToken) =>
        FromServiceResult(await _service.CreateAsync(dto, cancellationToken));
}
```

---

## URI命名規則

```
/api/{subsystem}/{screen}/{method}

例:
/api/hosho/hosho1010/search
/api/keiri/keiri1010/jikoToroku
/api/kyotsu/kyotsu0010/getCodeMaster
/api/demoitems   ← Demo用（[Route("api/[controller]")]）
```

- `subsystem`: 小文字ローマ字
- `screen`: サブシステム略称 + 4桁連番
- `method`: lowerCamelCase英語（Controller ActionMethod名と1:1対応）

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

## メッセージコード体系

```
I{SYS}2xxxx  → 正常（NrmList）   例: IHOSHO20001
W{SYS}3xxxx  → 警告（WrnList）   例: WHOSHO30001
E{SYS}4xxxx  → 業務エラー（ErrList） 例: EHOSHO40001
ESYS5xxxx    → システムエラー     例: ESYS50001
MSGX****     → フロントエンド用メッセージカタログ
APP****      → アプリケーションログ用
```

---

## 認証フロー

```
environment.auth.enabled = true の場合:
① MsalGuard → 未認証検知 → Entra ID ログイン画面
② Access Token取得 → sessionStorageに保存
③ MsalInterceptor → 全HTTPリクエストに Authorization: Bearer <JWT> 付与
④ Microsoft.Identity.Web → JWT検証
⑤ RoleMiddleware → UserRole テーブル（業務DB）でロール確認
⑥ 未認可 → 403表示のみ（アカウントロックなし）

environment.auth.enabled = false の場合（開発モック）:
① authGuard → true（スキップ）
② AuthContextService.ensureLoaded() → /api/auth/me でユーザー情報取得
```

---

## バリデーション責任分担

| 層 | 担当 |
|---|------|
| **Angular（フロントエンド）** | 必須 / 文字列長 / 形式 / 範囲 / 相関チェック |
| **.NET（バックエンド）** | マスタ存在確認 / 重複チェック / 業務ルール / FluentValidation |

---

## 命名規則

| 種別 | 規則 | 例 |
|------|------|---|
| Controller | PascalCase + `Controller` | `Hosho1010Controller` |
| Service Interface | `I` + PascalCase + `Service` | `IHosho1010Service` |
| DTO | PascalCase + `Dto` | `HoshoShinsaDto` |
| Entity | PascalCase | `HoshoShinsa` |
| DBテーブル | snake_case | `hosho_shinsa` |
| DBカラム | snake_case | `hosho_no` |
| Angular Component | kebab-case | `hosho-shinsa-list` |
| URI method | lowerCamelCase | `jikoToroku`, `search`, `update` |
| メッセージコード | `I/W/E + SYS + 5桁` | `EHOSHO40001` |

---

## 辞書ファイル一覧

Copilot Chat使用時は `#file:` で指定してください。

```
#file:docs/dictionary/01_terms.md
#file:docs/dictionary/02_old_new_mapping.md
#file:docs/dictionary/03_db/HOSHO_tables.md
#file:docs/dictionary/02_subsystems/HOSHO.md
```

---

## Copilot使用例

```
// Controller生成
#file:docs/dictionary/02_subsystems/HOSHO.md を参考に
HOSHO1010（保証審査入力）の Controller を作成してください。
- URI: /api/hosho/hosho1010/search
- BaseApiController を継承
- ServiceResult<T> → FromServiceResult() パターンを使用

// Angular Component生成
#file:docs/dictionary/01_terms.md を参考に
HoshoShinsa の一覧画面 Standalone Component を作成してください。
- ApiResponse<T> の unwrapApiResponse() を使用
- NotificationService.handleMessages() でトースト通知
- ngx-toastr / Reactive Forms を使用

// Service生成
#file:docs/dictionary/03_db/HOSHO_tables.md を参考に
IHosho1010Service インターフェースと実装クラスを作成してください。
- Repository<T> パターンを使用
- ServiceResult<T>.Success / Failure を返す
