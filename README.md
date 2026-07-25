# PorterYuan · 学习笔记

> 个人学习博客 / 数字花园，部署于 [porteryuan.top](https://porteryuan.top)。
> 基于 [Quartz v5](https://quartz.jzhao.xyz/) 构建，由 Cloudflare Pages 自动发布。

## 站点简介

- **内容定位**：公开的 AI / 编程学习笔记。
- **技术栈**：Quartz v5 + quartz-community 插件（中文 locale、深色模式、图谱、全文搜索、giscus 评论）。
- **部署**：Cloudflare Pages，生产分支 `v5`，构建命令 `npx quartz plugin install && npx quartz build`，输出目录 `public`，`NODE_VERSION=22`，Deploy command 留空。
- **统计**：Cloudflare Web Analytics 在边缘层注入，无需代码配置。

## 目录结构

```
personal-site/
├── content/                # 发布的笔记（Quartz 唯一内容源）
│   ├── index.md            # 站点首页
│   ├── 404.md              # 404 页面
│   ├── 学习路线图/           # 4 份公开学习路线图
│   └── 笔记/               # 831 篇技术/八股/算法笔记
├── quartz.config.yaml      # 站点配置：标题、主题、插件、ignorePatterns
├── quartz/                 # Quartz 框架源码（依赖）
├── public/                 # 构建产物（部署用，git 忽略）
└── docs/                   # 项目文档
```

## 本地预览

```bash
npx quartz build --serve     # 启动本地服务，默认 http://localhost:8080
```

## 构建与部署

```bash
npx quartz plugin install    # 拉取 quartz-community 插件（首次或新增插件时）
npx quartz build             # 构建静态站点到 public/
```

部署为 Git 驱动：推送到 `v5` 分支 → Cloudflare Pages 自动重建并上线。
**注意**：生产分支必须填 `v5`（非 `main`）；Deploy command 务必留空（填 wrangler 会失败）。

## ignorePatterns 说明

`quartz.config.yaml` 采用白名单式排除，当前忽略项：

| 模式 | 作用 |
|---|---|
| `private` / `templates` | 私密与模板目录 |
| `.obsidian` / `.trash` | Obsidian 元数据与回收站 |
| `.uploads` | 附件上传目录 |
| `.claude` | Claude Code 本地配置（含各目录 `settings.local.json`） |

## 日常维护

- **改笔记**：直接编辑 `content/` 下对应文件，`git push` 到 `v5` 即自动部署。
- **加笔记**：放入 `content/笔记/` 对应分类目录（保持现有层级），双链自动解析。
- **加路线图**：放入 `content/学习计划/`，无需改配置（已移出 ignore）。
