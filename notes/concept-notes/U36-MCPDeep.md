# U36 · MCP 深入（Model Context Protocol 协议工程）

> **核心问题**：怎么用 MCP（Model Context Protocol）构建生产级 agent 工具生态？MCP 协议本身、Server/Client 实现、Streamable HTTP 传输、与现有协议（Function Calling / A2A）的协同？
> **预计学时**：4h
> **对应 Phase**：Phase 2 / 进阶
> **状态**：2024-11 Anthropic 推出 → 2025 工业级标准（OpenAI / Google / Microsoft 全部支持）

---

## 1. 什么是 MCP

**MCP（Model Context Protocol）** = Anthropic 在 2024-11 推出的**统一工具协议**，让 LLM 应用以标准化方式连接外部数据源和工具。

> **核心定位**：**"USB-C for LLM applications"** — 一份 Server，所有支持 MCP 的客户端都能用。

**解决问题**：
- 之前每个 LLM 应用都要为每个数据源写一个集成（Notion / Slack / GitHub / DB...）
- 重复工作：每个集成都被写 N 次（Cursor、Claude Desktop、各个 agent framework）
- MCP 让**工具写一次，处处运行**

**vs A2A（U13）**：
- **MCP** = Agent ↔ 工具/数据（**纵向**）
- **A2A** = Agent ↔ Agent（**横向**）
- 两者**互补**：Agent 既用 MCP 调工具，又用 A2A 与其他 Agent 协作

---

## 2. MCP 三大核心概念

### 2.1 三大实体

| 实体 | 角色 | 例子 |
|------|------|------|
| **MCP Host** | LLM 应用 | Claude Desktop、Cursor、Cline、LangChain agent |
| **MCP Client** | Host 内部的协议客户端 | 每个 Host 内置的协议栈 |
| **MCP Server** | 暴露工具/资源的服务 | 一个 stdio 程序或 HTTP 服务 |

### 2.2 Server 暴露的三大原语

| 原语 | 作用 | 例子 |
|------|------|------|
| **Tools** | LLM 可调用的函数 | `get_weather(city)`、`search_docs(query)` |
| **Resources** | LLM 可读的上下文（类似 GET 端点）| `file://...`、`db://schema/users` |
| **Prompts** | 预制的 prompt 模板（user-controlled） | `/summarize-file` 模板 |

### 2.3 两种传输

| 传输 | 适合 | 性能 |
|------|------|------|
| **stdio** | 本地进程，单机部署 | 最快（无网络） |
| **Streamable HTTP** | 远程服务，多客户端 | 支持云端 |

---

## 3. 快速实现：FastMCP 写一个 Server

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Calculator")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

@mcp.tool()
async def fetch_weather(city: str) -> str:
    """Fetch weather for a city"""
    return f"Sunny in {city}"

@mcp.resource("doc://api")
def api_docs() -> str:
    return "API documentation: ..."

@mcp.prompt()
def review_code(code: str) -> str:
    return f"Please review the following code:\n{code}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
    # 或：mcp.run(transport="streamable-http", url="http://0.0.0.0:8000/mcp")
```

**关键点**：
- 用 `@mcp.tool()` 装饰器自动生成 JSON Schema
- docstring → 工具描述（LLM 用来决定何时调）
- type hints → 参数 schema

---

## 4. 主流框架集成

### 4.1 LangChain 集成

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain.agents import create_agent

server_params = StdioServerParameters(
    command="python",
    args=["/path/to/math_server.py"],
)

async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await load_mcp_tools(session)
        agent = create_agent("openai:gpt-4.1", tools)
        result = await agent.ainvoke({"messages": "what's (3 + 5) x 12?"})
```

### 4.2 多 MCP Server 集成

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "math": {
        "command": "python",
        "args": ["/path/to/math_server.py"],
        "transport": "stdio",
    },
    "weather": {
        "url": "http://localhost:8000/mcp",
        "transport": "http",
    },
})

tools = await client.get_tools()  # 同时加载多个 server 的所有 tool
agent = create_agent("openai:gpt-4.1", tools)
```

### 4.3 Claude Desktop 集成

```json
// ~/.config/claude_desktop_config.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/Documents"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..." }
    }
  }
}
```

重启 Claude Desktop → 自动加载所有 MCP tools。

---

## 5. Streamable HTTP 传输（远程 Server）

**MCP 在 2025-03 引入 Streamable HTTP** 替代原来的 HTTP+SSE：

```python
# Server 端
mcp.run(transport="streamable-http", url="http://0.0.0.0:8000/mcp")

# Client 端
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async with streamablehttp_client("http://localhost:3000/mcp") as (read, write, _):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await load_mcp_tools(session)
```

**Streamable HTTP 的优势**：
- 单 endpoint 处理双向 stream（避免 SSE 双向冲突）
- 支持 stateless server（k8s 友好）
- 支持 resumption（断点续传）
- 替代旧的 HTTP+SSE 模式（**2025-03 后已 deprecated**）

---

## 6. MCP vs Function Calling vs A2A

| 维度 | Function Calling | MCP | A2A |
|------|------------------|-----|-----|
| **范围** | 单个模型调用 | Agent ↔ 工具/数据 | Agent ↔ Agent |
| **标准化** | 每家 provider 不同 | 跨模型开放协议 | 跨 agent 框架 |
| **传输** | HTTP API | stdio / Streamable HTTP | HTTP + JSON |
| **典型使用** | 直接调 LLM | 给 agent 装"工具集" | 让 agent 们"对话" |
| **发现机制** | function schema 嵌入 prompt | 运行时 tool list | Agent Card |
| **生态** | 模型厂商 | Anthropic / OpenAI / Google / MS | Google 主推 |

**实战组合**：
```
LLM → Function Calling（直接调 LLM）
     → MCP（agent 调工具 / 读数据）
     → A2A（agent 之间协作）
```

---

## 7. MCP 在生产环境的 5 大挑战

### 7.1 安全性

**最大风险**：MCP server 能让 LLM 调用任何 tool — 包括删除文件、跑命令、转账。

**防护**：
- **白名单 tool**：只暴露必要的 tool（不要把整个 filesystem 暴露给不信任的 agent）
- **Permission scopes**：tool 内置权限检查（例：`read_file` vs `delete_file`）
- **HitL approval**：高风险 tool 走 U33 HITL
- **沙箱执行**：stdio server 跑在 Docker / microVM 里

### 7.2 Tool 数量膨胀

**问题**：MCP server 一旦多起来，agent 看到的 tool list 可能 100+，context 爆炸 + LLM 选择困难。

**解决方案**：
- **Tool ranking**：按相关性排序
- **Tool grouping**：按 domain 分组（"weather tools" / "DB tools"）
- **Lazy loading**：agent 自己 `request_tool(category)`（类似 progressive disclosure）
- **Tool 描述精炼**：description < 100 tokens

### 7.3 跨 server 状态管理

**问题**：MCP server 是无状态的，但 agent 状态跨 server 共享。

**解决方案**：
- Server 端：state 存 Redis / Postgres，session_id 作为 key
- Client 端：在 agent state 里维护 cross-server context

### 7.4 错误处理

```python
@mcp.tool()
async def fetch_data(query: str) -> str:
    try:
        result = await external_api(query)
        return result
    except TimeoutError:
        return "Request timed out, please try a simpler query"
    except Exception as e:
        return f"Error: {e}"
```

**原则**：永远返回 string（不让 exception 传到 LLM）；错误信息要 LLM 友好（说明下一步怎么做）。

### 7.5 可观测性

```python
from mcp.server.fastmcp import FastMCP
import logging

logger = logging.getLogger("mcp.server")

mcp = FastMCP("MyServer")

@mcp.tool()
async def my_tool(x: int) -> int:
    logger.info(f"my_tool called with x={x}")
    return x * 2
```

**追踪**：
- 用 OpenTelemetry 追踪 MCP call
- 上报 metrics：tool call count / latency / error rate
- 关联到 agent 的 trace（U35 AgentOps）

---

## 8. MCP 生态项目（2025）

| 项目 | 类型 | Stars | 用途 |
|------|------|-------|------|
| **MCP 官方**（modelcontextprotocol） | 规范 | - | https://modelcontextprotocol.io |
| **FastMCP**（Python） | 框架 | - | 快速写 Server |
| **MCP TypeScript SDK** | 框架 | - | TS 写 Server |
| **langchain-mcp-adapters** | 集成 | 3.5k+ | LangChain 接入 MCP |
| **mcp-cli** | CLI | - | 调试 MCP server |
| **@modelcontextprotocol/server-filesystem** | Server | - | filesystem 访问 |
| **@modelcontextprotocol/server-github** | Server | - | GitHub 集成 |
| **@modelcontextprotocol/server-postgres** | Server | - | PostgreSQL |
| **notion-mcp** | Server | - | Notion |
| **mcp-proxy** | 工具 | - | stdio ↔ HTTP 转换 |
| **inspector** | 调试 | - | MCP 调试器（类似 Postman） |

---

## 9. 实战：构建生产级 MCP Server

```python
# production_server.py
import os
import logging
from contextlib import asynccontextmanager
from mcp.server.fastmcp import FastMCP, Context

logger = logging.getLogger(__name__)

mcp = FastMCP("ProductionServer")

# 1. Tool with validation + auth
@mcp.tool()
async def query_db(sql: str, ctx: Context) -> list[dict]:
    """Execute a read-only SQL query against the analytics database.
    
    Use this when you need to fetch aggregated metrics. Always LIMIT results to 1000.
    """
    # 检查 ctx（per-request auth）
    user_id = ctx.request_context.get("user_id")
    if not user_id:
        return [{"error": "Authentication required"}]
    
    # 验证只读
    if not sql.strip().lower().startswith("select"):
        return [{"error": "Only SELECT queries are allowed"}]
    
    # 强制 LIMIT
    if "limit" not in sql.lower():
        sql += " LIMIT 1000"
    
    # 执行（带 timeout）
    try:
        async with asyncio.timeout(10):
            rows = await db.fetch(sql)
        return [dict(row) for row in rows]
    except Exception as e:
        logger.error(f"DB error: {e}", extra={"user_id": user_id})
        return [{"error": str(e)}]

# 2. Resource with caching
@mcp.resource("schema://users")
async def users_schema() -> str:
    """Get the users table schema"""
    return await cache.get_or_fetch("users_schema", db.get_schema, ttl=3600)

# 3. Prompt template
@mcp.prompt()
def summarize_incident(incident_id: str) -> str:
    return f"""Please investigate incident {incident_id} using the following tools:
- Use `query_db` to check recent error rates
- Use `fetch_logs` to retrieve relevant log entries  
- Use `create_jira_ticket` to create a tracking issue

Be specific and cite evidence from tool results."""

# 4. Lifecycle
@asynccontextmanager
async def lifespan():
    # 启动
    await db.connect()
    logger.info("MCP server started")
    yield
    # 关闭
    await db.disconnect()
    logger.info("MCP server stopped")

if __name__ == "__main__":
    mcp.run(
        transport="streamable-http",
        url="0.0.0.0:8000",
        lifespan=lifespan,
    )
```

---

## 10. 调试 MCP

### 10.1 MCP Inspector（官方调试器）

```bash
# 启动 MCP Inspector
npx @modelcontextprotocol/inspector

# 浏览器打开 http://localhost:5173
# 连接到 server.py，列出所有 tool / resource / prompt
# 直接测试每个 tool 的输入输出
```

### 10.2 常见调试问题

| 问题 | 原因 | 解决 |
|------|------|------|
| Client 找不到 server | 路径错 / 权限不足 | 检查 `command` 和 `args` 绝对路径 |
| Tool 没出现 | 装饰器没生效 | 检查 `@mcp.tool()` 而不是 `@tool` |
| 调 tool 报错 | 参数类型不匹配 | 检查 Python type hints |
| Server 卡住 | 死锁 / 异步 bug | 加 timeout |
| Streamable HTTP 404 | URL 路径错 | 默认是 `/mcp` 不是 `/` |

---

## 11. 面试视角

- **Q：MCP 是什么？解决了什么问题？**
  → Anthropic 2024-11 推出的"USB-C for LLM applications"——统一工具协议。一份 Server，所有支持 MCP 的客户端（Claude Desktop、Cursor、LangChain）都能用。**解决重复集成问题**。

- **Q：MCP 三大原语？**
  → Tools（可调用函数）/ Resources（可读上下文，GET 端点）/ Prompts（user-controlled 模板）。

- **Q：MCP 两种传输？**
  → stdio（本地进程，最快，无网络）和 Streamable HTTP（远程服务，多客户端，2025-03 新标准）。

- **Q：MCP vs Function Calling？**
  → Function Calling：模型厂商定义，prompt 内嵌 schema，HTTP 调 LLM。MCP：开放协议，独立 Server，stdio/HTTP，多客户端共享一份 Server。**MCP 是 Function Calling 的标准化和工具复用**。

- **Q：MCP vs A2A？**
  → MCP = Agent ↔ 工具/数据（**纵向**）。A2A = Agent ↔ Agent（**横向**）。两者**互补**。

- **Q：MCP 的 5 大生产挑战？**
  → ① 安全性（tool 权限、HitL 审批、沙箱）② Tool 数量膨胀（ranking、grouping、lazy load）③ 跨 server 状态管理（Redis / Postgres）④ 错误处理（返回 string 而非 exception）⑤ 可观测性（OpenTelemetry trace）。

- **Q：怎么用 LangChain 接 MCP？**
  → `langchain-mcp-adapters` 库。`MultiServerMCPClient({...})` 同时加载多个 server，`load_mcp_tools(session)` 转 LangChain tools，`create_agent(model, tools)` 创建 agent。

- **Q：MCP Streamable HTTP vs HTTP+SSE？**
  → Streamable HTTP 是 2025-03 新规范，单 endpoint 双向 stream，支持 stateless + resumption。HTTP+SSE 已被 deprecated。

- **Q：FastMCP 怎么用？**
  → `FastMCP("name")` → `@mcp.tool()` / `@mcp.resource()` / `@mcp.prompt()` 装饰器 → `mcp.run(transport="stdio" or "streamable-http")`。docstring → 工具 description，type hints → 参数 schema。

---

## 12. 必读材料

| 材料 | 链接 | 为什么读 |
|------|------|----------|
| **MCP 官方文档** | [modelcontextprotocol.io](https://modelcontextprotocol.io/) | 权威规范 |
| **MCP 规范 2025-03-26** | [modelcontextprotocol.io/specification/2025-03-26](https://modelcontextprotocol.io/specification/2025-03-26/basic) | 协议细节 |
| **Anthropic: Introducing MCP** | [anthropic.com/news/model-context-protocol](https://www.anthropic.com/news/model-context-protocol) | 发布博客 |
| **FastMCP** | [github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp) | Python 快速写 Server |
| **langchain-mcp-adapters** | [github.com/langchain-ai/langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) | LangChain 集成 |
| **MCP Inspector** | [github.com/modelcontextprotocol/inspector](https://github.com/modelcontextprotocol/inspector) | 调试器（类似 Postman） |
| **MCP Servers 集合** | [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) | 官方 server 实现（filesystem / github / postgres...）|
| **Claude Desktop MCP 集成** | [code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp) | Claude Code 集成 MCP |

---

## 13. 练习

1. **写一个简单 MCP Server**：用 FastMCP 写一个 todo 工具（add / list / complete），stdio 启动
2. **用 MCP Inspector 调试**：连到你写的 server，测试每个 tool
3. **多 server 集成**：写两个 server（math + weather），用 `MultiServerMCPClient` 同时加载
4. **Streamable HTTP**：把 server 改成 HTTP 模式，client 跨进程连接
5. **接入 LangGraph**：用 `langchain-mcp-adapters` 把 MCP tools 接入 LangGraph agent
6. **生产级实践**：给 server 加 auth / rate limit / OTel / error handling，跑 1000 次 tool call

---

## 14. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U6 Tool Use & Function Calling | MCP 是 Function Calling 的标准化 |
| U9 Multi-Agent | Agent 跨框架协作 |
| U13 A2A 协议 | A2A = Agent ↔ Agent（横向），MCP = Agent ↔ 工具（纵向）|
| U22 Context Engineering | MCP Resources 是 Context 的来源之一 |
| U23 Harness Engineering | MCP Server 是 Harness 组件 3（Tool Access）的实现 |
| U29 Agent Deploy | MCP stdio server 跑在 Docker 沙箱里（U29 部署）|
| U33 Human-in-the-loop | MCP 高风险 tool 走 HitL 审批 |
| U34 Guardrails | MCP 工具调用可加 Guardrails（input/output 验证）|

---

*最后更新：2026-06-30 新增 U36（MCP 深入）*
*来源：[MCP 官方文档](https://modelcontextprotocol.io) · [Anthropic 发布博客](https://www.anthropic.com/news/model-context-protocol) · [FastMCP](https://github.com/jlowin/fastmcp) · [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) · [Claude Code best practices](https://code.claude.com/docs/en/best-practices)*
