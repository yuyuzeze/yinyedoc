# 本番環境デプロイ手順（Azure VM + IIS 方式）

本システムは Angular 20（フロントエンド）と .NET 10 Web API（バックエンド）を同一の Azure VM 上の IIS に配置する構成を採用する。  
Azure Static Web App は使用しない。

---

## デプロイ構成

```
Azure VM（Windows Server 2022）
└── IIS 10 — Website: system.gov.local
    ├── / (Root)     → Angular 20 SPA（静的ファイル）   Pool_Frontend
    └── /api         → .NET 10 Web API（サブアプリ）    Pool_API（Kestrel in-process）
```

---

## 1. 事前準備（VM サーバー側）

1. **ASP.NET Core Hosting Bundle（.NET 10）のインストール**
   - 下載：https://dotnet.microsoft.com/download/dotnet → バージョン 10 → Hosting Bundle
   - インストール後、IIS を再起動する：
     ```powershell
     iisreset
     ```

2. **IIS URL Rewrite モジュール（バージョン 2.1）のインストール**
   - Angular SPA のクライアントサイドルーティングに必要
   - 下載：https://www.iis.net/downloads/microsoft/url-rewrite

3. **IIS で AspNetCoreModuleV2 が有効になっているか確認する**

---

## 2. ビルド & 発布

### 2.1 手動手順

```powershell
# ステップ1：Angular フロントエンドをビルド
cd frontend
npm install
ng build --configuration production

# ステップ2：.NET 10 Web API を発布
cd ..\backend\src\Api
dotnet publish -c Release -o ..\..\..\publish

# ステップ3：Angular ビルド産物を publish/wwwroot にコピー
#   Angular 20 のビルド産物は frontend/dist/frontend/browser/ 配下
New-Item -ItemType Directory -Force ..\..\..\publish\wwwroot
Copy-Item ..\..\..\..\frontend\dist\frontend\browser\* ..\..\..\publish\wwwroot\ -Recurse -Force
```

### 2.2 一键スクリプト

プロジェクトルートに `publish.ps1` が用意されている：

```powershell
cd D:\Code\Backend\C#\农林水产省
.\publish.ps1
```

実行後、`publish/` ディレクトリが完全なデプロイパッケージとなる。

---

## 3. IIS へのデプロイ

1. `publish/` ディレクトリ全体をサーバーにコピーする（例：`C:\inetpub\govsystem\`）
2. IIS マネージャーでサイトを追加する：
   - サイト名：`GovSystem`
   - 物理パス：`C:\inetpub\govsystem\`
   - ポート：`443`（HTTPS）
   - アプリケーションプール：**マネージドコードなし**（ASP.NET Core は自身のランタイムを管理）
3. サブアプリケーション `/api` を追加する：
   - 物理パス：`C:\inetpub\govsystem\`（同ディレクトリ）
   - アプリケーションプール：`Pool_API`（マネージドコードなし）
4. アプリケーションプールの ID にデプロイディレクトリへの**読み書き権限**を付与する（Azure SQL Managed Identity 接続のため）

---

## 4. IIS web.config の確認

`dotnet publish` 実行時に `web.config` が自動生成される。内容は以下のとおり：

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*"
             modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet" arguments=".\Api.dll"
                  stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout"
                  hostingModel="inprocess" />
    </system.webServer>
  </location>
</configuration>
```

問題発生時はログ確認のため `stdoutLogEnabled` を `true` に変更し、`logs` フォルダを作成する。

---

## 5. Angular SPA ルーティング対応（URL Rewrite）

Angular のクライアントサイドルーティングに対応するため、`wwwroot/` に `web.config` を配置する：

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="Angular Routes" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
            <add input="{REQUEST_URI}" pattern="^/api" negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

---

## 6. データベース設定（Azure SQL Database）

本番環境は Azure SQL Database に Managed Identity で接続する。`appsettings.prod.json` を確認する：

```json
{
  "ConnectionStrings": {
    "AppDB": "Server=tcp:appdb.database.windows.net,1433;Initial Catalog=AppDB;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;Authentication=Active Directory Managed Identity;"
  }
}
```

パスワードは不要。Azure VM の Managed Identity に Azure SQL への権限を付与すること。

---

## 7. デプロイ後の検証

```powershell
# 発布パッケージを単体で起動して確認する
cd publish
dotnet Api.dll
```

- `https://system.gov.local` → Angular 20 SPA のフロントエンド画面
- `https://system.gov.local/api/swagger` → Swagger UI（API テスト）

---

## 8. ローカル開発への影響

**影響なし。**

- ローカル開発時は `frontend/` で `ng serve`（Port 4200）、`backend/src/Api` で `dotnet run`（Port 5001）をそれぞれ起動する。
- `proxy.conf.json` により `/api` → `http://localhost:5001` に転送されるため、CORS 設定不要。
- 本デプロイ手順は Azure VM 上の IIS へのデプロイにのみ適用される。
