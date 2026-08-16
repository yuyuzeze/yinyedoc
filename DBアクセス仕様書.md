# DB アクセス仕様書

| 項目 | 内容 |
|------|------|
| 対象 | Linye Backend（`src`） |
| 読者 | 業務開発者 |
| 目的 | QueryGateway と Repository の使い分け、判定、実装式样を統一する |
| 関連 | `src/Infrastructure/Queries/README.md`、`.cursor/rules/db-access.mdc`、`src/Infrastructure/DDL/README.md` |

---

## 1. 目的

本システムは **画面読取用 SQL（旧システムから移植する SELECT を含む）** と **エンティティの書込み** を分離する。

- 旧システムの SQL は Catalog に抽出し、QueryKey で管理する。
- 単純な INSERT / UPDATE / DELETE は EF の汎用 Repository に任せる。
- 「簡単／複雑」では判断しない。**読取か書込みか** で機械的に選ぶ。

---

## 2. 方式一覧

業務コードから DB へ行く正式な入口は **2 つ**。起動時シードのみ例外。

| 方式 | 入口 | 実装 | 用途 |
|------|------|------|------|
| QueryGateway | `IQueryGateway` | Dapper（`DapperQueryGateway`） | 画面読取。Catalog の SQL を実行する |
| Repository | `IRepository<T>` | EF Core（`Repository<T>`） | 書込み、および Id で Entity を取得して直後に更新する |
| （例外）DbContext 直呼び | `AppDbContext` | EF Core | 起動時 `AuthDataSeeder` のみ。業務コードでは使わない |

どちらも同一の `AppDbContext`・同一 SQL Server 接続を共有する。接続文字列は `ConnectionStrings:DefaultConnection`。

```text
Controller / Middleware / Service
        │
        ├─ IQueryGateway  ──► DapperQueryGateway ──► GetDbConnection() ──► SQL Server
        │                         ▲ SQL は Queries/{Domain}/*Queries.cs
        │
        └─ IRepository<T> ──► Repository<T> ──► DbSet / SaveChanges ──► SQL Server
```

DI 登録（`AddRepositories`）：

- `IRepository<>` → `Repository<>`
- `IQueryGateway` → `DapperQueryGateway`

---

## 3. 判定ロジック（機械的。感覚で選ばない）

上から最初に当てはまった行を採用する。

```text
① 書込み（Insert / Update / Delete / Save）
        → IRepository<T>（EF）

② Id で Entity を取得し、直後に更新または削除する
        → IRepository<T>.GetByIdAsync

③ それ以外の読取（一覧、検索、画面 DTO、Join、集計、旧 SQL）
        → IQueryGateway + Queries/{Domain}/*Queries.cs（QueryKey 必須）

④ 迷ったら
        → デフォルトで Gateway
```

補足：

- 詳細画面が **表示のみ**（Join や画面 DTO が必要）→ Gateway に詳細 SQL を置く。
- 詳細取得の目的が **その Entity をすぐ Update / Delete** → Repository の `GetByIdAsync`。
- 旧システムの SELECT が長くても Gateway。Repository には SQL を置かない。

---

## 4. 判定表

| No | やりたいこと | 使う方式 | 置き場所 / メソッド |
|----|--------------|----------|---------------------|
| 1 | 画面一覧・検索・ページング | Gateway | 担当 Catalog に `XXX_Qnnn` |
| 2 | Join（例：UserRole + Department） | Gateway | 同上 |
| 3 | 画面 DTO / Row への射影 | Gateway | `*QueryModels.cs` に Row 型 |
| 4 | 集計・帳票・旧システム SELECT の移植 | Gateway | 担当 Catalog の末尾に追加 |
| 5 | 他業務の一覧を参照する（SQL はコピーしない） | Gateway | 相手 Catalog のフィールドを `nameof` 参照 |
| 6 | 1 件 INSERT | Repository | `AddAsync` |
| 7 | 1 件 UPDATE | Repository | `GetByIdAsync` → 項目変更 → `UpdateAsync` |
| 8 | 1 件 DELETE | Repository | `GetByIdAsync` → `RemoveAsync` |
| 9 | Id で Entity を取り、すぐ更新する | Repository | `GetByIdAsync` |
| 10 | Id で **表示用** の詳細を取る（Join あり） | Gateway | 詳細用 QueryKey を新設 |
| 11 | どれか分からない | Gateway | ④ に従う |

現コードの対応例：

| 呼び出し元 | 処理 | 方式 | Key / メソッド |
|------------|------|------|----------------|
| `DemoItemService.GetAllAsync` | DemoItem 一覧 | Gateway | `KYOTSU_Q001` |
| `AuthController.Me` | OID でロール取得 | Gateway | `KYOTSU_Q003` |
| `RoleMiddleware` | 認証後の業務ロール読込 | Gateway | `KYOTSU_Q003` |
| `SateiSampleService` | 査定から保証一覧を参照 | Gateway | `HOSHO_Q001`（跨業務参照） |
| `DemoItemService.Create/Update/Delete` | 登録・更新・削除 | Repository | `AddAsync` / `UpdateAsync` / `RemoveAsync` |
| `DemoItemService.GetByIdAsync` | Id で Entity 取得 | Repository | `GetByIdAsync`（更新系 CRUD の一部） |
| `AuthDataSeeder` | 起動時の初期データ | DbContext 直 | 業務では踏襲しない |

---

## 5. QueryGateway 仕様

### 5.1 役割

画面読取 SQL の **唯一の入口**。旧システムから持ち込んだ SELECT をここで管理する。

書込みは Repository を使う。Gateway 経由の更新は業務では行わない。

### 5.2 インタフェース

`Infrastructure.DataAccess.IQueryGateway`

| メソッド | 用途 | 業務での使用 |
|----------|------|----------------|
| `QueryAsync<T>` | 複数行 | 使用する |
| `QuerySingleOrDefaultAsync<T>` | 0 または 1 行 | 詳細読取で使用してよい（未使用でも可） |
| `ExecuteAsync` | 非照会 SQL | **業務では使わない**（書込みは Repository） |

引数：

| 引数 | 内容 |
|------|------|
| `queryKey` | Catalog の **フィールド名**。`nameof(KyotsuQueries.KYOTSU_Q001)` |
| `sql` | Catalog のフィールド値。`KyotsuQueries.KYOTSU_Q001` |
| `param` | Dapper パラメータ（例：`new { EntraObjectId = oid }`） |
| `cancellationToken` | 要求の中断 |

`queryKey` はログ（`QueryKey` / `ElapsedMs` / `Rows`）に残る。`const string Key = ...` は作らない。

### 5.3 Catalog 配置

```text
src/Infrastructure/Queries/
  Kyotsu/     ← 共通担当（KyotsuQueries / KyotsuQueryModels）
  Hosho/      ← 保証担当
  Satei/      ← 査定担当
```

担当外のフォルダに SQL を足さない。新業務が必要ならフォルダと `XxxQueries` を新設する。

### 5.4 QueryKey 形式

| 規則 | 内容 |
|------|------|
| 形式 | `{ドメイン}_Q{3桁連番}`。例：`KYOTSU_Q001`、`HOSHO_Q001`、`SATEI_Q001` |
| Key | Catalog の **フィールド名そのもの** |
| SQL | フィールドの値（raw string `""" ... """`） |
| 採番 | ドメイン内で末尾に +1。欠番は再利用しない |
| 呼び出し | `nameof(HoshoQueries.HOSHO_Q001)` と `HoshoQueries.HOSHO_Q001` を対で渡す |
| 跨業務 | 相手のフィールドを参照する。SQL テキストをコピーしない |

### 5.5 Row モデル

- ファイル：同じフォルダの `*QueryModels.cs`
- クラス名は用途が分かる名前（例：`DemoItemListRow`、`ActiveUserRoleRow`）
- SQL の列名とプロパティ名を一致させる（Dapper がマッピングする）
- 画面 DTO（`Api.Models.Dtos`）とは分ける。Service で Row → DTO に変換する

### 5.6 パラメータ

旧 SQL の埋め込み値は Dapper の `@Name` に置き換える。文字列結合で SQL を組まない。

```csharp
await _queries.QueryAsync<ActiveUserRoleRow>(
    nameof(KyotsuQueries.KYOTSU_Q003),
    KyotsuQueries.KYOTSU_Q003,
    new { EntraObjectId = oid },
    cancellationToken);
```

---

## 6. Repository 仕様

### 6.1 役割

エンティティの書込みと、**直後に更新するための Id 取得** のみ。

個別の `DemoItemRepository` / `UserRoleRepository` は作らない。例外はドメイン固有の複雑な書込みだけ（その場合も一覧検索 API は付けない）。

### 6.2 ホワイトリスト

`Infrastructure.Interfaces.IRepository<T>`

| 許可 | 禁止 |
|------|------|
| `GetByIdAsync` | `GetAll` |
| `AddAsync` | `Find(predicate)` |
| `UpdateAsync` | `Query()` / `IQueryable` 公開 |
| `RemoveAsync` | `Search*` / `List*` / `FindBy*` / `Report*` |
| | 複数テーブル `Include` の一覧 |

`Add` / `Update` / `Remove` は実装内で直ちに `SaveChangesAsync` する。

対象エンティティは `Infrastructure.Entities`。テーブル定義は `src/Infrastructure/DDL`（EF Migration は使わない）。新しい書込み対象テーブルは DDL と Entity と `AppDbContext` を揃えてから `IRepository<T>` を使う。

### 6.3 更新の式样

```csharp
var item = await _repository.GetByIdAsync(id, cancellationToken);
if (item is null)
    return /* NotFound */;

item.Name = dto.Name;
item.Description = dto.Description;
await _repository.UpdateAsync(item, cancellationToken);
```

---

## 7. 実装式样（手順）

### 7.1 画面読取 SQL を追加する（Gateway）

1. `Queries/**/*Queries.cs` と `XXX_Q` を検索し、同じ結果が取れる Key があればそれを使う。
2. なければ **担当 Catalog の末尾** にフィールドを追加する（連番 +1）。
3. 同じフォルダの `*QueryModels.cs` に Row 型を追加する。
4. Service は次の形だけを書く。SQL 文字列を Service / Controller に書かない。

```csharp
var rows = await _queries.QueryAsync<DemoItemListRow>(
    nameof(KyotsuQueries.KYOTSU_Q001),
    KyotsuQueries.KYOTSU_Q001,
    cancellationToken: cancellationToken);
```

5. 必要なら Row を画面 DTO に変換して返す。

Catalog 追加の型：

```csharp
/// <summary>DemoItem 一覧</summary>
public static readonly string KYOTSU_Q001 = """
    SELECT
        Id,
        Name,
        Description,
        CreatedAt
    FROM dbo.DemoItems
    ORDER BY Id
    """;
```

### 7.2 旧システム SQL を移植する

1. 旧 SELECT の業務担当（Kyotsu / Hosho / Satei）を決める。
2. 担当 `*Queries.cs` の末尾に `XXX_Qnnn` として貼る。
3. 埋め込み条件を `@Param` に置換する。
4. SELECT 列に合わせた Row 型を作る。
5. 画面からは Gateway のみ呼ぶ。Repository に移植しない。
6. 他業務が同じ結果を使う場合は、その Key を参照する（コピー禁止）。

跨業務参照の型（SQL は Hosho 担当が管理）：

```csharp
=> _queries.QueryAsync<HoshoListRow>(
    nameof(HoshoQueries.HOSHO_Q001),
    HoshoQueries.HOSHO_Q001,
    cancellationToken: cancellationToken);
```

### 7.3 単純 CRUD を追加する（Repository）

1. 対象テーブルの Entity と DDL、`AppDbContext` の `DbSet` があることを確認する。
2. Service のコンストラクタで `IRepository<対象Entity>` を受け取る。
3. 登録 → `AddAsync`、更新・削除 → `GetByIdAsync` のあと `UpdateAsync` / `RemoveAsync`。
4. 一覧・検索は 7.1 に戻る。Repository に検索メソッドを足さない。

同一 Service で混在してよい（現行 `DemoItemService`）。**一覧は Gateway、更新は Repository**。

### 7.4 新規業務ドメインを足す

1. `Queries/{Domain}/{Domain}Queries.cs` と `{Domain}QueryModels.cs` を作る。
2. 最初の Key は `{DOMAIN}_Q001`。
3. Service は `IQueryGateway` を注入する。Gateway 実装は増やさない。

---

## 8. 同一リクエストでの混在

Gateway と Repository は同じ接続を使う。**コミット済みのデータは双方から見える。** ただし次は禁止・注意とする。

| 状況 | 結果 | 開発者の対応 |
|------|------|----------------|
| Repository が `SaveChanges` したあと Gateway で読む | DB 上の最新が取れる | この順なら問題ない |
| Entity をメモリ上で変更し、未 Save のまま Gateway で読む | Dapper は Tracker を見ないため **古い行** になる | 先に Repository の更新を完了させる |
| Gateway の `ExecuteAsync` で更新した直後に `GetByIdAsync` | `FindAsync` が Tracker の **古い Entity** を返すことがある | `ExecuteAsync` で書かない |
| 同一リクエストで Gateway と Repository を `Task.WhenAll` する | 同一接続の並列は不安定 | 逐次 `await` する |

推奨順：

1. 書く → Repository（`SaveChanges` 完了まで待つ）
2. 画面用に読む → Gateway

---

## 9. 禁止事項

| 禁止 | 理由 |
|------|------|
| Repository に一覧・検索メソッドを足す | 読取 SQL の置き場が Catalog から崩れる |
| Service / Controller で `new SqlConnection` する | 接続とログが Gateway を迂回する |
| Service / Controller に SQL 文字列を直書きする | QueryKey 管理ができなくなる |
| 他業務の SQL テキストをコピーする | 修正が二重管理になる。フィールド参照にする |
| 業務コードから `AppDbContext` を直接使う | Seeder 以外は入口が 2 つに増える |
| Gateway の `ExecuteAsync` で業務更新する | EF Tracker と不整合になる |
| 「簡単だから Gateway で UPDATE」と判断する | 判定は読取／書込み。簡単／複雑ではない |
| 欠番になった QueryKey を再利用する | ログと旧 SQL 対応が壊れる |

---

## 10. 実装前チェックリスト

- [ ] 書込みか、画面読取か、判定表の行が 1 つに決まる
- [ ] 読取なら既存 `XXX_Q` を検索した
- [ ] 新規なら担当 Catalog の末尾で連番 +1、Row 型あり
- [ ] 呼び出しが `nameof(...)` + Catalog フィールドの対になっている
- [ ] SQL が Service / Controller に無い
- [ ] 他業務 SQL をコピーしていない
- [ ] 書込みなら `IRepository<T>` の 4 メソッドだけを使っている
- [ ] 同一リクエストで Gateway と Repository を並列実行していない

---

## 11. 関連ソース

| 種別 | パス |
|------|------|
| Gateway インタフェース | `src/Infrastructure/DataAccess/IQueryGateway.cs` |
| Gateway 実装 | `src/Infrastructure/DataAccess/DapperQueryGateway.cs` |
| Repository インタフェース | `src/Infrastructure/Interfaces/IRepository.cs` |
| Repository 実装 | `src/Infrastructure/Repositories/Repository.cs` |
| DbContext | `src/Infrastructure/Data/AppDbContext.cs` |
| Catalog（共通） | `src/Infrastructure/Queries/Kyotsu/` |
| Catalog（保証） | `src/Infrastructure/Queries/Hosho/` |
| Catalog（査定） | `src/Infrastructure/Queries/Satei/` |
| DI | `src/Api/Extensions/ServiceCollectionExtensions.cs` |
| 混在サンプル | `src/Api/Services/DemoItemService.cs` |
| 跨業務参照サンプル | `src/Api/Services/Satei/SateiSampleService.cs` |
| テーブル DDL | `src/Infrastructure/DDL/` |
