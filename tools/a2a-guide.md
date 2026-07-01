# Tool Guide · A2A (Agent-to-Agent Protocol)

> 覆盖：概念 / Agent Card / 快速写 Server/Client / 与 MCP 区分 / 面试高频问法

---

## 1. 核心概念

```
Agent A (A2A Client)
    ↓ HTTP + JSON (A2A Protocol)
Agent B (A2A Server)
```

**A2A = Agent-to-Agent 协议**，让两个 Agent 能够互相通信、协作完成任务

**解决的问题**：
- Agent 需要互相调用、共享上下文
- MCP 解决的是 Agent 与工具的通信
- A2A 解决的是 Agent 与 Agent 的通信

**关键组件**：
- **Agent Card**：描述 Agent 能力的元数据（JSON）
- **Task**：A2A 中的工作单元，带状态（pending/in-progress/completed）
- **Message**：Agent 间的消息格式

---

## 2. Agent Card

Agent Card 是 A2A 的核心，用于服务发现：

```json
{
  "name": "Code Review Agent",
  "description": "Review code for bugs and style issues",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "code-review",
      "name": "Code Review",
      "description": "Performs code review on provided source code"
    }
  ]
}
```

---

## 3. 快速写一个 A2A Server

### 3.1 基础 Server

```bash
pip install a2a-sdk
```

```python
from a2a.server import A2AServer
from a2a.types import Task, AgentCard

agent_card = AgentCard(
    name="Math Agent",
    description="Performs mathematical calculations",
    version="1.0.0",
    capabilities={"streaming": True, "pushNotifications": False},
    skills=[]
)

server = A2AServer(agent_card=agent_card)

@server.agent_handler("math")
async def calculate(expression: str) -> str:
    result = eval(expression)
    return str(result)

if __name__ == "__main__":
    server.start(host="0.0.0.0", port=8000)
```

### 3.2 带 Task 的 Server

```python
from a2a.server import A2AServer
from a2a.types import Task, TaskStatus

@server.agent_handler("process")
async def process_task(task: Task) -> Task:
    # 处理任务
    task.status = TaskStatus.COMPLETED
    task.output = {"result": "processed"}
    return task
```

---

## 4. 快速写一个 A2A Client

```python
from a2a.client import A2AClient

async def main():
    client = A2AClient("http://localhost:8000")
    
    # 发现 Agent
    agent_card = await client.discover_agent()
    print(f"Connecting to: {agent_card.name}")
    
    # 发送任务
    task = await client.send_task({
        "skill": "math",
        "input": {"expression": "2 + 3 * 4"}
    })
    
    print(f"Result: {task.output}")

asyncio.run(main())
```

---

## 5. A2A vs MCP

| 维度 | A2A | MCP |
|------|-----|-----|
| 定位 | Agent 与 Agent 通信 | Agent 与工具/服务通信 |
| 场景 | 多 Agent 协作 | 单 Agent 调用外部工具 |
| 协议层 | HTTP + JSON | stdio / HTTP + JSON |
| 状态 | 有 Task 状态管理 | 无状态 |

**关系**：A2A 和 MCP 是互补的，通常 Agent 既用 MCP 调用工具，又用 A2A 与其他 Agent 协作

---

## 6. 面试高频问法

### Q: A2A 协议是什么？解决什么问题？

→ Agent-to-Agent 协议，让多个 Agent 能够互相通信协作
→ 解决多 Agent 系统中的 Agent 间通信问题

### Q: A2A 和 MCP 的区别？

→ MCP：Agent 调用外部工具/服务
→ A2A：Agent 之间互相通信协作
→ 两者互补，Agent 通常同时使用两者

### Q: Agent Card 的作用？

→ 描述 Agent 能力的元数据，用于服务发现
→ 包含 name、description、capabilities、skills 等

### Q: A2A 的 Task 是什么？

→ A2A 中的工作单元，带状态（pending/in-progress/completed）
→ 用于跟踪协作任务的进度

---

*最后更新：2026-05-27*