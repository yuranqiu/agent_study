# Interview QA · Agent 部署与运维（Deploy & AgentOps）

> 来源：U29 Agent Deploy + U35 AgentOps
> 覆盖：4层架构 / Sandbox / Session 持久化 / Gateway / CI-CD / 可观测性 / Eval-driven 开发

---

## 一、部署架构篇

### Q: Agent 的 4 层部署架构是什么？每层职责是什么？

**Answer:**

```
┌────────────────────────────────────────────────────┐
│ L1: 用户接入层 (User Access)                         │
│   Web UI / IDE 插件 / CLI / API / IM 机器人        │
├────────────────────────────────────────────────────┤
│ L2: API Gateway (接入层)                             │
│   认证 / 限流 / 审计 / 路由 / 协议转换               │
├────────────────────────────────────────────────────┤
│ L3: Agent Orchestrator (编排层)                       │
│   LangGraph / CrewAI / 自定义 Harness                │
│   - Session 管理                                     │
│   - Memory 持久化                                    │
│   - State checkpoint                                │
├────────────────────────────────────────────────────┤
│ L4: Execution Sandbox (执行沙箱)                      │
│   Docker / gVisor / Firecracker                      │
│   - File / Shell / Network 隔离                       │
│   - Resource limits (CPU / Memory / Disk / Time)     │
└────────────────────────────────────────────────────┘
```

| 层级 | 技术栈 | 关键职责 |
|------|--------|----------|
| L1 用户接入 | Web / CLI / API | 协议适配、用户交互 |
| L2 Gateway | FastAPI / Kong / Helicone | 认证、限流、审计、路由 |
| L3 Orchestrator | LangGraph / CrewAI | 会话管理、状态持久化、工具调度 |
| L4 Sandbox | Docker / Firecracker | 隔离执行、资源限制、审计日志 |

**关键洞察**：L1-L3 是应用层（Python），L4 是 OS-level 隔离（基础设施层）。Agent 副作用（文件/网络/命令）全部在 L4 发生。

**面试追问：**
- 为什么 L3 和 L4 必须分离？→ Orchestrator 管理业务逻辑，Sandbox 负责安全隔离；混在一起会导致权限难以审计。
- 如果把 Sandbox 去掉，最大风险是什么？→ Agent 可以直接读写宿主机文件系统、访问内网、消耗无限资源。

---

### Q: 部署模式有哪些？什么时候选自建 vs 平台？

**Answer:**

| 模式 | 适合场景 | 代表 |
|------|----------|------|
| **自建** | 大规模、定制化、合规要求高 | 字节跳动、Anthropic |
| **LangGraph Platform** | LangChain 生态 | LangChain |
| **CrewAI Enterprise** | CrewAI 生态 | CrewAI |
| **Dify / Coze** | 可视化、低代码 | 字节跳动 / 字节 Coze |
| **OpenAI AgentKit** | 闭源托管 | OpenAI |
| **Firecracker / Lambda** | Serverless、高并发 | AWS Lambda |

**选型决策树**：

```
需要深度定制 / 内部合规？
├─ 是 → 自建（LangGraph + FastAPI + 自建 Sandbox）
└─ 否 → 快速上线？
    ├─ 是 → LangGraph Platform / Dify
    └─ 否 → 考虑 CrewAI Enterprise / OpenAI AgentKit
```

**面试追问：**
- 自建 Agent 平台和部署一个 Web 服务最大的区别？→ Agent 是有副作用的 LLM 循环，需要 sandbox + state 持久化 + eval pipeline，Web 服务是无状态 CRUD。

---

## 二、Sandbox 篇

### Q: 什么是 Sandbox？Agent 为什么必须沙箱？

**Answer:**

Agent 不是纯函数——它有副作用：写文件、执行命令、访问数据库、调用外部 API。Sandbox 就是 Agent 的"安全屋"。

**核心属性**：

| 属性 | 作用 | 实现方式 |
|------|------|----------|
| File 隔离 | 限制读写范围 | Docker volume / chroot |
| Network 隔离 | 限制可访问网络 | Docker network / iptables |
| Resource limits | 防资源耗尽 | cgroups / ulimit |
| Syscall 过滤 | 拦截危险调用 | seccomp / AppArmor |
| 可重置 | 每次运行干净状态 | Docker image / snapshot |
| 可观测 | 记录所有操作 | audit log / trace |

**面试追问：**
- 如果 Agent 只是"回答问题"不执行任何操作，还需要 Sandbox 吗？→ 不需要。Sandbox 为有副作用操作设计；纯问答 Agent 可以用无沙箱部署（但建议至少有限流）。

---

### Q: 常见 Sandbox 实现方案对比？

**Answer:**

隔离强度从左到右递增：

```
最强隔离 ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ → 最弱隔离
─────────────────────────────────────────────────
完整 VM  | Firecracker |  gVisor  |  Docker  |  Process
─────────────────────────────────────────────────
启动慢，安全最高                         启动快，安全低
```

| 方案 | 隔离强度 | 启动时间 | 适用场景 |
|------|---------|----------|----------|
| **Process** | 弱 | 0s | 不可信代码禁止 |
| **Docker** | 中 | 1-2s | 一般 Coding Agent |
| **gVisor** | 强 | 1-2s | 多租户 / 半可信环境 |
| **Firecracker** | 强 | <125ms | Serverless / 高 QPS |
| **完整 VM** | 最强 | 秒-分钟 | 金融/医疗等高合规 |

**选型建议**：
- Coding Agent（如 OpenHands）→ Docker 默认
- Computer Use Agent → 完整 VM 或 Firecracker
- 高并发 API Agent → Firecracker microVM
- 多租户 SaaS → gVisor 或 Docker + 强化 seccomp

**面试追问：**
- Docker 和 Firecracker 的核心安全区别？→ Docker 共享内核，有容器逃逸风险；Firecracker 是 microVM，每个实例有独立内核，隔离更强。
- gVisor 的 sentry 和 gofer 是什么？→ senty 在用户态拦截 syscall，gofer 处理文件系统访问，双重隔离。

---

### Q: 用 Docker 实现 Agent 沙箱的代码示例？

**Answer:**

```dockerfile
# agent-sandbox/Dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    python3.11 python3-pip git curl \
    && rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash agent
USER agent
WORKDIR /home/agent
# 限制只能访问 /home/agent 目录
# 网络：白名单域名（运行时通过 Docker network 控制）
```

```python
# 用 Docker SDK 启动 sandbox
import docker

client = docker.from_env()

def run_in_sandbox(code: str, timeout: int = 30):
    """在沙箱内执行 Python 代码"""
    container = client.containers.run(
        "agent-sandbox:latest",
        command=f"python -c '{code}'",
        volumes={"/host/project": {"bind": "/home/agent/project", "mode": "rw"}},
        network_mode="bridge",          # 受限网络
        mem_limit="512m",               # 内存上限
        cpu_quota=50000,                # 50% CPU
        detach=True,
        remove=True,                    # 运行完自动销毁
    )
    try:
        result = container.wait(timeout=timeout)
        logs = container.logs().decode()
        return {"status": result["StatusCode"], "logs": logs}
    except Exception as e:
        container.kill()
        raise
```

**关键安全配置**：
- `--read-only`：根文件系统只读
- `--network=none` 或 `--network=bridge`：限制网络
- `--cap-drop=ALL`：去掉所有 Linux capabilities
- `--security-opt=no-new-privileges`：禁止提权

**面试追问：**
- Docker `remove=True` 为什么重要？→ 防止残留文件泄露到下个任务；每次都是干净环境。
- 如何防止容器内进程逃逸到宿主机？→ 用 `--security-opt seccomp=...` 限制 syscall；gVisor/Firecracker 更安全；定期更新 Docker 版本修补 CVE。

---

### Q: Computer Use Agent 需要什么级别的隔离？

**Answer:**

Computer Use Agent（GUI Agent）能"操作任何程序"，需要 **完整 OS 隔离**：

| 方案 | 厂商 | 隔离方式 |
|------|------|----------|
| **Anthropic Computer Use** | Anthropic | macOS Sandbox / Docker / VM |
| **OpenAI Operator** | OpenAI | 浏览器沙箱 + 远程 VM |
| **browser-use** | 开源 | Playwright headless + Docker |
| **ByteDance UI-TARS** | 字节跳动 | Docker + 远程 VM |

**典型架构**：
```
用户 → API Gateway → Orchestrator → 远程 VM（snapshot 启动）
                                           ↓
                                    Computer Use Agent
                                    (在 VM 内操作 GUI)
                                           ↓
                                    完成后销毁 VM
```

**关键点**：
- 每次任务启动一个**快照 VM**，任务结束销毁
- VM 内无持久化数据（或只保留必要截图/日志）
- 网络可选：完全隔离（只允许浏览器访问白名单）或完整互联网

**面试追问：**
- 为什么不直接用 Docker 跑 GUI Agent？→ Docker 共享内核，GUI 环境（如浏览器）可能有 side-channel 攻击；完整 VM 内核隔离更安全。
- Firecracker 适合 Computer Use 吗？→ 适合！AWS Lambda 用 Firecracker，启动 <125ms，隔离强，成本低。

---

## 三、Session / State 持久化篇

### Q: Agent Session 怎么持久化？Postgres + Redis 怎么配合？

**Answer:**

Agent 是有状态的——跨多个 turn 持续，跨 session 恢复。生产部署需要持久化：

```sql
-- sessions 表：存储完整 agent state
CREATE TABLE sessions (
    id UUID PRIMARY KEY,
    user_id TEXT NOT NULL,
    state JSONB NOT NULL,           -- 整个 agent state（LangGraph 的 State）
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ,
    checkpoint_index INT
);

-- messages 表：对话历史
CREATE TABLE messages (
    id UUID PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    role TEXT,                      -- user / assistant / tool
    content JSONB,
    tool_call JSONB,
    created_at TIMESTAMPTZ
);

-- checkpoints 表：状态快照
CREATE TABLE checkpoints (
    id UUID PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    step_index INT,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ
);
```

**Postgres + Redis 分工**：
- **Postgres**：持久化 conversation history + tool calls + final state
- **Redis**：活跃 session 的 fast access（带 TTL），热数据缓存

```python
# 读写策略
def get_session(session_id):
    # 先查 Redis 热缓存
    cached = redis.get(f"session:{session_id}")
    if cached:
        return json.loads(cached)
    # 缓存未命中，查 Postgres
    row = postgres.query("SELECT state FROM sessions WHERE id = %s", session_id)
    # 回写 Redis（TTL 1 小时）
    redis.setex(f"session:{session_id}", 3600, row["state"])
    return row["state"]
```

**面试追问：**
- 为什么不只用 Postgres？→ 每次读都查 DB 太慢；Redis 做热缓存降低延迟。
- 为什么不只用 Redis？→ Redis 是内存数据库，成本高，不适合存全量历史；Postgres 持久化 + 便宜。
- LangGraph 怎么持久化？→ LangGraph 自带 `PostgresSaver` / `RedisSaver` checkpointer，直接插入/恢复 state。

---

### Q: 什么是 Checkpoint & Resume？为什么重要？

**Answer:**

Checkpoint = Agent 执行过程中的状态快照。Resume = 从快照恢复继续执行。

```python
def save_checkpoint(session_id: str, state: dict, step_index: int):
    """保存检查点"""
    postgres.execute(
        "INSERT INTO checkpoints (session_id, state, step_index) "
        "VALUES (%s, %s, %s)",
        (session_id, json.dumps(state), step_index)
    )

def resume_from_checkpoint(session_id: str, target_step: int):
    """从指定步骤恢复"""
    row = postgres.execute(
        "SELECT state FROM checkpoints "
        "WHERE session_id = %s AND step_index <= %s "
        "ORDER BY step_index DESC LIMIT 1",
        (session_id, target_step)
    ).fetchone()
    return json.loads(row["state"])
```

**为什么重要**：
- Agent 执行可能很长（10+ 步 tool call），中间可能超时/失败
- 没有 checkpoint = 失败后从头开始 = 浪费 token + 时间
- 支持"断点续跑"是生产 Agent 的基本要求

**LangGraph 实现**：
```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
app = graph.compile(checkpointer=checkpointer)

# 自动保存每个 step 的 checkpoint
config = {"configurable": {"thread_id": "user-123"}}
result = app.invoke(inputs, config)
# 下次用同一个 thread_id 调用会自动 resume
```

**面试追问：**
- Checkpoint 存在哪里？→ 可以存在 Postgres（持久化）、Redis（快但易失）、S3（冷备份）。
- Checkpoint 粒度？→ 每个 step 保存一次（LangGraph 默认），或只在关键决策点保存（节省空间）。

---

### Q: Agent 的 Memory 持久化栈是什么？

**Answer:**

| 层级 | 存储 | 内容 | TTL |
|------|------|------|-----|
| **短期** | Redis | 最近 N 轮对话 | 会话级 |
| **长期** | Postgres + VectorDB | 用户偏好、关键决策 | 永久 |
| **项目记忆** | 文件系统 (AGENTS.md) | 项目知识、规范 | 永久 |

**生产级 Memory 栈**：
- **短期（对话内）**：Redis sorted set，按时间戳排序，保留最近 20 轮
- **长期（跨会话）**：Postgres 存原始数据，VectorDB（如 Qdrant/Pinecone）存 embedding，支持语义检索
- **持久项目记忆**：AGENTS.md / NOTES.md（U26），Git 版本控制

**面试追问：**
- VectorDB 和 Redis 各解决什么问题？→ Redis 解决"最近说了什么"（时序），VectorDB 解决"以前说过什么相关的"（语义）。

---

## 四、Gateway 篇

### Q: Agent API Gateway 的核心职责是什么？

**Answer:**

| 职责 | 说明 | 实现方式 |
|------|------|----------|
| **认证** | 验证调用者身份 | JWT / API Key / OAuth2 |
| **授权** | 多租户隔离 / RBAC | 用户级 / Team 级 / Resource 级 |
| **限流** | 防止 abuse / 成本控制 | Per-user / Per-IP / Token 预算 |
| **审计** | 完整请求记录 | Request/Response log + trace |
| **路由** | 版本管理 / 灰度发布 | Header 路由 / Canary |
| **可观测** | 统一埋点 | Metrics + Trace + Logs |
| **协议转换** | HTTP / gRPC / WebSocket | 中间件 |

**和传统 Gateway 的区别**：
- 传统 Gateway：路由 + 限流 + 认证
- LLM Gateway：额外要考虑 **token 预算**（按 token 数限流，不是按请求数）、**prompt cache 策略**、**cost tracking**

**面试追问：**
- 为什么 LLM 应用需要 token 级限流而不是请求级？→ 一次请求可能消耗 100K token（长 context + reasoning model），按请求数限流无法防止成本突增。

---

### Q: 自建 Gateway vs 用现成方案怎么选？

**Answer:**

| 方案 | 优势 | 劣势 | 适合场景 |
|------|------|------|----------|
| **FastAPI + 中间件** | 完全灵活 | 需自己实现所有功能 | 自建、定制化高 |
| **Kong / Traefik** | 成熟、插件丰富 | 对 LLM 特性支持弱 | 标准 API 管理 |
| **LangServe** | LangChain 原生 | 生态绑定 | LangChain 项目 |
| **Helicone** | LLM 专用、自动监控 | SaaS 或自托管成本 | 快速接入、统一监控 |
| **OpenRouter** | 模型路由 + 缓存 | 第三方依赖 | 多模型 fallback |

**推荐组合**：
- 快速启动：**Helicone**（LLM 专用 gateway，自动 caching + rate limit + cost tracking）
- 自建：**FastAPI + 自研中间件** + **Langfuse**（可观测）

**面试追问：**
- LangServe 和普通 FastAPI 路由的区别？→ LangServe 是 LangChain 生态的部署工具，自动处理 streaming、batch、schema 转换；FastAPI 更灵活但需自己处理。

---

### Q: 用 Helicone 做 LLM Gateway 的代码示例？

**Answer:**

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://oai.helicone.ai/v1",
    default_headers={
        "Helicone-Auth": f"Bearer {HELICONE_API_KEY}",
        "Helicone-User-Id": "user-123",      # 按用户跟踪
        "Helicone-Session-Id": "session-456",  # 按会话跟踪
    }
)

# 自动记录所有请求 + cost tracking
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}]
)
```

Helicone 自动提供：
- 每次请求的 token 用量（input / output / cache hit）
- 延迟分布（P50 / P95 / P99）
- 按 user / session / model 的 cost 聚合
- Prompt cache 命中率

**面试追问：**
- Helicone 和 Langfuse 的区别？→ Helicone 侧重 gateway（路由 + 缓存 + 限流），Langfuse 侧重可观测性（trace + eval + 监控）。两者常配合使用。

---

## 五、CI/CD for Agent 篇

### Q: Agent 的 CI/CD 和传统软件 CI/CD 有什么不同？

**Answer:**

| 维度 | 传统软件 | Agent |
|------|----------|-------|
| **输出** | 确定性（相同输入 → 相同输出） | 概率性（相同输入 → 不同输出） |
| **测试** | Unit / Integration / E2E | Unit + **Prompt Eval** + Tool Mock + Eval |
| **回归检测** | 断言精确匹配 | 语义匹配 + LLM-as-judge |
| **部署单元** | 代码版本 | Prompt + Model + Code 三版本 |
| **监控** | Error rate / Latency | Error rate + **Quality score** + Cost |

**核心差异**：LLM 输出是概率性的 → 传统"断言精确匹配"不够用 → 需要 **eval-driven** 的方式。

**面试追问：**
- 传统 assert A == B 为什么对 LLM 不够？→ 同样问"法国的首都是什么"，模型可能回答"巴黎"或"Paris"，都正确但字符串不同。

---

### Q: 什么是 Prompt Regression Test？怎么实现？

**Answer:**

Prompt Regression Test = 确保 prompt / model 改动后，输出质量不下降。

```python
# 用 Promptfoo 跑 prompt 回归
def test_summarization_prompt():
    cases = load_golden_cases()  # 100 个样例
    for case in cases:
        result = agent.invoke(case.input)
        
        # 1. 语义匹配（不是精确匹配）
        assert semantic_match(result, case.expected_output)
        
        # 2. 成本约束
        assert cost_within_budget(result, max_tokens=500)
        
        # 3. 延迟约束
        assert latency_under(result, max_seconds=3)
```

```yaml
# promptfooconfig.yaml
prompts:
  - "你是一个 helpful 助手。\n问题：{{question}}\n回答："
  - |
    你是 helpful 助手。请用 Markdown 格式回答。
    
    问题：{{question}}
    
    回答：

providers:
  - openai:gpt-4o
  - anthropic:claude-3-5-sonnet-20241022

tests:
  - vars:
      question: "什么是 RAG？"
    assert:
      - type: contains
        value: "检索"
      - type: llm-rubric
        value: "回答应该解释 RAG = Retrieval + Augmentation + Generation"
```

**面试追问：**
- Golden set 多少条够？→ 50-500 条，覆盖核心场景；生产级通常 200+ 条，分基础 / 边界 / 失败 case。
- Semantic match 怎么实现？→ 用 embedding 相似度（cosine > 0.9）或 LLM-as-judge。

---

### Q: Tool Mock 怎么实现？为什么需要？

**Answer:**

Tool Mock = 在测试中模拟外部工具调用，避免依赖真实外部服务。

```python
import pytest
from unittest import mock

@pytest.fixture
def mock_search_tool():
    with mock.patch("tools.search", return_value=mock_search_results):
        yield

def test_agent_uses_search(mock_search_tool):
    result = agent.run("search for AI agent trends")
    
    # 1. 验证确实调用了 search
    assert "search" in result.tool_calls
    
    # 2. 验证结果正确使用
    assert "AI agent" in result.output.lower()
```

**为什么需要**：
- 外部服务可能不稳定 / 收费 / 慢
- 需要测试边界情况（如搜索返回空结果、超时）
- 测试可重复、可离线运行

**面试追问：**
- Mock 和 Integration Test 的区别？→ Mock 隔离外部依赖，速度快；Integration Test 用真实服务，验证端到端。两者互补。

---

### Q: Agent 的完整 CI/CD Pipeline 是什么样的？

**Answer:**

```
PR → Unit test → Prompt eval → Integration test → E2E (少量) → Staging → Canary 5% → 100%
```

```yaml
# .github/workflows/agent-ci.yml
name: Agent CI
on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: pip install -e .
      - run: pytest tests/unit/

  prompt-eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm install -g promptfoo
      - run: promptfoo eval
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

  deepeval-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: pip install deepeval
      - run: deepeval test run tests/eval/

  redteam:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm install -g promptfoo
      - run: promptfoo redteam run
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

  e2e:
    runs-on: ubuntu-latest
    steps:
      - run: python tests/e2e/test_agent_flow.py
```

**关键**：每个 prompt / model / agent code 改动都自动跑全量 eval。

**面试追问：**
- CI 跑 eval 的成本怎么控制？→ 用 gpt-4o-mini 跑 eval（便宜），生产才用 gpt-4o；或对 golden set 做采样（每次跑 20%）。
- Canary 阶段怎么评估？→ 对比新/旧版本的 quality score + latency + error rate，达标才全量。

---

## 六、可观测性篇

### Q: LLM 应用的可观测性（Observability）包括什么？

**Answer:**

传统可观测性 = Metrics + Logs + Traces。LLM 应用额外增加：

```
传统 3 支柱              LLM 应用扩展
───────────────────    ───────────────────
Metrics (指标)       →   + Token 用量 / Cost / Quality score
Logs (日志)          →   + Prompt / Completion / Tool call 日志
Traces (链路)        →   + LLM call 级 trace + Tool call trace
```

**核心指标**：

| 指标 | 说明 | 工具 |
|------|------|------|
| **Latency** | TTFT / E2E latency | Prometheus + Grafana |
| **Token usage** | Input / Output / Cache hit | Langfuse / Helicone |
| **Cost per query** | 单次请求成本 | Langfuse / Helicone |
| **Error rate** | LLM error / Tool error | Sentry / Prometheus |
| **Quality score** | LLM-as-judge 评分 | Langfuse / DeepEval |
| **Tool call accuracy** | 工具调用成功率 | 自定义 |

**面试追问：**
- TTFT 是什么？→ Time To First Token，首个 token 返回时间；用户感知延迟的核心指标。

---

### Q: Langfuse 怎么用？核心概念？

**Answer:**

Langfuse = 开源 LLM 观测平台，兼容 OpenTelemetry。

**核心概念**：
- **Trace**：一次完整请求的链路（可能包含多个 LLM call + tool call）
- **Span**：trace 内的单个操作（如一次 LLM call、一次 tool call）
- **Generation**：一次 LLM 生成（input prompt → output completion）
- **Score**：对 generation 的质量评分

```python
import langfuse
from langfuse import Langfuse

langfuse = Langfuse()

# 创建 trace
trace = langfuse.trace(name="agent-run", user_id="user-123")

# 记录 LLM call
generation = trace.generation(
    name="llm-call",
    model="gpt-4o",
    input={"messages": messages},
    output={"content": response},
    usage={"prompt_tokens": 100, "completion_tokens": 50}
)

# 记录 tool call
trace.span(
    name="tool-call",
    input={"tool": "search", "args": {"query": "AI"}},
    output={"results": [...]}
)

# 记录质量评分（可事后或自动）
generation.score(
    name="quality",
    value=0.9,
    comment="回答准确"
)
```

**面试追问：**
- Langfuse 和 LangSmith 的区别？→ LangSmith 是 LangChain 官方，生态集成好但闭源；Langfuse 开源可自部署，兼容 OpenTelemetry，更灵活。

---

### Q: Helicone 作为 LLM Gateway 的核心功能？

**Answer:**

Helicone = LLM 专用 gateway，提供：
- 统一接入（OpenAI / Anthropic / 自部署）
- 自动 caching（prompt cache / semantic cache）
- Rate limit + cost tracking
- 自托管或 SaaS

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://oai.helicone.ai/v1",
    default_headers={
        "Helicone-Auth": f"Bearer {HELICONE_API_KEY}",
        "Helicone-User-Id": "user-123",
    }
)
```

**Dashboard 提供**：
- Token 用量趋势（按 user / model / endpoint）
- 延迟分布（P50 / P95 / P99）
- Cost  breakdown
- Prompt cache 命中率

**面试追问：**
- Helicone 的 caching 策略？→ 默认按 exact prompt cache（相同 prompt 返回缓存）；支持 semantic cache（相似 prompt 返回缓存）。

---

## 七、Eval-Driven Development 篇

### Q: 什么是 Eval-Driven Development？为什么对 Agent 特别重要？

**Answer:**

Eval-Driven Development = 每次改动（prompt / model / agent code）都必须先跑 eval 集，达标才能合并。

```
写新 prompt → 跑 prompt eval → 优化 → 跑 deepeval → 优化 → 跑 ragas → ...
                                                            ↓
                                                     达标 → commit
```

**对 Agent 特别重要**：
- LLM 输出是概率性的，改了 prompt 可能意外破坏现有能力
- 没有 eval = 赌博，靠人工感觉判断质量
- 团队协作时，eval 集是**共享的质量标尺**

**核心原则**：
- **没有 eval 集就不要改 prompt**
- **每个 prompt 改动都要跑 eval**
- **每周跑完整 E2E suite**

**面试追问：**
- Eval-Driven 和 TDD 的区别？→ TDD 是写代码前先写测试；Eval-Driven 是改 prompt/model 前先跑评估。都是"先验证，后提交"的思想。

---

### Q: Agent 评估的 4 层体系是什么？

**Answer:**

| 层级 | 评估内容 | 工具 |
|------|----------|------|
| **L1 通用能力** | 知识 / 推理 / 数学 | lm-eval-harness / MMLU / GSM8K |
| **L2 任务能力** | 编程 / Agent 任务 | SWE-bench / AgentBench / GAIA |
| **L3 业务能力** | 垂直场景 | 自建 golden set + Promptfoo |
| **L4 用户反馈** | 真实使用质量 | A/B test + 用户评分 + 质量采样 |

**面试追问：**
- SWE-bench 评估什么？→ 真实 GitHub issue 的编程修复能力，是目前编程 Agent 的主要 benchmark。
- 自建 golden set 要注意什么？→ 覆盖核心场景（50+ 条）、包含边界 case、定期根据 prod failure 扩充。

---

### Q: 主流 Eval 工具怎么选？Promptfoo / DeepEval / RAGAS 对比？

**Answer:**

| 工具 | 定位 | 优势 | 适用场景 |
|------|------|------|----------|
| **Promptfoo** | LLM 评估 + red team | 声明式、CI 友好、简单 | 快速 prototype / CI 集成 |
| **DeepEval** | pytest 风格评估 | 14+ metrics、可自定义 | 单元测试风格、团队已有 pytest |
| **RAGAS** | RAG 评估 | 4 大指标是事实标准 | RAG pipeline 质量评估 |
| **LangSmith** | 一体化平台 | Trace + Eval + Dataset | LangChain 生态 |
| **Langfuse** | 开源可观测 | 可自部署、OpenTelemetry | 开源 / 需要自托管 |

**选型建议**：
- 快速上手 → **Promptfoo**（一行 yaml）
- pytest 风格 → **DeepEval**
- RAG 专项 → **RAGAS**
- 生产监控 → **Langfuse** + **RAGAS**
- LangChain 项目 → **LangSmith**

**组合使用**：Promptfoo（prompt 回归）+ DeepEval（trajectory 质量）+ RAGAS（RAG 质量）

**面试追问：**
- Promptfoo 和 DeepEval 能一起用吗？→ 能，它们互补：Promptfoo 做 prompt 级回归，DeepEval 做 trajectory 级深度评估。

---

### Q: 生产 Agent 的监控和告警怎么设计？

**Answer:**

**关键 SLI / SLO**：

| SLI | SLO 目标 |
|-----|----------|
| 可用性 | 99.9% |
| End-to-end latency | P95 < 5s |
| LLM latency | P95 < 3s |
| Error rate | < 0.5% |
| Token cost / query | 视业务 |
| Sandbox escape attempts | 0 |

**监控工具栈**：
- **Metrics**：Prometheus + Grafana
- **Logs**：Loki / ELK
- **Trace**：OpenTelemetry / Jaeger / Langfuse
- **LLM-specific**：Langfuse / Helicone / Arize Phoenix
- **Alert**：Alertmanager / PagerDuty

**异常检测**：
- **成本突增**（某个 user 烧了 10x token）→ 自动限流 + 告警
- **失败率突增**（上游 LLM 挂了）→ 自动 fallback
- **长尾延迟**（reasoning model 思考太久）→ 设 max timeout + fallback
- **沙箱逃逸尝试**（agent 试图访问 /etc/shadow）→ 立即 kill + 告警

**面试追问：**
- 怎么定义"成本突增"的阈值？→ 基于历史基线（如过去 7 天同用户 P95），突增 3x 触发告警；或设绝对预算（单用户单日 $10）。

---

## 八、综合实战篇

### Q: 从笔记本到生产，部署 Agent 的完整路径是什么？

**Answer:**

**阶段 1：本地原型**
- 本地跑通（Docker / 直接 Python）
- 用 Promptfoo 写 10 条 eval

**阶段 2：测试环境**
- 接入 Gateway（FastAPI + 认证）
- 加 Sandbox（Docker）
- Session 持久化（Postgres + Redis）
- CI pipeline 接入（promptfoo + deepeval）

**阶段 3：预发布**
- 接入可观测（Langfuse / Helicone）
- 跑完整 eval 集（golden set 200+ 条）
- Red team 测试（promptfoo redteam）

**阶段 4：生产灰度**
- Canary 5% 流量
- 监控 SLO（latency / error / cost / quality）
- 自动回滚配置

**阶段 5：生产全量**
- 100% 流量
- 持续监控 + 定期 eval
- 自动扩充 golden set（从 prod failure 收集）

**面试追问：**
- 什么时候从阶段 2 进入阶段 3？→ 当 eval 集通过率 > 95%，red team 无高危漏洞，sandbox 安全审计通过。
- 生产出问题了怎么快速回滚？→ 一键回滚（prompt 版本 + model 版本 + agent code 版本都纳入版本管理）。

---

### Q: Agent 的 Disaster Recovery 怎么设计？

**Answer:**

| 场景 | 恢复策略 |
|------|----------|
| **Agent 进程崩溃** | K8s restart / 自动重启 |
| **Session 丢失** | Postgres 持久化 + 定期备份；从最新 checkpoint resume |
| **LLM 上游故障** | Circuit breaker + Fallback chain（多 provider） |
| **数据损坏** | S3 备份 + 定期快照 |
| **Region 故障** | Multi-region 部署 + 数据同步 |

**Checkpoint 策略**：
- 每 N 步保存 checkpoint（如每 5 步）
- 关键决策点强制保存
- Checkpoint 存在 Postgres（主） + S3（冷备）

**面试追问：**
- Multi-region 部署 Agent 有什么特殊考虑？→ Session 数据需要跨 region 同步（Postgres streaming replication）；LLM API 调用可能需要就近接入。

---

### Q: Agent 的 Cost 优化策略有哪些？

**Answer:**

| 策略 | 说明 | 节省比例 |
|------|------|----------|
| **Prompt Cache** | 相同 prompt 复用 | 50-90% |
| **Semantic Cache** | 相似 prompt 返回缓存 | 30-50% |
| **Model Routing** | 简单问题用小模型 | 50-80% |
| **Streaming** | 边生成边返回（TTFT 降低） | 用户体验 |
| **Token 限制** | 单次请求 max token 上限 | 防意外 |
| **Batch API** | 批量请求折扣 | 20-50% |

**代码示例（Prompt Cache）**：
```python
from langchain.cache import InMemoryCache
from langchain_openai import ChatOpenAI

# 开启 prompt cache
import langchain
langchain.llm_cache = InMemoryCache()

llm = ChatOpenAI(model="gpt-4o")
# 相同 prompt 第二次调用直接返回缓存
```

**面试追问：**
- Prompt Cache 和 Semantic Cache 的区别？→ Prompt Cache 要求完全相同的 prompt；Semantic Cache 用 embedding 找相似 prompt，缓存命中率更高但可能有精度损失。

---

### Q: 面试中怎么回答"怎么把 Agent 从笔记本部署到生产"？

**Answer:**

**回答框架（5 步）**：

**Step 1：架构分层**
"4 层架构：L1 接入 → L2 Gateway（认证/限流/审计）→ L3 Orchestrator（LangGraph 管理 session/state）→ L4 Sandbox（Docker/gVisor 隔离执行）"

**Step 2：安全隔离**
"Agent 有副作用（文件/网络/命令），必须沙箱。Docker 做默认隔离，Computer Use 用完整 VM。每次运行 snapshot 重置，audit log 全记录。"

**Step 3：状态管理**
"Session 持久化：Postgres 存完整 state + messages（JSONB），Redis 缓存热数据。LangGraph checkpointer 自动 save/resume。"

**Step 4：可观测 + CI/CD**
"CI/CD 跑 prompt eval + tool mock + golden set。生产监控用 Langfuse/Helicone：trace + cost + latency + quality score。Eval-driven：每次 prompt 改动必须跑 eval 集。"

**Step 5：灰度 + 回滚**
"Canary 5% → 监控 SLO → 全量。一键回滚（prompt + model + code 版本化）。"

**加分项**：
- 提到 Circuit breaker + Fallback chain（多 LLM provider）
- 提到 Red team 测试（promptfoo redteam）
- 提到 Drift Detection（监控 quality score 趋势）

**面试追问：**
- 如果只有 1 个工程师，怎么最小化部署复杂度？→ 用 LangGraph Platform（托管）+ Helicone（gateway）+ Docker（sandbox），避免自建基础设施。

---

### Q: 设计一个支持多租户的 Agent 平台，关键考虑是什么？

**Answer:**

**多租户隔离维度**：

| 维度 | 隔离方式 |
|------|----------|
| **数据隔离** | 每个 tenant 独立 DB schema / row-level security |
| **资源隔离** | Per-tenant rate limit / token budget |
| **沙箱隔离** | 每个 tenant 的 agent 跑在独立 sandbox |
| **认证隔离** | Per-tenant OAuth / JWT |

**架构示例**：
```
用户 → Gateway（JWT 验证 + tenant 识别）
         ↓
   Orchestrator（按 tenant 路由到独立 sandbox pool）
         ↓
   Sandbox Pool（tenant-A 的 sandbox / tenant-B 的 sandbox）
```

**关键挑战**：
- **Noisy Neighbor**：一个 tenant 烧太多资源影响其他人 → per-tenant resource limit
- **数据泄露**：tenant A 不能访问 tenant B 的数据 → row-level security + sandbox isolation
- **Cost 分摊**：每个 tenant 独立计费 → token-level billing

**面试追问：**
- 多租户 Agent 和单租户最大的区别？→ 隔离从"进程级"升级到"数据+资源+沙箱"三级隔离；计费从"按请求"变成"按 token + 资源"。

---

### Q: LangGraph + Postgres 实现 Session 持久化的完整示例？

**Answer:**

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.graph import StateGraph, MessagesState
from langchain_openai import ChatOpenAI

# 1. 初始化 Postgres checkpointer
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/agent_db"
)

# 2. 定义 Agent graph
workflow = StateGraph(MessagesState)
workflow.add_node("agent", lambda state: llm.invoke(state["messages"]))
app = workflow.compile(checkpointer=checkpointer)

# 3. 使用（自动持久化）
config = {"configurable": {"thread_id": "user-session-123"}}

# 第一次调用
result = app.invoke(
    {"messages": [("user", "你好")]},
    config
)

# 第二次调用（自动从 checkpoint resume）
result = app.invoke(
    {"messages": [("user", "继续")]},
    config  # 同一个 thread_id
)
```

**Postgres 表结构**（自动创建）：
- `checkpoints`：每个 step 的 state snapshot
- `writes`：每个 step 的写操作记录

**面试追问：**
- Postgres checkpointer 的瓶颈在哪？→ 每个 step 都要写 DB；高并发场景可能需要 Redis 做热缓存，或批量异步写。

---

### Q: Circuit Breaker 在 Agent 系统中的作用和实现？

**Answer:**

Circuit Breaker = 上游 LLM 或工具挂掉时，不再发请求，快速失败。

**状态机**：
```
CLOSED（正常） → OPEN（熔断） → HALF-OPEN（试探）
     ↓               ↑                  ↓
  请求正常        连续失败 N 次        试探成功 → CLOSED
                                  试探失败 → OPEN
```

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED / OPEN / HALF-OPEN

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF-OPEN"
            else:
                raise CircuitOpenError("Circuit is open")

        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise

    def on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

**在 Agent 中的应用**：
```python
# LLM 调用加 circuit breaker
cb_llm = CircuitBreaker(failure_threshold=5, recovery_timeout=60)

def call_llm(messages):
    return cb_llm.call(llm.invoke, messages)
```

**面试追问：**
- Circuit Breaker 和 Retry 的区别？→ Retry 是"失败后重试"；Circuit Breaker 是"连续失败后停止重试，快速失败"。两者配合：Retry 在 CLOSED 状态，Circuit Breaker 在 OPEN 状态。

---

### Q: 怎么设计 Agent 的 Golden Test Set（黄金测试集）？

**Answer:**

**Golden Test Set = Agent 质量的"标尺"**，必须覆盖：

| 类别 | 占比 | 说明 |
|------|------|------|
| **基础场景** | 40% | 核心功能正常流程 |
| **边界 case** | 30% | 边界值、异常输入 |
| **Tool 调用** | 20% | 工具选择正确性、参数正确性 |
| **失败恢复** | 10% | 错误处理、重试逻辑 |

```jsonl
{"input": "什么是 RAG？", "expected": "包含'检索'和'生成'", "category": "basic_qa"}
{"input": "1000000 的平方根", "expected": "1000", "category": "basic_qa"}
{"input": "搜索一个不存在的内容", "expected": "承认不知道", "category": "edge_case"}
{"input": "北京天气", "expected": "调用 get_weather(city='北京')", "category": "tool_use"}
{"input": "数据库连接失败时的处理", "expected": "fallback 到缓存", "category": "failure_recovery"}
```

**管理**：
- 版本化：`tests/golden/v1.0/`, `tests/golden/v1.1/`
- 自动扩充：从 prod failure 收集（`user_rating < 3`）
- 定期 review：每季度 review 一次，淘汰过时 case

**面试追问：**
- Golden set 多少条够？→ 50-500 条；核心场景 50 条必须全过，扩展场景 200+ 条，边界 case 100+ 条。

---

### Q: Eval-Driven Development 的具体工作流？

**Answer:**

```
1. 写新 prompt
   ↓
2. 本地跑 promptfoo（10 条 golden set）
   ↓
3. 达标 → 提交 PR
   ↓
4. CI 跑全量 eval（promptfoo + deepeval + ragas）
   ↓
5. 不达标 → 优化 prompt → 回到步骤 2
   ↓
6. 达标 → 合并 → Staging 跑完整 E2E
   ↓
7. Staging 达标 → Canary 5%
   ↓
8. Canary SLO 达标 → 全量
   ↓
9. 生产监控（5% sampling eval + drift detection）
   ↓
10. 收集 failure → 扩充 golden set → 回到步骤 1
```

**核心原则**：
- **没有 eval 集就不要改 prompt**
- **每次 prompt 改动都要跑 eval**
- **生产 failure 自动回炉 golden set**

**面试追问：**
- Eval 集怎么保证不泄漏敏感数据？→ Golden set 用匿名化数据，不含 PII；定期 audit。

---

## 九、面试总结

### 必背概念速查

| 概念 | 一句话 |
|------|--------|
| **4层架构** | L1接入 → L2 Gateway → L3 Orchestrator → L4 Sandbox |
| **Sandbox 选型** | Docker 默认 / gVisor 多租户 / Firecracker 高并发 / VM 高合规 |
| **Session 持久化** | Postgres 持久化 + Redis 热缓存 + LangGraph checkpointer |
| **Gateway 职责** | 认证 + 限流（token级）+ 审计 + 路由 + 可观测 |
| **CI/CD** | Unit → Prompt eval → Integration → E2E → Staging → Canary |
| **Eval-Driven** | 改 prompt 前先跑 eval，不达标不允许合并 |
| **可观测性** | Metrics + Logs + Traces + LLM-specific（token / cost / quality） |
| **从笔记本到生产** | 原型 → 测试环境（sandbox + gateway + CI） → 预发布（eval + red team） → 灰度 → 全量 |

### 高频追问链路

```
问：Agent 怎么安全执行有副作用操作？
→ Sandbox 隔离（Docker/gVisor/Firecracker）
→ 追问：Firecracker 和 Docker 安全区别？
→ 追问：Computer Use 需要什么级别隔离？

问：Agent session 怎么持久化？
→ Postgres + Redis + LangGraph checkpointer
→ 追问：Checkpoint 粒度？
→ 追问：Redis 为什么不能做唯一存储？

问：Agent 的 CI/CD 有什么特殊？
→ Prompt regression test + Tool mock + Eval-driven
→ 追问：Golden set 多少条够？
→ 追问：CI 跑 eval 成本怎么控制？
```

---

## 十、必读材料

| 材料 | 为什么读 |
|------|----------|
| [Docker 官方文档](https://docs.docker.com/) | 沙箱基础 |
| [gVisor](https://gvisor.dev/) | 强沙箱实现 |
| [Firecracker](https://github.com/firecracker-microvm/firecracker) | Serverless 沙箱 |
| [LangGraph Checkpointing](https://langchain-ai.github.io/langgraph/concepts/persistence/) | Agent state 持久化 |
| [Langfuse](https://langfuse.com/) | LLM 可观测性 |
| [Helicone](https://www.helicone.ai/) | LLM Gateway |
| [Promptfoo](https://promptfoo.dev/) | Prompt 评估 |
| [DeepEval](https://docs.confident-ai.com/) | pytest 风格评估 |
| [RAGAS](https://docs.ragas.io/) | RAG 评估 |
| [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | Agent 安全基线 |
| [12-Factor App](https://12factor.net/) | 生产应用方法论 |
| [Anthropic: Computer Use safety](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use) | GUI Agent 沙箱设计 |

---

## 十一、练习

1. **Docker 沙箱**：用 Docker 跑一段 Python 代码，限制只能访问 /tmp 目录
2. **LangGraph + Postgres**：用 LangGraph 的 PostgresSaver 做能 resume 的 agent
3. **Circuit Breaker**：实现一个简单的 circuit breaker，验证上游挂掉时不再发请求
4. **Helicone 接入**：在 OpenAI client 上加 Helicone 跑 1000 个请求，看 cost/latency dashboard
5. **CI Pipeline**：写一个 GitHub Action，跑 prompt regression test
6. **Sandbox 逃逸测试**：故意在 sandbox 里尝试 `cat /etc/shadow`，看是否被拦截
7. **Golden Set 构建**：收集 100 条自己的 Agent 测试 case，分基础 / 边界 / tool / 失败四类
8. **Drift Detection**：监控 RAGAS 分数 30 天，观察趋势并设置告警

---

## 十二、与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U18 Agent 安全与红队 | Sandbox / permission / 审计是安全的具体实现 |
| U22 Context Engineering | Session 持久化 = Context 跨 turn / 跨 session 保持 |
| U23 Harness Engineering | Sandbox / observability / circuit breaker 是 Harness 11 组件的具体实现 |
| U24 Reasoning Model | Reasoning Model 慢+贵 → 更需要 cost / latency 工程 |
| U25 Computer Use | Computer Use 必须配强 sandbox（VM） |
| U28 LLM Inference | LLM 调用是 production 的瓶颈，要 cache / fallback / 监控 |
| U21 Loop Engineering | Loop Engineering 跨 session 持续 → 依赖 deployment 层的 session 持久化 |
| U34 Guardrails | Safety metrics 来自 U34 |
| U35 AgentOps | 本文件直接来源 |

---

*最后更新：2026-07-01*
*来源：U29 Agent Deploy + U35 AgentOps + Docker/gVisor/Firecracker 文档 + LangGraph checkpointer + Langfuse/Helicone*
