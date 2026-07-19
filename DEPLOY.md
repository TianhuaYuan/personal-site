# 部署说明：个人学习站点 → Cloudflare Pages

本站点由 Quartz v5 从 Obsidian vault 的「笔记/」生成。以下是从本地构建到上线的完整步骤。
项目位置：`D:\Project\personal-site`

## 一、本地预览（已验证可跑）

```bash
cd D:\Project\personal-site
npx quartz build --serve      # 构建并启动本地服务，访问 http://localhost:8080
```

> 注意：本沙箱对「批量删除 public/ 目录」有安全拦截（>50 文件需确认），
> 导致 `build --serve` 在清理步骤报错。在你自己的机器上直接运行即可，无此限制。
> 临时验证已用 `python -m http.server 8080 --directory public` 托管已构建产物，确认返回 200 / 68KB。

## 二、上线方式 A：GitHub + Cloudflare Git 自动部署（量传推荐，本地已就绪）

本地仓库已初始化并提交到分支 `v5`（commit `3abb981`）。只需把本地仓库连到 GitHub 空仓库，
Cloudflare 连接后即每次 `git push` 自动部署——这是量传阶段最省心的方案。

1. 在 GitHub 新建一个**空仓库**（建议名 `personal-site`；**不要**勾选 README / .gitignore / LICENSE，本地已有）：
   - 可见性 Public 或 Private 均可（Cloudflare Pages 都支持）。
2. 把本地仓库推到 GitHub（上游 remote 已移除，需加回你自己的地址）：
   ```bash
   cd D:\Project\personal-site
   git remote add origin https://github.com/<你的用户名>/personal-site.git
   git push -u origin v5
   ```
   > 关键：当前分支是 `v5`（与 Quartz 上游约定一致），**不是** `main`。
   > Cloudflare 的「生产分支」必须填 `v5`，否则检测不到提交。
3. 登录 Cloudflare Dashboard → **Workers & Pages** → **Create** → **Pages** → **Connect to Git** → 选该仓库。
4. 构建设置：
   - Framework preset：**None**
   - Build command：`npx quartz plugin install && npx quartz build`
   - Build output directory：`public`
   - Production branch：`v5`
   - Node.js version：**22**（在「环境变量」加 `NODE_VERSION=22`；package.json 的 `engines` 也已锁 `>=22`）
5. 首次部署完成后获得 `https://<项目名>.pages.dev`，点开即可访问。之后每次 `git push` 自动重建上线。

## 三、上线方式 B：wrangler 直传（试点最快）

```bash
npx wrangler login          # 浏览器 OAuth 授权（需你的 Cloudflare 账号）
npx wrangler pages deploy public
```
部署后终端会给出 `*.pages.dev` 地址。

## 四、上线后必做：回填 baseUrl

拿到真实地址（如 `personal-site-xxxx.pages.dev` 或自定义域名 `tianhua.dev`）后，
编辑 `quartz.config.yaml`：

```yaml
configuration:
  baseUrl: personal-site-xxxx.pages.dev   # 自定义域名则填 tianhua.dev（裸域名，不带 https）
```

改完重新构建 + 部署，否则 RSS / sitemap / SEO 链接会指向错误地址。

## 五、自定义域名（待定）

1. 在域名注册商处把域名 NS 交由 Cloudflare 管理，或添加 CNAME 记录指向 `*.pages.dev`。
2. Cloudflare Pages → Custom domains → 添加域名，按提示验证。
3. 同步把 `baseUrl` 改成裸域名并重部署。

## 六、当前试点状态

- 已发布内容：仅 `08-MCP协议.md`（单篇试点）。
- 已验证：Quartz v5 构建成功、Mermaid（3 个图）渲染链路、[[双链]] 输出、本地 HTTP 200。
- 未发布（私密）：`学习计划/`、`其他/cr-agent-项目计划书.md`、`00-全局导航.md`（量传时作为首页）。
- 待办：量传整棵 `笔记/` 树 → 设 `00-全局导航.md` 为首页 → 扫描跨文件夹断链 → 飞书笔记导入验证。

## 七、配置文件要点（quartz.config.yaml）

| 项 | 当前值 | 说明 |
|----|--------|------|
| pageTitle | 田华 · 技术学习笔记 | 站点标题 |
| locale | zh-CN | 中文日期/语言 |
| analytics.provider | null | 试点关闭外部分析，上线后可开 |
| baseUrl | tianhua.pages.dev | 上线后回填真实地址 |
| ignorePatterns | 含 学习计划/其他/.obsidian 等 | 白名单式排除私密 |
| footer.links | {} | 量传时填入 GitHub / 简历链接 |
| 插件 | 46 个（graph/backlinks/search/explorer/mermaid 等） | 原生双链生态 |
