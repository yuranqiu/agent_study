# U29 · Agent 部署 & Sandbox DevOps（部署与沙箱工程）

> **核心问题**：怎么把 Agent 从"我笔记本上能跑"变成"生产环境 7×24 小时稳定服务"？沙箱、Docker、gateway、CI/CD 怎么设计？
> **预计学时**：4h
> **对应 Phase**：Phase 4 / 项目实战
> **状态**：生产工程必修（2025-2026 DevOps 基础）

---

## 1. 为什么需要 Agent Deploy & Sandbox

Agent 在生产环境面临：
- **执行风险**：Coding Agent / Computer Use Agent 真的能"删文件、跑命令、访问数据库"——必须沙箱
- **可扩展性**：从 1 个用户到 10 万个用户
- **可观测性**：用户报告"我昨天让它做的事没成功"——怎么 debug
- **可恢复性**：agent 跑挂了怎么恢复（重启 / 续跑 / 状态保存）
- **多租户**：不同用户之间不能互相影响
- **安全合规**：金融/医疗 agent 要审计、加密、权限控制

**核心洞察**：Agent 不只是 LLM 调用，是**有副作用的 LLM 循环**。副作用 = 文件 / DB / API / 操作系统调用 = 需要 sandbox。

---

## 2. 4 层部署架构

```
┌────────────────────────────────────────────┐
│ L1: 用户接入层                              │
│   Web UI / CLI / IDE / API / IM 机器人     │
├────────────────────────────────────────────┤
│ L2: API Gateway / 鉴权 / 限流               │
│   Kong / 自建 FastAPI Gateway               │
├────────────────────────────────────────────┤
│ L3: Agent Orchestrator                      │
│   LangGraph / CrewAI / Custom Harness       │
│   - Session 管理                            │
│   - Memory 持久化                            │
│   - State checkpoint                        │
├────────────────────────────────────────────┤
│ L4: Execution Sandbox                       │
│   - Docker / Firecracker / gVisor           │
│   - File / Shell / Network 隔离              │
│   - Resource limits (CPU / memory / disk)    │
└────────────────────────────────────────────┘
```

**关键**：
- **L1-L3 是 Python**（应用层）
- **L4 是 OS-level 隔离**（基础设施层）

---

## 3. Sandbox：Agent 的"安全屋"

### 3.1 沙箱的核心属性

| 属性 | 作用 | 实现 |
|------|------|------|
| **File 隔离** | Agent 只能读写指定目录 | Docker volume / chroot |
| **Network 隔离** | 限制可访问的网络 | Docker network / iptables |
| **Resource limits** | 限制 CPU / memory / disk / time | cgroups / ulimit |
| **Syscall 过滤** | 拦截危险系统调用 | seccomp / AppArmor |
| **可重置** | 每次跑完恢复到干净状态 | Docker image / snapshot |
| **可观测** | 记录所有操作 | audit log / trace |

### 3.2 沙箱实现层级

```
最强隔离 ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ → 最弱隔离
─────────────────────────────────────────────────
VM     |   gVisor  |   Firecracker  |  Docker  |  Process
─────────────────────────────────────────────────
启动慢，安全最高                            启动快，安全低
```

| 方案 | 隔离强度 | 启动时间 | 适用场景 |
|------|---------|----------|----------|
| **Process** | 弱 | 0s | 不可信代码禁止 |
| **Docker** | 中 | 1-2s | 一般 coding agent |
| **gVisor** | 强 | 1-2s | 多租户 / 半可信 |
| **Firecracker** | 强 | < 125ms | Serverless / 高 QPS |
| **完整 VM** | 最强 | 秒-分钟 | 金融/医疗等高合规 |

### 3.3 Coding Agent 的沙箱实践

**Claude Code / OpenHands** 都用 Docker 作为默认沙箱：

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
# 网络限制：只能访问白名单域名
```

```python
# 用 Docker SDK 启动 sandbox
import docker

client = docker.from_env()

def run_in_sandbox(code, timeout=30):
    container = client.containers.run(
        "agent-sandbox:latest",
        command=f"python -c '{code}'",
        volumes={"/host/project": {"bind": "/home/agent/project", "mode": "rw"}},
        network_mode="bridge",  # 受限网络
        mem_limit="512m",
        cpu_quota=50000,  # 50% CPU
        detach=True,
        remove=True,
    )
    try:
        result = container.wait(timeout=timeout)
        logs = container.logs().decode()
        return {"status": result["StatusCode"], "logs": logs}
    except Exception as e:
        container.kill()
        raise
```

### 3.4 Computer Use 的更强隔离

GUI Agent 因为能"操作任何程序"，需要**完整 OS 隔离**：

| 方案 | 厂商 | 特点 |
|------|------|------|
| **Anthropic Computer Use** | Anthropic | macOS 沙箱 / Docker / VM |
| **OpenAI Operator** | OpenAI | 浏览器沙箱 + 远程 VM |
| **browser-use** | 开源 | Playwright headless + Docker |
| **ByteDance UI-TARS** | 字节 | Docker + 远程 VM |

**典型架构**：
```
用户 → API Gateway → Orchestrator → 远程 VM (snapshot 启动)
                                       ↓
                                  Computer Use Agent
                                  (在 VM 内操作)
                                       ↓
                                  完成后销毁 VM
```

---

## 4. Session / State 管理

Agent 是**有状态的**——跨多个 turn 持续，跨 session 恢复（详见 U21）。生产部署要解决：

### 4.1 Session 存储

```python
# Postgres + Redis 模式
- Postgres: 持久化 conversation history + tool calls + final state
- Redis: 活跃 session 的 fast access（带 TTL）
```

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY,
    user_id TEXT NOT NULL,
    state JSONB NOT NULL,  -- 整个 agent state
    created_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ,
    checkpoint_index INT
);

CREATE TABLE messages (
    id UUID PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    role TEXT,  -- user / assistant / tool
    content JSONB,
    tool_call JSONB,
    created_at TIMESTAMPTZ
);
```

### 4.2 Checkpoint & Resume

```python
def save_checkpoint(session_id, state, step_index):
    db.execute(
        "INSERT INTO checkpoints (session_id, state, step_index) VALUES (%s, %s, %s)",
        (session_id, json.dumps(state), step_index)
    )

def resume_from_checkpoint(session_id, step_index):
    row = db.execute(
        "SELECT state FROM checkpoints WHERE session_id = %s AND step_index <= %s ORDER BY step_index DESC LIMIT 1",
        (session_id, step_index)
    ).fetchone()
    return json.loads(row["state"])
```

### 4.3 Memory 持久化

详见 U8 + U11。**生产级 Memory 栈**：
- **短期**：Redis（最近 N 轮对话）
- **长期**：Postgres + 向量库（用户偏好、关键决策）
- **持久项目记忆**：AGENTS.md / NOTES.md（详见 U26）

---

## 5. Gateway：API 入口

### 5.1 Gateway 的核心职责

| 职责 | 说明 |
|------|------|
| **认证** | JWT / API key / OAuth |
| **授权** | RBAC / multi-tenant isolation |
| **限流** | 用户级 / 全局级 / token 预算 |
| **审计** | 完整 request/response 记录 |
| **路由** | 不同 endpoint / 版本 / 灰度 |
| **可观测** | 统一埋点 / metrics / trace |
| **协议转换** | HTTP / gRPC / WebSocket |

### 5.2 自建 vs 用现成

| 方案 | 适合 |
|------|------|
| **FastAPI + 中间件** | 自建，灵活 |
| **Kong / Traefik** | 标准网关 |
| **Cloud API Gateway** | AWS API Gateway / Azure APIM |
| **LangServe** | LangChain 生态 |
| **OpenRouter / Helicone** | LLM 专用，路由 + 缓存 + 监控 |

### 5.3 LLM 专用 Gateway：Helicone

[Helicone](https://www.helicone.ai/) 是 LLM 专用 gateway：
- 统一接入 OpenAI / Anthropic / 自部署
- 自动 caching / rate limit / cost tracking
- 自托管或 SaaS

```python
# 用 Helicone 做网关
from openai import OpenAI
client = OpenAI(
    base_url="https://oai.helicone.ai/v1",
    default_headers={
        "Helicone-Auth": f"Bearer {HELICONE_API_KEY}",
        "Helicone-User-Id": user_id,
    }
)
# 自动记录 + cost tracking
```

---

## 6. CI/CD for Agent

Agent 的 CI/CD 比传统软件复杂——需要测 prompt、测 tool calling、测多 step 流程。

### 6.1 测试金字塔

```
       E2E（少量，golden cases）
       ─────────────────────────
      Integration（中等，prompt regression）
      ───────────────────────────────────
     Unit（多，tool / format / parser）
     ───────────────────────────────────
```

### 6.2 Prompt Regression Test

```python
# 用 LangSmith / Promptfoo 跑 prompt 回归
def test_summarization_prompt():
    cases = load_golden_cases()  # 100 个样例
    for case in cases:
        result = agent.invoke(case.input)
        assert semantic_match(result, case.expected_output)
        assert cost_within_budget(result, max_tokens=500)
        assert latency_under(result, max_seconds=3)
```

### 6.3 Tool Mock

```python
@pytest.fixture
def mock_search_tool():
    with mock.patch("tools.search", return_value=mock_search_results):
        yield

def test_agent_uses_search(mock_search_tool):
    result = agent.run("search for AI agent trends")
    assert mock_search_tool.called
    assert "search" in result.tool_calls
```

### 6.4 Eval-driven Development

- **每个 prompt 改动都要跑 eval 集**
- **每个 tool 改动都要跑 integration test**
- **每周跑完整 E2E suite**

详见 U12 Agent 评估标准。

### 6.5 Deployment Pipeline

```
PR → Unit test → Prompt eval → Integration test → E2E (少量) → Staging → Canary 5% → 100%
```

- **Staging**：用真实数据子集跑
- **Canary**：5% 流量新版本，对比 SLO
- **回滚**：自动 / 一键

---

## 7. 监控 & Alerting

### 7.1 关键 SLI / SLO

| SLI | SLO 目标 |
|-----|----------|
| **可用性** | 99.9% |
| **End-to-end latency** | P95 < 5s |
| **LLM latency** | P95 < 3s |
| **Error rate** | < 0.5% |
| **Token cost / query** | 视业务 |
| **Sandbox escape attempts** | 0 |

### 7.2 监控工具

- **Metrics**：Prometheus + Grafana
- **Logs**：Loki / ELK
- **Trace**：OpenTelemetry / Jaeger / Langfuse
- **LLM-specific**：Langfuse / Helicone / Arize Phoenix
- **Alert**：Alertmanager / PagerDuty

### 7.3 异常检测

- **成本突增**（某个 user 烧了 10x token）→ 自动限流 + 告警
- **失败率突增**（上游 LLM 挂了）→ 自动 fallback
- **长尾延迟**（reasoning model 思考太久）→ 设 max timeout + fallback
- **沙箱逃逸尝试**（agent 试图访问 /etc/shadow）→ 立即 kill + 告警

---

## 8. Production Checklist

部署 Agent 到生产前必查：

**Sandbox**：
- [ ] File 隔离（只能访问指定目录）
- [ ] Network 隔离（白名单 / 完全隔离）
- [ ] Resource limits（CPU / memory / time）
- [ ] Syscall 过滤（可选，看威胁模型）
- [ ] 沙箱可重置（每次跑完恢复）
- [ ] 完整 audit log

**Reliability**：
- [ ] Circuit breaker（上游挂掉时不再打）
- [ ] Fallback chain（多 provider）
- [ ] Retry with exponential backoff
- [ ] Health check endpoint
- [ ] Graceful shutdown

**Observability**：
- [ ] Request 级别 metrics（latency / token / cost）
- [ ] Trace（跨 step / 跨 tool）
- [ ] Structured logging
- [ ] Alert 配置
- [ ] Error 聚合（Sentry 等）

**Security**：
- [ ] 认证 / 授权
- [ ] Rate limit（per user / per IP / per token budget）
- [ ] 危险操作 human-in-the-loop gate
- [ ] 数据加密（at rest / in transit）
- [ ] 审计日志（compliance）

**Performance**：
- [ ] Streaming（TTFT < 500ms）
- [ ] Caching（prompt cache / semantic cache）
- [ ] Concurrency 控制
- [ ] Batching（如适用）

**Disaster Recovery**：
- [ ] Session 持久化（Postgres + S3 备份）
- [ ] Checkpoint 机制
- [ ] 一键回滚
- [ ] Multi-region（高级）

---

## 9. 部署模式：自建 vs 平台

| 模式 | 适合 | 代表 |
|------|------|------|
| **自建** | 大规模、定制、合规要求高 | 字节、阿里、Anthropic |
| **LangSmith / LangGraph Platform** | LangChain 生态 | LangChain |
| **CrewAI Enterprise** | CrewAI 生态 | CrewAI |
| **Dify / Coze** | 可视化、低代码 | 字节 / Coze |
| **OpenAI AgentKit / Anthropic Agent SDK** | 闭源 + 托管 | OpenAI / Anthropic |
| **Nexent** | 零代码 Harness | ModelEngine |
| **LoopX / Loom** | 编程 Agent harness | 独立开源 |

---

## 10. 面试视角

- **Q：Agent 怎么安全执行有副作用的操作（写文件 / 跑命令）？**
  → **Sandbox 隔离**：Docker 限制文件 + 网络 + 资源；gVisor / Firecracker 提供更强隔离；完整 VM 用于高合规场景。每次跑完 snapshot 重置。完整 audit log + 危险操作 human-in-the-loop。

- **Q：Agent session 怎么持久化？**
  → Postgres 存 state + messages（JSONB），Redis 缓存活跃 session。LangGraph 自带 checkpointer（Postgres / Redis backend）。每 N 步或关键决策点保存 checkpoint，支持 resume。

- **Q：怎么设计 Agent 的 API gateway？**
  → 认证（JWT / OAuth）+ 授权（RBAC / 多租户）+ 限流（per user / per token budget）+ 审计 + 路由 + 协议转换。LLM 专用考虑用 Helicone / OpenRouter 做统一接入 + 监控。

- **Q：Agent 怎么做到 99.9% 可用？**
  → Circuit breaker + Fallback chain（多 LLM provider）+ Retry with backoff + Health check + Graceful shutdown + Auto-scaling（用 K8s HPA）。

- **Q：CI/CD for Agent 有什么特殊？**
  → 需要 prompt regression test + tool mock + eval 集。Pipeline：unit → prompt eval → integration → E2E → staging → canary → production。Eval-driven development。

- **Q：Computer Use Agent 怎么安全部署？**
  → 完整 OS 隔离（远程 VM snapshot 启动 / Firecracker microVM）。用户操作完全在隔离 VM 内，不影响主系统。完成后销毁 VM，状态只保留必要数据。

- **Q：怎么监控 LLM 应用的 cost？**
  → 埋点每 request 的 token 数（input / output / cache hit / miss）；按 user / model / endpoint 聚合；设预算告警；用 Langfuse / Helicone / OpenLLMetry。

---

## 11. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Docker 官方文档](https://docs.docker.com/) | 沙箱基础 |
| [gVisor](https://gvisor.dev/) | 强沙箱 |
| [Firecracker](https://github.com/firecracker-microvm/firecracker) | Serverless 沙箱 |
| [LangGraph Checkpointing](https://langchain-ai.github.io/langgraph/concepts/persistence/) | Agent state 持久化 |
| [Langfuse](https://langfuse.com/) | LLM observability |
| [Helicone](https://www.helicone.ai/) | LLM gateway |
| [OpenLLMetry](https://opentelemetry.io/) | OpenTelemetry for LLM |
| [Anthropic: Computer Use safety](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use) | GUI Agent 沙箱设计 |
| [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | Agent 安全基线 |
| [12-Factor App](https://12factor.net/) | 生产应用经典方法论 |

---

## 12. 练习

1. **Docker 沙箱**：用 Docker 跑一段 Python 代码，限制只能访问 /tmp 目录
2. **LangGraph + Postgres**：用 LangGraph 的 PostgresSaver 做一个能 resume 的 agent
3. **Circuit breaker**：实现一个简单的 circuit breaker，验证"上游挂掉时不再发请求"
4. **Helicone 接入**：在 OpenAI client 上加 Helicone 跑 1000 个请求，看 cost / latency dashboard
5. **CI pipeline**：写一个 GitHub Action，跑 prompt regression test
6. **Sandbox 逃逸测试**：故意在 sandbox 里尝试 `cat /etc/shadow`，看是否被拦截

---

## 13. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U18 Agent 安全与红队 | Sandbox / permission / 审计是安全的具体实现 |
| U22 Context Engineering | Session 持久化 = Context 跨 turn / 跨 session 保持 |
| U23 Harness Engineering | Sandbox / observability / circuit breaker 是 Harness 11 组件的具体实现 |
| U24 Reasoning Model | Reasoning Model 慢+贵 → 更需要 cost / latency 工程 |
| U25 Computer Use | Computer Use 必须配强 sandbox（VM） |
| U28 LLM Inference | LLM 调用是 production 的瓶颈，要 cache / fallback / 监控 |
| U21 Loop Engineering | Loop Engineering 跨 session 持续 → 依赖 deployment 层的 session 持久化 |

---

*最后更新：2026-06-30 新增 U29（Agent 部署 & Sandbox DevOps）*
*来源：Docker/gVisor/Firecracker 文档、LangGraph checkpointer、Langfuse/Helicone、Anthropic Computer Use 安全设计、12-Factor App*
