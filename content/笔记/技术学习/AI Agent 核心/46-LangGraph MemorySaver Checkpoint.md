---
title: "LangGraph MemorySaver Checkpoint"
created: "2026-07-21"
tags:
  - 技术学习
  - ai
  - langgraph
  - checkpoint
  - agent
---

# LangGraph MemorySaver Checkpoint

---

> **一句话**：** MemorySaver 是 LangGraph 内置的 Checkpoint 持久化机制——每个 Node 执行后自动保存 State 快照，通过 `thread_id` 隔离不同会话。生产环境从 MemorySaver（内存）升级到 SqliteSaver（本地文件）或 PostgresSaver（数据库）。

---

## 一、为什么需要 Checkpoint？

上一讲 [[45-LangGraph框架：StateGraph-Node-Edge-Checkpoint|LangGraph 框架]] 讲了 Node 之间靠 State 传数据。但有一个问题没解决：**State 存在哪里？**

```text
没有 Checkpoint：
  Node A 执行 → State 在内存 → Node B 执行 → State 在内存 → 服务重启 → 💥 全丢

有 Checkpoint：
  Node A 执行 → State 自动写入 Checkpoint → Node B 执行 → State 自动写入 Checkpoint
  → 服务重启 → 从 Checkpoint 恢复 → 继续执行
```

Checkpoint 就是 **Agent 的存档点**——跟玩游戏存盘一个道理。

---

## 二、MemorySaver 工作原理

```mermaid
sequenceDiagram
    participant User as 用户
    participant Graph as StateGraph
    participant Node as Node函数
    participant CP as MemorySaver

    User->>Graph: graph.invoke(state, thread_id)
    Graph->>CP: 读取 thread_id 的上一个 checkpoint
    CP-->>Graph: 返回上次的 State（如果有）
    Graph->>Node: 执行节点，传入 State
    Node-->>Graph: 返回更新后的 partial State
    Graph->>CP: 保存新 State 快照（带序号）
    Graph->>Node: 执行下一个节点...
    Note over Graph,CP: 每个节点执行后都自动存档
```

MemorySaver 在每次 Node 执行后做两件事：

1. **合并 State**：把 Node 返回的 `dict` 合并到现有 State
2. **序列化并存储**：把完整的 State 存到内存字典 `{thread_id: [checkpoint_1, checkpoint_2, ...]}`

---

## 三、thread_id 隔离机制

```python
"""
thread_id = 会话隔离的钥匙。
不同 thread_id 的 State 完全独立，互不干扰。
"""
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, END

# 编译时挂载 MemorySaver
graph = builder.compile(checkpointer=MemorySaver())

# 用户 A 的会话
result_a = graph.invoke(
    {"messages": [HumanMessage(content="我叫张三")]},
    config={"configurable": {"thread_id": "user-a"}},
)

# 用户 B 的会话 —— 完全独立，不知道张三
result_b = graph.invoke(
    {"messages": [HumanMessage(content="我叫什么名字？")]},
    config={"configurable": {"thread_id": "user-b"}},
)
# user-b 问"我叫什么"，LLM 不会回答"张三"——因为 State 是隔离的

# 用户 A 继续对话 —— 能记住自己是张三
result_a2 = graph.invoke(
    {"messages": [HumanMessage(content="我叫什么名字？")]},
    config={"configurable": {"thread_id": "user-a"}},
)
# user-a 问"我叫什么"，LLM 回答"张三"——因为同一 thread_id 共享 State
```

```mermaid
flowchart LR
    subgraph MemorySaver内存
        T1["thread_id: user-a<br/>State: {name: 张三}"]
        T2["thread_id: user-b<br/>State: {}"]
    end
    R1["user-a 请求"] --> T1
    R2["user-b 请求"] --> T2
```

**要点：** `thread_id` 就是会话 ID。每个用户/对话一个 thread_id，State 互不干扰。这不叫"多租户"，叫**会话隔离**。

---

## 四、get_state：调试和状态检查

```python
"""
get_state() 让你在任意时刻查看某个 thread 的当前状态。
调试神器——不用 print 满天飞。
"""
# 查看 user-a 的当前状态
current_state = graph.get_state(
    config={"configurable": {"thread_id": "user-a"}}
)

print(current_state.values)          # 完整的 State 字典
print(current_state.next)            # 下一个要执行的节点（空 = 已结束）
print(current_state.metadata)        # 元数据（step 序号、执行时间等）

# 查看 user-a 的完整执行历史
history = list(graph.get_state_history(
    config={"configurable": {"thread_id": "user-a"}}
))

for snapshot in history:
    print(f"Step {snapshot.metadata['step']}: next={snapshot.next}")
    # Step 3: next=()  ← 已结束
    # Step 2: next=('answer',)
    # Step 1: next=('tool',)
    # Step 0: next=('router',)
```

get_state 的三个核心字段：

| 字段 | 含义 | 用途 |
|------|------|------|
| `values` | 当前完整的 State 字典 | 看 LLM 的中间输出、工具结果 |
| `next` | 下一个要执行的节点名 | 判断流程走到哪了 |
| `metadata` | step 序号、source、写入时间 | 追踪执行路径 |

---

## 五、三种持久化后端对比

```mermaid
flowchart TD
    subgraph 开发["开发阶段"]
        M["MemorySaver<br/>内存存储"]
    end
    subgraph 单机["单机部署"]
        S["SqliteSaver<br/>SQLite 文件"]
    end
    subgraph 生产["生产环境"]
        P["PostgresSaver<br/>PostgreSQL 数据库"]
    end
    M -->|"上测试环境"| S
    S -->|"上生产"| P
```

| 后端 | 存储位置 | 持久化 | 适用场景 | 优点 | 缺点 |
|------|----------|--------|----------|------|------|
| **MemorySaver** | 内存 dict | 重启丢失 | 本地开发、原型 | 零配置、最快 | 进程死数据没 |
| **SqliteSaver** | 本地 .db 文件 | 持久 | 单机部署、测试 | 文件落地、易备份 | 不支持并发写 |
| **PostgresSaver** | PostgreSQL | 持久 | 生产多实例 | 并发安全、多副本 | 需要 PG 实例 |

```python
# MemorySaver —— 开发用
from langgraph.checkpoint.memory import MemorySaver
graph = builder.compile(checkpointer=MemorySaver())

# SqliteSaver —— 单机测试用
from langgraph.checkpoint.sqlite import SqliteSaver
graph = builder.compile(checkpointer=SqliteSaver.from_conn_string("checkpoints.db"))

# PostgresSaver —— 生产用（推荐）
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost:5432/agent_db"
)
checkpointer.setup()  # 自动建表
graph = builder.compile(checkpointer=checkpointer)
```

> [!note] 三个后端实现了**同一个 Checkpointer 接口**，切换只需改一行构造代码。`builder.compile(checkpointer=xxx)` 的参数是接口类型，Graph 不关心底层是什么。

---

## 六、Human-in-the-loop：利用 Checkpoint 实现人工审批

这是 Checkpoint 最实用的场景——**人在回路**：

```python
"""
Human-in-the-loop 模式：
1. 在敏感操作前 interrupt() 暂停
2. 人工审查当前 State
3. 批准 → 继续执行 / 拒绝 → 回退
"""
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt

def refund_node(state: AgentState) -> dict:
    """退款节点 —— 高危操作，需人工审批。"""
    # interrupt() 暂停执行，把 State 和控制权交给人
    approval = interrupt({
        "question": f"确认退款 {state['amount']} 元给用户 {state['user_id']}？",
        "context": state["order_info"],
    })
    # 人点了"批准"后，从这里继续
    if approval == "approved":
        # 执行实际退款
        execute_refund(state["order_id"], state["amount"])
        return {"refund_status": "done"}
    else:
        return {"refund_status": "cancelled"}

# 编译
graph = builder.compile(checkpointer=MemorySaver())

# 第一次调用 → 会在 interrupt() 处暂停
config = {"configurable": {"thread_id": "order-12345"}}
try:
    graph.invoke(state, config)
except GraphInterrupt:
    pass  # 正常——正在等人审批

# 查看当前卡在哪
snapshot = graph.get_state(config)
print(snapshot.next)  # ('refund_node',) ← 卡在退款节点

# 人批准后 → 从 interrupt 点继续
graph.invoke(
    Command(resume="approved"),  # ← 把"批准"传回 interrupt()
    config,
)
```

```mermaid
sequenceDiagram
    participant User as 用户
    participant Graph as StateGraph
    participant Node as refund_node
    participant Human as 审批人

    User->>Graph: invoke(state)
    Graph->>Node: 执行 refund_node
    Node->>Human: interrupt("确认退款？")
    Note over Graph: 暂停，保存 checkpoint
    Human-->>Graph: approve
    Graph->>Node: 继续执行（resume="approved"）
    Node-->>Graph: 返回结果
    Graph-->>User: 最终 State
```

---

## 七、项目实战对应

### ai-resume：MemorySaver 用于 Reflexion 状态

ai-resume 的 Reflexion（自我反思）环节中，每次反思迭代的状态通过 MemorySaver 保存，便于查看反思轨迹：

```python
# ai-resume 项目中 Reflexion 的 checkpoint 使用
graph = builder.compile(checkpointer=MemorySaver())
config = {"configurable": {"thread_id": f"resume-{resume_id}"}}

# 第一次生成
result = graph.invoke({"resume_text": text}, config)

# 反思不通过 → 重新生成，同一 thread_id 会叠加 State
result = graph.invoke({"feedback": "技能描述不够具体"}, config)

# 查看反思历史
for snapshot in graph.get_state_history(config):
    print(f"反思轮次: {snapshot.values.get('reflection_round')}")
```

### cr-agent：生产级 PostgresSaver

cr-agent 项目使用 Supervisor-Worker StateGraph + Checkpoint 实现代码审查中的状态回溯：

详见：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/01-Supervisor-Worker编排与StateGraph.md#记忆级graph.py+state.py逐行精读|cr-agent: StateGraph]]

## 相关链接

- 目录：[[00-AI]]
- 上一篇：[[45-LangGraph框架：StateGraph-Node-Edge-Checkpoint]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/03_LangGraph状态机.md#Checkpoint 机制|ai-resume: LangGraph状态机]]
- 项目实践：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/01-Supervisor-Worker编排与StateGraph.md#🔴 记忆级：graph.py + state.py 逐行精读|cr-agent: Supervisor-Worker编排]]

---

→ [[技术学习清单#LangChain + LangGraph]]
