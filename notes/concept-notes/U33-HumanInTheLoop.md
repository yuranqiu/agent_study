# U33 · Human-in-the-loop 设计（人在环设计）

> **核心问题**：怎么在 Agent 执行链中**合理嵌入人类决策点**，让"该人类决策"的部分真正由人决定，同时不让 Agent 变得不可用？
> **预计学时**：5h
> **对应 Phase**：Phase 4 / 面试
> **状态**：生产必修（2024-2026 Agent 安全 + 监管 + 框架标准化的核心方向）

---

## 1. 什么是 Human-in-the-loop（HITL）

**HITL** = 在 Agent 自动化流程中**保留或插入人类决策点**的范式。

**为什么需要**：
- **安全**：危险操作（删数据库 / 支付 / 发送）需要人类确认
- **质量**：关键决策（医疗 / 法律 / 投资）需要人类判断
- **合规**：监管要求（金融 / 医疗 / EU AI Act Article 14）要求 human oversight
- **信任**：用户对 agent 的信任来自"我能干预"
- **学习**：人类反馈是 Agent 改进的核心信号（RLHF、DPO 都来自 HITL）

**vs 纯自动化**：
- 纯自动化：Agent 自主决策 + 执行
- HITL：Agent 提议 + 人类决策 + Agent 执行
- Pure human：人类决策 + 人类执行（没有 agent）

**HITL 是中间态，平衡效率和安全**。Harness 11 组件中第 11 个就是 **Intervention Recording**（见 U23）。

---

## 2. 4 大 HITL 模式

### 2.1 Approval（批准）

**Pattern**：Agent 提议 action，**人类批准**后执行。

```
Agent: "我要执行 X"
Human: [Approve] / [Reject] / [Modify]
Agent: 执行 / 不执行
```

**适用**：高风险操作（删文件 / 转账 / 发送邮件 / 部署）

**实现**：
```python
def dangerous_action(payload):
    # 1. Agent 提议
    action = agent.propose(payload)
    
    # 2. 人类审查（UI 弹窗 / Slack 通知 / webhook）
    approved = human_review(action)
    
    # 3. 决策
    if approved:
        return execute(action)
    elif approved.modified:
        return execute(approved.payload)
    else:
        return "Rejected"
```

### 2.2 Interrupt（中断）

**Pattern**：Agent 执行中**主动中断**，等人类输入。

```
Agent: "需要更多信息"
  ↓ interrupt
Human: 提供信息
  ↓ resume
Agent: 继续
```

**适用**：需要补充信息 / 不确定 / 关键决策点

### 2.3 Oversight（监督）

**Pattern**：Agent **自主运行**，但人类**实时监督**（dashboard / alert）。

**适用**：低风险但高频（自动化流水线 / 监控 agent）

### 2.4 Confidence-based Delegation（基于置信度的委托）

**Pattern**：Agent **自己评估**置信度，**低置信度**才转人工。

```python
def should_delegate_to_human(response, confidence_threshold=0.7):
    # 1. self-consistency 测置信度
    samples = [llm.invoke(prompt) for _ in range(5)]
    consistency = most_common(samples) / len(samples)
    if consistency > 0.9:
        return samples[0], False
    else:
        return None, True
```

---

## 3. 5 级风险分级

| 风险等级 | 例子 | HITL 模式 | 通知渠道 |
|----------|------|-----------|----------|
| **L0 低** | 查天气、读文件 | 无 HITL | - |
| **L1 中低** | 写文件、跑命令 | Oversight | Slack 通知 |
| **L2 中** | 改 DB、API 调用 | Approval | UI 弹窗 |
| **L3 中高** | 支付、发送、删除 | Approval + 二次确认 | Slack + 二次确认 |
| **L4 高** | 不可逆操作（生产部署） | Multi-party approval | Email + 多渠道 |
| **L5 极高** | 法律、医疗、军事 | Mandatory human review | 强制 + 审计 |

详见 §11 多 reviewer 协议。

---

## 4. 三大主流框架的 HITL 实现

### 4.1 LangGraph：`interrupt()` + `Command(resume)` 规范模式

**LangGraph 是当前 HITL 设计最成熟的框架**。canonical 模式（来自官方 docs）：

```python
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

def approval_node(state: State):
    # Pause and ask for approval
    approved = interrupt("Do you approve this action?")
    
    # When you resume, Command(resume=...) returns that value here
    return {"approved": approved}

# Compile with checkpointer (use durable in production)
app = graph.compile(checkpointer=PostgresSaver(...))

# 1. 启动：跑直到 interrupt 暂停
config = {"configurable": {"thread_id": "thread-1"}}
stream = app.stream_events({"input": "data"}, config=config, version="v3")
final = stream.output  # 驱动流到底

# 2. 检测中断
if stream.interrupted:
    print(stream.interrupts)
    # > (Interrupt(value='Do you approve this action?'),)

# 3. 人类决策，resume
resumed = app.stream_events(Command(resume=True), config=config, version="v3")
```

**6 条 LangGraph interrupt 铁律**（必读）：

1. **节点重跑原则**：resume 时，节点从**开头**重跑（不是从 interrupt 之后续）
2. **不要 try/except 包 interrupt()**（会破坏信号机制）
3. **每个节点最多调用一次 interrupt()**（用 state 而不是 while 循环）
4. **interrupt 之前的所有代码会重跑**（所以不要在前面写不可重入的副作用）
5. **payload 必须 JSON-serializable**（否则无法 checkpoint）
6. **thread_id 是持久游标**（生产环境用稳定 ID，不能随机）

**反模式示例（要避免）**：
```python
# ❌ 错误：while 循环里反复调用 interrupt
def bad_node(state):
    while True:
        answer = interrupt("Question?")  # 每次 resume 都会重跑整个 while
        if is_valid(answer):
            return {"answer": answer}
        # 错误！这样会指数级重跑之前的代码
```

**正确模式：state + 条件边**：
```python
# ✓ 正确：用 state 保存上一次的 question，conditional edge 循环
def get_age_node(state: FormState):
    question = state.get("pending_question") or "What is your age?"
    answer = interrupt(question)  # 只调一次
    if isinstance(answer, int) and answer > 0:
        return {"age": answer, "pending_question": None}
    # 失败时更新 question，下次 resume 自动重问
    return {"pending_question": f"'{answer}' is not a valid age. Please enter a positive number."}

def route(state):
    return END if state.get("age") is not None else "collect_age"
```

**interrupt 在 tool 里**（推荐用于把 approval 逻辑下沉到 tool）：
```python
from langchain.tools import tool
from langgraph.types import interrupt

@tool
def send_email(to: str, subject: str, body: str):
    """Send an email to a recipient."""
    
    # Pause before sending
    response = interrupt({
        "action": "send_email",
        "to": to, "subject": subject, "body": body,
        "message": "Approve sending this email?"
    })
    
    if response.get("action") == "approve":
        return f"Email sent to {response.get('to', to)}"
    return "Email cancelled by user"
```

**优势**：approval 逻辑跟着 tool 走，可被多个 agent 复用。LLM 自然调 tool，interrupt 自动 pause。

### 4.2 OpenAI Agents SDK：`RunState` + `interruptions` + `needs_approval`

**OpenAI Agents SDK（2025）的 HITL 设计比 LangGraph 更"声明式"**——把 approval 内置在 tool 装饰器里。

```python
from agents import Agent, Runner, function_tool, RunState

# 1. 标记需要 approval 的 tool
@function_tool(needs_approval=True)
async def cancel_order(order_id: int) -> str:
    return f"Cancelled order {order_id}"

# 2. 条件式 approval（基于参数）
async def requires_review(_ctx, params, _call_id) -> bool:
    return "refund" in params.get("subject", "").lower()

@function_tool(needs_approval=requires_review)
async def send_email(subject: str, body: str) -> str:
    return f"Sent '{subject}'"

agent = Agent(
    name="Support agent",
    tools=[cancel_order, send_email],
)

# 3. 运行：approval 自动出现
result = await Runner.run(agent, "Send an email about refund")
print(result.interruptions)  # → [ToolApprovalItem(...)]

# 4. 持久化 + 跨进程 resume
state = result.to_state()  # → RunState
STATE_PATH.write_text(state.to_string())

# 5. 之后（可能跨进程、跨天）重新加载
state = await RunState.from_string(agent, STATE_PATH.read_text())

# 6. 决策：approve 或 reject
for interruption in result.interruptions:
    if user_approved:
        state.approve(interruption, always_approve=False)  # sticky decision
    else:
        state.reject(interruption, rejection_message="Risk too high")

# 7. 继续
result = await Runner.run(agent, state)
```

**OpenAI 模式 vs LangGraph 模式对比**：

| 维度 | LangGraph | OpenAI Agents SDK |
|------|-----------|-------------------|
| **触发方式** | 显式调 `interrupt()` | 装饰器声明 `needs_approval` |
| **状态管理** | Checkpointer（Postgres / Redis）| `RunState.to_string()` / `from_string()` |
| **Sticky decision** | 手动管理 | `always_approve=True` |
| **多 agent 嵌套** | 需手动 sync | `Agent.as_tool()` 自动支持 |
| **Hosted tools** | 需自己实现 | MCP / Shell / ApplyPatch 内置 |
| **Realtime / Voice** | 需自己实现 | WebSocket 内置 |

**OpenAI 模式的 4 个杀手特性**：
- **Sticky decision**：`always_approve=True` 缓存决策，run 内后续同 tool 自动通过
- **分层 approval**：run-wide `tool_error_formatter` + per-call `rejection_message` 双层定制
- **嵌套 agent approval**：`Agent.as_tool(needs_approval=...)`，approval 自动上浮到外层
- **跨进程持久化**：`RunState` JSON 序列化，可以存数据库几天后 resume

### 4.3 Microsoft AutoGen：3 种 human_input_mode

AutoGen 的 HITL 模型最简单——3 个全局模式：

```python
from autogen import ConversableAgent

agent = ConversableAgent(
    "agent",
    llm_config={"config_list": [...]},
    human_input_mode="ALWAYS",  # 或 "TERMINATE" / "NEVER"
)
```

| 模式 | 行为 | 适用 |
|------|------|------|
| `NEVER` | 不问人类，纯自主 | 完全自主 agent |
| `TERMINATE`（默认）| 满足终止条件时才问 | 大部分场景 |
| `ALWAYS` | 每次都问 | 高风险 / 学习模式 |

**用 `UserProxyAgent` 模拟人类**：
```python
user_proxy = ConversableAgent(
    "user_proxy",
    llm_config=False,  # no LLM
    human_input_mode="ALWAYS",  # always ask human
)
user_proxy.initiate_chat(agent, message="...")
```

**AutoGen 特色**：`max_consecutive_auto_reply` + `is_termination_msg` 配合，实现"自动 N 步 + 出错问人类"。

### 4.4 Claude Code：Hooks（PreToolUse 拦截）

**Claude Code 把 HITL 做成"bash hooks"**——在每个 tool 调用前自动跑一个命令：

```json
// ~/.claude/settings.json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "if echo $CLAUDE_TOOL_INPUT | grep -qE 'rm -rf|drop table'; then exit 2; fi"
      }]
    }]
  }
}
```

- `exit 0`：放行 tool
- `exit 2`：拒绝 tool（stderr 作为 reason 反馈给 Claude）
- 任何其他 code：放行但警告

**优势**：零代码、shell-based、可版本管理（hooks 在 repo 里）。**生产必加**——比如禁止 `rm -rf /`、禁止 `git push --force`、禁止 `drop table`。

### 4.5 框架对比

| 框架 | 触发方式 | 状态管理 | 学习曲线 | 适用 |
|------|----------|----------|----------|------|
| **LangGraph** | 显式 `interrupt()` | Checkpointer | 中 | 自建复杂 agent |
| **OpenAI Agents SDK** | 声明式 `needs_approval` | `RunState` | 低 | OpenAI 生态 |
| **AutoGen** | 3 个 `human_input_mode` | 自动 | 低 | 多 agent 对话 |
| **CrewAI** | `human_input=True` 参数 | 简单 | 低 | CrewAI 生态 |
| **Claude Code** | Hooks（bash 脚本）| 无 | 极低 | 终端 coding agent |

---

## 5. HITL 的设计原则

### 5.1 4 条原则

1. **不打断**：只在真正需要时打断用户（别让用户烦）
2. **可逆**：默认让 agent 做，错了能 undo
3. **可解释**：让用户知道 agent 在做什么、为什么
4. **可学习**：用户的反馈回流到 system，提升下一次

### 5.2 4 个反模式

| 反模式 | 后果 | 修复 |
|--------|------|------|
| **每步都问** | 用户烦死，agent 不可用 | 按风险分级 |
| **从不问** | 灾难性错误 | 强制 L2+ 必须有 approval |
| **问但用户不懂** | 用户瞎点 | 提供 explanation + 影响评估 |
| **反馈不回流** | 同样错误反复发生 | 把反馈写入 memory / 训练数据 |

---

## 6. UI / UX 设计

HITL 的成败 80% 在 UI 设计。

### 6.1 关键 UI 元素

- **Diff view**：让用户看到"要改什么"
- **Preview**：让用户看到"改完会怎样"
- **Rollback button**：错了能 undo
- **Explanation**：agent 为什么这样做
- **Alternatives**：3-5 个 alternative actions
- **History**：之前人类决策过什么

### 6.2 渠道选择

| 场景 | 渠道 |
|------|------|
| **低风险 + 高频** | UI 弹窗 / toast |
| **中风险** | Slack / Teams 通知 + 按钮 |
| **高风险** | Email / SMS + 二次确认 |
| **L4+ 不可逆** | 多人审批流（多渠道） |
| **离线 / 异步** | Queue + dashboard |

### 6.3 Async vs Sync HITL

- **Sync**（同步）：Agent 暂停等人类（适合 sync workflow）
- **Async**（异步）：Agent 把 action 加入队列，人类异步批处理（适合高 QPS）

详见 §8 生产级 HITL 架构。

---

## 7. 实战：完整的 LangGraph HITL Agent

```python
from typing import TypedDict, Literal, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.types import interrupt, Command
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    next_action: dict | None
    approved: bool | None

def agent_node(state: AgentState):
    """LLM 决定下一步 action"""
    response = llm.invoke(state["messages"])
    return {"messages": [response], "next_action": parse_action(response)}

def should_request_approval(state: AgentState) -> Literal["approval", "execute", "end"]:
    action = state.get("next_action")
    if not action:
        return "end"
    
    risk = classify_risk(action)  # §3 风险分级
    if risk >= 2:
        return "approval"
    return "execute"

def approval_node(state: AgentState):
    """人类审批"""
    decision = interrupt({
        "question": f"Approve this action?",
        "action": state["next_action"],
        "risk_level": classify_risk(state["next_action"]),
        "context": state["messages"][-3:],  # 最近 3 轮
    })
    
    if decision.get("action") == "approve":
        return {"approved": True}
    elif decision.get("action") == "modify":
        return {"next_action": decision["payload"], "approved": True}
    else:
        return {"approved": False}

def execute_node(state: AgentState):
    """执行 action"""
    if not state.get("approved"):
        return {"messages": [{"role": "system", "content": "User rejected the action."}]}
    result = execute_action(state["next_action"])
    return {"messages": [{"role": "tool", "content": str(result)}], "approved": None, "next_action": None}

# Build graph
builder = StateGraph(AgentState)
builder.add_node("agent", agent_node)
builder.add_node("approval", approval_node)
builder.add_node("execute", execute_node)

builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", should_request_approval, {
    "approval": "approval", "execute": "execute", "end": END
})
builder.add_edge("approval", "execute")
builder.add_edge("execute", "agent")

# 关键：checkpointer + interrupt_before
graph = builder.compile(
    checkpointer=PostgresSaver.from_conn_string(DATABASE_URL),
    interrupt_before=["approval"],  # 在 approval 节点之前 pause
)

# 运行
config = {"configurable": {"thread_id": "user-123"}}
for chunk in graph.stream({"messages": [user_input]}, config=config):
    if "__interrupt__" in chunk:
        # UI 显示 approval
        decision = ui_prompt(chunk["__interrupt__"])
        graph.update_state(config, decision)
        continue
    print(chunk)
```

---

## 8. 实战：生产级 HITL 架构

```
┌────────────┐
│   Agent    │
└─────┬──────┘
      ↓
┌────────────┐
│  Risk      │  ← 按 L0-L5 评估
│  Classifier│
└─────┬──────┘
      ↓
   ┌──┴──┐
   ↓     ↓
L0-L1  L2+
   ↓     ↓
  Auto  HITL Queue
         ↓
   ┌─────┼──────┐
   ↓     ↓      ↓
 Slack Email  Dashboard
   ↓
 Human decides
   ↓
 Back to Agent
```

### 8.1 Risk Classifier

```python
RISK_KEYWORDS = {
    "delete": 3, "drop": 4, "truncate": 5,
    "send": 2, "email": 2, "slack": 2,
    "payment": 4, "transfer": 4, "refund": 3,
    "deploy": 4, "production": 4, "release": 3,
    "file_write": 1, "api_call": 1, "shell": 2,
}

def classify_risk(action: str, target: str) -> int:
    score = 0
    for kw, weight in RISK_KEYWORDS.items():
        if kw in action.lower() or kw in target.lower():
            score = max(score, weight)
    return score
```

### 8.2 HITL Queue

```python
import redis
import json

r = redis.Redis()

def enqueue_for_review(action, payload, context, risk_level):
    if risk_level < 2:
        return execute(action, payload)
    
    task_id = str(uuid.uuid4())
    r.setex(
        f"hitl:{task_id}",
        3600,
        json.dumps({
            "action": action, "payload": payload,
            "context": context, "risk_level": risk_level,
        }),
    )
    notify_human(task_id, risk_level)  # Slack / email
    return {"status": "pending_review", "task_id": task_id}

def process_human_decision(task_id, decision):
    task = json.loads(r.get(f"hitl:{task_id}"))
    if decision == "approve":
        return execute(task["action"], task["payload"])
    elif decision == "reject":
        return {"status": "rejected"}
    elif decision.startswith("modify:"):
        new_payload = json.loads(decision[len("modify:"):])
        return execute(task["action"], new_payload)
```

---

## 9. Confidence-based Delegation（基于置信度的 HITL）

HITL 的关键挑战是**什么时候转人工**。简单规则是按 risk level 转，更智能是按 LLM 置信度转。

### 9.1 置信度估算方法

| 方法 | 原理 | 成本 |
|------|------|------|
| **Self-consistency** | 采样 N 次，多数一致 → 高置信 | 高（N × 单次 cost）|
| **Logit-based** | token 概率低 → 答案不确定 | 低（不需要再调 LLM）|
| **Verifier model** | 用小 LLM 验证答案 | 中 |
| **Entropy-based** | 答案分布的熵 | 中 |
| **Likelihood ratio** | 多个候选的概率比 | 中 |

### 9.2 Self-consistency 实战

```python
def compute_confidence(prompt, n_samples=5):
    """用 self-consistency 算置信度"""
    samples = [llm.invoke(prompt) for _ in range(n_samples)]
    
    # 归一化并去重
    from collections import Counter
    counter = Counter(samples)
    most_common, count = counter.most_common(1)[0]
    
    consistency = count / n_samples
    return most_common, consistency

def should_delegate(prompt, threshold=0.7):
    answer, confidence = compute_confidence(prompt)
    if confidence >= threshold:
        return answer, False  # 高置信，不转人工
    return None, True  # 低置信，转人工
```

### 9.3 校准（Calibration）

**LLM 输出的置信度经常过度自信**——0.9 的 logit 可能只有 0.6 的实际准确率。

校准方法：
- **Temperature scaling**：调 temperature 平滑置信度
- **Platt scaling**：用 logistic regression 把 logit 校准到准确率
- **Conformal prediction**：给出保证覆盖率的预测集

```python
# 用历史反馈数据校准
def calibrate(raw_logit, true_accuracy_history):
    """简单的 Platt scaling"""
    from sklearn.linear_model import LogisticRegression
    X = np.array([s['logit'] for s in true_accuracy_history]).reshape(-1, 1)
    y = np.array([s['correct'] for s in true_accuracy_history])
    model = LogisticRegression().fit(X, y)
    return model.predict_proba([[raw_logit]])[0, 1]
```

---

## 10. 多 Reviewer 协议 / Approval Quorum

**单 reviewer HITL 风险**：reviewer 犯错 / 疲劳 / 主观偏差。**生产高风险场景需要 quorum（多人审批）**。

### 10.1 Quorum 模式

| 模式 | 协议 | 适用 |
|------|------|------|
| **Unanimous（一致同意）** | 全部 reviewer approve | 极高风险（医疗 / 金融） |
| **Majority（多数同意）** | > 50% approve | 高风险（生产部署） |
| **Threshold（阈值）** | N / M approve | 一般高风险 |
| **Weighted（加权）** | 不同 reviewer 不同权重 | 大型组织 |
| **Hierarchical（分层）** | 先低层、再高层 | 监管要求 |

### 10.2 实现

```python
class QuorumHITL:
    def __init__(self, required_approvals: int, total_reviewers: int):
        self.required = required_approvals
        self.total = total_reviewers
        self.votes = {}  # reviewer_id -> decision
    
    def record_vote(self, reviewer_id: str, decision: str):
        self.votes[reviewer_id] = decision
        approvals = sum(1 for v in self.votes.values() if v == "approve")
        rejections = sum(1 for v in self.votes.values() if v == "reject")
        
        if approvals >= self.required:
            return "approved"
        if rejections > self.total - self.required:
            return "rejected"
        if len(self.votes) >= self.total:
            return "rejected"  # 时间到，没凑够 approval
        return "pending"
```

### 10.3 Reviewer 角色分离

- **Maker（生成）**：LLM agent 提议 action
- **Checker（验证）**：另一个 LLM / 简单规则检查
- **Approver（批准）**：人类 final decision

```
Maker (LLM) → Checker (rules/small LLM) → Approver (Human)
```

---

## 11. HITL × RLHF / Active Learning

**HITL 的核心价值不只是"安全"，还是"持续学习的数据源"**。

### 11.1 三层 learning 闭环

```
TURN N:
  Agent 提议 → 人类 approve/reject/modify
       ↓
  写入 log + training set
       ↓
TURN N+1:
  微调 model（DPO / SFT）
       ↓
TURN N+M:
  Agent 表现更好
```

### 11.2 Active Learning from HITL

不只是"写 memory"，是**主动选择最有信息量的样本让人来标**：

```python
def select_samples_for_hitl_labeling(agent_outputs, n=10):
    """主动学习：选最不确定的样本让人标"""
    uncertainties = [compute_uncertainty(o) for o in agent_outputs]
    
    # Top-N 最不确定的
    indices = np.argsort(uncertainties)[-n:]
    return [agent_outputs[i] for i in indices]
```

### 11.3 HITL → DPO 数据

```python
# 把 HITL 决策直接转成 DPO 偏好数据
def hitl_to_dpo(log: list[HumanDecision]):
    pairs = []
    for decision in log:
        if decision.choice == "modify":
            # 修正后的（chosen）vs agent 原版（rejected）
            pairs.append({
                "prompt": decision.context["prompt"],
                "chosen": decision.corrected_payload,
                "rejected": decision.original_payload,
            })
    return pairs

# 定期 retrain agent
def retrain_agent():
    pairs = hitl_to_dpo(load_human_decisions())
    dataset = Dataset.from_list(pairs)
    trainer = DPOTrainer(model=agent, args=DPOConfig(...))
    trainer.train()
```

**关键**：HITL 决策是**最真实的偏好数据**——比 GPT-4 蒸馏更可靠，因为是人类在真实场景下做的决策。

---

## 12. HITL for Long-Running Agent（Loop Engineering 视角）

**Long-running agent 跨 session 持续工作**（详见 U21 Loop Engineering）。HITL 在这种场景下有特殊挑战。

### 12.1 Async HITL with Loop Engineering

```python
# LoopX / Loom 风格的 HITL
class LoopEngineHITL:
    def __init__(self):
        self.queue = HITLQueue()
        self.state = LoopState()  # 见 U21
    
    def propose_action(self, action):
        risk = classify_risk(action)
        if risk >= 2:
            task_id = self.queue.enqueue(action)
            return {"status": "pending_human", "task_id": task_id}
        else:
            return execute(action)
    
    def check_pending_decisions(self):
        """Loop 每个 tick 检查人类决策"""
        for task_id in self.queue.pending():
            if decision := self.queue.get_decision(task_id):
                self.handle_decision(task_id, decision)
```

### 12.2 跨 session 决策持久化

关键：HITL 决策必须持久化到 LoopState，否则 agent 重启会丢失决策历史。

```python
# 写到 LoopState（不是 agent state）
loop_state.record_hitl_decision(
    task_id=task_id,
    decision=decision,
    context=context,
    timestamp=datetime.now(),
)
# 下次 loop 启动会从 LoopState 加载
```

### 12.3 Background HITL Review

异步 HITL（§6.3）天然适合 loop engineering——agent 继续工作，人类后台 review：

```
Loop tick 1: 提议 5 个 actions → 4 个自动执行，1 个入 HITL 队列
Loop tick 2: agent 继续工作，人类异步 review 队列里的 1 个
Loop tick 3: 人类决策到了，agent 接到信号，调整后续 action
```

---

## 13. 反馈学习（Feedback Loop）

HITL 不仅是"安全机制"，还是**学习信号**。

### 13.1 收集人类反馈

```python
@dataclass
class HumanFeedback:
    task_id: str
    decision: Literal["approve", "reject", "modify"]
    feedback: str  # 人类评论
    corrected_payload: dict | None
    timestamp: datetime
```

### 13.2 写入 Memory / 训练集

```python
def save_feedback(feedback: HumanFeedback):
    # 1. 写入长期 memory（让 agent 未来不再犯同样错）
    memory.save({
        "type": "human_feedback",
        "context": feedback.context,
        "decision": feedback.decision,
        "correction": feedback.corrected_payload,
    })
    
    # 2. 写入 SFT/DPO 训练集
    if feedback.decision in ("reject", "modify"):
        training_data.append({
            "prompt": feedback.context["prompt"],
            "rejected": feedback.context["original_payload"],
            "chosen": feedback.corrected_payload or default_payload,
        })
```

---

## 14. Interrupt 规则（LangGraph 6 大铁律 + 实战避坑）

| 规则 | 说明 | 错误示例 | 正确示例 |
|------|------|----------|----------|
| **节点重跑** | resume 时节点从开头跑 | 假设 `interrupt` 之后接着跑 | 把副作用放在 interrupt 之后 |
| **不要 try/except** | try/except 会吞掉 interrupt 信号 | `try: interrupt() except: ...` | 正常 call，不 try |
| **每节点一次** | 节点只能调一次 `interrupt()` | `while True: interrupt()` | state + conditional edge |
| **可重入** | interrupt 之前代码会重跑 | 写 `if redis.setnx(): interrupt()` | 写幂等逻辑 |
| **JSON-serializable** | payload 必须能 JSON 序列化 | 传 `datetime` 对象 | 传 `datetime.isoformat()` 字符串 |
| **thread_id 稳定** | 生产用稳定 ID | `thread_id = uuid4()` | 用用户 ID + session ID |

---

## 15. 面试视角

- **Q：HITL 4 大模式？5 级风险分级？**
  → Approval / Interrupt / Oversight / Confidence-based。L0 查天气（无 HITL）→ L1 写文件（Oversight）→ L2 改 DB（Approval）→ L3 支付（Approval + 二次确认）→ L4 部署（Multi-party）→ L5 法律医疗（Mandatory review）。

- **Q：LangGraph 的 interrupt 怎么用？6 大铁律？**
  → 显式调 `interrupt(payload)`，然后用 `Command(resume=...)` resume。铁律：① 节点重跑 ② 不要 try/except ③ 每节点一次 ④ 可重入 ⑤ JSON-serializable ⑥ thread_id 稳定。

- **Q：LangGraph vs OpenAI Agents SDK 的 HITL 区别？**
  → LangGraph：显式 `interrupt()`，需自己管 checkpointer。OpenAI：声明式 `needs_approval`，内置 `RunState` 持久化 + sticky decision + 嵌套 agent approval。OpenAI 更"声明式"，LangGraph 更"灵活"。

- **Q：HITL 怎么测？**
  → 三层：① 单元：手动 mock 人类输入 ② 集成：跑真实 case 记录人类决策时间 ③ 指标：review latency（决策多久了）/ override rate（人类 override agent 比例）/ error rate（错误决策率）。

- **Q：HITL 怎么改进 agent？**
  → 三层 learning 闭环：① 短期：写 memory（避免同样错）② 中期：写 SFT/DPO 训练集（retrain）③ 长期：continuous learning pipeline。HITL 决策是**最真实的偏好数据**。

- **Q：Quorum 协议？**
  → 多人审批：unanimous（一致同意）/ majority（多数）/ threshold（N/M）/ weighted（加权）/ hierarchical（分层）。L4+ 必加。

- **Q：HITL 的反模式？**
  → ① 每步都问（用户烦）② 从来不问（灾难）③ 问但用户不懂（瞎点）④ 反馈不回流（同样错反复）。修复：按风险分级 + 解释 + 影响 + 反馈写入 memory。

- **Q：HITL 怎么和 Confidence-based 配合？**
  → 三段式：先 compute confidence（self-consistency / logit-based）→ 风险分级 → 决策：confidence 高 + 风险低 → 自动；confidence 低 + 风险高 → 必须人工；中间情况 → 异步 HITL 队列。

- **Q：OpenAI Agents SDK 的 needs_approval 怎么用？**
  → `@function_tool(needs_approval=True)` 标记需 approval 的 tool；可用 callable 动态判断（参数相关）。运行后 `result.interruptions` 包含 `ToolApprovalItem` 列表，`state.approve(interruption)` / `state.reject(interruption)` 决策，resume run。`always_approve=True` 缓存 sticky decision。

- **Q：HITL 和 U21 Loop Engineering 怎么配合？**
  → Loop Engineering 的 agent 跨 session 工作，HITL 决策必须**持久化到 LoopState**（不是 agent state）。Loop 每个 tick 检查 HITL 队列，async 决策不阻塞 loop 运转。

---

## 16. 必读论文（10+ 篇）

> 按"理论 → 工程 → 应用"分层整理。每篇 1-2 句说明为什么读。

### 16.1 经典 / 基础

| 论文 | 年份 | 为什么要读 |
|------|------|-----------|
| [Agent-Agnostic Human-in-the-Loop Reinforcement Learning](https://arxiv.org/abs/1701.04079) | 2017 | RL + HITL 的开山论文，定义 teacher advice 协议 |
| [Reinforcement Learning with Human Feedback (InstructGPT)](https://arxiv.org/abs/2203.02155) | 2022 | RLHF 经典，HITL 偏好数据训练的奠基 |

### 16.2 HITL for Agents（2024-2025 工业级）

| 论文 | 年份 | 为什么要读 |
|------|------|-----------|
| [Human-In-the-Loop Software Development Agents](https://arxiv.org/abs/2411.12924) | 2024 | 多 agent 软件开发中 HITL 模式（Atlassian 实战） |
| [Human-In-The-Loop Software Development Agents: Challenges and Future Directions](https://arxiv.org/abs/2506.11009) | 2025 | 同上作者团队，challenges + future directions（必读） |
| [Enabling Self-Improving Agents to Learn at Test Time With Human-In-The-Loop Guidance](https://arxiv.org/abs/2507.17131) | 2025 | Test-time learning，HITL 在 agent 持续改进中的角色 |
| [CentaurEval: Benchmarking Human-in-the-Loop Value in Agentic Coding](https://arxiv.org/abs/2512.04111) | 2025 | 怎么测 HITL 的"价值"（不是"有没有"） |
| [Handoff Debt: The Rediscovery Cost When Coding Agents Take Over Interrupted Tasks](https://arxiv.org/abs/2606.02875) | 2025 | Agent 接手被中断任务的"rediscovery cost"，HITL 设计参考 |
| [When Users Change Their Mind: Evaluating Interruptible Agents in Long-Horizon Web Navigation](https://arxiv.org/abs/2604.00892) | 2025 | 用户中途改主意的场景，long-horizon HITL |
| [IHBench: Evaluating Post-Interruption Recovery in Voice Agents](https://arxiv.org/abs/2606.19595) | 2025 | Voice agent 中断后恢复的评测 |
| [Optimizing LLMs for Dynamic Constraints through Human-in-the-Loop Discriminators](https://arxiv.org/abs/2410.15163) | 2024 | 用 HITL discriminator 满足动态约束 |
| [Governance by Construction for Generalist Agents](https://arxiv.org/abs/2605.20874) | 2025 | Generalist agent 的 governance（HITL 是核心机制）|
| [FinHarness: An Inline Lifecycle Safety Harness for Finance LLM Agents](https://arxiv.org/abs/2605.27333) | 2025 | 金融 agent 的 lifecycle safety harness（HITL + 风险分级） |
| [Zero-Shot LLMs in Human-in-the-Loop RL: Replacing Human Feedback for Reward Shaping](https://arxiv.org/abs/2503.22723) | 2025 | 用 LLM 替代部分人类反馈，HITL 的 cost reduction |
| [Continual Harness: Online Adaptation for Self-Improving Foundation Agents](https://arxiv.org/abs/2605.09998) | 2025 | Online adaptation 的 harness，HITL 在其中 |

### 16.3 HITL + RLHF / Training

| 论文 | 年份 | 为什么要读 |
|------|------|-----------|
| [DPO 论文](https://arxiv.org/abs/2305.18290) | 2023 | HITL 偏好数据的现代用法 |
| [DeepSeek R1](https://arxiv.org/abs/2501.12948) | 2025 | GRPO + RLVR 用 verifiable reward 替代部分人类标注 |
| [Anthropic Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 2025 | 实战 context engineering，HITL 是核心 |

### 16.4 监管 / 合规

| 文档 | 年份 | 为什么要读 |
|------|------|-----------|
| [EU AI Act Article 14 (Human Oversight)](https://artificialintelligenceact.eu/article/14/) | 2024 | 欧盟 AI 法案对 human oversight 的法律要求 |
| [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | 2025 | 行业安全标准 |
| [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) | 2023 | 美国国家标准 |

---

## 17. 必读教程（4 大框架官方）

| 教程 | 链接 | 为什么读 |
|------|------|----------|
| **LangGraph: Interrupts** | [docs.langchain.com/oss/python/langgraph/interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) | **最权威**，canonical interrupt() + Command(resume) 模式，含 5 大常见 pattern + 6 大铁律 |
| **OpenAI Agents SDK: Human-in-the-loop** | [openai.github.io/openai-agents-python/human_in_the_loop](https://openai.github.io/openai-agents-python/human_in_the_loop/) | 声明式 needs_approval + RunState 持久化 |
| **Microsoft AutoGen: Human Feedback in Agents** | [microsoft.github.io/autogen/0.2/docs/tutorial/human-in-the-loop](https://microsoft.github.io/autogen/0.2/docs/tutorial/human-in-the-loop) | 3 种 human_input_mode 详解 |
| **Anthropic: Claude Code Hooks** | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks) | PreToolUse 拦截 + 工业级 HITL 实践 |
| **Cobus Greyling: HITL Patterns** | [cobusgreyling.substack.com](https://cobusgreyling.substack.com) | 行业布道者，HITL + Loop Engineering 视角 |

---

## 18. 练习

1. **LangGraph Interrupt**：用 `interrupt_before` 实现一个需要人类确认的 agent
2. **OpenAI needs_approval**：写一个 `cancel_order` tool 用 `needs_approval=True`，用 `RunState` 持久化到 disk，跨进程 resume
3. **AutoGen 3 modes**：分别用 NEVER / TERMINATE / ALWAYS 跑同一个对话，对比人类介入次数
4. **Claude Code Hooks**：写一个 PreToolUse hook，拦截 `rm -rf` 和 `git push --force`
5. **Risk Classifier**：写一个函数，检测 action 的 risk level（关键词 + LLM 双层）
6. **HITL Queue**：用 Redis 实现一个 async HITL queue，支持多 reviewer + quorum
7. **Confidence-based**：用 self-consistency 实现 LLM 置信度估算
8. **Feedback Loop**：实现"人类决策 → 写入 memory + 训练集 → 后续不再犯"的 pipeline
9. **HITL + DPO**：把 HITL 决策转成 DPO 训练数据，跑一次 retrain
10. **Cross-session HITL**：用 LoopX 风格的 LoopState 持久化 HITL 决策，跨 session 恢复

---

## 19. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U9 Multi-Agent | HITL 是 Agent 协作中的"人类是 special agent"模式 |
| U18 Agent 安全与红队 | HITL 是防御深度的一部分 |
| U21 Loop Engineering | Long-running agent 跨 session 工作，HITL 必须持久化到 LoopState |
| U22 Context Engineering | HITL 决策可写入 context 避免重复问 |
| U23 Harness Engineering | HITL 是 Harness 组件 11（Intervention Recording）的具体实现 |
| U24 Reasoning Model | Reasoning model + HITL：reasoning_effort 高 → confidence 高 → 可能不需 HITL |
| U30 Post-training | HITL 决策 → SFT/DPO/RLHF 训练数据 |
| U31 Structured Output | HITL 通常需要结构化的"action 提议"才能 review |
| U34 Guardrails | HITL 在 Guardrails 之上，提供人工兜底 |
| U35 AgentOps | HITL latency / override rate 是关键 metric |

---

*最后更新：2026-06-30 深度增强版（v2.0）*
*内容来源：[LangGraph 官方 Interrupts 文档](https://docs.langchain.com/oss/python/langgraph/interrupts) · [OpenAI Agents SDK HITL](https://openai.github.io/openai-agents-python/human_in_the_loop/) · [Microsoft AutoGen HITL](https://microsoft.github.io/autogen/0.2/docs/tutorial/human-in-the-loop) · Anthropic Claude Code Hooks · 10+ arxiv 论文*
