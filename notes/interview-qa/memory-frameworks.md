# Interview QA · Memory 框架篇

> 来源：U11-Memory框架对比 + 主流框架文档
> 覆盖：LangChain / CrewAI / AutoGen / smolagents / LlamaIndex

---

## 一、短期记忆（Short-term Memory）

### Q: 各框架如何实现短期记忆？

| 框架 | 实现方式 | 代码示例 |
|------|---------|---------|
| **LangChain** | 完整支持，4种类型 | `ConversationBufferWindowMemory(k=5)` |
| **CrewAI** | 封装LangChain，`memory=True` | 黑盒设计 |
| **AutoGen** | 无内置，需手动维护 | 手动 `history = []` |
| **smolagents** | 无，依赖 context window | 不适用 |
| **LlamaIndex** | 无 | 不适用 |

### Q: LangChain 短期记忆有哪些类型？

**Answer:**

| 类型 | 类 | 特点 |
|------|-----|------|
| Buffer | `ConversationBufferMemory` | 保留所有历史 |
| Window | `ConversationBufferWindowMemory` | 只留最近N条 |
| Summary | `ConversationSummaryMemory` | LLM生成摘要 |
| Entity | `EntityMemory` | 记住实体（人/地点/组织）|

```python
# 滑动窗口示例
from langchain.memory import ConversationBufferWindowMemory
memory = ConversationBufferWindowMemory(memory_key="chat_history", k=5, return_messages=True)
```

### Q: AutoGen 没有内置 Memory 怎么实现？

**Answer:**

```python
# 手动维护历史
history = []
def chat_with_history(agent, message):
    history.append({"role": "user", "content": message})
    response = agent.generate_reply(messages=history)
    history.append({"role": "assistant", "content": response})
    return response
```

---

## 二、长期记忆（Long-term Memory）

### Q: 各框架如何实现长期记忆？

| 框架 | 长期记忆支持 | 实现方式 |
|------|------------|---------|
| **LangChain** | 完整 | `VectorStoreRetrieverMemory` + FAISS/Chroma/Pinecone |
| **CrewAI** | Crew级别 | `Crew(memory=True)` 跨任务共享 |
| **LlamaIndex** | 索引驱动 | `VectorStoreIndex` 检索而非独立Memory |
| **AutoGen** | 无 | 需自行接入向量库 |
| **smolagents** | 无 | 需自行接入向量库 |

### Q: LangChain VectorStore-backed Memory 如何用？

**Answer:**

```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

memory = VectorStoreRetrieverMemory(
    vectorstore=FAISS.from_documents([], OpenAIEmbeddings()),
    retriever_kwargs={"k": 3},
)

memory.save_context({"input": "用户喜欢简洁代码"}, {"output": "已记录"})
relevant = memory.load_memory_variables({"input": "这次怎么写"})
```

### Q: LangChain 和 LlamaIndex 长期记忆的区别？

**Answer:**
- **LangChain**：独立的 Memory 模块，save/recall 接口
- **LlamaIndex**：本质是索引驱动的检索，不是独立 Memory 模块，通过 QueryEngine 跨会话检索

---

## 三、反思机制（Reflection）

### Q: Reflexion 反思机制在各框架如何实现？

| 框架 | 反思机制 | 实现方式 |
|------|---------|---------|
| LangChain | 需手动实现 | `ConversationSummaryMemory` + 自定义循环 |
| CrewAI | 内置 | Hierarchical Process + Manager Agent |
| AutoGen | 需手动编排 | GroupChat + 评估Agent |
| smolagents | 无 | 不支持 |
| LlamaIndex | 无 | 不适用 |

### Q: LangChain 如何实现 Reflexion？

**Answer:**

```python
def reflexion_loop(task: str, max_attempts: int = 3):
    for attempt in range(max_attempts):
        result = agent.run(task)
        if evaluate(result):
            return result  # 成功
        # 反思失败原因
        反思文本 = reflect(task, result, success=False)
        memory.save_context(
            {"input": f"上次任务：{task}"},
            {"output": 反思文本}
        )
```

---

## 四、框架对比总结

### Q: 什么场景选什么框架？

**Answer:**

```
需要 Memory？
├── 只需要短期（对话）
│   ├── 需要高定制 → LangChain
│   └── 快速上手 → CrewAI
│
├── 需要短期 + 长期
│   ├── LangChain（完整支持）
│   └── CrewAI（开箱即用）
│
├── 多Agent协作
│   ├── 串行任务 → CrewAI Sequential
│   └── 需要协调者 → CrewAI Hierarchical
│
└── 只需要长期（检索）
    └── LlamaIndex
```

### Q: 框架 Memory 能力总览？

| 维度 | LangChain | CrewAI | AutoGen | smolagents | LlamaIndex |
|------|-----------|--------|---------|------------|------------|
| 定位 | 全功能框架 | Multi-Agent编排 | 多Agent对话 | 轻量级Agent | 数据检索专家 |
| 短期记忆 | 完整 | 基础 | 无 | 无 | 无 |
| 长期记忆 | 完整 | Crew级别 | 无 | 无 | 索引驱动 |
| 反思机制 | 需手动 | 内置 | 需编排 | 无 | 无 |
| 学习曲线 | 陡峭 | 平缓 | 平缓 | 最平缓 | 中等 |

---

## 五、统一 Memory 接口

### Q: 如何设计跨框架的统一 Memory 接口？

**Answer:**

```python
from abc import ABC, abstractmethod
from typing import List

class BaseMemory(ABC):
    @abstractmethod
    def save(self, key: str, value: str) -> None:
        pass

    @abstractmethod
    def recall(self, query: str, k: int = 3) -> List[str]:
        pass

    @abstractmethod
    def clear(self) -> None:
        pass

class LangChainMemory(BaseMemory):
    def __init__(self, k: int = 5):
        from langchain.memory import ConversationBufferWindowMemory
        self.memory = ConversationBufferWindowMemory(k=k)

    def save(self, key: str, value: str) -> None:
        self.memory.save_context({"input": key}, {"output": value})

    def recall(self, query: str, k: int = 3) -> List[str]:
        result = self.memory.load_memory_variables({})
        return [result.get("history", "")]

    def clear(self) -> None:
        self.memory.clear()
```

---

## 六、关联资源

- [LangChain Memory 文档](https://python.langchain.com/docs/concepts/memory/)
- [CrewAI Memory](https://docs.crewai.com/concepts/memory)
- [AutoGen ConversableAgent](https://microsoft.github.io/autogen/)
- [Reflexion 论文](https://arxiv.org/abs/2303.11366)
- [mem0（专门Agent记忆层）](https://github.com/mem0ai/mem0)

---

*最后更新：2026-05-27*
