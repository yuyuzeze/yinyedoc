# 农林水产省 Demo — 环境搭建与操作手册

---

## 一、前提条件（安装软件）

### 1.1 安装 Node.js

- 下载地址：https://nodejs.org/
- 推荐版本：**Node.js 20 LTS**（或 18 LTS 也可以）
- 安装后验证：

```powershell
node --version    # 应输出 v20.x.x 或 v18.x.x
npm --version     # 应输出 10.x.x 或 9.x.x
```

### 1.2 安装 Angular CLI（全局）

Node.js 安装完后，打开命令行执行：

```powershell
npm install -g @angular/cli@18
```

安装后验证：

```powershell
ng version
```

应显示 Angular CLI 18.x.x。

> **注意**：如果公司网络有代理，可能需要配置 npm 代理：
> ```powershell
> npm config set proxy http://proxy.example.com:8080
> npm config set https-proxy http://proxy.example.com:8080
> ```

### 1.3 安装 .NET SDK

- 下载地址：https://dotnet.microsoft.com/download
- 推荐版本：**.NET 8.0 SDK**（LTS 长期支持版）
- 安装后验证：

```powershell
dotnet --version   # 应输出 8.0.x
```

本 Demo 使用 **.NET 8.0** 与 **Angular 18** 构建。

### 1.4 安装 Visual Studio 2022（后端开发用）

- 下载地址：https://visualstudio.microsoft.com/
- 选 **Community**（免费）或 Professional / Enterprise
- 安装时勾选工作负载：**ASP.NET 和 Web 开发**
- 安装完后也可以用来打开 `backend/Nogyosuisan.sln`

### 1.5 安装 VS Code（前端开发用，可选）

- 下载地址：https://code.visualstudio.com/
- 推荐扩展：
  - Angular Language Service
  - Prettier
  - ESLint
  - C# Dev Kit（如果也想在 VS Code 里写后端）

---

## 二、用 Visual Studio 从零创建 .NET 8 Web API 项目（参考）

> 本 Demo 已包含完整后端代码，以下仅作**参考**说明如何用 Visual Studio 创建。

### 步骤

1. 打开 Visual Studio 2022 → 点击 **创建新项目**
2. 搜索模板 **ASP.NET Core Web API** → 选中 → 下一步
3. 填写：
   - 项目名称：`Api`
   - 位置：`D:\Code\Backend\C#\农林水产省\backend\src\`
   - 解决方案名称：`Nogyosuisan`
4. 框架选 **.NET 8.0 (Long Term Support)**
5. 勾选：
   - ✅ Configure for HTTPS（可选，开发阶段可以不勾）
   - ✅ Use controllers
   - ✅ Enable OpenAPI support（即 Swagger）
   - ❌ Do not use top-level statements（保持取消，用顶级语句更简洁）
6. 点击 **创建**

创建后：
- 右键解决方案 → 添加 → 新建项目 → **类库 (Class Library)** → 分别创建 `Domain`、`Application`、`Infrastructure`
- 在各项目上右键 → 添加 → 项目引用 → 按以下关系勾选：
  - `Application` 引用 `Domain`
  - `Infrastructure` 引用 `Domain` + `Application`
  - `Api` 引用 `Application` + `Infrastructure`

---

## 三、当前 Demo 项目结构

```
农林水产省/
├── backend/                        # .NET 后端
│   ├── Nogyosuisan.sln             # 解决方案文件
│   └── src/
│       ├── Api/                    # Web API 入口
│       │   ├── Controllers/        #   DemoItemsController.cs
│       │   ├── Properties/         #   launchSettings.json
│       │   ├── wwwroot/            #   前端构建产物放这里（部署时）
│       │   ├── Program.cs          #   启动配置
│       │   ├── appsettings.json
│       │   └── Api.csproj
│       ├── Application/            # 应用层（DTO、服务接口/实现）
│       ├── Domain/                 # 领域层（实体）
│       └── Infrastructure/         # 基础设施（EF Core、Repository）
├── frontend/                       # Angular 18 前端
│   ├── src/
│   │   ├── app/
│   │   │   ├── core/               # models、services
│   │   │   └── features/           # 页面组件
│   │   └── environments/
│   ├── proxy.conf.json             # 开发代理配置
│   ├── angular.json
│   └── package.json
├── publish.ps1                     # 一键发布脚本
└── init.md                         # 本文件
```

---

## 四、本地开发（前后端分离模式）

开发时前后端**各跑一个端口**，互不影响：

| 服务 | 端口 | 说明 |
|------|------|------|
| 前端 Angular | http://localhost:4200 | `ng serve` 启动 |
| 后端 .NET API | http://localhost:5001 | `dotnet run` 启动 |

前端通过 `proxy.conf.json` 将 `/api` 请求自动转发到后端 5001，无需配置 CORS。

### 4.1 启动后端

```powershell
cd backend\src\Api
dotnet run
```

- 启动后浏览器打开：http://localhost:5001/swagger
- 可以用 Swagger 页面直接测试 API（Try it out）
- 首次运行会自动创建 SQLite 数据库文件 `nogyosuisan.db`

**用 Visual Studio 启动**：
- 打开 `backend\Nogyosuisan.sln`
- 将 `Api` 设为启动项目
- 按 F5（调试）或 Ctrl+F5（不调试运行）

### 4.2 启动前端

```powershell
cd frontend
npm install        # 首次需要，安装依赖
ng serve           # 或 npm run start
```

- 启动后浏览器打开：http://localhost:4200
- 修改代码会自动热更新（HMR），不需要手动刷新

### 4.3 联调验证

1. 确保后端在 5001 运行
2. 前端在 4200 运行
3. 打开 http://localhost:4200 → 看到"Demo 项目列表"
4. 点"新增"→ 填写名称和描述 → 提交
5. 列表中应出现刚才创建的数据
6. 可以编辑、删除

---

## 五、发布（单站点模式，用于部署到 IIS）

发布后前端和后端合并到一个目录中，由 .NET 统一托管。

### 5.1 手动步骤

```powershell
# 第一步：构建前端
cd frontend
ng build

# 第二步：发布后端
cd ..\backend\src\Api
dotnet publish -c Release -o ..\..\..\publish

# 第三步：复制前端产物到 publish/wwwroot
#   Angular 18 构建产物在 frontend/dist/frontend/browser/ 下
mkdir ..\..\..\publish\wwwroot -Force
Copy-Item ..\..\..\..\frontend\dist\frontend\browser\* ..\..\..\publish\wwwroot\ -Recurse -Force
```

### 5.2 使用一键脚本

项目根目录已有 `publish.ps1`：

```powershell
cd D:\Code\Backend\C#\农林水产省
.\publish.ps1
```

执行完后 `publish/` 目录即为完整的可部署包。

### 5.3 验证发布包

```powershell
cd publish
dotnet Api.dll
```

打开 http://localhost:5001 → 应看到前端页面，http://localhost:5001/api/demoitems → 返回 API 数据。

---

## 六、部署到 IIS

### 6.1 准备服务器

1. 安装 **ASP.NET Core Hosting Bundle**（对应 .NET 8 版本）
   - 下载：https://dotnet.microsoft.com/download/dotnet → 选版本 → Hosting Bundle
   - 安装后**重启 IIS**（`iisreset`）

2. 确认 IIS 模块中有 **AspNetCoreModuleV2**

### 6.2 部署步骤

1. 将 `publish/` 目录整个复制到服务器，例如 `C:\inetpub\nogyosuisan\`
2. 打开 IIS 管理器 → 添加网站：
   - 网站名：`Nogyosuisan`
   - 物理路径：`C:\inetpub\nogyosuisan\`
   - 端口：`80`（或其他）
   - 应用程序池：选 **无托管代码**（因为 ASP.NET Core 自己管理运行时）
3. 确保应用程序池标识对该目录有**读写权限**（SQLite 需要写 .db 文件）

### 6.3 IIS web.config

发布时 `dotnet publish` 会自动生成 `web.config`，内容类似：

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

如需启用日志排查问题，将 `stdoutLogEnabled` 改为 `true`，并创建 `logs` 文件夹。

### 6.4 数据库配置（正式环境）

默认使用 SQLite，正式环境可改为 SQL Server。修改 `appsettings.json`：

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=数据库服务器;Database=Nogyosuisan;User Id=sa;Password=xxx;TrustServerCertificate=True;"
  }
}
```

同时需要：
1. Infrastructure 项目引用 `Microsoft.EntityFrameworkCore.SqlServer`
2. Program.cs 中 `UseSqlite` 改为 `UseSqlServer`

---

## 七、当前版本与后续升级说明

本 Demo 已使用 **.NET 8.0** 与 **Angular 18**。若日后需要升级（例如到 .NET 9 或 Angular 19），可参考以下步骤。

### 7.1 升级 .NET（例如 net8.0 → net9.0）

将每个项目的 `<TargetFramework>` 改为目标版本（如 `net9.0`），并升级对应 NuGet 包版本（如 EF Core 9.0.x、AspNetCore 9.0.x）。

### 7.2 升级 Angular（例如 18 → 19）

修改 `frontend/package.json` 中所有 `@angular/*` 与 `@angular-devkit/build-angular`、`@angular/cli`、`@angular/compiler-cli` 的版本号，然后删除 `frontend/node_modules` 文件夹，再执行：

```powershell
cd frontend
npm install
ng build
```

### 7.3 验证

```powershell
cd backend
dotnet build
```

无错误即升级完成。

---

## 八、常见问题

### Q: ng serve 报错 "port 4200 already in use"
A: 执行 `ng serve --port 4201` 换端口，或关掉占用 4200 的进程。

### Q: dotnet run 报错 "address already in use"
A: 有另一个进程在占用 5001。用任务管理器关掉，或改 `launchSettings.json` 里的端口。

### Q: 前端页面显示但数据加载不了
A: 确认后端在运行，且 `proxy.conf.json` 中 target 端口与后端端口一致。

### Q: 部署到 IIS 后 404
A: 确认安装了 ASP.NET Core Hosting Bundle 并重启了 IIS。确认应用程序池设为"无托管代码"。

### Q: SQLite 数据库文件权限问题
A: 确保 IIS 应用程序池标识对部署目录有**写入权限**。
