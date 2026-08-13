# PorterYuan 的学习笔记

我的个人学习站，主要放 AI 和编程相关的笔记，线上在 [porteryuan.top](https://porteryuan.top)。

用 [Quartz v5](https://quartz.jzhao.xyz/) 搭的，加了几个 quartz-community 插件：中文 locale、深色模式、双向链接图谱、全文搜索、giscus 评论。部署在 Cloudflare Pages，推 `v5` 分支就自动重建上线。

## 内容

`content/` 是唯一的笔记来源，按主题分目录：

- `笔记/` — 技术笔记，下面按 AI 与 Agent、Leetcode、工程化与运维、计算机基础、语言与框架 分类
- `学习路线图/` — 4 份公开路线图（Agent 方法论、LeetCode、八股文、技术学习）
- `index.md`、`404.md` — 首页和 404 页

笔记大多是从 Obsidian 同步过来的，双链在构建时自动解析成图谱。

## 本地跑起来

```bash
npx quartz build --serve   # 本地预览，默认 http://localhost:8080
```

## 构建与部署

```bash
npx quartz plugin install   # 拉取 quartz-community 插件（首次或新增插件时）
npx quartz build            # 构建静态站到 public/
```

部署是 git 驱动的：推 `v5` 分支 → Cloudflare Pages 自动重建。两个容易踩的坑：

- 生产分支要填 `v5`，不是 `main`
- Cloudflare 里 Deploy command 必须留空，填了 wrangler 之类会构建失败

## 维护

- **改笔记**：直接编辑 `content/` 下对应文件，push 就上线
- **加笔记**：放进 `content/笔记/` 对应分类目录，双链自动解析
- **加路线图**：放进 `content/学习路线图/`，这个目录已移出 ignorePatterns，不用改配置

`quartz.config.yaml` 里 ignorePatterns 目前排除：

| 模式 | 排除内容 |
|---|---|
| `private` / `templates` | 私密笔记和模板 |
| `.obsidian` / `.trash` | Obsidian 元数据和回收站 |
| `.uploads` | 附件上传目录 |
| `.claude` | Claude Code 本地配置 |
