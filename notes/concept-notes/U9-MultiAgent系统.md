# U9 · Multi-Agent 系统

> **核心问题**：多个 Agent 怎么协作和通信？
> **预计学时**：5h
> **对应 Phase**：Phase 5

---

## 学习目标（Gate 检验）

1. 能画出三种协作模式（sequential / hierarchical / concurrent）的架构图
2. 能用 crewAI 或 LangGraph 实现一个 writer + reviewer 双 Agent 协作
3. 能解释 Agent 间通信协议的设计要点
4. 能说出多 Agent 系统中资源竞争的处理方式

---

## 1. 什么时候需要 Multi-Agent

单一 Agent 的瓶颈：
- 角色冲突：一个 Agent 既要当规划者又要是执行者
- 能力上限：一个模型的上下文和工具集是固定的
- 可维护性：Prompt 太长难以管理

Multi-Agent 解决的：
- 角色分离：Planner 负责规划，Executor 负责执行
- 能力扩展：不同 Agent 调用不同模型
- 并行提速：多个 Agent 同时工作

---

## 2. 三种协作模式

### Sequential（串行）

```
User → Agent A → Agent B → Agent C → Final Output
```

一个 Agent 的输出是下一个的输入。简单但慢。

适用场景：写作流程（初稿 → 改写 → 校对）

### Hierarchical（层级）

```
              Planner
            /    |    \
          Writer Reviewer Critic
            |      |      |
            v      v      v
          Draft  Report  Critique
            \      |      /
              Synthesizer
                  ↓
             Final Output
```

有明确的上下级关系，上级分配任务，下级执行，结果汇总给上级。

适用场景：复杂的创作和审核流程

### Concurrent（并行）

```
              User Request
         /         |         \
    Agent A   Agent B     Agent C
     (搜索)    (计算)     (查文档)
         \         |         /
              Aggregator
                  ↓
             Final Output
```

多个 Agent 同时工作，结果汇合后统一输出。

适用场景：多信息源并行收集（如"帮我搜集 X 公司信息，要财务、技术、团队三方面"）

---

## 3. Agent 间通信协议

### 共享消息（Shared Message Board）

所有 Agent 读写同一个消息队列。

```
Agent A 写消息 → 消息队列 → Agent B 读取 → 处理 → 写回
```

优点：简单，解耦
缺点：消息顺序和依赖管理复杂

### 邮箱模式（Mailbox）

每个 Agent 有自己的收件箱，消息发送有明确的收件人。

```python
class Agent:
    def __init__(self, name):
        self.name = name
        self.mailbox = []

    def receive(self, message):
        self.mailbox.append(message)

    def send(self, recipient, content):
        recipient.receive({"from": self.name, "content": content})
```

### 共享工具（Shared Tools）

多个 Agent 共用同一个工具集，避免重复定义。

```python
shared_tools = [search_tool, calculator_tool, file_tool]

writer_agent = Agent("writer", tools=shared_tools + [writing_tool])
reviewer_agent = Agent("reviewer", tools=shared_tools + [review_tool])
```

---

## 4. 资源竞争与冲突处理

### 问题场景

多个 Agent 同时需要同一个 Tool（如写文件），或同时修改同一份数据。

### 处理策略

| 策略 | 实现方式 | 适用场景 |
|------|---------|---------|
| 互斥锁（Mutex）| 同一时间只有一个 Agent 能用某个 Tool | 写文件等独占操作 |
| 队列化（Queue）| 按顺序排队使用共享资源 | 资源有限但可共享 |
| 版本冲突检测 | 写入前检查版本，冲突则重试 | 数据库、多人协作 |
| 层级优先级 | 上级 Agent 优先 | 紧急任务优先处理 |

---

## 5. 实战：用 crewAI 实现双 Agent 协作

### 安装

```bash
pip install crewai crewai-tools
```

### 代码

```python
from crewai import Agent, Task, Crew

# 定义 Writer Agent
writer = Agent(
    role="专业写作助手",
    goal="根据用户提供的关键词写一篇结构清晰的文章",
    backstory="你是一个资深编辑，擅长把复杂的技术概念写成通俗易懂的文章。",
    verbose=True,
)

# 定义 Reviewer Agent
reviewer = Agent(
    role="文章审核专家",
    goal="检查文章的质量，给出修改建议",
    backstory="你是一个严格的审稿人，关注文章准确性、逻辑性和可读性。",
    verbose=True,
)

# 定义 Task
write_task = Task(
    description="以'AI Agent'为关键词，写一篇 500 字左右的介绍性文章，结构：引言、正文、结语。",
    agent=writer,
    expected_output="一篇结构完整的文章",
)

review_task = Task(
    description="审核 Writer 写的文章，检查逻辑错误和表达问题，给出 3 条具体修改建议。",
    agent=reviewer,
    expected_output="3 条修改建议列表",
)

# 创建 Crew（sequential 模式）
crew = Crew(
    agents=[writer, reviewer],
    tasks=[write_task, review_task],
    process="sequential",  # sequential / hierarchical
)

# 启动
result = crew.kickoff()
print(result)
```

---

## 6. 实战：用 LangGraph 实现 Multi-Agent

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    task: str
    draft: str
    review: str
    final: str

def writer_node(state: AgentState) -> AgentState:
    """Writer Agent 节点"""
    draft = f"关于「{state['task']}」的文章初稿..."
    return {"draft": draft}

def reviewer_node(state: AgentState) -> AgentState:
    """Reviewer Agent 节点"""
    review = f"建议：1) 补充更多例子 2) 结尾太仓促 3) 调整段落顺序"
    return {"review": review}

def synthesizer_node(state: AgentState) -> AgentState:
    """汇总节点"""
    return {"final": f"综合意见后的终稿: {state['draft']} + {state['review']}"}

# 构建图
graph = StateGraph(AgentState)
graph.add_node("writer", writer_node)
graph.add_node("reviewer", reviewer_node)
graph.add_node("synthesizer", synthesizer_node)

graph.set_entry_point("writer")
graph.add_edge("writer", "reviewer")
graph.add_edge("reviewer", "synthesizer")
graph.add_edge("synthesizer", END)

app = graph.compile()

# 运行
result = app.invoke({"task": "AI Agent 的发展趋势", "draft": "", "review": "", "final": ""})
print(result["final"])
```

---

## 完成 Gate 检验

1. **Q1**：画 sequential / hierarchical / concurrent 三种协作模式的架构图。
2. **Q2**：crewAI 和 LangGraph 在 Multi-Agent 编排上的核心区别是什么？
3. **Q3**：Agent 间通信协议的设计要点有哪些？
4. **Q4**：当两个 Agent 同时需要写同一个文件时，怎么处理冲突？

---

## 练习清单

- [ ] 用 crewAI 实现 writer + reviewer 双 Agent 协作，跑通完整流程
- [ ] 用 LangGraph 实现一个简单的 multi-agent 流程图
- [ ] 设计一个 hierarchical 模式的 multi-agent 系统（如 planner + writer + reviewer + editor）
- [ ] 实现一个带共享工具的多 Agent 系统

---

## 关联资源

- [LangGraph Multi-Agent 教程](https://python.langchain.com/docs/tutorials/multi_agent/)
- [crewAI 官方文档](https://docs.crewai.com/)
- [AutoGen 文档](https://microsoft.github.io/autogen/)（微软 multi-agent 框架）
- [Magentic-One](https://github.com/microsoft/autogen)（微软 multi-agent 系统）

---

## 7. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U5 ReAct 范式|每个 Agent 节点内部用 ReAct 循环|
|U6 Tool Use|Multi-Agent 系统依赖 U6 的 tool 定义和调用机制|
|U8 Memory|Agent 间通信和状态共享依赖 Memory 机制|
|U13 A2A 协议|A2A 是 Agent 间通信的标准协议，U9 的多 Agent 通信可基于 A2A|
|U36 MCP 深入|MCP 是 Agent 调用工具的标准，Multi-Agent 系统离不开 MCP|
|U37 Subagents|主 Agent + Sub-agent 是最常见的 Multi-Agent 模式|
|U21 Loop Engineering|Multi-Agent loop 是 Loop Engineering 的核心场景|

## 8. Multi-Agent 生产模式深度

### 8.1 A2A + MCP 联合架构

完整的多 Agent 系统通常同时使用 A2A（横向）和 MCP（纵向）：

```
Agent A (Planner)                    Agent B (Executor)
    │                                       │
    │── A2A: send_task("分析需求") ──────────>│ 接收任务
    │<── A2A: TaskResult ────────────────────│ 返回分析
    │                                       │
    │  MCP: list_tools()                    │  MCP: list_tools()
    │  MCP: call_tool("search") ───────────────>│
    │  MCP: call_tool("code_exec") ──────────────>│
```

**关键**：A2A 解决 Agent 间通信，MCP 解决每个 Agent 的工具调用，两者互补不替代。

### 8.2 Agent 间状态管理

Multi-Agent 系统的状态管理是核心难题：

|模式|状态位置|优点|缺点|
|---|---|---|---|
|共享内存|所有 Agent 读写同一份 state|简单|并发冲突|
|消息传递|Agent 只通过消息交换信息|解耦|延迟高|
|事件溯源|所有状态变更记录为事件|可审计|复杂|
|CRDT|冲突可合并的数据结构|无冲突|学习成本高|

### 8.3 面试能说的话

> 'Multi-Agent 通信用 A2A 协议（Agent Card + Task + Message），工具调用用 MCP 协议（Tool + Resource + Prompt）。状态管理推荐用消息传递（LangGraph 的 State 对象）——简单、可审计、可回放。'

### 8.4 生产级 Multi-Agent 设计原则

1. **单一职责**：每个 Agent 只做一件事（Planner / Researcher / Writer / Reviewer）
2. **接口契约**：Agent 间通信用明确的 schema（JSON Schema 或 Pydantic）
3. **超时与重试**：每个子任务要有 timeout + retry + fallback
4. **可观测性**：trace 每个 Agent 的输入输出，方便 debug
5. **成本控制**：并行 Agent 的 token 消耗可能爆炸，要设 budget
*最后更新：2026-05-26*