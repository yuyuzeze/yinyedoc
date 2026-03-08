# 前端部署到 Azure Static Web App

## 已做的改动

- **`.github/workflows/azure-static-web-apps-frontend.yml`**  
  仅针对 `frontend/` 的 push/PR 触发，构建 Angular 并部署到 Azure Static Web App。  
  - `app_location`: `frontend`  
  - `output_location`: `dist/frontend`  
  - 未配置 `api_location`，只部署静态前端。

## 你需要完成的步骤

1. **把代码推到 GitHub**
   - 在 GitHub 新建仓库（或使用现有仓库）。
   - 本地添加 remote 并 push（包含 `frontend` 和 `.github/workflows`）。

2. **在 Azure 创建 Static Web App 并连接 GitHub**
   - 打开 [Azure Portal](https://portal.azure.com) → 创建资源 → **Static Web App**。
   - 在「部署」步骤中：
     - 选择 **GitHub**，授权并选择本仓库、分支（如 `main`）。
     - 若 Azure 自动生成 workflow，可先跳过或删除，使用本仓库已有的 `azure-static-web-apps-frontend.yml`。
   - 创建完成后，在 Static Web App 的 **概览** 或 **管理部署令牌** 中复制 **部署令牌**。

3. **在 GitHub 仓库中配置 Secret**
   - 仓库 → **Settings** → **Secrets and variables** → **Actions**。
   - 新建 Secret，名称：`AZURE_STATIC_WEB_APPS_API_TOKEN`，值：上一步复制的部署令牌。

4. **触发部署**
   - 对 `frontend/` 做一次提交并 push 到 `main`（或 `master`），或合并到该分支的 PR，即可触发 workflow 并部署。

## 本地调试是否受影响？

**不受影响。**

- 工作流只在 **GitHub 上** 在 push/PR 时运行，不会改你本机环境。
- 本地仍然在 `frontend` 目录执行：
  - `npm install`
  - `npm start`（或 `ng serve`）
- `proxy.conf.json` 继续把 `/api` 转到 `http://localhost:5001`，本地调试方式不变。
- 只有部署到 Azure 后，线上环境的 `/api` 需要单独配置（例如后端部署到 Azure App Service 或其它地址，再在 Static Web App 里配置代理或环境变量）。

## 生产环境 API 地址（可选）

当前前端使用相对路径 `/api`。若后端部署在别的主机（例如 Azure App Service），需要：

- 在 `frontend/src/environments/` 为 production 使用单独配置（如 `environment.prod.ts`），或在构建时通过 Angular 的 `fileReplacements` 注入生产 API 根地址；
- 或在 Azure Static Web App 的 `staticwebapp.config.json` 中配置代理，把 `/api` 转发到你的后端 URL。

如需，我可以根据你当前 `environment` 结构写出具体配置示例。
