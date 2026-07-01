# U11 · Memory 框架对比

> **核心问题**：各主流框架如何实现 Memory？优劣势是什么？
> **预计学时**：4h
> **对应 Phase**：Phase 2

---

## 学习目标（Gate 检验）

1. 能说出 LangChain / CrewAI / AutoGen / smolagents / LlamaIndex 的 Memory 实现差异
2. 能根据场景选择合适的框架 Memory 方案
3. 能解释短期记忆（滑动窗口）和长期记忆（向量库）在各框架的实现
4. 能实现一个跨框架的统一 Memory 接口

---

## 1. 为什么对比 Memory 框架

Memory 是 Agent 的核心组件，但各框架的设计思路差异巨大：

| 维度 | LangChain | CrewAI | AutoGen | smolagents | LlamaIndex |
|------|-----------|--------|---------|------------|------------|
| 定位 | 全功能框架 | Multi-Agent编排 | 多Agent对话 | 轻量级Agent | 数据检索专家 |
| Memory 支持 | 完整 | 基础 | 无内置 | 极简 | 仅长期记忆 |
| 学习曲线 | 陡峭 | 平缓 | 平缓 | 最平缓 | 中等 |

选错框架的代价：Memory 设计不兼容会导致后续扩展困难。

---

## 2. 短期记忆：滑动窗口 vs 摘要压缩

### 2.1 LangChain

```python
from langchain.memory import ConversationBufferWindowMemory

memory = ConversationBufferWindowMemory(
    memory_key="chat_history",
    k=5,  # 保留最近5轮
    return_messages=True,
)
```

LangChain 提供最完整的短期记忆实现：

| 类型 | 类 | 特点 |
|------|-----|------|
| Buffer | `ConversationBufferMemory` | 保留所有历史 |
| Window | `ConversationBufferWindowMemory` | 只留最近N条 |
| Summary | `ConversationSummaryMemory` | LLM生成摘要 |
| Entity | `EntityMemory` | 记住实体 |

### 2.2 CrewAI

```python
from crewai import Agent

agent = Agent(
    role="分析师",
    goal="分析数据",
    memory=True,  # 开启记忆
)
```

CrewAI 的短期记忆是黑盒设计，只暴露 `memory=True` 参数，底层使用 LangChain 的 Memory 实现。

### 2.3 AutoGen

AutoGen **没有内置短期记忆**，需要自己实现：

```python
from autogen import AssistantAgent

# 需要手动维护历史
history = []

def chat_with_history(agent, message):
    history.append({"role": "user", "content": message})
    response = agent.generate_reply(messages=history)
    history.append({"role": "assistant", "content": response})
    return response
```

### 2.4 smolagents

smolagents 的 CodeAgent 没有独立 Memory 模块，依赖模型自身的 context window。

### 2.5 对比总结

| 框架 | 短期记忆支持 | 实现方式 | 可定制性 |
|------|------------|----------|---------|
| LangChain | 完整 | 显式类 | 高 |
| CrewAI | 基础 | 封装LangChain | 低 |
| AutoGen | 无 | 需手动实现 | 灵活 |
| smolagents | 无 | 依赖context | 无 |
| LlamaIndex | 无 | 不适用 | 不适用 |

---

## 3. 长期记忆：向量库记忆

### 3.1 LangChain

```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

memory = VectorStoreRetrieverMemory(
    vectorstore=FAISS.from_documents([], OpenAIEmbeddings()),
    retriever_kwargs={"k": 3},
)

memory.save_context(
    {"input": "用户喜欢简洁代码"},
    {"output": "已记录"}
)

relevant = memory.load_memory_variables({"input": "这次怎么写"})
```

支持：FAISS / Chroma / Pinecone / Milvus / Weaviate 等。

### 3.2 LlamaIndex

LlamaIndex 的 Memory 本质上是 **索引驱动的检索**，不是独立的 Memory 模块：

```python
from llama_index.core import VectorStoreIndex

index = VectorStoreIndex.from_documents(docs)
query_engine = index.as_query_engine()

# 跨会话检索历史
result = query_engine.query("用户之前的偏好是什么")
```

### 3.3 CrewAI

CrewAI 的长期记忆通过 **Crew 级别的 memory** 实现：

```python
from crewai import Crew

crew = Crew(
    agents=[researcher, writer],
    tasks=[task1, task2],
    memory=True,  # Crew级别记忆，跨任务共享
)
```

底层使用向量库存储任务输出。

### 3.4 AutoGen / smolagents

均无内置长期记忆，需要自行接入外部向量库。

---

## 4. 反思机制：Reflexion 在各框架的实现

### 4.1 LangChain 的 Reflexion 支持

LangChain 通过 `ConversationSummaryMemory` 间接支持反思——LLM 生成摘要时隐含反思能力。

完整 Reflexion 需要自己实现：

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

### 4.2 CrewAI 的任务级反思

CrewAI 的 `Process` 支持 hierarchical 模式，Manager Agent 隐含评估和协调能力：

```python
crew = Crew(
    agents=[researcher, writer, reviewer],
    tasks=[...],
    process="hierarchical",  # Manager 协调 + 评估
)
```

### 4.3 AutoGen 的群聊评估

AutoGen 的 `GroupChat` 支持让多个 Agent 相互评估：

```python
group_chat = GroupChat(
    agents=[writer, reviewer, editor],
    max_turn=3,
)
```

reviewer 可以给 writer 提供反馈，隐含反思机制。

### 4.4 对比总结

| 框架 | 反思机制 | 实现方式 |
|------|---------|---------|
| LangChain | 需手动实现 | `ConversationSummaryMemory` + 自定义 |
| CrewAI | 内置 | Hierarchical Process + Manager Agent |
| AutoGen | 需手动编排 | GroupChat + 评估Agent |
| smolagents | 无 | 不支持 |
| LlamaIndex | 无 | 不适用 |

---

## 5. 框架选择决策树

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

### 场景选择

| 场景 | 推荐框架 | 原因 |
|------|---------|------|
| 复杂Agent定制 | LangChain | Memory类型最全 |
| 快速搭AI应用 | CrewAI | 开箱即用 |
| 多Agent对话 | AutoGen | 群聊机制成熟 |
| 极简需求 | smolagents | 无额外复杂度 |
| 知识库问答 | LlamaIndex | 检索深度优化 |

---

## 6. 实现：统一 Memory 接口

为了跨框架兼容性，抽象统一接口：

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any

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

# LangChain 实现
class LangChainMemory(BaseMemory):
    def __init__(self, k: int = 5):
        from langchain.memory import ConversationBufferWindowMemory
        from langchain_openai import ChatOpenAI
        self.memory = ConversationBufferWindowMemory(k=k)

    def save(self, key: str, value: str) -> None:
        self.memory.save_context({"input": key}, {"output": value})

    def recall(self, query: str, k: int = 3) -> List[str]:
        result = self.memory.load_memory_variables({})
        return [result.get("history", "")]

    def clear(self) -> None:
        self.memory.clear()

# 使用示例
if __name__ == "__main__":
    memory: BaseMemory = LangChainMemory(k=5)
    
    memory.save("用户喜欢Python", "好的，已记住")
    memory.save("用户想要Web应用", "好的，帮你搭")
    
    results = memory.recall("用户想要什么")
    print(results)
```

---

## 完成 Gate 检验

1. **Q1**：LangChain、CrewAI、AutoGen 的 Memory 实现差异是什么？
2. **Q2**：短期记忆（滑动窗口）在各框架如何实现？哪个框架支持最完整？
3. **Q3**：长期记忆（向量库）在 LangChain 和 LlamaIndex 的实现有什么区别？
4. **Q4**：Reflexion 反思机制在 LangChain 如何实现？CrewAI 如何间接支持反思？
5. **Q5**：什么场景下应该选择 LangChain？什么场景下应该选择 CrewAI？

---

## 练习清单

- [ ] 用 LangChain 实现一个带 `ConversationBufferWindowMemory` 的对话 Agent
- [ ] 用 CrewAI 的 `memory=True` 开启 Agent 记忆，对比差异
- [ ] 给 AutoGen Agent 手动实现滑动窗口记忆
- [ ] 用 LangChain 的 `VectorStoreRetrieverMemory` 实现长期记忆
- [ ] 实现一个统一的 `BaseMemory` 接口，兼容 LangChain 和自定义实现

---

## 关联资源

- [LangChain Memory 文档](https://python.langchain.com/docs/concepts/memory/)
- [CrewAI Memory](https://docs.crewai.com/concepts/memory)
- [AutoGen ConversableAgent](https://microsoft.github.io/autogen/)
- [Reflexion 论文](https://arxiv.org/abs/2303.11366)
- [mem0（专门Agent记忆层）](https://github.com/mem0ai/mem0)

---

## 7. 与本仓库其他单元的关系

|关联单元|关系|
||---|---|
|U8 Memory 与 Reflection|U8 讲 Memory 原理，U11 对比框架实现|
|U9 Multi-Agent|Multi-Agent 系统的 Memory 选型依赖 U11 的对比|
|U38 Context Management|短期记忆（滑动窗口）是 Context Management 的基础|
|U40 Verification|Memory 的正确性依赖 U40 的验证技术|
|U35 AgentOps|AgentOps 监控 Memory 的使用和性能|
*最后更新：2026-05-27*