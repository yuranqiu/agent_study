# Tool Guide · LangChain / LangGraph

> 覆盖：安装 / 核心概念 / 最小可运行代码 / 常见坑点 / 面试高频问法

---

## 1. 安装

```bash
pip install langchain langchain-openai langchain-community langgraph
```

---

## 2. 核心概念速查

### 2.1 模块对应关系

| 模块 | 作用 | 关键类 |
|------|------|--------|
| `langchain-core` | 基础抽象：Runnable, PromptTemplate, OutputParser | `BaseLanguageModel` |
| `langchain-openai` | OpenAI GPT 系列接口 | `ChatOpenAI` |
| `langchain-community` | 第三方集成：FAISS, Chroma, SerpAPI | 各种 VectorStore |
| `langgraph` | 有状态 Multi-Agent 编排，支持 Cycles | `StateGraph`, `END` |

### 2.2 关键概念

**Runnable**：一切皆可链（`.invoke()` / `.batch()` / `.stream()`）
**Chain**：用 `|` pipe 组装顺序调用
**Agent**：动态选 Tool 的决策者
**Memory**：跨会话记住对话内容
**Retriever**：从向量库检索相关文档

---

## 3. 最小可运行代码

### 3.1 Basic Chain（翻译链）

```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

prompt = PromptTemplate.from_template("把 {text} 翻译成 {language}:")
llm = ChatOpenAI(model="gpt-4o-mini")

chain = prompt | llm
result = chain.invoke({"text": "你好", "language": "English"})
print(result.content)
```

### 3.2 带 Output Parser

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser

chain = prompt | llm | CommaSeparatedListOutputParser()
result = chain.invoke({"text": "编程语言", "language": "中文"})
# result = ["Python", "Java", "JavaScript"]
```

### 3.3 RAG Chain（检索增强生成）

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain.chains import RetrievalQA

# 建向量库
vectorstore = FAISS.from_texts(docs, OpenAIEmbeddings())
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# RAG chain
qa = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    retriever=retriever
)
result = qa.invoke("什么是 RAG？")
```

### 3.4 ReAct Agent（LangGraph 版）

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini")
tools = [search_tool, calculator_tool]

agent = create_react_agent(model, tools)
result = agent.invoke({"messages": [("user", "北京天气怎么样？")]})
```

### 3.5 自定义 Tool

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """查询城市天气"""
    return f"{city} 天气晴，25°C"

# 用法：
# agent = create_react_agent(model, [get_weather])
```

### 3.6 Memory（对话记忆）

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain

memory = ConversationBufferMemory()
conversation = ConversationChain(llm, memory=memory)

conversation.invoke("我叫张三")
conversation.invoke("我叫什么？")
# 输出：张三
```

---

## 4. LangGraph（状态机 + Cycles）

### 4.1 核心概念

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class State(TypedDict):
    messages: list
    step: str

graph = StateGraph(State)
graph.add_node("process", lambda state: {"step": "done"})
graph.set_entry_point("process")
graph.add_edge("process", END)
app = graph.compile()
```

### 4.2 Conditional Edge（条件分支）

```python
def route(state):
    if state["step"] == "done":
        return "end"
    return "continue"

graph.add_conditional_edges("process", route, {"end": END, "continue": "process"})
```

### 4.3 Checkpointer（持久化状态）

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 带 thread_id 恢复对话
config = {"configurable": {"thread_id": "user-123"}}
result = app.invoke({"messages": [("user", "hi")]}, config)
```

---

## 5. 常见坑点

| 坑 | 说明 | 解法 |
|----|------|------|
| `Provider not found` | 没装对应集成包 | `pip install langchain-openai` |
| Tool 不生效 | `create_react_agent` 没传 tools | `create_react_agent(model, tools=[...])` |
| Memory 太长 | 对话历史无限增长 | 用 `ConversationWindowMemory` 或 Summary |
| Token 超出 | prompt + context 超过 window | 加 `max_tokens` 或截断 |
| LCEL pipe 报错 | 前一步输出格式不匹配后一步 | 检查 Runnable 接口，必要时加 `RunnableLambda` |
| Stream 不 work | 没用 `.stream()` | chain 是 Runnable 就能 `.stream()` |

---

## 6. 面试高频问法

### Q: LangChain 的核心组件？

→ Models / Prompts / Chains / Agents / Memory / Indexes
→ 其中 Chains = 顺序组合，Agents = 动态决策

### Q: LCEL 是什么？

→ LangChain Expression Language，用 `|` pipe 组合 chain
→ 统一 Runnable 接口，支持同步/异步/流式

### Q: LangChain 和 LangGraph 的区别？

→ LangChain：固定顺序的 Chain，适合简单流程
→ LangGraph：添加状态机 + Cycles + 人机交互，适合复杂 Agent 编排

### Q: 如何实现多 Agent 协作？

→ 用 LangGraph 的 `StateGraph` + `add_node()` 添加多个 Agent
→ 用 `add_conditional_edges` 实现路由

### Q: LangSmith 是什么？

→ 可观测平台，自动追踪 chain/agent 执行轨迹
→ 环境变量集成，无需改代码

---

*最后更新：2026-05-27*