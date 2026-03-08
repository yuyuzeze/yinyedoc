# 农林水产省 Demo

前后端分离 Demo 环境，基于 Angular 18 + .NET 8 Web API + EF Core + SQLite。

## 环境要求

- **Node.js** 18+（推荐 20 LTS）
- **.NET SDK** 8.0
- 本地无需安装 SQL Server，Demo 使用 SQLite 单文件数据库

## 项目结构

```
├── backend/                 # .NET Web API（Clean Architecture）
│   ├── src/
│   │   ├── Api/             # Web API 入口、Controllers
│   │   ├── Application/      # 应用层（DTO、服务接口与实现）
│   │   ├── Domain/          # 领域实体
│   │   └── Infrastructure/  # EF Core、DbContext、Repository
│   └── Nogyosuisan.sln
├── frontend/                # Angular 18 SPA
│   ├── src/app/
│   │   ├── core/            # 模型、服务
│   │   └── features/        # 功能模块（Demo 列表与表单）
│   └── proxy.conf.json      # 开发时代理到后端
└── README.md
```

## 本地运行

### 1. 启动后端 API

```bash
cd backend/src/Api
dotnet run
```

- API 地址：**http://localhost:5001**
- Swagger：**http://localhost:5001/swagger**
- 首次运行会自动创建 SQLite 数据库文件 `nogyosuisan.db`（位于 Api 项目目录）

### 2. 启动前端

```bash
cd frontend
npm install
ng serve
```

- 前端地址：**http://localhost:4200**
- 通过 `proxy.conf.json` 将 `/api` 请求代理到 `http://localhost:5001`，无需单独配置 CORS

### 3. 联调验证

1. 浏览器打开 http://localhost:4200
2. 在列表中点击「新增」创建一条 Demo 数据，或先在 http://localhost:5001/swagger 中用 **Try it out** 调用 `POST /api/demoitems` 插入数据
3. 刷新前端列表，应能看到数据；可进行编辑、删除

## 仅测 API（不启动前端）

- 打开 http://localhost:5001/swagger
- 使用 **DemoItems** 下的 `GET /api/demoitems`、`POST /api/demoitems` 等进行请求测试

## 数据库

- **Demo 默认**：SQLite，连接字符串 `Data Source=nogyosuisan.db`
- 如需改用 **SQL Server LocalDB**，在 `backend/src/Api/appsettings.Development.json` 中修改：

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=NogyosuisanDemo;Trusted_Connection=True;"
}
```

并在 Infrastructure 项目中引用 `Microsoft.EntityFrameworkCore.SqlServer`，在 Api 的 `Program.cs` 中改用 `UseSqlServer(connectionString)`。

## 技术栈

| 层       | 技术 |
|----------|------|
| 前端     | Angular 18（Standalone）、TypeScript、SCSS |
| 后端     | .NET 8 Web API、C# |
| ORM/数据 | EF Core 8、SQLite（可换 LocalDB） |
| 文档     | Swagger/OpenAPI |

按上述步骤即可在本地完整运行并联调前后端 Demo。
