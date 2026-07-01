# Tool Guide · smolagents

> 覆盖：安装 / 核心概念 / 最小可运行代码 / 面试高频问法

---

## 1. 安装

```bash
pip install smolagents
```

---

## 2. 核心概念

**LangChain Agent**：通过 Tool Definition 让模型**选择**工具
**smolagents CodeAgent**：模型**直接写 Python 代码**执行，不需要预定义 Tool

→ 模型生成代码 → execute → 返回结果
→ 更轻量，但要求模型有强代码生成能力

---

## 3. 最小可运行代码

### 3.1 CodeAgent（直接生成代码执行）

```python
from smolagents import CodeAgent, LocalAgent

agent = CodeAgent(model="gpt-4o-mini", tools=[])
result = agent.run("帮我写一个快速排序算法")
print(result)
```

### 3.2 带 Tool 的 CodeAgent

```python
from smolagents import CodeAgent
from smolagents.tools import WebSearchTool, PythonExecuteTool

agent = CodeAgent(
    model="gpt-4o-mini",
    tools=[WebSearchTool(), PythonExecuteTool()]
)
result = agent.run("搜索 AI 最新进展并写一段测试代码")
```

### 3.3 Multi-Agent

```python
from smolagents import MultiAgent

researcher = CodeAgent(model="gpt-4o-mini", name="researcher")
writer = CodeAgent(model="gpt-4o-mini", name="writer")

team = MultiAgent(agents=[researcher, writer])
result = team.run("研究量子计算最新进展并写成文章")
```

---

## 4. 工具类型

| Tool | 作用 |
|------|------|
| `PythonExecuteTool` | 执行 Python 代码 |
| `WebSearchTool` | 搜索网页 |
| ` visitscrapeTool` | 抓取网页内容 |
| `FileWriteTool` | 写文件 |
| `BashTool` | 执行 shell 命令 |

---

## 5. 面试高频问法

### Q: smolagents 和 LangChain Agent 的核心区别？

→ LangChain：通过 Tool Definition 让模型选择工具
→ smolagents：模型直接生成 Python 代码执行，不需要预定义 Tool

### Q: 什么时候用 smolagents？

→ 需要轻量嵌入的场景
→ 模型代码生成能力强
→ 不需要复杂工具抽象

### Q: CodeAgent 的执行流程？

→ 模型生成 Python 代码 → 执行 → 返回结果
→ 代码在沙箱环境运行，保证安全

---

*最后更新：2026-05-27*