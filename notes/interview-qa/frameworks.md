# Interview QA · 框架篇

> 来源：Navedphysicist/100-Agentic-AI-Interview-Questions
> 覆盖：LangChain / CrewAI / AutoGen / MCP / smolagents / Dify

---

## 一、LangChain 篇

### Q: LangChain 是什么？解决了什么问题？

**Answer:**
- 开源框架，构建 LLM 应用的全生态工具箱
- 解决：LLM 与外部工具/数据/流程的集成问题
- 核心模块：Models / Prompts / Chains / Agents / Memory / Indexes

---

### Q: LangChain Expression Language (LCEL) 是什么？

**Answer:**
- 用 `|`  pipe 操作符组合 chain 组件的声明式语言
- 统一 Runnable 接口：同步/异步/流式/批处理统一抽象
- 示例：
```python
prompt | model | output_parser
```
- 优势：声明式组合、可链式调用、统一的流式支持

---

### Q: LangChain 中 Chain 和 Agent 的区别？

**Answer:**
- **Chain**：固定顺序执行，输出传给下一步（像管道）
- **Agent**：动态决策，根据输入决定下一步调用哪个 Tool
- Chain 是"预设流程"，Agent 是"自主决策"
- LangGraph 提供了带状态机 + Cycles 的 Agent 编排能力

---

### Q: LangChain 支持哪些 Memory 类型？

**Answer:**
- **ConversationBufferMemory**：保留所有历史消息
- **ConversationWindowMemory**：只保留最近 N 条
- **EntityMemory**：记住实体（人/地点/组织）
- **SummaryMemory**：用 LLM 摘要历史
- **VectorStore-backed Memory**：语义检索长期记忆
- **组合使用**：短期用 Buffer，长期用 Vector

---

### Q: LangChain 如何实现 RAG？

**Answer:**

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from langchain_openai import ChatOpenAI
from langchain.chains import RetrievalQA

vectorstore = FAISS.from_texts(docs, OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

qa_chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    retriever=retriever
)
result = qa_chain.invoke("问题")
```

---

### Q: LangChain Agent 有哪些类型？

**Answer:**

| Agent | 描述 |
|-------|------|
| `zero-shot-react-description` | 通用 ReAct，根据 tool description 选择 |
| `conversational-react-description` | 对话式，带 Memory |
| `chat-with-select-tool` | 多 Tool 选择场景 |
| `Self-ask-with-search` | 结合搜索的自我提问 |
| `ReAct-docstore` | 结合文档库的 ReAct |

---

### Q: LangSmith 是什么？和 LangChain 的关系？

**Answer:**
- LangSmith = LangChain 的可观测平台
- 自动追踪 chain/agent 执行轨迹、捕获状态转换
- 支持：Prompt 测试、数据集管理、性能指标、Cost 追踪
- 通过环境变量无缝集成，无需改代码

---

## 二、CrewAI 篇

### Q: CrewAI 核心概念是什么？

**Answer:**
- **Agent** = role + goal + backstory + tools
- **Task** = description + expected_output + agent + async_execution
- **Crew** = agents + tasks + process(sequential/hierarchical)
- **Process**：sequential（串行）/ hierarchical（层级管理器）

---

### Q: CrewAI 的 Sequential 和 Hierarchical Process 区别？

**Answer:**
- **Sequential**：任务按顺序执行，前一个输出作为下一个输入
- **Hierarchical**：Manager Agent 协调，动态分配任务给 Worker
- 选择：线性流程用 Sequential；需要协调者用 Hierarchical

---

### Q: CrewAI 如何传递 Task 上下文？

**Answer:**

```python
research_task = Task(description="研究 AI 趋势", agent=researcher)
writing_task = Task(
    description="写文章",
    agent=writer,
    context=[research_task]  # 前置任务结果传入
)
# writer agent 会收到 research_task 的输出作为 context
```

---

### Q: CrewAI 如何添加自定义 Tool？

**Answer:**

```python
from crewai import Agent
from crewai.tools import tool

@tool("Database Search")
def search_db(query: str) -> str:
    """搜索数据库，args: query - 搜索词"""
    return f"Results for: {query}"

analyst = Agent(
    role="数据分析师",
    goal="分析数据",
    tools=[search_db]
)
```

---

### Q: CrewAI Memory 如何开启？

**Answer:**
- Agent 级别：`Agent(..., memory=True)`
- Crew 级别：`Crew(..., memory=True)`
- 存储任务输出、交互上下文，支持跨任务记忆

---

## 三、AutoGen 篇

### Q: AutoGen 的核心组件是什么？

**Answer:**
- **AssistantAgent**：LLM 驱动，执行代码/生成响应
- **UserProxyAgent**：模拟用户，可触发代码执行/人类输入
- **GroupChat**：多 Agent 群聊协调
- **ConversableAgent**：基类，支持自定义对话行为

---

### Q: AutoGen 如何实现多 Agent 对话？

**Answer:**

```python
from autogen import AssistantAgent, UserProxyAgent

assistant = AssistantAgent("assistant", llm_config={"model": "gpt-4o-mini"})
user = UserProxyAgent("user")

user.initiate_chat(assistant, message="写一个快排")
# 自动来回对话，直到完成或达到 max_turn
```

---

### Q: AutoGen 的代码执行机制？

**Answer:**
- UserProxyAgent 内置 code execution
- 设置 `code_execution_config={"work_dir": ".", "use_docker": False}`
- 支持 Python 代码自动执行（需要 Python 环境）
- 安全性：通过 code_execution_config 限制可执行的操作

---

## 四、MCP 篇

### Q: MCP 是什么？解决了什么问题？

**Answer:**
- MCP (Model Context Protocol) = AI 应用与外部工具的标准协议
- 解决 N×M 问题：不需要为每个模型 × 每个工具写独立适配
- 核心角色：
  - **MCP Host**：AI 应用（Claude Desktop、Cursor）
  - **MCP Client**：Host 内的客户端
  - **MCP Server**：暴露工具/资源/prompt 的服务端

---

### Q: MCP 和 Function Calling 的区别？

**Answer:**

| 维度 | MCP | Function Calling |
|------|-----|-----------------|
| 范围 | 协议标准 | Provider 特定 API |
| 模型支持 | 跨所有 LLM | 仅特定模型 |
| 能力 | Tools + Resources + Prompts | 仅 Functions |
| 发现机制 | 自动 list_tools | 手动告知 |
| 传输 | stdio/HTTP/SSE | HTTP |

---

### Q: 用 FastMCP 写一个 MCP Server

**Answer:**

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

---

### Q: LangChain 如何接入 MCP Tools？

**Answer:**

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "math": {"transport": "stdio", "command": "python", "args": ["math_server.py"]}
})

tools = await client.get_tools()
agent = create_agent("claude-sonnet-4-5-20250929", tools)
result = await agent.ainvoke({"messages": [{"role": "user", "content": "Add 5 and 3"}]})
```

---

## 五、smolagents 篇

### Q: smolagents 和 LangChain Agent 的核心区别？

**Answer:**
- **LangChain**：通过 Tool Definition 让模型选择工具
- **smolagents (CodeAgent)**：模型直接写 Python 代码执行，不需要预定义 Tool
- CodeAgent = 模型自己生成代码 → 执行 → 返回结果
- 更轻量，但需要模型有较强的代码生成能力

---

### Q: smolagents 的核心用法

**Answer:**

```python
from smolagents import CodeAgent, LocalAgent

agent = CodeAgent(model="gpt-4o-mini", tools=[])
result = agent.run("帮我写一个排序算法")
# 模型直接生成 Python 代码并执行
```

---

## 六、Dify 篇

### Q: Dify 是什么？和 LangChain 的区别？

**Answer:**
- Dify = 开源 LLM App 平台，有 UI，可视化编排
- LangChain = 代码框架，需要写 Python
- Dify 适合：不想写代码的业务人员，快速搭 AI 应用
- Dify 底层调用 LangChain / LangGraph

---

### Q: Dify 能搭什么类型的应用？

**Answer:**
- Chatbot（单轮/多轮）
- Agent（带 ReAct/Function Call）
- Workflow（流程编排）
- RAG Pipeline（知识库问答）
- 各类 AI 应用（客服/文档/审批）

---

## 五、LlamaIndex 篇

### Q: LlamaIndex 和 LangChain 的核心区别？

**Answer:**
- **LangChain**：全能框架，Tool/Chain/Memory/Agent 全覆盖，RAG 是其中一小部分
- **LlamaIndex**：专注 RAG + 数据检索，索引/检索质量深度优化
- 选择：如果场景 70% 是 RAG + 私有数据 → 用 LlamaIndex；需要复杂 Agent 编排 → 用 LangChain

---

### Q: LlamaIndex 的核心流程？

**Answer:**
- Document → Node（切分）→ Index（向量化）→ Retriever（检索）→ QueryEngine（组装 LLM 生成）
- 核心概念：Document / Node / Index / Retriever / QueryEngine / Response Synthesizer

---

### Q: Document / Node / Index 的关系？

**Answer:**
- **Document**：原始文档（PDF/Markdown/文本），含 text + metadata
- **Node**：Document 切分后的块，保留父子关系和 metadata
- **Index**：索引结构（Vector/Keyword/Composite），加速检索

---

### Q: 什么是 ComposableGraph？

**Answer:**
- 把多个索引组合成图，查询时自动跨索引检索
- 适用：不同数据源建不同索引，再组合查询

---

### Q: 如何提升 RAG 检索质量？

**Answer:**
1. 调 chunk_size（512 左右）+ chunk_overlap
2. 用元数据过滤缩小范围
3. 加 Node 后处理（重排序/关键词过滤）
4. 用 HyDE（假设性文档增强）
5. 用更好的 embedding（bge-large-zh）

---

### Q: LlamaIndex 支持哪些向量库？

**Answer:**
- 支持：Chroma / FAISS / Milvus / Pinecone / Weaviate / Qdrant
- 一切兼容 LangChain VectorStore 的都支持

---

*最后更新：2026-05-27*