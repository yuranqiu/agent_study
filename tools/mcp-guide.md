# Tool Guide · MCP (Model Context Protocol)

> 覆盖：概念 / FastMCP 写 Server / LangChain 接入 / 面试高频问法

---

## 1. 核心概念

```
MCP Host (AI 应用, Claude Desktop/Cursor)
    ↓ MCP Client
MCP Server (暴露 tools/resources/prompts)
```

**解决的问题**：每个模型 × 每个工具都要单独适配 → N×M 问题
**MCP = 统一协议**，一份 server 所有支持 MCP 的客户端都能用

---

## 2. 快速写一个 MCP Server（FastMCP）

```bash
pip install fastmcp
```

### 2.1 最小 Server

```python
from fastmcp import FastMCP

mcp = FastMCP("Calculator")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two numbers"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### 2.2 带 Resources

```python
@mcp.resource("doc://api")
def get_api_docs() -> str:
    """返回 API 文档"""
    return "# API Documentation\n\n..."

# 客户端连接后自动 discover resources
```

### 2.3 带 Prompts

```python
@mcp.prompt()
def code_review(code: str) -> str:
    """代码审查 prompt"""
    return f"Review this code:\n{code}"
```

### 2.4 HTTP Server

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http", url="http://0.0.0.0:8000/mcp")
```

---

## 3. LangChain 接入 MCP

```bash
pip install langchain-mcp-adapters
```

### 3.1 stdio 模式

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

async def main():
    client = MultiServerMCPClient({
        "math": {
            "transport": "stdio",
            "command": "python",
            "args": ["math_server.py"]
        }
    })

    tools = await client.get_tools()
    agent = create_agent("claude-sonnet-4-5-20250929", tools)
    result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "Add 5 and 3"}]
    })
    print(result)

asyncio.run(main())
```

### 3.2 HTTP 模式

```python
client = MultiServerMCPClient({
    "weather": {
        "transport": "streamable-http",
        "url": "http://localhost:8000/mcp"
    }
})

tools = await client.get_tools()
```

### 3.3 多 Server

```python
client = MultiServerMCPClient({
    "math": {"transport": "stdio", "command": "python", "args": ["math.py"]},
    "weather": {"transport": "streamable-http", "url": "http://localhost:8000/mcp"}
})
```

---

## 4. MCP vs Function Calling

| 维度 | MCP | Function Calling |
|------|-----|-----------------|
| 范围 | 协议标准 | Provider 特定 |
| 模型 | 跨所有 LLM | 仅特定模型 |
| 内容 | Tools + Resources + Prompts | 仅 Functions |
| 发现 | 自动 list_tools | 手动告知 |
| 传输 | stdio / HTTP / SSE | HTTP only |

---

## 5. 面试高频问法

### Q: MCP 是什么？解决了什么问题？

→ Model Context Protocol，AI 与外部工具的通信标准协议
→ 解决 N×M 集成问题：不需要为每个模型×每个工具单独适配

### Q: MCP Host / Client / Server 关系？

→ Host：AI 应用（Claude Desktop、Cursor）
→ Client：Host 内协议客户端
→ Server：暴露工具/资源/prompt 的独立进程

### Q: MCP 和 Function Calling 的核心区别？

→ Function Calling 是 Provider 特定（OpenAI 只能 OpenAI 模型用）
→ MCP 是跨模型的统一协议

### Q: 用 FastMCP 写 Server 的基本步骤？

→ `FastMCP("name")` → `@mcp.tool()` 装饰器 → `mcp.run(transport="stdio")`

---

*最后更新：2026-05-27*