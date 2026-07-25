---
title: "Aggregator 聚合与 E2E 链路"
tags:
  - 项目笔记
  - cr-agent
created: "2026-07-21"
---

# Aggregator 聚合与 E2E 链路

> 评测面板的聚合层要解决"数据怎么呈现"的问题——Tab 切换还是追加、Chart.js 还是 ECharts、样本列表平铺还是分组折叠。最终的布局方案兼顾了信息密度和认知负担，但验证环节被 browser_evaluate 坑了一路。

## 一、背景

在开发3时做好了评测 summary API，在开发4时是把数据"可视化"。评测面板的核心价值：让用户一眼看到"这个 Code Review 系统在 26 条测试样本上表现如何"，用图表 + 数字 + 详情三层信息架构，满足"概览 → 对比 → 下钻"的认知路径。

但有很多设计选择：评测面板放哪儿？跟报告区什么关系？图表用什么库？样本列表怎么组织不显得乱？每个选择都涉及信息架构的取舍。

## 二、逐个讲：问题 → 怎么想 → 怎么解

### 1. 评测面板放哪儿 → Tab 切换

**问题**：右栏已经是报告区，评测面板是放下面、还是新页面、还是 Tab 切换？

**怎么想**：三个选项。A：报告区下面追加——页面太长。B：新页面——单文件 HTML 没有路由系统。C：Tab 切换——报告和评测共用右栏，互斥显示。

方案 C 最合理——报告和评测都是"右栏内容"，Tab 切换符合"同一区域不同视图"的语义。

**怎么解**：在 panel-header 里加两个 Tab（报告/评测），点击切换 report 和 eval-view 的 `hidden` 属性。同时控制 `btn-copy-md` 和 `btn-eval-refresh` 的显隐。

### 2. Chart.js CDN 加载失败怎么办 → 双层降级

**问题**：CDN 可能被墙或网络问题，Chart.js 加载不了图表就空白。

**怎么想**：两道防线。第一道：`<script onerror>` 设置 `window.__chartJsFailed=true` 标记。第二道：`renderEvalChart` 里检查 `typeof Chart === "undefined"`，如果没加载显示降级文字。

**怎么解**：`<script onerror="window.__chartJsFailed=true">` + `renderEvalChart` 开头检查标记，失败就替换 canvas 容器为"Chart.js 加载失败"提示。用户还能看到总览卡片和样本列表，只是少了图表。

### 3. 样本列表 26 条怎么组织 → 两层折叠

**问题**：26 条样本平铺很长，用户难以快速找到感兴趣的分类。

**怎么想**：两层折叠。第一层按 category 分组（security/quality/performance/structure），默认折叠。第二层每个样本点击展开详情。用户先选分类再选样本，认知路径清晰。

**怎么解**：`renderSampleGroups` 按 category 分组，每组有 header 和 body。点击 header 切换 `.open` class，通过 CSS `.eval-cat-group.open .eval-cat-body { display: block }` 控制展开。

### 4. browser_evaluate 返回 null → 换验证策略

**问题**：浏览器自动化的 `browser_evaluate` 一直返回 null，无法用 JS 检查 DOM 状态。

**怎么想**：三个替代方案。A：截图 + 视觉判断——需要 vision 能力。B：网络请求日志——证明 API 被调用了。C：PowerShell 调 API 验证数据结构 + 代码审查确认前端逻辑。

方案 B+C 最可靠。

**怎么解**：用 `browser_network_requests` 看 `GET /api/v1/evaluation/summary` 请求成功，Chart.js 加载成功。PowerShell 验证 API 返回正确的数据结构。代码审查确认前端渲染逻辑。

## 三、关键决策与取舍

| 决策 | 选择 | 取舍 |
|------|------|------|
| Tab 切换而非追加/新页面 | 布局不变，用户心智简单 | 不能同时看报告和评测，但信息密度太高不宜同屏 |
| 4 卡片用 grid 而非 flex | `repeat(4, 1fr)` 更直接 | 卡片宽度固定，长数字可能溢出 |
| Chart.js 柱状图而非雷达图 | 对比清晰，一眼看出分类差异 | 不如雷达图"酷炫"，但实用性优先 |
| CSS class 切换而非 JS 创建详情 | 代码简单，状态在 DOM 里 | 26 条预渲染 DOM 节点多一点，但无所谓 |
| 双层折叠样本列表 | 先选分类再选样本 | 多了一层点击，但对 26 条样本来说认知负担降低更多 |
| 前端不改数据格式 | mock → 真实切换时零改动 | 数据格式是前后端契约，必须设计时考虑兼容性 |

## 四、踩坑

1. **端口 8765 被占用**。上次 dev_server 没杀干净。用 `Get-NetTCPConnection -LocalPort 8765` 找进程杀掉。教训：开发服务器长时间运行后记得检查端口占用。

2. **browser_evaluate 一直返回 null**。无论脚本简单还是复杂都 null。怀疑是工具的返回值序列化有 bug。放弃用它检查 DOM，改用网络请求 + PowerShell + 代码审查三重验证。教训：工具不可靠时换思路——用其他可观测信号替代直接 DOM 检查。

3. **Chart.js 主题切换时颜色不跟随**。Chart.js 画在 canvas 上不会自动跟随 CSS 变量。在 `renderEvalChart` 里手动判断 `data-theme` 属性传对应的 textColor。在 `toggleTheme` 里加一行——评测面板可见时重绘图表。

4. **mock 数据格式要"像真的"**。一开始 mock 数据做得太随意——每类样本分数完全平均分布，看起来假。后来调成"security 偏高 / performance 偏低 / quality 和 structure 居中"的分布，才像真实评测数据。这个细节虽然不影响功能，但看到会说"这数据一眼假"。教训：mock 数据也要讲究逼真度。

## 五、常见疑问

**Q1：为什么用 Chart.js 而不是 D3.js 或 ECharts？** A：三个原因。Chart.js 够用（分组柱状图 10 行代码搞定）。体积小（~200KB），CDN 加载快。项目其他地方没用图表库，不想为了一个图表引入 ECharts（~1MB）或 D3（学习成本高）。以后做复杂可视化再换不迟。

**Q2：Chart.js 加载失败怎么办？** A：双层降级。`<script onerror>` 标记 + `renderEvalChart` 检查，失败显示"Chart.js 加载失败"提示文字。用户还能看总览卡片和样本列表——核心信息不依赖图表。

**Q3：主题切换时图表颜色怎么跟着变？** A：Chart.js 画在 canvas 上，不会自动跟随 CSS 变量。在 `renderEvalChart` 里手动判断 `data-theme` 传对应的 textColor。`toggleTheme` 里加一行重绘逻辑。

**Q4：26 条样本全预渲染，性能有没有问题？** A：~260 个 DOM 节点，现代浏览器微秒级渲染。详情默认折叠不影响布局计算。以后扩展到几千条再考虑虚拟滚动，但当前 YAGNI。

---

## 相关笔记

- [[01-技术研读/00-17条架构决策总览|00 17条架构决策总览]]
- [[01-技术研读/03-Aggregator去重排序与报告渲染|03 Aggregator去重排序与报告渲染]]
- [[05_三层容错与并发bug|05 三层容错与并发bug]]

## 技术学习笔记

- [[15-Agent架构与工具调用|Agent架构与工具调用]]
- [[10-RAG 评估：检索指标 + 生成指标|RAG评估]]
- [[39-Agent评测体系：benchmark-case-指标设计|Agent评测体系]]
- [[笔记/技术学习/TypeScript React/05-React-Router路由系统|React-Router路由系统]]
- [[笔记/技术学习/TypeScript React/06-React Hooks深入|React Hooks深入]]

**Q5：评测面板从 mock 切到真实数据时，前端需要改什么？** A：一行都不用改。因为后端 API 的响应格式（total / composite_avg / prf_avg / by_category / per_sample）完全没变，只是数据从伪随机变成了真实的 rule_based 结果。这就是面向接口设计的好处——前后端通过接口契约解耦，后端换数据源前端无感。唯一改的是展示层：加了一张"评测模式"卡片告诉用户当前是规则基线还是 LLM 评的。
