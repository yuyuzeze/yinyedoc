# 前后端拆成两个工程（两个 Git 仓库）指南

拆成两个仓库后，可以各自独立 push，分别触发**前端自动部署到 Azure Static Web App**、**后端自动部署到 Azure App Service**，互不干扰。

---

## 一、拆分思路

| 仓库 | 内容 | 部署目标 | 触发 |
|------|------|----------|------|
| **前端仓库**（如 `nogyosuisan-web`） | 当前 `frontend/` 目录下的全部内容，放到新仓库**根目录** | Azure Static Web App | push → 构建 Angular → 部署 |
| **后端仓库**（如 `nogyosuisan-api`） | 当前 `backend/` 目录下的全部内容，放到新仓库**根目录** | Azure App Service | push → 构建 .NET → 部署 |

两个仓库各自有独立的 `.github/workflows`，实现「改哪里就自动打包部署哪里」。

---

## 二、操作步骤

### 1. 准备前端独立仓库

1. 在 GitHub 新建空仓库（如 `nogyosuisan-web`）。
2. 本地新建目录并初始化为 Git：
   ```bash
   mkdir nogyosuisan-web && cd nogyosuisan-web
   git init
   ```
3. **把当前项目里 `frontend` 目录下的所有文件复制到新仓库根目录**（不要带一层 `frontend`）：
   - 即新仓库根目录直接是 `angular.json`、`package.json`、`src/`、`public/` 等。
4. 复制本仓库中准备好的前端用文件：
   - 从 `docs/split-repos/frontend/.github/workflows/azure-static-web-apps.yml` 复制到新仓库的 `.github/workflows/`。
   - 从 `docs/split-repos/frontend/.gitignore` 复制到新仓库根目录的 `.gitignore`（仅保留 Node/Angular 相关）。
5. 提交并推送：
   ```bash
   git add .
   git commit -m "Initial frontend repo"
   git remote add origin https://github.com/你的用户名/nogyosuisan-web.git
   git push -u origin main
   ```
6. 在 Azure 创建 Static Web App，连接该前端仓库，并在 GitHub 仓库 Settings → Secrets 中配置 `AZURE_STATIC_WEB_APPS_API_TOKEN`。之后每次 push 会自动打包并部署。

**前端生产环境 API 地址**：部署后前端会请求 `/api`。若后端单独部署在 App Service，需在 Static Web App 的 `staticwebapp.config.json` 里配置代理，把 `/api` 转到后端 URL；或在前端用环境变量/构建时注入生产 API 根地址（例如 `https://你的后端.azurewebsites.net`）。

---

### 2. 准备后端独立仓库

1. 在 GitHub 新建空仓库（如 `nogyosuisan-api`）。
2. 本地新建目录并初始化为 Git：
   ```bash
   mkdir nogyosuisan-api && cd nogyosuisan-api
   git init
   ```
3. **把当前项目里 `backend` 目录下的所有内容复制到新仓库根目录**：
   - 即新仓库根目录是 `Nogyosuisan.sln`、`src/`（Api、Application、Domain、Infrastructure）等。
4. 复制本仓库中准备的后端用文件：
   - 从 `docs/split-repos/backend/.github/workflows/azure-app-service.yml` 复制到新仓库的 `.github/workflows/`。
   - 从 `docs/split-repos/backend/.gitignore` 复制到新仓库根目录的 `.gitignore`（仅保留 .NET 相关）。
5. **按下面「后端 workflow 配置」**修改 workflow 里的 App Service 名称、发布路径等（如有需要）。
6. 在 Azure 创建 App Service（.NET 8），在 GitHub 仓库 Settings → Secrets 中配置 `AZURE_WEBAPP_PUBLISH_PROFILE`（或使用 OIDC）。
7. 提交并推送，即可实现 push 后自动打包并部署到 App Service。

---

## 三、本地开发方式（拆分后）

- **前端**：在 `nogyosuisan-web` 里 `npm install`、`npm start`；通过 `proxy.conf.json` 把 `/api` 指到本地后端（如 `http://localhost:5001`）。
- **后端**：在 `nogyosuisan-api` 里用 Visual Studio 或 `dotnet run --project src/Api` 跑起来。
- 两个工程分别打开两个终端/两个 IDE 窗口即可，和现在「前后端同仓」时的本地联调方式一致，只是目录拆成两个仓库。

---

## 四、本仓库里预置的「拆分用」文件

以下文件是给**拆分后的两个新仓库**使用的模板，不要直接在当前 monorepo 里当主 workflow 用：

- `docs/split-repos/frontend/` — 前端独立仓库用的 workflow（`.github/workflows/azure-static-web-apps.yml`）、`.gitignore`。复制时把 **frontend 目录里的内容** 放到新仓库根目录，并把 `.github`、`.gitignore` 放到新仓库对应位置。
- `docs/split-repos/backend/`  — 后端独立仓库用的 workflow（`.github/workflows/azure-app-service.yml`）、`.gitignore`。复制时把 **backend 目录里的内容** 放到新仓库根目录，并把 `.github`、`.gitignore` 放到新仓库对应位置。记得在 workflow 里把 `AZURE_WEBAPP_NAME` 改成你的 App Service 名称，并在 GitHub Secrets 中配置 `AZURE_WEBAPP_PUBLISH_PROFILE`。

---

## 五、可选：保留当前仓库

若希望保留当前 `angulardemo` 仓库作为备份或 monorepo 历史，可以只把「前端副本」「后端副本」推到两个新仓库，无需删除当前仓库。之后日常开发与部署以两个新仓库为准即可。
