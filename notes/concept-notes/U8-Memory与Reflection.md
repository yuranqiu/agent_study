# U8 · Memory 与 Reflection

> **核心问题**：Agent 怎么记住和反思？
> **预计学时**：5h
> **对应 Phase**：Phase 2

---

## 学习目标（Gate 检验）

1. 能区分短期记忆（滑动窗口）和长期记忆（向量库）的适用场景
2. 能实现一个带滑动窗口的对话 Agent（超过 5 轮对话后压缩历史）
3. 能解释 Reflexion 反思机制的三个步骤
4. 能说清楚上下文压缩和摘要的区别

---

## 1. 为什么 Agent 需要 Memory

LLM 本身没有状态——每次 API 调用是独立的，用完即忘。

但实际任务需要：
- 多轮对话的上下文（短期记忆）
- 跨会话的持久知识（长期记忆）
- 失败后的经验积累（反思）

Memory 是 Agent 系统里补齐 LLM 缺陷的组件。

---

## 2. 短期记忆：滑动窗口

### ConversationBufferWindowMemory

只保留最近 N 条对话，超过的丢弃或压缩。

```python
from langchain.memory import ConversationBufferWindowMemory
from langchain_openai import ChatOpenAI
from langchain.agents import AgentType, Tool, create_react_agent

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

memory = ConversationBufferWindowMemory(
    memory_key="chat_history",
    k=5,  # 只保留最近 5 轮对话
    return_messages=True,
)

# 每次对话后更新记忆
memory.save_context({"input": "我想去北京旅游"}, {"output": "北京有什么好玩的？"})
# 超过 5 轮后，最早的会被丢弃
```

### 缺点

- 早期信息完全丢失
- 适合短对话，不适合长任务

---

## 3. 摘要式压缩

### 思路

当历史太长时，把历史压缩成摘要，释放 context 空间。

```python
# LangChain 的 ConversationSummaryMemory
from langchain.memory import ConversationSummaryMemory

memory = ConversationSummaryMemory(
    memory_key="chat_history",
    llm=llm,  # 用 LLM 生成摘要
)

# 随着对话进行，历史被压缩成摘要
memory.save_context(
    {"input": "我想了解机器人的历史"},
    {"output": "机器人起源于1950年代的科幻小说..."}
)
# 对话很长后，chat_history 变成一段摘要而非完整对话
```

### 摘要 vs 滑动窗口

| 方式 | 保留内容 | 丢失内容 |
|------|---------|---------|
| 滑动窗口 | 最近 N 条 | 早期所有细节 |
| 摘要 | 压缩后的全历史语义 | 原始细节（但语义仍在） |

---

## 4. 长期记忆：向量库记忆

### 思路

把重要记忆向量化为存在向量库里，检索时召回相关记忆。

```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

# 把每次对话的重要信息存入向量库
memory = VectorStoreRetrieverMemory(
    vectorstore=FAISS.from_documents([], OpenAIEmbeddings()),
    retriever_kwargs={"k": 3},  # 召回 Top-3 相关记忆
    memory_key="past_context",
)

memory.save_context(
    {"input": "用户偏好简洁的代码风格"},
    {"output": "了解，已记住偏好简洁风格"}
)

# 新对话时检索相关记忆
relevant = memory.load_memory_variables({"input": "这次代码怎么写"})
print(relevant["past_context"])
```

### 适用场景

- Personal Agent（记住用户偏好、习惯）
- 跨会话的知识积累
- 需要引用历史经验的任务

---

## 5. Reflexion：语言反馈反思

### 三步机制

```
1. Agent 执行任务，产生结果
2. Evaluator 评估结果（对/错）
3. 如果失败，Agent 反思失败原因，写成语言反馈
4. 把反思结果作为记忆，下一轮任务时检索使用
```

### 核心思想

不是通过梯度更新（RL），而是用语言反馈让 Agent 记住"上次为什么失败"。

### 论文原文核心

> "Given the sparse feedback signals (e.g., binary success/failure), we found it beneficial to incorporate linguistic feedback as another source of information to facilitate the Agent's learning."

### 简单实现

```python
def reflect(task: str, result: str, success: bool) -> str:
    """生成反思文本"""
    prompt = f"""任务：{task}
结果：{result}
成功与否：{"成功" if success else "失败"}

请反思：如果失败了，原因是什么？下次怎么能做得更好？用 2~3 句话简洁回答。"""

    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
    )
    return resp.choices[0].message.content

# 示例
reflection = reflect(
    task="帮用户写一个排序算法",
    result="用了冒泡排序，时间复杂度 O(n²)",
    success=True
)
print(reflection)
```

---

## 6. 实现：带 Memory 的对话 Agent

```python
from langchain.memory import ConversationBufferWindowMemory
from langchain_openai import ChatOpenAI
from langchain.prompts import PromptTemplate

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 带记忆的 Prompt
MEMORY_PROMPT = PromptTemplate.from_template("""你是一个人性化的助手。

之前的对话历史：
{chat_history}

当前用户：{input}
你的回复：""")

def chat(question: str, memory: ConversationBufferWindowMemory) -> str:
    # 1. 加载历史
    history = memory.load_memory_variables({})

    # 2. 生成回复
    resp = llm.invoke(
        MEMORY_PROMPT.format(
            chat_history=history.get("chat_history", ""),
            input=question,
        )
    )

    # 3. 保存历史
    memory.save_context({"input": question}, {"output": resp.content})

    return resp.content

# 使用
memory = ConversationBufferWindowMemory(k=5, return_messages=True)
print(chat("我想学 Python", memory))      # 第 1 轮
print(chat("推荐一些教程", memory))       # 第 2 轮
print(chat("学费多少", memory))           # 第 3 轮
# 第 6 轮时，第 1 轮已被丢弃
```

---

## 完成 Gate 检验

1. **Q1**：短期记忆（滑动窗口）和长期记忆（向量库）的核心区别是什么？各适合什么场景？
2. **Q2**：摘要式压缩和滑动窗口各会丢失什么信息？
3. **Q3**：Reflexion 的三个步骤是什么？为什么用语言反馈而非梯度更新？
4. **Q4**：实现一个带 k=3 滑动窗口记忆的对话 Agent，超过 3 轮后验证最早对话被丢弃。

---

## 练习清单

- [ ] 用 `ConversationBufferWindowMemory` 实现一个带滑动窗口的对话 Agent
- [ ] 实现一个摘要式压缩的对话记忆（`ConversationSummaryMemory`）
- [ ] 实现 Reflexion 反思机制：任务执行 → 评估 → 反思 → 存储
- [ ] 用向量库实现长期记忆：记住用户偏好，下次对话时召回

---

## 关联资源

- [Generative Agents 论文 - Memory 部分](https://arxiv.org/abs/2304.03442)
- [Reflexion 论文](https://arxiv.org/abs/2303.11366)
- [LangChain Memory 文档](https://python.langchain.com/docs/concepts/memory/)
- [mem0](https://github.com/mem0ai/mem0)（专门的 Agent 记忆层）

---


## 11. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U1 LLM 基础|Memory 的存储和检索依赖 LLM 的理解能力|
|U2 Embedding 与 VectorDB|长期记忆（向量库）依赖 U2 的 Embedding 技术|
|U9 Multi-Agent|Agent 间的状态共享依赖 Memory 机制|
|U11 Memory 框架对比|U8 讲原理，U11 对比各框架的实现|
|U38 Context Management|短期记忆（滑动窗口）是 Context Management 的基础|
*最后更新：2026-05-26*
---

## 7. mem0 架构深度（参考 [mem0ai/mem0](https://github.com/mem0ai/mem0)）

**mem0** 是 2024-2025 主流的 Agent Memory 框架，在 SWE-bench Verified 上比全 context 高 26% 准确率。

### 7.1 4 层 Memory 架构

```
┌──────────────────────────────────────────┐
│  L1: Working Memory（in-context）         │  ← 当前 conversation
│      滑动窗口 / summary 压缩             │
├──────────────────────────────────────────┤
│  L2: Short-term Memory（session）         │  ← 当前 session 的事实
│      Session-scoped Redis / SQLite        │
├──────────────────────────────────────────┤
│  L3: Long-term Memory（user profile）     │  ← 跨 session 持续
│      VectorDB + 实体抽取 + 重要性评分    │
├──────────────────────────────────────────┤
│  L4: Procedural / Tool Memory             │  ← 学到的 skills
│      Tool call history + success rate     │
└──────────────────────────────────────────┘
```

### 7.2 mem0 核心实现（简化 50 行）

```python
import openai
from mem0 import Memory

class AgentWithMem0:
    def __init__(self):
        self.memory = Memory()  # 初始化 mem0
        self.client = openai.OpenAI()
        self.user_id = "user_123"

    def add_memory(self, messages):
        """自动从对话中抽取事实"""
        # mem0 用 LLM 抽取: "User lives in Paris" → fact
        # 然后存到 VectorDB
        self.memory.add(messages, user_id=self.user_id)

    def search_memory(self, query, limit=5):
        """检索相关记忆"""
        return self.memory.search(query, user_id=self.user_id, limit=limit)

    def chat(self, user_input):
        # 1. 检索相关记忆
        memories = self.search_memory(user_input)
        memory_context = "\n".join(f"- {m['memory']}" for m in memories)

        # 2. 把记忆塞进 system prompt
        system = f"""你是 helpful 助手。
用户的历史信息：
{memory_context}

回答时考虑这些信息。"""

        # 3. 调 LLM
        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": system},
                {"role": "user", "content": user_input},
            ],
        )

        # 4. 把这轮对话存为记忆（异步）
        self.add_memory([
            {"role": "user", "content": user_input},
            {"role": "assistant", "content": response.choices[0].message.content},
        ])

        return response.choices[0].message.content

# 使用
agent = AgentWithMem0()
print(agent.chat("我下周要去 Paris 旅行，有什么建议？"))
# → "根据你的喜好，Paris 适合..."
print(agent.chat("我住的酒店叫什么？"))  # 2 个月后再问
# → "你住在 Hôtel du Louvre..."（从 memory 召回）
```

### 7.3 mem0 vs 传统 Vector Store Memory

| 维度 | Vector Store Memory | mem0 |
|------|---------------------|------|
| **数据粒度** | 整个 chunk | 抽取的"事实"（"User lives in Paris"）|
| **重要性评分** | ❌ | ✅（自动评分）|
| **记忆衰减** | ❌ | ✅（30 天未访问 → 降权）|
| **更新机制** | 重新 add | ✅（UPDATE / DELETE / NOOP）|
| **效果** | baseline | +26% SWE-bench |

**核心洞察**：传统 Vector Store 把 chunk 当成"记忆"，但 chunk 是**原始数据**；mem0 把 chunk 抽取成**事实**（fact），事实才是"知识"。

---

## 8. 3 大 Memory 框架对比

| 框架 | 架构 | 优势 | 适用 |
|------|------|------|------|
| **mem0** | 4 层 + LLM 抽取 fact | 自动化 + 衰减 | 通用 agent |
| **Letta** | MemGPT 风格（分层 + 工具） | 大 context 友好 | 长 session agent |
| **Zep** | 时序 + 实体图 | 时间感知 | 时序任务 |
| **LangChain Memory** | 单层（Buffer / Summary）| 简单 | 小项目 |

### 8.1 选型决策

```
需求：通用 agent 长期记忆
├── 标准 + 易用 → mem0
├── 长 session（>100k tokens）→ Letta / MemGPT
├── 时序推理（"上周说过什么"）→ Zep
└── 简单 demo → LangChain ConversationBufferMemory
```

---

## 9. Reflexion 论文深入

**Reflexion**（Shinn et al., 2023）= 让 agent 用**自然语言反思**自己的失败。

### 9.1 三步循环

```
1. Act：agent 尝试任务
2. Evaluate：外部评估（test / user / heuristic）判定 success / failure
3. Reflect：if failure → LLM 生成 "我为什么会失败？我下次应该怎么做？"
4. 反思写入 memory，下次再尝试时参考
```

### 9.2 实战代码（50 行）

```python
class ReflexionAgent:
    def __init__(self, llm, max_trials=3):
        self.llm = llm
        self.memory = []  # 反思历史
        self.max_trials = max_trials

    def act(self, task):
        # 用所有反思 + 当前任务 调 LLM
        reflection_text = "\n".join(
            f"反思 {i+1}: {r}" for i, r in enumerate(self.memory)
        )
        prompt = f"""任务：{task}

过去的反思（避免重复错误）：
{reflection_text if reflection_text else '（无）'}

请输出你的答案。"""
        return self.llm.invoke(prompt)

    def evaluate(self, task, response):
        """外部评估（实际由用户 / test 决定）"""
        # 示例：检查是否包含某个关键词
        return "正确" in response or "success" in response.lower()

    def reflect(self, task, response, success):
        if success:
            return None
        # 让 LLM 反思
        reflect_prompt = f"""你刚才尝试任务：{task}
你的答案是：{response}
但任务失败了。

请反思：
1. 你的答案哪里错了？
2. 你应该用什么不同的方法？
3. 哪些关键信息你遗漏了？

请用 2-3 句话简洁回答。"""
        return self.llm.invoke(reflect_prompt)

    def run(self, task):
        for trial in range(self.max_trials):
            response = self.act(task)
            success = self.evaluate(task, response)
            if success:
                return response
            reflection = self.reflect(task, response, success)
            self.memory.append(reflection)
        return response  # max trials reached
```

**关键**：反思是**自然语言**（不是向量），可以被 LLM 重新理解、引用、组合。

---

## 10. 实战：3 个 Memory 优化技巧

### 10.1 实体优先（Entity-First）

```python
# ❌ 差：把整个对话当 memory
memory.add(chat_history)  # 噪音大

# ✓ 好：只存关键事实
memory.add("User: lives in Paris")
memory.add("User: works at Anthropic")
memory.add("User: prefers Python over JS")
```

### 10.2 时间衰减（Time Decay）

```python
def get_relevant_memories(memory_db, query, current_time):
    all_memories = memory_db.search(query, limit=20)

    def recency_score(memory, now):
        age_days = (now - memory.created_at).days
        return 0.5 ** (age_days / 30)  # 30 天半衰期

    def relevance_score(memory, query):
        return memory.similarity_to(query)

    # 综合分数
    return sorted(
        all_memories,
        key=lambda m: 0.7 * relevance_score(m, query) + 0.3 * recency_score(m, current_time),
        reverse=True,
    )[:5]
```

### 10.3 主动遗忘（Active Forgetting）

```python
def maybe_forget(memory):
    """定期清理低质量记忆"""
    if memory.access_count < 3 and memory.age_days > 90:
        memory.delete()
    if memory.importance_score < 0.3:
        memory.delete()
```

**原则**：Memory 不是越多越好，是**精**越好。
