# Phase 1：基础夯实

> **目标**：消除基础知识短板，跑通第一个 API 调用，理解核心概念
> **对应集群**：C1 + C2 + C3 + C4 基础
> **建议顺序**：U1 → U3 → U-DL → U2 → U4 → U6 → U5

---

## 集群覆盖

本阶段聚焦在四个集群的基础部分：

| 集群 | 单元 | 状态 |
|------|------|------|
| **C1 · LLM 与 Prompt** | U1, U3, U-DL | 基础必须 |
| **C2 · Embedding 与检索** | U2 | 基础必须 |
| **C3 · RAG** | U4 | 基础必须 |
| **C4 · Agent 范式** | U5, U6 | 入门必须 |

进入 C5（Memory）、C6（Multi-Agent）、C7（评估与安全）、C8（多模态与编程 Agent）之前，需要先完成本阶段所有 Gate。

---

## 模块 1：Python 工程能力

- 面向对象：类、继承、抽象类、`Protocol`（类型检查）
- 异步编程：`async`/`await`、`asyncio`并发、`aiohttp`
- 装饰器：`@property`、`@cache`、`@functools.wraps`
- 类型提示：`Optional`、`List`、`Dict`、`TypedDict`
- 上下文管理：`with` 语句、自定义 `__enter__/__exit__`
- 日志：`logging` 模块、格式化、handler

**学习策略**：边学边用，不单独花时间，遇到再查。

---

## 模块 2：环境与工具链

- 熟练调用 OpenAI / Claude / DeepSeek API
- `uv` 或 `conda` 管理 Python 环境
- `pdb` / `debugpy` 断点调试，能看懂 LLM 返回的原始 JSON
- Jupyter Notebook 流畅使用
- 会看 LangSmith / OpenAI Playground 的 trace

---

## 模块 3：U1 · LLM 基础

**必学概念**
- Next-token prediction 本质（不是"回答"，是"续写"）
- Transformer attention 机制（至少能直观解释）
- Temperature / Top-p 采样：为什么同一个问题两次回答不一样
- Context Window：模型能看到多长的文本
- Instruction Tuning / RLHF：模型怎么学会遵循指令
- GPT-4 / Claude / DeepSeek 模型能力对比

**必读材料**
- [Jay Alammar 可视化文章](https://jalammar.github.io/illustrated-chatgpt/)（30 分钟）
- [Andrej Karpathy 视频](https://www.youtube.com/watch?v=kexM9Ars3Mw)（1小时）
- [OpenAI Tokenizer 工具](https://platform.openai.com/tokenizer)

**练习**
- 用 `tiktoken` 对中文句子做 token 计数
- 对比 temperature=0 和 temperature=1.0 的输出差异（至少 3 句）

**笔记**：`notes/concept-notes/U1-LLM基础.md`

---

## 模块 4：U3 · Prompt 工程

**必学概念**
- Zero-shot / Few-shot / Chain-of-Thought 区别
- System Prompt / User Prompt / Assistant Message 各什么时候用
- Structured Output：JSON Mode / Pydantic Mode
- Prompt 注入与防御（安全意识）

**必读材料**
- [Prompt Engineering Guide](https://www.promptingguide.ai/)（通读 2 小时）
- [Anthropic Prompt Engineering 指南](https://docs.anthropic.com/en/docs/prompt-engineering/)

**练习**
- 同一问题用 zero-shot / one-shot / CoT 三种方式提问
- 让模型按 Pydantic schema 输出，验证格式

**笔记**：`notes/concept-notes/U3-Prompt工程.md`

---

## 模块 5：U-DL · 深度学习扫盲

**必学概念**
- Transformer 为什么需要 Attention（3Blue1Brown 视频 15min）
- Autoregressive generation
- GPT 的能力来源：Scaling Law + Next-token prediction
- RLHF 大框架：Reward Model + PPO

**练习**
- 能用自己的话解释：Transformer Attention、Autoregressive generation
- 理解 RLHF 大框架

**笔记**：`notes/concept-notes/UD-DL深度学习扫盲.md`

---

## 模块 6：U2 · Embedding 与 VectorDB

**必学概念**
- 从 Word2Vec 到 Sentence Embedding 的演进
- 余弦相似度、点积作为向量距离
- ANNS：HNSW、IVF 的直观理解
- Embedding 模型选型（text-embedding-3-small / bge-m3 等）

**必读材料**
- [OpenAI Embeddings 官方文档](https://platform.openai.com/docs/guides/embeddings)
- [HNSW 可视化](https://arxiv.org/abs/1608.07834)（只看 Introduction）
- [FAISS Quick Start](https://github.com/facebookresearch/faiss/blob/main/INSTALL.md)

**练习**
- 用 OpenAI Embedding API 对 10 句话做向量化和余弦相似度检索
- 用 FAISS 或 Chroma 建一个本地向量库

**笔记**：`notes/concept-notes/U2-Embedding与VectorDB.md`

---

## 模块 7：U4 · RAG 基础

**必学概念**
- RAG = Retrieval + Augmentation + Generation 三段流程
- Chunking 策略：固定窗口、重叠、语义分割
- Reranking：为什么需要重排序
- HyDE（假设性文档嵌入）概念

**必读材料**
- [RAG 实战：用 LangChain 从 0 搭一个](https://python.langchain.com/docs/tutorials/rag/)
- [Chunking 策略对比](https://pub.aimind.io/2024/chunking-strategies-for-rag/)

**练习**
- 找一篇 PDF，用 LangChain 做一次完整 RAG 流程
- 对比不同 chunk_size（256 / 512 / 1024）对检索质量的影响

**笔记**：`notes/concept-notes/U4-RAG基础.md`

---

## 模块 8：U6 · Tool Use 与 Function Calling

**必学概念**
- Tool Definition 的 JSON Schema（name / description / parameters）
- OpenAI Function Calling vs Claude Tool Use 异同
- Tool Choice 策略（强制用某个 tool / auto）
- 多 tool 调用时的顺序和依赖处理

**必读材料**
- [OpenAI Function Calling 文档](https://platform.openai.com/docs/guides/function-calling)
- [Claude Tool Use 文档](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)

**练习**
- 定义 3 个 tool（天气查询 + 搜索 + 计算器），用 OpenAI API 实现完整调用链
- 观察模型在什么情况下选择不调用 tool

**工具**：`tools/langchain-guide.md`
**笔记**：`notes/concept-notes/U6-ToolUse与FunctionCalling.md`

---

## 模块 9：U5 · ReAct 范式

**必学概念**
- ReAct 四步循环：Thought → Action → Observation → Next Thought
- 与纯 Reasoning（CoT）和纯 Action（Function Call）的区别
- 什么时候 ReAct 比其他方案更好

**必读材料**
- [ReAct 论文](https://arxiv.org/abs/2210.03629)（Abstract + Figure 1）
- [ReAct in LangChain](https://python.langchain.com/docs/tutorials/agent_executor/)

**练习**
- 画 ReAct 四步循环的图（手画或 Mermaid）
- 用 LangChain 的 `ReAct agents` 实现一个带 3 个 tool 的 agent

**笔记**：`notes/concept-notes/U5-ReAct范式.md`

---

## 收尾项目：ReAct Agent 原型

**需求**
1. 至少 3 个 Tool（天气查询、搜索、计算器等有实际功能的）
2. Tool Calling 通过 OpenAI Functions/Tools 格式
3. ReAct 循环可配置 max iterations（10步后强制停止）
4. 输出 trace log，看到每一步的 Reason / Action / Observation
5. 支持流式输出（streaming）

**加分项**
- 支持 Memory（ConversationBufferWindow）
- 支持切换不同模型（OpenAI / Claude / DeepSeek）
- 用 Pydantic 做输入输出 schema 验证

---

## Phase 1 完成检验 Gate

通过以下 4 项才能进入下一阶段：

1. ✅ 能手写一个带类型提示的 `async def run()` agent 循环
2. ✅ 能调用 OpenAI API 实现一次 tool-calling（带 function definition）
3. ✅ 能解释 ReAct 四步循环的本质（不需要背论文）
4. ✅ ReAct Agent 原型能跑通至少 3 个 tool 的完整调用链

---

## 学习原则

1. **先跑通，再理解**：先让代码跑起来，看到结果，再去理解原理
2. **每个练习都要动手**：不只看教程，要自己敲一遍
3. **不懂就问**：遇到问题不要卡超过 20 分钟，先问再继续
4. **记录卡点**：每个卡点记下来，复盘时统一解决
5. **别追求完美**：理解 80% 就继续往前走，不要在一个点上死磕

---

*调整记录：2026-06-30 重构为知识集群视图，去除时间线*
