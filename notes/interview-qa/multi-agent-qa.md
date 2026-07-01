# Interview QA · Multi-Agent 系统

> 来源：Navedphysicist/100-Agentic-AI-Interview-Questions + U9-MultiAgent系统 + U37-Subagents + vibe-agentic-interview-notes
> 覆盖：协作模式 / 通信协议 / 资源竞争 / LangGraph State / Sub-agent 隔离 / Topology / Writer-Reviewer Loop / 系统设计

---

## 一、协作模式篇

### Q1: Multi-Agent 系统中 Sequential / Hierarchical / Concurrent 三种模式的核心区别？各适用什么场景？

**Answer:**

```
Sequential（串行）:
  User → Planner → Writer → Reviewer → Final
  特点：严格顺序，一个 Agent 的输出是下一个的输入
  优势：逻辑清晰、易 debug
  劣势：慢，无法并行

Hierarchical（层级）:
              Planner (Manager)
           /     |      \
      Writer  Reviewer  Critic
           \    |      /
           Synthesizer → Final
  特点：有明确的上下级，上级分配、下级执行、结果汇总
  优势：职责清晰，可扩展角色
  劣势：Manager 可能成为瓶颈

Concurrent（并行）:
         User Request
      /        |        \
  Agent A   Agent B   Agent C   ← 同时启动
  (搜索)    (计算)    (查文档)
      \        |        /
         Aggregator → Final
  特点：多 Agent 同时工作，结果汇合
  优势：最快，充分利用并行
  劣势：结果合并复杂，可能有冲突
```

| 模式 | 复杂度 | 延迟 | 可扩展性 | 典型场景 |
|------|--------|------|---------|---------|
| Sequential | 低 | 高 | 差 | 写作流水线（草稿→改写→校对）|
| Hierarchical | 中 | 中 | 好 | 复杂创作审核（Planner+Writer+Reviewer）|
| Concurrent | 高 | 低 | 最好 | 多源并行研究（财务+技术+团队）|

**追问：** 一个系统能同时混用三种模式吗？→ 可以，如 Manager 内部用 Concurrent，Manager 之间用 Sequential

---

### Q2: Hierarchical 模式中 Manager Agent 的设计要点是什么？

**Answer:**

Manager Agent 的职责：
1. **任务分解**：把用户需求拆成可并行的子任务
2. **Agent 分配**：根据子任务类型选合适的 Worker
3. **结果汇总**：收集所有 Worker 的输出，整合成最终答案
4. **质量控制**：判断输出是否满足要求，不满足则重新分配

```python
# Manager 伪代码
class Manager:
    def handle(self, user_request):
        # 1. 分解
        subtasks = self.decompose(user_request)
        # 2. 并行分发
        results = parallel_map(subtasks, self.assign_worker)
        # 3. 汇总
        return self.synthesize(results)

    def assign_worker(self, subtask):
        if subtask.type == "search": return SearchAgent()
        if subtask.type == "write":  return WriterAgent()
        if subtask.type == "review": return ReviewerAgent()
```

**关键设计选择：**
- Manager 是否参与执行？→ 通常不参与，纯编排
- Manager 的 LLM 用什么？→ 通常用最强的模型（如 Opus），因为它要做决策
- 失败重试由谁处理？→ Manager 负责决定重试还是放弃

---

### Q3: Concurrent 模式中如何解决多个 Agent 同时写同一个资源的问题？

**Answer:**

三种策略：

**策略1：互斥锁（Mutex）**
```python
class ResourceLock:
    def __init__(self):
        self._lock = False

    async def acquire(self, agent_id):
        while self._lock:
            await asyncio.sleep(0.1)
        self._lock = True
        self._owner = agent_id

    def release(self):
        self._lock = False

# 使用
lock = ResourceLock()
await lock.acquire("agent_a")
write_file("output.md", content)
lock.release()
```

**策略2：队列化（Queue）**
```python
from asyncio import Queue

write_queue = Queue()

async def agent_write(agent_id, content):
    await write_queue.put((agent_id, content))

async def writer_worker():
    while True:
        agent_id, content = await write_queue.get()
        # 串行写入，无冲突
        write_file(f"output_{agent_id}.md", content)
        write_queue.task_done()
```

**策略3：版本冲突检测**
```python
class VersionedFile:
    def __init__(self, path):
        self.path = path
        self.version = 0

    def write(self, agent_id, content, expected_version):
        if expected_version != self.version:
            raise ConflictError(f"版本冲突：期望 {expected_version}，实际 {self.version}")
        # 写入
        with open(self.path, 'w') as f:
            f.write(content)
        self.version += 1
```

| 策略 | 适用场景 | 缺点 |
|------|---------|------|
| Mutex | 真正独占的资源 | 其他 Agent 必须等待 |
| Queue | 可顺序处理 | 可能产生排队延迟 |
| 版本冲突 | 可合并的写操作 | 冲突时需要合并逻辑 |

**追问：** 如果多个 Agent 都要读同一个文件，需要锁吗？→ 不需要，读操作可以并发；但如果一个 Agent 在读另一个在写，就需要 Reader-Writer Lock

---

## 二、通信协议篇

### Q4: A2A 协议是什么？和 MCP 的区别？

**Answer:**

```
A2A（Agent-to-Agent）= Agent 之间的"对话协议"
MCP（Model Context Protocol）= Agent 和 Tool 之间的"调用协议"

类比：
- A2A = 两个同事怎么发消息（HTTP / gRPC）
- MCP = 一个员工怎么调用公司的各种工具（内部 API）
```

| 维度 | A2A | MCP |
|------|-----|-----|
| **通信双方** | Agent ↔ Agent | Agent ↔ Tool / Resource |
| **解决什么问题** | Agent 发现、任务委派、结果返回 | Tool 标准化调用 |
| **核心概念** | Agent Card, Task, Artifact | Resource, Tool, Prompt |
| **协议层** | HTTP / SSE / gRPC | JSON-RPC |
| **角色** | 横向（Agent 之间） | 纵向（Agent 向下） |

**关键理解**：A2A 和 MCP 是互补关系，不是竞争关系：

```python
# Agent A 要搜索 + 计算结果
# A2A：Agent A → Agent B（搜索 Agent）
# MCP：Agent B 内部调用 search_tool / calculator_tool

# Agent A 发送 A2A 任务
response = await a2a_client.send_task(
    agent="search-agent",
    task="搜索 Python 3.12 新特性"
)

# Agent B（搜索 Agent）内部用 MCP 调用工具
search_result = mcp_client.call_tool("web_search", query="Python 3.12 new features")
```

**追问：** 为什么不用 MCP 实现 Agent 间通信？→ MCP 设计目标是 Tool/Resource 暴露，不是 Agent 编排。A2A 有任务状态管理（submitted → working → completed），这是 MCP 没有的。

---

### Q5: 共享消息（Shared Message Board）和邮箱模式（Mailbox）各有什么优劣？

**Answer:**

**共享消息板：**
```python
class MessageBoard:
    def __init__(self):
        self.messages = []

    def post(self, message):
        self.messages.append(message)

    def read_all(self):
        return self.messages

# 所有 Agent 读写同一个 board
board = MessageBoard()
agent_a.post({"type": "search_result", "content": "..."})
agent_b.post({"type": "draft", "content": "..."})
# Agent C 读取所有消息，汇总
```

- 优点：简单，天然支持广播
- 缺点：消息顺序不确定、Agent 需要自己过滤、难以追踪因果关系

**邮箱模式：**
```python
class Mailbox:
    def __init__(self):
        self.inbox = []

    def send(self, to, message):
        to.inbox.append(message)

    def receive(self):
        return self.inbox.pop(0)  # FIFO

# 每个 Agent 有自己的邮箱
alice = Agent("alice")
bob = Agent("bob")
alice.send(bob, {"task": "write draft"})
bob.send(alice, {"result": "draft done"})
```

- 优点：明确收件人、FIFO 保证顺序、天然点对点
- 缺点：广播效率低（需要遍历发送）、中间件实现复杂度高

| 维度 | 共享消息板 | 邮箱模式 |
|------|-----------|---------|
| 解耦程度 | 高 | 中 |
| 消息定位 | 需要过滤 | 直接投递 |
| 广播效率 | 高 | 低 |
| 追踪调试 | 难 | 易 |

---

### Q6: Agent 之间传递什么？完整消息格式设计？

**Answer:**

```python
from dataclasses import dataclass, field
from typing import Any, Optional
from enum import Enum

class MessageType(Enum):
    TASK_ASSIGN = "task_assign"     # Manager → Worker：分配任务
    TASK_RESULT = "task_result"     # Worker → Manager：返回结果
    QUERY = "query"                 # 请求信息
    RESPONSE = "response"           # 回应请求
    ERROR = "error"                 # 错误通知
    CONTROL = "control"             # 控制信号（停止/重试）

@dataclass
class AgentMessage:
    id: str                          # 消息唯一 ID（用于追踪）
    type: MessageType
    sender: str                      # 发送者 ID
    recipient: str                   # 接收者 ID（"broadcast" 表示广播）
    payload: dict[str, Any]          # 实际内容
    timestamp: float                 # 发送时间
    parent_id: Optional[str] = None  # 父消息 ID（用于追踪对话链）
    metadata: dict = field(default_factory=dict)  # trace_id, priority 等
```

**实际使用示例：**
```python
# Manager 分发任务
msg = AgentMessage(
    id=generate_uuid(),
    type=MessageType.TASK_ASSIGN,
    sender="manager",
    recipient="writer_agent",
    payload={"task": "写一篇关于 RAG 的文章", "word_count": 500},
    timestamp=time.time(),
    trace_id="session_abc123",  # 用于全链路追踪
)

# Worker 返回结果
result_msg = AgentMessage(
    id=generate_uuid(),
    type=MessageType.TASK_RESULT,
    sender="writer_agent",
    recipient="manager",
    payload={"draft": "...", "tokens_used": 1200},
    timestamp=time.time(),
    parent_id=msg.id,  # 指向任务消息
    metadata={"latency_ms": 3500},
)
```

**追问：** trace_id 有什么用？→ 分布式追踪，可以完整还原一次用户请求经过了哪些 Agent、每段花了多少时间，是调试 Multi-Agent 系统的关键

---

## 三、LangGraph State 设计篇

### Q7: LangGraph State 对象怎么设计？和普通 Multi-Agent 的 state 有什么区别？

**Answer:**

```python
from typing import TypedDict, Annotated
import operator

# 基础 State 定义
class AgentState(TypedDict):
    # 必填字段
    messages: Annotated[list, operator.add]   # 消息历史（自动合并）
    task: str                                  # 当前任务描述
    # 可选：中间结果
    draft: str
    review: str
    final: str
    # 控制字段
    next_agent: str                            # 下一个要执行的 Agent
    iteration: int                             # 迭代次数（防止死循环）
```

**关键设计点：**

1. **Annotated + operator.add**：消息自动合并，不会覆盖
```python
# 两个节点都往 messages 里加东西
def writer_node(state):
    return {"messages": [AIMessage(content="draft")]}

def reviewer_node(state):
    return {"messages": [AIMessage(content="review")]}

# LangGraph 自动把两条消息合并到列表里
```

2. **next_agent 控制路由**：
```python
def router(state):
    if state.get("has_issues"):
        return "writer"    # 有修改意见，回去重写
    return "synthesizer"   # 没问题，去汇总

graph.add_conditional_edges("reviewer", router, {
    "writer": "writer",
    "synthesizer": "synthesizer"
})
```

3. **iteration 防止死循环**：
```python
def writer_node(state):
    if state["iteration"] >= 3:
        return {"next_agent": "synthesizer"}  # 强制结束
    return {"draft": new_draft, "iteration": state["iteration"] + 1}
```

**vs 普通 Multi-Agent state**：

| 特性 | LangGraph State | 普通 Multi-Agent |
|------|----------------|------------------|
| 状态共享 | 内置，所有节点共享 | 需要手动传递 |
| 消息合并 | 自动（Annotated） | 手动 append |
| 路由 | 条件边（conditional_edges）| 手动 if/else |
| 持久化 | checkpoint 内置 | 需要自己实现 |
| 循环控制 | 内置（iteration 字段） | 手动 |

---

### Q8: LangGraph 中如何设计多 Agent 共享同一个 State？

**Answer:**

所有 Agent 节点读写同一个 State 对象，这是 LangGraph 的核心特性：

```python
class SharedState(TypedDict):
    task: str                          # 原始任务
    research: str                      # Agent A 的输出
    draft: str                         # Agent B 的输出
    review_feedback: str               # Agent C 的输出
    final_answer: str                  # 最终答案
    agents_spawned: Annotated[list[str], operator.add]  # 追踪哪些 Agent 参与过
    error: Optional[str]

def researcher(state: SharedState) -> SharedState:
    result = web_search(state["task"])
    return {"research": result}

def writer(state: SharedState) -> SharedState:
    # 可以读到 researcher 的结果
    prompt = f"基于以下研究：{state['research']}\n写文章：{state['task']}"
    draft = llm.generate(prompt)
    return {"draft": draft}

def reviewer(state: SharedState) -> SharedState:
    # 可以读到 researcher 和 writer 的结果
    review = llm.generate(f"审核文章：{state['draft']}")
    return {"review_feedback": review}

# 构建图：所有节点共享同一个 SharedState
builder = StateGraph(SharedState)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("reviewer", reviewer)

builder.add_edge(START, "researcher")
builder.add_edge("researcher", "writer")
builder.add_edge("writer", "reviewer")
builder.add_edge("reviewer", END)
```

**为什么这样设计好？**
- 无需显式传递参数，节点间自动可见
- 每个节点只读自己需要的字段（如 reviewer 不需要知道 research 的细节）
- 天然支持 checkpoint/resume（state 序列化）

**追问：** 如果某个 Agent 不需要看到另一个 Agent 的完整输出，怎么隔离？→ 在 State 里拆分子 State，用 subgraph；或者只传摘要（summarize）给下游

---

## 四、Sub-agent 隔离与拓扑篇

### Q9: Sub-agent 隔离 Context 为什么有效？从 Attention 机制角度解释

**Answer:**

**核心机制：Attention 的局部性**

Transformer 的 Attention 复杂度是 O(n²)，context 越大，每步推理越慢、越贵。Sub-agent 隔离本质上是：

```
主 Agent Context (100K tokens):
  [系统提示 + 历史对话 + 工具定义 + 文件内容 + ...]
  ↑ 如果 subagent 的操作结果直接塞进来，上面这块会变得更大

Sub-agent Context (8K tokens):
  [精简的系统提示 + 只传必要的输入 + 工具定义]
  ↑ 隔离后，subagent 在 8K 里做事，不会"污染"主 Agent 的 100K
```

**为什么隔离有效——三点：**

1. **Attention 不污染**：subagent 的推理链（thought/action/observation）完全在独立 context 里，不会出现在主 agent 的 attention window 里

2. **token 预算隔离**：subagent 自己的 token 消耗独立计算，不会占用主 agent 的 context budget

3. **fresh context 效应**：reviewer subagent 不会被 writer 的推理过程"影响"——它只看到最终代码，做出更客观的判断

```python
# 对比：无隔离 vs 有隔离

# ❌ 无隔离（上下文爆炸）
main_context = [
    {"role": "user", "content": "review this code"},
    {"role": "assistant", "content": "let me read 50 files..."},
    {"role": "assistant", "content": "file 1 content..."},
    # ... 50 个文件的内容全塞进 context
    {"role": "assistant", "content": "draft code..."},
    {"role": "assistant", "content": "review result"},
]

# ✓ 有隔离（subagent）
main_context = [
    {"role": "user", "content": "review this code"},
    {"role": "assistant", "content": "委派 reviewer subagent..."},
    {"role": "assistant", "content": "reviewer 摘要: 发现 3 个问题..."},  # 只有摘要
]
# subagent 自己内部读 50 个文件，主 agent 完全不知情
```

**追问：** subagent 返回摘要会不会丢失细节？→ 会，所以需要设计摘要策略（如让 subagent 返回结构化结果而非自由文本）

---

### Q10: Star topology 和 Mesh topology 有什么区别？各自适用什么场景？

**Answer:**

**Star Topology（星型）：**
```
        Coordinator (Center)
       /    |     |      \
   Agent A Agent B Agent C Agent D
```
- 所有 Agent 通过中心节点通信
- Agent 之间不直接通信
- 中心节点负责路由和协调

**Mesh Topology（网状）：**
```
   Agent A ←→ Agent B
      ↕         ↕
   Agent C ←→ Agent D
      ↕         ↕
   Agent E ←→ Agent F
```
- Agent 之间直接通信
- 去中心化
- 每个 Agent 自己决定和谁通信

| 特性 | Star Topology | Mesh Topology |
|------|--------------|--------------|
| 中心节点 | 必须有 | 无 |
| 通信路径 | 都经过中心 | 直连 |
| 延迟 | 中心可能瓶颈 | 通常更低 |
| 容错 | 中心挂则全挂 | 部分节点挂不影响整体 |
| 可扩展性 | 中心 Agent 成本高 | 可水平扩展 |
| 调试难度 | 容易（都在中心路由）| 难（消息流向分散）|

**代码对比：**

```python
# Star Topology
class StarCoordinator:
    def __init__(self):
        self.agents = {}

    def register(self, name, agent):
        self.agents[name] = agent

    def route(self, from_agent, to_agent, message):
        # 所有消息经过中心
        return self.agents[to_agent].receive(message)

# 使用
coordinator = StarCoordinator()
coordinator.register("planner", PlannerAgent())
coordinator.register("writer", WriterAgent())
coordinator.route("user", "planner", {"task": "write article"})
```

```python
# Mesh Topology
class MeshAgent:
    def __init__(self, name):
        self.name = name
        self.peers = {}  # Agent 知道所有其他 Agent

    def register_peer(self, name, agent):
        self.peers[name] = agent

    def send(self, peer_name, message):
        return self.peers[peer_name].receive(message)

    def broadcast(self, message):
        return {name: agent.receive(message) for name, agent in self.peers.items()}
```

**场景选择：**
- Star：Manager-Worker 模式，如 CrewAI 的 hierarchical 流程
- Mesh：对等协作，如 AutoGen 的 group chat

**追问：** 实际产品里用哪种多？→ Star 更常见，因为更容易控制和 debug；Mesh 适合需要灵活协作的场景（如 brainstorming）

---

### Q11: 如何选择 Star vs Mesh？给出决策框架

**Answer:**

```
需要中心控制（权限/审计/路由）？ → Star
不需要，Agent 之间平等协作？     → Mesh

Agent 数量少（<5）？            → 两者皆可
Agent 数量多（>10）？            → Star（否则消息风暴）

需要容错（个别 Agent 可能挂）？ → Mesh
需要统一日志/审计？              → Star
```

---

## 五、Writer/Reviewer Loop 篇

### Q12: Writer/Reviewer Loop 模式是什么？为什么 Review 要用独立 Context？

**Answer:**

**基本模式：**
```
Writer 写初稿 → Reviewer 审核 → 有修改意见 → Writer 修改 → Reviewer 再审 → ... → 通过或达到上限
```

```python
async def writer_reviewer_loop(task: str, max_iterations: int = 3):
    """Writer/Reviewer Loop 完整流程"""
    draft = None
    review = None

    for i in range(max_iterations):
        # 1. Writer 生成/修改
        if draft is None:
            prompt = f"写一篇关于 {task} 的文章"
        else:
            prompt = f"根据以下修改意见修改文章：\n修改意见：{review}\n当前文章：{draft}"

        draft = await writer_agent.run(prompt)

        # 2. Reviewer 审核（关键：独立 context！）
        review = await reviewer_agent.run(
            f"审核这篇文章：\n{draft}\n检查：逻辑错误、表达问题、事实准确性"
        )

        # 3. 判断是否通过
        if review.get("approved"):
            return draft

    return draft  # 达到上限，返回最新版本
```

**为什么 Reviewer 要用独立 Context？**

```
问题场景：
Writer Context: [用户需求 + Writer的推理 + 初稿 + ...]
                              ↓
如果 Reviewer 在同一个 Context 里：
  → Reviewer 会看到 Writer 的推理过程（"我觉得这个逻辑是对的"）
  → 被 Writer 的"自我感觉良好"影响
  → 审核标准变松，发现不了问题

独立 Context：
Writer Context: [用户需求 + 推理 + 初稿]  →  只把初稿传给 Reviewer
Reviewer Context: [审核要求 + 初稿]         →  没有 Writer 的推理，更客观
```

**实际效果对比：**
- 同 Context 审核：通常只发现 1-2 个问题（受 writer 影响）
- 独立 Context 审核：通常发现 3-5 个问题（fresh perspective）

---

### Q13: Writer/Reviewer Loop 中如何防止死循环？

**Answer:**

**问题：** Writer 改 A → Reviewer 说改 B → Writer 改 B → Reviewer 说改 A → ...

**解决方案：**

**方案1：迭代次数限制（最简单）**
```python
for i in range(max_iterations):  # 最多 3 轮
    ...
return draft  # 超限返回
```

**方案2：修改内容去重检测**
```python
def has_real_changes(new_draft: str, old_draft: str) -> bool:
    """检测新版本和旧版本是否有实质变化"""
    # 简单方法：比较长度
    # 进阶：计算编辑距离 / embedding 相似度
    similarity = cosine_similarity(embed(old_draft), embed(new_draft))
    return similarity < 0.95  # 相似度超过 95% 认为没有实质变化

# 在循环中检测
if i > 0 and not has_real_changes(draft, previous_draft):
    logger.warning("Writer 没有实质修改，终止循环")
    break
```

**方案3：Reviewer 评分阈值**
```python
# Reviewer 返回结构化评分
review = {
    "score": 8,           # 0-10 分
    "approved": score >= 7,
    "critical_issues": [],  # 严重问题列表
}

# 严重问题必须修复，评分达标可提前结束
if review["approved"] and not review["critical_issues"]:
    break
```

| 方案 | 复杂度 | 可靠性 | 适用场景 |
|------|--------|--------|---------|
| 次数限制 | 低 | 中 | 快速原型 |
| 去重检测 | 中 | 高 | 生产系统 |
| 评分阈值 | 高 | 最高 | 高质量要求场景 |

**追问：** 工业界实际用哪种？→ 通常三者结合：次数限制 + 评分阈值，去重检测作为可选增强

---

## 六、资源竞争与冲突处理篇

### Q14: 多个 Agent 同时调用同一个 Tool 怎么处理？

**Answer:**

**场景：** 3 个 Agent 同时要写同一个文件 `output.md`

**问题：** 最终写入取决于谁最后写，结果是 nondeterministic 的

**解决方案：**

```python
class ToolMutex:
    """Tool 级别互斥锁"""
    def __init__(self):
        self._locks: dict[str, asyncio.Lock] = {}

    async def acquire(self, tool_name: str, agent_id: str):
        if tool_name not in self._locks:
            self._locks[tool_name] = asyncio.Lock()
        await self._locks[tool_name].acquire()
        logger.info(f"{agent_id} 获得 {tool_name} 锁")

    def release(self, tool_name: str, agent_id: str):
        self._locks[tool_name].release()
        logger.info(f"{agent_id} 释放 {tool_name} 锁")

# 使用
mutex = ToolMutex()

async def agent_write(agent_id, content):
    await mutex.acquire("write_file", agent_id)
    try:
        write_file("output.md", content)
    finally:
        mutex.release("write_file", agent_id)
```

**更优雅的方案：Tool 层封装**

```python
class ThreadSafeWriteTool:
    """自动处理并发的写工具"""

    def __init__(self, path: str):
        self.path = path
        self._queue = asyncio.Queue()
        self._version = 0

    async def write(self, agent_id: str, content: str):
        """排队写入，版本号递增"""
        await self._queue.put((agent_id, content))
        # 实际写入是串行的
        agent, content = await self._queue.get()
        with open(self.path, 'w') as f:
            f.write(content)
        self._version += 1
        return {"version": self._version, "writer": agent}
```

**追问：** 如果两个 Agent 要修改同一份 Vector DB 的数据怎么办？→ 类似方案：每个修改操作带版本号，写入前检查版本，冲突则返回 ConflictError，由上层决定重试还是合并

---

### Q15: 版本冲突检测（Optimistic Concurrency Control）在 Multi-Agent 中怎么用？

**Answer:**

```python
class VersionedResource:
    """带版本控制的共享资源"""
    def __init__(self):
        self._data = {}
        self._versions = {}  # key → version number

    def read(self, key: str) -> tuple[Any, int]:
        """读取时返回当前版本号"""
        return self._data.get(key), self._versions.get(key, 0)

    def write(self, key: str, value: Any, expected_version: int, agent_id: str) -> bool:
        """写入时检查版本号（OCC）"""
        current = self._versions.get(key, 0)
        if current != expected_version:
            raise ConflictError(
                f"Agent {agent_id}: 版本冲突 "
                f"(期望 {expected_version}，实际 {current})"
            )
        self._data[key] = value
        self._versions[key] = current + 1
        return True

# 使用示例
resource = VersionedResource()

# Agent A 读
data, v = resource.read("config")

# Agent B 也在读
data2, v2 = resource.read("config")

# Agent A 写（成功）
resource.write("config", {"model": "gpt-4"}, v, "agent_a")

# Agent B 写（失败，版本号不对）
try:
    resource.write("config", {"model": "claude"}, v2, "agent_b")
except ConflictError:
    # Agent B 需要重新读取、合并、重试
    new_data, new_v = resource.read("config")
    merged = merge(data2, new_data)
    resource.write("config", merged, new_v, "agent_b")
```

**OCC vs Lock-based：**

| 策略 | 性能 | 复杂度 | 适用场景 |
|------|------|--------|---------|
| OCC（乐观） | 读多写少时高 | 中 | 冲突不频繁的场景 |
| Lock（悲观） | 写多时稳定 | 低 | 写频繁的场景 |
| 无锁（CRDT） | 最高 | 高 | 需要最终一致性的场景 |

---

## 七、架构与设计篇

### Q16: Star topology 和 Mesh topology 在代码层面怎么实现？各有什么 trade-off？

**Answer:**

**Star Topology 实现：**

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class StarHub:
    """星型拓扑中心节点"""
    agents: dict[str, Any]
    message_log: list[dict]

    def register(self, name: str, agent: Any):
        self.agents[name] = agent

    def send(self, from_agent: str, to_agent: str, message: dict):
        """所有消息必须经过 Hub"""
        self.message_log.append({
            "from": from_agent,
            "to": to_agent,
            "message": message,
            "timestamp": time.time(),
        })
        return self.agents[to_agent].receive(message)

    def broadcast(self, from_agent: str, message: dict):
        """广播：发送给所有其他 Agent"""
        results = {}
        for name, agent in self.agents.items():
            if name != from_agent:
                results[name] = agent.receive(message)
        return results

# 使用
hub = StarHub(agents={})
hub.register("manager", ManagerAgent())
hub.register("writer", WriterAgent())
hub.register("reviewer", ReviewerAgent())

# 所有通信都经过 hub
hub.send("manager", "writer", {"task": "write draft"})
hub.send("writer", "reviewer", {"draft": "..."})
hub.send("reviewer", "manager", {"feedback": "..."})
```

**Mesh Topology 实现：**

```python
@dataclass
class MeshNode:
    """网状拓扑节点"""
    name: str
    agent: Any
    peers: dict[str, "MeshNode"]

    def connect(self, peer: "MeshNode"):
        """直接建立连接"""
        self.peers[peer.name] = peer
        peer.peers[self.name] = self

    def send(self, peer_name: str, message: dict):
        """直接发送给对等节点"""
        if peer_name not in self.peers:
            raise ValueError(f"未知对等节点: {peer_name}")
        return self.peers[peer_name].agent.receive(message)

    def broadcast(self, message: dict):
        """广播给所有已连接的节点"""
        return {
            name: peer.agent.receive(message)
            for name, peer in self.peers.items()
        }

# 使用：需要手动建立连接
alice = MeshNode("alice", AliceAgent(), {})
bob = MeshNode("bob", BobAgent(), {})
charlie = MeshNode("charlie", CharlieAgent(), {})

alice.connect(bob)
bob.connect(charlie)
alice.connect(charlie)  # 需要显式建立全连接
```

| 对比 | Star | Mesh |
|------|------|------|
| 通信路径 | 2 跳（A→Center→B）| 1 跳（A→B）|
| 日志/审计 | 统一在 Center | 分散在各节点 |
| 新节点加入 | 只需注册到 Center | 需要和所有其他节点建连接 |
| 网络故障 | 单点风险 | 部分隔离 |
| 消息风暴 | 不可能 | 需要 TTL / 去重 |

---

### Q17: LangGraph State 对象设计的核心原则和常见陷阱？

**Answer:**

**核心原则：**

1. **State 是单一数据源**：所有节点读同一个 State，不传参
2. **Annotated 处理合并**：列表类型用 `Annotated[list, operator.add]` 自动合并
3. **不可变更新**：每个节点返回新的 State 片段，LangGraph 合并
4. **控制字段和业务字段分离**：`next_agent` 和 `task` 分属不同字段

**完整 State 设计模板：**

```python
from typing import TypedDict, Annotated, Optional
import operator
from langchain_core.messages import BaseMessage

class MultiAgentState(TypedDict):
    # === 消息历史（自动合并）===
    messages: Annotated[list[BaseMessage], operator.add]

    # === 任务相关 ===
    task: str                              # 原始任务
    task_decomposition: list[str]          # 分解后的子任务

    # === 各 Agent 输出 ===
    research_result: str
    draft: str
    review_feedback: str
    final_answer: str

    # === 控制字段 ===
    next: str                              # 下一个节点
    iteration: int                         # 迭代计数
    error: Optional[str]                   # 错误信息

    # === 元数据 ===
    trace_id: str                          # 追踪 ID
    start_time: float                      # 开始时间
```

**常见陷阱：**

```python
# ❌ 陷阱1：忘记 Annotated，导致消息被覆盖
class BadState(TypedDict):
    messages: list  # 没有 Annotated，后写节点的消息会覆盖先写的

# ✓ 正确
class GoodState(TypedDict):
    messages: Annotated[list, operator.add]  # 自动合并

# ❌ 陷阱2：State 里塞太多不需要的字段
class BloatedState(TypedDict):
    task: str
    full_file_content: str          # 10万 token，但只有 1 个节点需要
    all_search_results: str         # 同上
    intermediate_calc: str          # 中间计算，不需要持久化

# ✓ 正确：按需放在节点内部，不要放 State
def search_node(state):
    # 临时变量，不需要放 State
    full_content = expensive_search(state["task"])
    summary = summarize(full_content)
    return {"search_summary": summary}  # 只存摘要

# ❌ 陷阱3：忘记处理循环终止
def writer_node(state):
    # 没有检查 iteration，会无限循环
    return {"draft": new_draft, "next": "reviewer"}

# ✓ 正确
def writer_node(state):
    if state["iteration"] >= 3:
        return {"next": "synthesizer"}  # 强制退出
    return {"draft": new_draft, "iteration": state["iteration"] + 1, "next": "reviewer"}
```

---

## 八、面试系统设计题

### Q18: 设计一个 Multi-Agent 内容创作系统，从零到生产部署

**Answer:**

这是一个典型的系统设计题，按以下框架回答：

**第一步：需求澄清**

```
功能需求：
- 输入：主题关键词、字数要求、风格
- 输出：结构化的文章 + 引用来源
- 流程：搜索 → 写作 → 审核 → 修改 → 定稿

非功能需求：
- 延迟：< 30 秒
- 可靠性：失败重试，人工可介入
- 成本：平均每次 < ¥0.5
```

**第二步：架构设计**

```
                    ┌──────────────────┐
                    │   API Gateway    │
                    │   (FastAPI)      │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   Manager Agent  │  ← 顶层编排
                    │ (GPT-4 / Opus)   │
                    └────────┬─────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
    │ Search Agent│  │ Writer Agent│  │Reviewer Agent│
    │ (Sonnet)    │  │ (Sonnet)    │  │ (Opus)       │
    │ + WebSearch  │  │ + WriteTool │  │ + ReadOnly   │
    └─────────────┘  └──────┬──────┘  └──────────────┘
                            │
                    ┌───────▼────────┐
                    │  Synthesizer   │
                    │  汇总 + 格式   │
                    └───────────────┘
```

**第三步：关键设计决策**

1. **State 管理**：LangGraph StateGraph，checkpoint 支持断点续跑
2. **通信协议**：A2A（Agent 间）+ MCP（Tool 调用）
3. **资源竞争**：WebSearch 用队列（外部 API rate limit）；文件写用 Mutex
4. **错误处理**：每个 Agent 有独立 try-catch，失败通知 Manager 决定重试/降级
5. **可观测性**：每个 Agent 有独立 trace_id，LangSmith / Phoenix 追踪

**第四步：技术选型**

| 组件 | 选择 | 理由 |
|------|------|------|
| 编排 | LangGraph | 原生 StateGraph，支持条件路由 |
| 模型 | Opus(Manager) / Sonnet(Worker) | Manager 需要强推理，Worker 够用 |
| 向量搜索 | Qdrant Cloud | 搜索 Agent 需要 RAG |
| 消息队列 | Redis Streams | Agent 间异步通信 |
| 部署 | Docker Compose | 开发；K8s 生产 |

**追问：** 如果 Review 一直不通过怎么办？→ 最多 3 轮迭代，然后人工介入（HITL）

---

### Q19: 设计一个 Multi-Agent 代码审查系统

**Answer:**

```
需求：提交代码 → 多维度审查 → 汇总报告

架构：
┌─────────────┐
│ 提交者提交  │
└──────┬──────┘
       │
┌──────▼──────────────────────────────────┐
│  Manager (分类审查维度，分发任务)        │
└───┬──────────┬──────────┬──────────┬────┘
    │          │          │          │
┌───▼───┐  ┌───▼───┐  ┌───▼───┐  ┌──▼────┐
│Security│ │ Style │ │Tests  │ │Perf  │  ← 并发审查
│Reviewer│ │Checker│ │Checker│ │Checker│
│(Opus)  │ │(Haiku)│ │(Sonnet)│ │(Sonnet)│
└───┬────┘  └───┬────┘  └───┬────┘  └──┬────┘
    │          │          │          │
    └──────────┴──────────┴──────────┘
               │
        ┌──────▼──────┐
        │  Aggregator │  ← 汇总所有审查意见
        │  (Opus)     │
        └─────────────┘
```

```python
class CodeReviewSystem:
    def __init__(self):
        self.agents = {
            "security": SubAgent(system=SECURITY_PROMPT, tools=[Read, Grep], model="opus"),
            "style": SubAgent(system=STYLE_PROMPT, tools=[Read], model="haiku"),
            "tests": SubAgent(system=TEST_PROMPT, tools=[Read, Bash], model="sonnet"),
            "perf": SubAgent(system=PERF_PROMPT, tools=[Read, Grep], model="sonnet"),
        }

    async def review(self, code: str, file_path: str) -> dict:
        # 并发审查
        reviews = await asyncio.gather(*[
            agent.run(f"Review: {code}\nFile: {file_path}")
            for agent in self.agents.values()
        ])

        # 汇总
        aggregator = Agent(system=AGGREGATOR_PROMPT, model="opus")
        result = aggregator.run(
            f"汇总以下审查意见：\n" + "\n".join(reviews)
        )
        return result
```

**关键设计：**
- 不同维度用不同 Agent（专业化）
- 并发执行（降低延迟）
- Aggregator 是唯一用 Opus 的节点（需要强推理来整合多维度反馈）
- 每个 Reviewer 只 Read + Grep（不能 Write，防止误改代码）

---

## 九、面试 Tips

### Q20: Multi-Agent 系统设计面试怎么答？给出答题框架

**Answer:**

**万能框架：需求 → 架构 → 状态 → 通信 → 容错 → 成本**

```
1. 需求澄清（1-2 分钟）
   - 输入输出是什么？
   - 需要几个 Agent？各自角色？
   - 延迟/成本/可靠性要求？

2. 架构草图（2-3 分钟）
   - 画图：Star / Mesh / Hierarchical
   - 标出每个 Agent 的职责
   - 标出通信路径（A2A）和 Tool 调用（MCP）

3. State 设计（1-2 分钟）
   - LangGraph StateGraph or 自实现？
   - State 里放什么？放多少？
   - 如何避免 context 爆炸？

4. 通信协议（1 分钟）
   - A2A：Agent 间任务委派
   - MCP：Agent 内部的 Tool 调用
   - 消息格式：带 ID、type、timestamp、trace_id

5. 冲突处理（1 分钟）
   - 资源竞争：Mutex / Queue / OCC
   - 死循环防护：迭代限制 + 去重检测

6. 容错与可观测性（1 分钟）
   - 每个 Agent 独立 error handling
   - trace_id 全链路追踪
   - HITL 人工介入点

7. 成本估算（1 分钟）
   - 每个 Agent 的 token 消耗
   - 总延迟 = max(并行 Agent 延迟) + 串行部分
```

**评分标准：**
- 是否能区分 A2A 和 MCP（很多人混淆）
- 是否提到 context 隔离（Sub-agent 的关键）
- 是否有实际的代码片段（证明不是背书）

---

### Q21: 怎么避免 Multi-Agent 系统中的上下文爆炸（Context Explosion）？

**Answer:**

**问题根源：**
```
Agent A 输出 10K tokens → Agent B 接收 → Agent B 输出 15K tokens → Agent C 接收...
最终：Context 指数增长，超出窗口，成本暴涨
```

**六大防御策略：**

**1. Sub-agent 隔离（最有效）**
```python
# 不要让所有 Agent 共享同一个 context
# 每个 subagent 有自己的 context window
result = subagent.run(task)  # 独立 8K context
return summarize(result)      # 只回传 500 token 摘要
```

**2. 摘要回传（Summarize-as-you-go）**
```python
def summarize_and_pass(content: str, max_tokens: int = 500) -> str:
    """把长内容压缩成摘要再传给下一个 Agent"""
    if len(tokenize(content)) <= max_tokens:
        return content
    return llm.generate(f"总结以下内容，不超过 {max_tokens} tokens：\n{content}")
```

**3. 结构化输出（Structured Output）**
```python
# ❌ 自由文本：1000 tokens
reviewer_output = "这篇文章有很多问题，首先..."

# ✓ 结构化：200 tokens
reviewer_output = {
    "approved": False,
    "score": 5,
    "issues": [
        {"line": 23, "type": "logic", "desc": "因果倒置"},
        {"line": 45, "type": "fact", "desc": "数据过时"},
    ]
}
```

**4. 选择性传递（Pass only what's needed）**
```python
# 主 Agent 不需要知道 subagent 的完整推理过程
subagent_output = subagent.run(task)

# 只传结果，不传中间步骤
main_agent_input = {
    "task": "继续下一步",
    "subagent_result": subagent_output.final_answer,  # 只传最终结果
    # 不传 subagent 的 messages / thought / action 链
}
```

**5. 增量更新（Delta Updates）**
```python
# ❌ 每次都传全量 State
state = {"messages": [...所有历史消息...]}

# ✓ 增量：只传新增部分
state = {"messages": [new_message]}  # 自动合并
```

**6. RAG 替代全文传递**
```python
# ❌ 把 100 个文件全塞进 context
context = read_all_files(directory)

# ✓ 用 RAG 检索相关片段
relevant_chunks = vector_db.search(query, top_k=5)
context = "\n".join(chunk.content for chunk in relevant_chunks)
```

| 策略 | 防御效果 | 实现成本 |
|------|---------|---------|
| Sub-agent 隔离 | ★★★★★ | 中 |
| 摘要回传 | ★★★★ | 低 |
| 结构化输出 | ★★★★ | 中 |
| 选择性传递 | ★★★ | 低 |
| 增量更新 | ★★★ | 低（框架内置）|
| RAG | ★★★★ | 高 |

**面试一句话总结：** "避免上下文爆炸的核心思想是 **each agent only sees what it needs**，用子任务隔离 + 摘要压缩 + 结构化输出三层防御。"

---

### Q22: 实际工业级 Multi-Agent 系统有哪些？它们的架构特点？

**Answer:**

| 系统 | 架构 | 特点 |
|------|------|------|
| **CrewAI** | Hierarchical + Sequential | 角色定义简单，process 参数切换模式 |
| **AutoGen** | Mesh (GroupChat) | 灵活，但调试难 |
| **LangGraph** | StateGraph (可任意拓扑) | 最灵活，生产级持久化 |
| **Claude Code** | Star + Subagent | 主从模式，subagent 定义简单 |
| **OpenAI Swarm** | Handoff (Mesh) | 极简，handoff 机制 |
| **Magentic-One** | Hierarchical | 微软，Orchestrator + 4 Worker |

**面试追问：** 你用过哪个？→ 诚实回答，可以对比优缺点

---

### Q23: Multi-Agent 系统怎么测试？

**Answer:**

**分层测试策略：**

```python
# 1. 单元测试：单个 Agent
def test_writer_agent():
    agent = WriterAgent()
    result = agent.run("写一篇关于 AI 的文章")
    assert len(result) > 100
    assert "AI" in result

# 2. 集成测试：两个 Agent 协作
async def test_writer_reviewer_loop():
    result = await writer_reviewer_loop("AI", max_iterations=2)
    assert result is not None
    assert not result.get("error")

# 3. 端到端测试：完整流程
async def test_full_pipeline():
    system = ContentCreationSystem()
    result = await system.run("Python 3.12 新特性")
    assert "draft" in result
    assert "review" in result
    assert result.get("iteration", 0) < 5  # 不会死循环

# 4. 回归测试：prompt 变更不影响行为
def test_agent_output_consistency():
    agent = ReviewerAgent()
    result = agent.run(standard_input)
    assert result["score"] >= 5  # 评分不能突然变低
```

**关键测试点：**
- 循环终止（不会无限迭代）
- 错误传播（子 Agent 失败，Manager 能感知）
- 资源释放（Mutex 是否正常释放）
- Token 消耗（防止 cost 爆炸）

---

### Q24: Multi-Agent 系统中的 Human-in-the-Loop 怎么实现？

**Answer:**

```python
class HITLManager:
    """Human-in-the-Loop 管理器"""

    def __init__(self):
        self.pending_approvals = {}

    async def request_approval(self, agent_name: str, action: dict):
        """请求人工审批"""
        approval_id = generate_uuid()
        self.pending_approvals[approval_id] = {
            "agent": agent_name,
            "action": action,
            "status": "pending",
        }
        # 通知人工（WebSocket / 邮件 / Slack）
        await notify_human(f"Agent {agent_name} 请求审批: {action}")
        return approval_id

    async def wait_for_approval(self, approval_id: str, timeout: int = 300):
        """等待人工审批"""
        import asyncio
        for _ in range(timeout):
            if self.pending_approvals[approval_id]["status"] == "approved":
                return True
            elif self.pending_approvals[approval_id]["status"] == "rejected":
                return False
            await asyncio.sleep(1)
        return False  # 超时

# 在 Agent 中使用
async def writer_agent(task):
    draft = await generate_draft(task)
    # 关键操作请求人工审批
    approval_id = await hitl.request_approval("writer", {"action": "publish", "content": draft})
    if await hitl.wait_for_approval(approval_id):
        return draft
    else:
        return {"error": "审批被拒绝"}
```

**HITL 设计要点：**
- 审批点在哪里？→ 写操作、发送操作、模型选择变更
- 超时策略？→ 超时后是 fail-safe（停止）还是 fail-open（继续）？
- 审计日志？→ 谁在什么时候审批了什么

---

### Q25: 如果让你优化一个现有的 Sequential Multi-Agent 系统，你会从哪些方面入手？

**Answer:**

**分析现有系统：**
```python
# 现有 Sequential 流程（慢）
User → Search(5s) → Write(10s) → Review(8s) → Total: 23s
```

**优化策略：**

**1. 识别可并行的步骤**
```python
# Search 和 Write 不能并行，但 Review 可以在 Write 的同时开始部分工作
# 如：Reviewer 提前准备审核标准（不需要等 draft）
User → Search(5s) → Write(10s) + ReviewPrep(并行, 0s) → Review(8s)
```

**2. 缓存中间结果**
```python
# 如果搜索主题相同，缓存搜索结果
cache = RedisCache()

def search_node(state):
    cache_key = f"search:{hash(state['task'])}"
    if cached := cache.get(cache_key):
        return {"research": cached}
    result = web_search(state["task"])
    cache.set(cache_key, result, ttl=3600)
    return {"research": result}
```

**3. 模型降级**
```python
# 简单任务用更便宜的模型
def select_model(task_complexity: str) -> str:
    if task_complexity == "simple":
        return "haiku"    # 0.25x cost
    elif task_complexity == "medium":
        return "sonnet"   # 1x cost
    return "opus"         # 3x cost
```

**4. 结果复用**
```python
# Writer 不需要每次都从头写
# 如果有类似任务的 draft，可以复用 + 修改
similar_draft = vector_db.search_similar(state["task"], top_k=1)
prompt = f"基于以下类似文章修改：\n{similar_draft}\n新任务：{state['task']}"
```

**5. 错误降级**
```python
# Reviewer 失败时，降级到简单检查
try:
    review = await reviewer_agent.run(draft)
except Exception:
    review = {"approved": True, "fallback": True}  # 降级：直接通过
```

| 优化 | 延迟降低 | 成本降低 | 复杂度 |
|------|---------|---------|--------|
| 并行识别 | 20-30% | - | 中 |
| 缓存 | 50-80% | 高 | 低 |
| 模型降级 | - | 40-60% | 低 |
| 结果复用 | 30-50% | 中 | 高 |
| 错误降级 | 提升可用性 | - | 低 |

---

## 十、代码实战

### Q26: 手写一个带冲突处理的 Multi-Agent 任务队列系统

**Answer:**

```python
import asyncio
import time
from dataclasses import dataclass, field
from typing import Any
from enum import Enum

class TaskStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"

class TaskConflictError(Exception):
    pass

@dataclass
class Task:
    id: str
    agent_id: str
    action: str
    params: dict
    status: TaskStatus = TaskStatus.PENDING
    result: Any = None
    error: str = None
    version: int = 0  # 用于 OCC

class AgentTaskQueue:
    """带冲突处理的 Agent 任务队列"""

    def __init__(self):
        self._queue = asyncio.Queue()
        self._locks = {}           # 资源级别互斥
        self._versions = {}        # 资源版本号
        self._results = {}

    def acquire_lock(self, resource: str, agent_id: str):
        """获取资源锁（阻止其他 Agent 同时修改同一资源）"""
        if resource not in self._locks:
            self._locks[resource] = {"locked": False, "owner": None}

        lock_info = self._locks[resource]
        if lock_info["locked"] and lock_info["owner"] != agent_id:
            raise TaskConflictError(
                f"资源 {resource} 被 {lock_info['owner']} 锁定，{agent_id} 需要等待"
            )

        lock_info["locked"] = True
        lock_info["owner"] = agent_id

    def release_lock(self, resource: str, agent_id: str):
        """释放资源锁"""
        if resource in self._locks:
            self._locks[resource] = {"locked": False, "owner": None}

    def check_version(self, resource: str, expected_version: int) -> bool:
        """OCC 版本检查"""
        current = self._versions.get(resource, 0)
        if current != expected_version:
            raise TaskConflictError(
                f"版本冲突：期望 {expected_version}，实际 {current}"
            )
        return True

    def increment_version(self, resource: str):
        self._versions[resource] = self._versions.get(resource, 0) + 1

    async def submit(self, task: Task):
        """提交任务"""
        await self._queue.put(task)

    async def run(self, agent_fn):
        """任务执行循环"""
        while True:
            task = await self._queue.get()
            task.status = TaskStatus.RUNNING
            try:
                # 执行前检查资源
                if task.params.get("resource"):
                    resource = task.params["resource"]
                    self.acquire_lock(resource, task.agent_id)
                    self.check_version(resource, task.params.get("expected_version", 0))

                # 执行
                result = await agent_fn(task)

                # 更新版本
                if task.params.get("resource"):
                    self.increment_version(task.params["resource"])
                    self.release_lock(task.params["resource"], task.agent_id)

                task.result = result
                task.status = TaskStatus.COMPLETED
                self._results[task.id] = task

            except TaskConflictError as e:
                task.error = str(e)
                task.status = TaskStatus.FAILED
                # 冲突时重新排队（带退避）
                await asyncio.sleep(1)
                await self.submit(task)

            self._queue.task_done()

# 使用示例
async def main():
    queue = AgentTaskQueue()

    # Agent A：写文件
    await queue.submit(Task(
        id="t1", agent_id="writer",
        action="write_file",
        params={"resource": "output.md", "content": "Hello A", "expected_version": 0}
    ))

    # Agent B：写同一个文件（会冲突）
    await queue.submit(Task(
        id="t2", agent_id="reviewer",
        action="write_file",
        params={"resource": "output.md", "content": "Hello B", "expected_version": 0}
    ))

    # 启动消费者
    await asyncio.gather(
        queue.run(write_file_fn),   # 单消费者，串行处理
    )
    await queue._queue.join()
```

---

### Q27: LangGraph 中实现条件路由 + 循环 + 中断的完整示例

**Answer:**

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
import operator

class ReviewState(TypedDict):
    task: str
    draft: str
    review: str
    revision_count: int
    max_revisions: int
    approved: bool
    final: str

def writer(state: ReviewState) -> ReviewState:
    """Writer 节点"""
    if state["revision_count"] == 0:
        prompt = f"写一篇文章：{state['task']}"
    else:
        prompt = f"根据修改意见修改：\n意见：{state['review']}\n当前版本：{state['draft']}"

    draft = llm.generate(prompt)
    return {"draft": draft}

def reviewer(state: ReviewState) -> ReviewState:
    """Reviewer 节点（独立 context）"""
    review = llm.generate(
        f"审核文章：\n{state['draft']}\n"
        f"返回 JSON: {{'approved': bool, 'feedback': str, 'score': int}}"
    )
    review_data = parse_json(review)
    return {
        "review": review_data["feedback"],
        "approved": review_data["approved"],
    }

def should_continue(state: ReviewState) -> str:
    """条件路由：是否继续循环"""
    if state["approved"]:
        return "synthesizer"
    if state["revision_count"] >= state["max_revisions"]:
        return "synthesizer"  # 达到上限，强制结束
    return "writer"

def synthesizer(state: ReviewState) -> ReviewState:
    """汇总节点"""
    return {"final": f"最终版本：{state['draft']}"}

def human_review(state: ReviewState) -> ReviewState:
    """人工审核节点（HITL）"""
    # 在实际部署中，这里会暂停等待人工输入
    # LangGraph 中断机制：返回 Command 暂停
    from langgraph.types import Command
    return Command(
        resume={"approved": True},  # 人工批准后继续
        update={"final": state["draft"]}
    )

# 构建图
graph = StateGraph(ReviewState)
graph.add_node("writer", writer)
graph.add_node("reviewer", reviewer)
graph.add_node("synthesizer", synthesizer)
graph.add_node("human_review", human_review)

# 边
graph.add_edge(START, "writer")
graph.add_edge("writer", "reviewer")

# 条件边：Reviewer 后分叉
graph.add_conditional_edges(
    "reviewer",
    should_continue,
    {
        "writer": "writer",           # 需要修改，回去重写
        "synthesizer": "synthesizer", # 通过，去汇总
    }
)

graph.add_edge("synthesizer", "human_review")  # 人工审核
graph.add_edge("human_review", END)

app = graph.compile()

# 运行
result = app.invoke({
    "task": "RAG 系统设计",
    "revision_count": 0,
    "max_revisions": 3,
    "approved": False,
    "draft": "",
    "review": "",
    "final": "",
})
```

**追问：** LangGraph 的中断机制怎么用？→ `Command(pause=True)` 暂停图执行，等待外部输入（如人工审批），然后用 `app.invoke(Command(resume={...}))` 恢复

---

## 十一、速查表

### Multi-Agent 速查

| 概念 | 一句话 |
|------|--------|
| Sequential | 一个输出 → 下一个输入，简单但慢 |
| Hierarchical | Manager 分配，Worker 执行，结果汇总 |
| Concurrent | 多 Agent 同时工作，结果合并 |
| A2A | Agent 间的通信协议（横向）|
| MCP | Agent 和 Tool 的协议（纵向）|
| Sub-agent | 独立 context + tool + model 的子 Agent |
| State isolation | 隔离 context 避免污染 + 降低 token 消耗 |
| OCC | 乐观并发控制，版本号检测冲突 |
| Writer-Reviewer Loop | 交替生成和审核，独立 context 保证客观性 |

---

## 面试追问链（示例）

```
问：Multi-Agent 和单 Agent 加 tool 的区别？
答：单 Agent 是"一个员工用多个工具"，Multi-Agent 是"多个员工各司其职"
追问：什么时候必须用 Multi-Agent？
答：角色职责差异大、需要并行、或单个 context 塞不下时

问：A2A 和 MCP 的区别？
答：A2A 是 Agent 之间的对话协议，MCP 是 Agent 调用工具的标准
追问：能不能只用 MCP 实现 Agent 通信？
答：技术上可以 hack，但不推荐；MCP 没有任务状态管理、没有 Agent 发现机制

问：Sub-agent 隔离 context 为什么比传摘要更有效？
答：隔离是"完全不见"，传摘要是"看了再总结"；隔离避免了 attention 污染
追问：subagent 的 context 多大合适？
答：取决于任务复杂度，8K-32K；原则是够用但不浪费
```

---

## 关联资源

- [LangGraph Multi-Agent 教程](https://python.langchain.com/docs/tutorials/multi_agent/)
- [crewAI 官方文档](https://docs.crewai.com/)
- [AutoGen 文档](https://microsoft.github.io/autogen/)
- [A2A 协议规范](https://agents.json.org/) (Google A2A)
- [MCP 规范](https://modelcontextprotocol.io/)
- [Claude Code Sub-agents](https://code.claude.com/docs/en/sub-agents)
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)
- [LangGraph Subgraphs](https://langchain-ai.github.io/langgraph/concepts/subgraphs/)

---

## 与本仓库其他单元的关系

| 关联单元 | 关系 |
|---------|------|
| U5 ReAct | 每个 Agent 节点内部用 ReAct 循环 |
| U6 Tool Use | Multi-Agent 依赖 U6 的 Tool 定义 |
| U8 Memory | Agent 间通信依赖 Memory 机制 |
| U13 A2A | A2A 是 Agent 间通信的标准协议 |
| U36 MCP | MCP 是 Agent 调用工具的标准 |
| U37 Subagents | 主+Sub-agent 是最常见的 Multi-Agent 模式 |
| U21 Loop Engineering | Multi-Agent loop 是核心场景 |

---

*最后更新：2026-07-01*
