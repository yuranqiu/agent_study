# Agent 学习路线仓库

> 目标：校招准备，理论 + 实践双线并行

## 目录结构

```
agent_study/
├── README.md                    # 路线总览（本文）
├── roadmap/
│   ├── phase-1-fundamentals.md  # 第1阶段：基础夯实
│   ├── _meta.json                # 路线元数据
│   └── ...                       # 其他阶段文件
├── notes/
│   ├── paper-notes/              # 论文笔记
│   ├── concept-notes/            # 核心概念笔记（U1-U41 + U-DL）
| `prompt-qa.md` | Prompt 工程（U3）：Zero-shot/Few-shot/CoT / System Prompt 4 要素 / Anthropic 12 法则 / XML 标签 / Pydantic Mode / Prompt 注入防御 / Temperature 调参 | 🆕 ~40 题 |
│   └── interview-qa/            # 面试题整理
| `planning-qa.md` | Planning 与推理（U7）：GSM-SF / ToT / PoT / Plan-and-Solve / LLM Compiler / ADaPT / 子目标缓存 / 状态回溯 / 失败恢复 / 系统设计 | 🆕 ~30 题 |
├── projects/
│   └── README.md                # 项目实践记录
| `context-harness-qa.md` | Context Engineering & Harness Engineering：5大组件 / Just-in-time / Long-horizon技术 / 11组件 / H0-H3 / Claude Code案例 | 🆕 ~21 题 |
│   ├── _tracker.md              # 进度追踪
│   └── weekly-reviews/          # 每周复盘
└── tools/                       # 框架/工具速查指南
```

---

## 当前阶段

- **Phase 1：基础夯实**
- 主题：Python 工程 + AI 基础 + Agent 通识

---

## 学习原则

1. **先建最小循环，再深入**：先跑通一个 50-150 行的最小 agent，再研究框架和源码
2. **代码驱动理论**：每个概念都要有配套的最小可运行代码，不能只记笔记
3. **优先官方文档和开源项目**：读官方博客、论文、真实代码库，少看二手总结
4. **定期复盘**：每周做一次复盘，总结进度，更新路线
5. **最小改动原则**：路线调整只追加、不推翻之前已确定的内容
6. **面试导向**：每个知识点问自己"这能怎么在面试中讲"
7. **Eval 先于扩展**：加功能前先加测试评估，不要盲目加 agent
8. **模型训练不是重心**：Agent 工程岗核心竞争力是系统设计 + Prompt + API，不要求自己训练模型

---

## 如何更新路线

1. 在 `notes/concept-notes/Ux-*.md` 中追加新笔记
2. 更新 `roadmap/_meta.json` 中相关单元的状态
3. 在 `progress/weekly-reviews/` 中记录本次调整的原因

---

## 知识地图（Knowledge Map）

> 以"主题集群（Cluster）→ 单元（Unit）"组织，弱化时间线。每单元 = 学习 → 笔记 → 练习 → 过 Gate。

### Cluster 总览

| 集群 | 单元 | 核心问题 |
|------|------|----------|
| **C1 · LLM 与 Prompt 基础** | U1, U3, U14, U24, U30, U32, U-DL | 模型怎么生成？怎么跟它说话？o1/R1 怎么用？怎么后训练？怎么微调？ |
| **C2 · Embedding 与检索** | U2, U16, U19 | 文本怎么变向量？怎么让检索"好"？向量库怎么选？ |
| **C3 · RAG** | U4, U15 | 怎么让模型用你自己的知识回答？怎么从"能用"到"好用"？ |
| **C4 · Agent 范式** | U5, U6, U7 | Agent 怎么把推理和行动交替？怎么用工具？怎么拆解任务？ |
| **C5 · Memory 与状态** | U8, U11 | Agent 怎么记住和反思？各框架 Memory 有何差异？ |
| **C6 · Multi-Agent 与协议** | U9, U13, U36 | 多 Agent 怎么协作？A2A / MCP 协议？ |
| **C7 · 评估与安全** | U10, U12, U18, U34, U35 | 怎么评估 / 攻击 / 防御？Guardrails 怎么设防？AgentOps 怎么做？ |
| **C8 · 多模态与编程 Agent** | U17, U20, U25 | Agent 怎么感知图像？怎么操作 GUI？编程 Agent 怎么工作？ |
| **C9 · Loop Engineering 与长跑 Agent** | U21, U27 | 怎么让 Agent 跨 session 持续工作？怎么定义"目标"？ |
| **C10 · Agent 运行时工程（Context + Harness）** | U22, U23, U31, U33, U37, U38, U40 | 塞什么进 context？怎么调度？Subagent / 验证 / HITL 怎么用？ |
| **C11 · Agent 能力工程化（Skills + AGENTS.md）** | U26, U41, U42 | 怎么把"项目知识"以可发现、可加载的方式提供给 Agent？ + Agent 全栈项目学习路线 + 面试真题与学习顺序 |
| **C12 · 生产工程（Inference + Deploy）** | U28, U29, U39 | 怎么让 LLM 调用更快更便宜？怎么安全部署？Permission & Auto Mode？ |

---

### C1 · LLM 与 Prompt 基础

#### U1 · LLM 基础

**核心问题**：GPT 是怎么生成下一个词的？

**必学概念**
- Next-token prediction 本质（不是"回答"，是"续写"）
- Transformer attention 机制（至少能直观解释）
- Temperature / Top-p 采样：为什么同一个问题两次回答不一样
- Context Window：模型能看到多长的文本
- Instruction Tuning / RLHF：模型怎么学会遵循指令
- GPT-4 / Claude / DeepSeek 模型能力对比

**必读材料**
- [ChatGPT 究竟是怎么工作的？—— Jay Alammar 可视化](https://jalammar.github.io/illustrated-chatgpt/)（图解版，30 分钟）
- [GPT 是如何工作的—— Andrej Karpathy 视频](https://www.youtube.com/watch?v=kexM9Ars3Mw)（1小时）
- [OpenAI Tokenizer 工具](https://platform.openai.com/tokenizer)（自己动手试）

**练习**
- 用 `tiktoken` 或 OpenAI SDK 查一个中文句子被切成多少个 token
- 对比 temperature=0 和 temperature=1.0 的输出差异（至少 3 句）

**笔记/资料**：`notes/concept-notes/U1-LLM基础.md`

#### U3 · Prompt 工程

**核心问题**：怎么跟模型有效说话？

**必学概念**
- Zero-shot / Few-shot / Chain-of-Thought 区别
- System Prompt / User Prompt / Assistant Message 各什么时候用
- Structured Output：JSON Mode / Pydantic Mode
- Prompt 注入与防御（安全意识）

**必读材料**
- [Prompt Engineering Guide](https://www.promptingguide.ai/)（必读，通读 2 小时）
- [Anthropic Prompt Engineering 指南](https://docs.anthropic.com/en/docs/prompt-engineering/)

**练习**
- 对同一个问题，分别用 zero-shot / one-shot / cot 三种方式提问，对比输出
- 让模型按 Pydantic schema 输出，验证格式

**笔记/资料**：`notes/concept-notes/U3-Prompt工程.md`

#### U14 · DSPy

**核心问题**：如何用声明式编程范式构建 LLM 应用？

**必学概念**
- Signature / Module / Optimizer（Teleprompter）
- 声明式 Prompt 编程 vs 字符串拼接
- 编译器如何自动优化 Prompt 与示例选择
- BootstrapFewShot / MIPRO 等优化算法
- DSPy 与传统 Prompt 工程的差异：评测驱动 vs 经验驱动

**必读材料**
- [DSPy 官方文档](https://dspy.ai/)
- [DSPy 论文](https://arxiv.org/abs/2310.03714)

**练习**
- 用 DSPy 实现一个 RAG 流程，定义 Signature + Module
- 跑一次 BootstrapFewShot 优化器，对比优化前后的输出

**笔记/资料**：`notes/concept-notes/U14-DSPy.md` · 速查：`tools/dspy-guide.md` · 面试题：`notes/interview-qa/dspy-qa.md`

#### U-DL · 深度学习扫盲（轻量版）

**核心问题**：理解 LLM 原理所需的 DL 基础

**必学概念**
- Transformer 为什么需要 Attention（3Blue1Brown 视频 15min）
- Autoregressive generation
- GPT 的能力来源：Scaling Law + Next-token prediction
- RLHF 大框架：Reward Model + PPO

**必学清单**
- [3Blue1Brown Attention 视频](https://www.youtube.com/watch?v=eMlx5fFNoKY)
- 选读 InstructGPT 论文 Abstract + Section 3（2h）
- 选体验：用 unsloth 对小模型跑一次 LoRA（3h，走通流程即可）

**笔记/资料**：`notes/concept-notes/UD-DL深度学习扫盲.md`


#### U24 · Reasoning Model / Test-time Compute（推理模型）

**核心问题**：为什么 OpenAI o1/o3、DeepSeek R1、Gemini Thinking 这类"推理模型"突然改变了 Agent 设计的范式？Agent 工程师该怎么用它们？

**必学概念**
+- Reasoning Model 范式：回答前先生成长的内部 chain of thought
+- Test-time compute scaling law：用更多推理时间换更高准确率
+- 何时用 Reasoning vs 标准 LLM：数学/科学/规划用 Reasoning，简单 QA/对话用标准
+- 5 大设计模式：双模型协作 / Cascade 先快后慢 / 推理 + RAG / 推理 + Tool / 推理 + Extended Thinking
+- 局限：慢（10s-60s+）、贵（5-10x）、不可控、不能 streaming

**必读材料**
+- [OpenAI o1 System Card](https://openai.com/index/openai-o1-system-card/) — 官方权威
+- [OpenAI o3 Blog](https://openai.com/index/openai-o3/) — AIME / GPQA / Codeforces SOTA
+- [DeepSeek R1 Paper](https://arxiv.org/abs/2501.12948) — 开源推理模型
+- [Anthropic: Claude 3.7 Sonnet](https://www.anthropic.com/news/claude-3-7-sonnet) — Extended thinking 模式

**练习**
+- 用同一道 AIME 题，分别用 GPT-4o / o1 / DeepSeek R1 解，对比正确率 + 延迟
+- 跑 o3-mini 的 `reasoning_effort=low` vs `high`，对比成本/质量
+- 实现一个 Cascade Agent：先 mini 模型试，置信度低升级 o1

**笔记/资料**：`notes/concept-notes/U24-ReasoningModel.md`


#### U30 · Post-training 与 Alignment（后训练）

**核心问题**：预训练后的 LLM 怎么变成能用的 chat / reasoning / agent？SFT / DPO / GRPO / RLVR 怎么选？

**必学概念**
+- Post-training 全景：SFT → RM → RLHF / DPO / GRPO → RLVR
+- **SFT**（监督微调）：(prompt, response) 数据继续训练，起点
+- **RLHF**（PPO）：训练 RM + PPO 优化，但训练不稳定
+- **DPO**：直接用偏好数据优化，不需要 RM 和 PPO
+- **GRPO**：DeepSeek R1 用的算法，组内相对排名作为 advantage
+- **RLVR**：rule-based reward（数学/代码），没有 reward hacking
+- **R1 训练流程**：SFT → GRPO+RLVR → 拒采 SFT → 通用 RLHF

**必读材料**
+- [InstructGPT 论文](https://arxiv.org/abs/2203.02155) — RLHF 经典
+- [DPO 论文](https://arxiv.org/abs/2305.18290)
+- [DeepSeek R1 论文](https://arxiv.org/abs/2501.12948) — GRPO + R1 训练
+- [HuggingFace TRL 文档](https://huggingface.co/docs/trl) — 官方训练库

**练习**
+- 用 trl SFTTrainer 在 Alpaca 数据上微调
+- 用 trl DPOTrainer 训练偏好对齐
+- 用 trl GRPOTrainer 训练数学推理

**笔记/资料**：`notes/concept-notes/U30-PostTraining.md`

---

#### U32 · Fine-tuning 实践（微调实战）

**核心问题**：怎么用 SFT / DPO / GRPO 把开源 base model 训练成专属 agent 模型？

**必学概念**
+- 何时该 fine-tuning vs prompt engineering vs RAG
+- 3 大微调范式：SFT / LoRA / QLoRA
+- 工具栈：LLaMA-Factory（中文友好）/ trl（官方）/ unsloth（资源友好）
+- 数据准备：人工标注 / GPT-4 蒸馏 + 人类 review
+- 训练监控：loss / reward / KL divergence
+- 部署：vLLM serve + OpenAI 兼容 API
+- 常见坑：数据少 / 灾难性遗忘 / 复读

**必读材料**
+- [LLaMA-Factory 文档](https://github.com/hiyouga/LLaMA-Factory) — 中文首选
+- [unsloth 文档](https://docs.unsloth.ai/) — 资源友好
+- [QLoRA 论文](https://arxiv.org/abs/2305.14314)
+- [HuggingFace TRL 文档](https://huggingface.co/docs/trl)

**练习**
+- 用 LLaMA-Factory WebUI SFT 一个 7B 模型
+- 用 unsloth 做 QLoRA 训练
+- 用 GPT-4 蒸馏 1000 条 agent 数据

**笔记/资料**：`notes/concept-notes/U32-FineTuning.md`

---

### C2 · Embedding 与检索

#### U2 · Embedding 与 VectorDB

**核心问题**：怎么让模型"找到"相关知识？

**必学概念**
- 从 Word2Vec 到 Sentence Embedding 的演进
- 余弦相似度、点积作为向量距离
- ANNS：HNSW、IVF 的直观理解
- Embedding 模型选型（text-embedding-3-small / bge-m3 等）

**必读材料**
- [OpenAI Embeddings 官方文档](https://platform.openai.com/docs/guides/embeddings)
- [HNSW 可视化](https://arxiv.org/abs/1608.07834)（只看 Introduction）
- [FAISS 官方 tutorial](https://github.com/facebookresearch/faiss/blob/main/INSTALL.md)（跑通 Quick Start）

**练习**
- 用 OpenAI Embedding API 对 10 句话做向量化和余弦相似度检索
- 用 FAISS 或 Chroma 建一个本地向量库，实现相似语句检索

**笔记/资料**：`notes/concept-notes/U2-Embedding与VectorDB.md`

#### U16 · Embedding 进阶

**核心问题**：怎么让 Embedding 适配你的专属场景？

**必学概念**
- Embedding 微调：领域适配、对比学习
- Late Interaction：ColBERT / ColPali 等
- Cross-Encoder vs Bi-Encoder 差异
- 多语言 / 多模态 Embedding
- Embedding 评估：MTEB benchmark

**必读材料**
- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard)
- [ColBERT 论文](https://arxiv.org/abs/2004.12832)

**练习**
- 用 bge-m3 对中文语料做向量化，对比 text-embedding-3-small 的检索效果
- 尝试用 sentence-transformers 微调一个领域 Embedding

**笔记/资料**：`notes/concept-notes/U16-Embedding进阶.md`

#### U19 · 向量数据库深度对比

**核心问题**：如何在不同场景下正确选型向量数据库？

**必学概念**
- FAISS / Chroma / Qdrant / Milvus / Weaviate / Pinecone 横向对比
- 部署模式：嵌入式 / 服务端 / 云托管
- 索引类型：HNSW / IVF / DiskANN / ScaNN
- 混合检索：BM25 + 向量 + Filter
- 性能、成本、运维复杂度

**必读材料**
- [Qdrant 文档](https://qdrant.tech/documentation/)
- [Milvus 文档](https://milvus.io/docs)

**练习**
- 用同一份语料在 FAISS / Chroma / Qdrant 上对比检索时延和准确率
- 实现一个 BM25 + 向量混合检索

**笔记/资料**：`notes/concept-notes/U19-向量数据库深度对比.md`

---

### C3 · RAG

#### U4 · RAG 基础

**核心问题**：怎么让模型用你自己的知识回答？

**必学概念**
- RAG = Retrieval + Augmentation + Generation 三段流程
- Chunking 策略：固定窗口、重叠、语义分割
- Reranking：为什么需要重排序
- HyDE（假设性文档嵌入）概念

**必读材料**
- [RAG 实战：用 LangChain 从 0 搭一个](https://python.langchain.com/docs/tutorials/rag/)（跑通例子）
- [Chunking 策略对比](https://pub.aimind.io/2024/chunking-strategies-for-rag/)（选读）

**练习**
- 找一篇 PDF，用 LangChain 做一次完整 RAG 流程（切分 → 向量化 → 检索 → 生成）
- 对比不同 chunk_size（256 / 512 / 1024）对检索质量的影响

**笔记/资料**：`notes/concept-notes/U4-RAG基础.md`

#### U15 · RAG 进阶

**核心问题**：怎么让 RAG 从"能用"到"好用"？

**必学概念**
- 高级 Chunking：语义分割、层次化切分、Late Chunking
- Query Rewriting / Multi-Query / Step-Back Prompting
- Reranker 模型：bge-reranker / Cohere Rerank
- GraphRAG：实体-关系图谱增强检索
- Self-RAG / Corrective-RAG：带反思的 RAG
- RAG 评估：RAGAS / TruLens

**必读材料**
- [GraphRAG 论文](https://arxiv.org/abs/2404.16130)
- [RAGAS 文档](https://docs.ragas.io/)

**练习**
- 给 RAG 流程加 Reranker，对比加前后 top-5 准确率
- 实现一个最小 GraphRAG：抽取实体 → 建图 → 检索 → 生成

**笔记/资料**：`notes/concept-notes/U15-RAG进阶.md` · 面试题：`notes/interview-qa/advanced-rag.md`

---

### C4 · Agent 范式

#### U5 · ReAct 范式

**核心问题**：Agent 怎么把推理和行动交替起来？

**必学概念**
- ReAct 四步循环：Thought → Action → Observation → Next Thought
- 与纯 Reasoning（CoT）和纯 Action（Function Call）的区别
- 什么时候 ReAct 比其他方案更好

**必读材料**
- [ReAct 论文](https://arxiv.org/abs/2210.03629)（读 Abstract + Figure 1，30 分钟）
- [ReAct in LangChain](https://python.langchain.com/docs/tutorials/agent_executor/)（跑通 example）

**练习**
- 画 ReAct 四步循环的图（手画或 Mermaid）
- 用 LangChain 的 `ReAct agents` 实现一个带 3 个 tool 的 agent

**工具文件**：`tools/langchain-guide.md`
**面试资料**：`notes/interview-qa/agent-core.md`（ReAct 篇）

#### U6 · Tool Use 与 Function Calling

**核心问题**：模型怎么调用外部工具？

**必学概念**
- Tool Definition 的 JSON Schema（name / description / parameters）
- OpenAI Function Calling vs Claude Tool Use 异同
- Tool Choice 策略（强制用某个 tool / auto）
- 多 tool 调用时的顺序和依赖处理
- CodeAgent 风格：让模型直接写代码执行（vs 显式 tool calling）

**必读材料**
- [OpenAI Function Calling 文档](https://platform.openai.com/docs/guides/function-calling)
- [Claude Tool Use 文档](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)

**练习**
- 定义 3 个 tool（天气查询 + 搜索 + 计算器），用 OpenAI API 实现完整调用链
- 观察模型在什么情况下选择不调用 tool

**工具文件**：`tools/langchain-guide.md`（Tool 定义部分）· `tools/smolagents-guide.md`（CodeAgent 风格）
**面试资料**：`notes/interview-qa/agent-core.md`（Tool Use 篇）

#### U7 · Planning 与 Reasoning

**核心问题**：Agent 怎么拆解复杂任务？

**必学概念**
- 任务分解：GSM-SF（Go Step by Step）
- 规划方法：ToT（Tree of Thoughts）、CoT、PoT
- 子目标缓存和状态回溯
- 失败重试和路径恢复
- Plan-and-Execute 范式：先规划再执行

**必读材料**
- [Tree of Thoughts 论文](https://arxiv.org/abs/2305.10601)（读 Abstract + Figure 1）
- [Self-Consistency 论文](https://arxiv.org/abs/2203.11171)（选读）

**练习**
- 对一个多步问题，手动拆解成 5 步以内的子目标
- 用 ToT 风格实现一个简单问题的多路径探索

**工具文件**：`tools/langchain-guide.md`（LangGraph Conditional Edge 部分）
**面试资料**：`notes/interview-qa/agent-core.md`（Planning 篇）

---

### C5 · Memory 与状态

#### U8 · Memory 与 Reflection

**核心问题**：Agent 怎么记住和反思？

**必学概念**
- 短期记忆：ConversationBufferWindow
- 长期记忆：VectorStore-backed Memory
- 反思机制：Reflexion 语言反馈
- Summarization 和上下文压缩
- Token 经济性：什么该进 Context、什么该压缩或丢弃

**必读材料**
- [Generative Agents 论文](https://arxiv.org/abs/2304.03442)（读 Section 3 Memory 部分）
- [Reflexion 论文](https://arxiv.org/abs/2303.11366)（读 Section 3）

**练习**
- 实现一个带滑动窗口记忆的对话 agent（超过 5 轮对话后压缩历史）
- 用 Reflection 机制让 agent 自我纠错一个简单任务

**工具文件**：`tools/langchain-guide.md`（Memory 部分）
**面试资料**：`notes/interview-qa/agent-core.md`（Memory 篇）

#### U11 · Memory 框架对比

**核心问题**：各主流框架如何实现 Memory？优劣势是什么？

**必学概念**
- LangChain / CrewAI / AutoGen / smolagents / LlamaIndex 的 Memory 实现差异
- 短期 vs 长期：滑动窗口 / 摘要 / 向量库 / 用户偏好
- 反思机制在各框架的实现
- 跨框架统一 Memory 接口设计

**必读材料**
- [LangChain Memory 文档](https://python.langchain.com/docs/modules/memory/)
- [mem0 论文/文档](https://docs.mem0.ai/)

**练习**
- 用同一份对话数据，分别在 LangChain / CrewAI 上跑一遍，对比行为差异
- 抽象一个 `BaseMemory` 接口，分别实现 LangChain 版和手写向量库版

**笔记/资料**：`notes/concept-notes/U11-Memory框架对比.md` · 面试题：`notes/interview-qa/memory-frameworks.md`

---

### C6 · Multi-Agent 与协议

#### U9 · Multi-Agent 系统

**核心问题**：多个 Agent 怎么协作和通信？

**必学概念**
- Agent 间通信协议（共享消息 / 邮箱）
- 角色划分：Planner / Writer / Reviewer / Executor
- 协作模式：sequential / hierarchical / concurrent
- 共享工具和资源竞争
- 群聊机制：AutoGen GroupChat
- Handoff 机制：OpenAI Swarm

**必读材料**
- [LangGraph Multi-Agent 教程](https://python.langchain.com/docs/tutorials/multi_agent/)（跑通）
- [crewAI 官方文档](https://docs.crewai.com/)（理解 agent 角色和任务流）

**练习**
- 用 crewAI 或 LangGraph 实现一个 writer + reviewer 双 agent 协作

**工具文件**：`tools/langchain-guide.md`（LangGraph Multi-Agent）/ `tools/crewai-guide.md` / `tools/autogen-guide.md`
**面试资料**：`notes/interview-qa/frameworks.md`（CrewAI 篇）+ `notes/interview-qa/coding-agent.md`（Sub-agent 篇）

#### U13 · A2A 协议（Agent-to-Agent Protocol）

**核心问题**：Agent 之间如何互相通信、发现彼此、协作完成任务？

**必学概念**
- A2A vs MCP：A2A 解决 Agent 间通信，MCP 解决 Agent 与工具通信
- Agent Card：描述 Agent 能力的元数据（JSON）
- Task / Message / Part 状态机
- 服务发现 + 长任务（async task）
- 与 LangGraph / CrewAI 等框架的关系

**必读材料**
- [A2A 协议规范](https://github.com/a2a-mcp-go/a2a)
- [Google A2A 公告](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)

**练习**
- 写一个最小 A2A Server + Client，跑通消息收发
- 写一个 Agent Card，描述自己的 ReAct Agent

**笔记/资料**：`notes/concept-notes/U13-A2A协议.md` · 速查：`tools/a2a-guide.md`


#### U36 · MCP 深入（Model Context Protocol）

**核心问题**：怎么用 MCP（Model Context Protocol）构建生产级 agent 工具生态？Server/Client 实现、Streamable HTTP 传输、与 Function Calling / A2A 的协同？

**必学概念**
+- MCP 三大实体：Host（LLM 应用）/ Client（协议客户端）/ Server（暴露 tools+resources+prompts）
+- 三大原语：**Tools**（可调用函数）/ **Resources**（GET 端点，类文件读取）/ **Prompts**（user-controlled 模板）
+- 两种传输：**stdio**（本地进程，最快）/ **Streamable HTTP**（远程服务，2025-03 新标准）
+- MCP vs Function Calling：跨模型开放协议 vs 模型厂商定义
+- MCP vs A2A：Agent ↔ 工具（**纵向**）vs Agent ↔ Agent（**横向**）
+- FastMCP 写 Server：`@mcp.tool()` / `@mcp.resource()` / `@mcp.prompt()` 装饰器
+- LangChain 集成：`langchain-mcp-adapters` + `MultiServerMCPClient`
+- 5 大生产挑战：安全性 / Tool 数量膨胀 / 跨 server 状态 / 错误处理 / 可观测性
+- Tool 描述精炼（< 100 tokens）+ Lazy loading + Permission scopes

**必读材料**
+- [MCP 官方文档](https://modelcontextprotocol.io/) — 权威规范
+- [MCP 规范 2025-03-26](https://modelcontextprotocol.io/specification/2025-03-26/basic) — 协议细节
+- [FastMCP](https://github.com/jlowin/fastmcp) — Python 快速写 Server
+- [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) — LangChain 集成
+- [MCP Inspector](https://github.com/modelcontextprotocol/inspector) — 调试器

**练习**
+- 用 FastMCP 写一个 todo 工具 server，stdio 启动
+- 用 MCP Inspector 调试你的 server
+- 用 `MultiServerMCPClient` 集成多个 server
+- 接入 LangGraph 跑通
+- 生产级实践：加 auth / rate limit / OTel

**笔记/资料**：`notes/concept-notes/U36-MCPDeep.md`

---

### C7 · 评估与安全

#### U10 · Agent 评估与安全

**核心问题**：怎么评估和防护 Agent？

**必学概念**
- Agent 评估：任务完成率、step 效率、错误恢复率
- Agent 安全：提示词注入、工具滥用、无限循环
- 红队测试和对抗样本
- Agent 的可解释性和 trace

**必读材料**
- [SWE-bench 论文](https://arxiv.org/abs/2310.06770)（理解评测数据集设计思路）
- [AgentGym 论文](https://arxiv.org/abs/2406.12023)（选读）

**练习**
- 给自己的 ReAct Agent 加一个 max_iterations 限制和 trace log
- 尝试 Prompt 注入攻击，看 agent 是否泄露内部信息

**面试资料**：`notes/interview-qa/agent-core.md`（评估篇）+ `notes/interview-qa/coding-agent.md`（安全设计篇）

#### U12 · Agent 评估标准

**核心问题**：如何建立一套完整的 Agent 评估标准体系？

**必学概念**
- 评估维度：任务完成率 / 工具调用正确率 / 步骤效率 / 成本 / 鲁棒性
- 评测基准：SWE-bench / AgentBench / GAIA / AgentGym / WebArena
- 过程评估：轨迹（trajectory）评估
- 端到端评估 vs 单元评估
- Human-in-the-loop 评估方法
- 评估成本与自动化

**必读材料**
- [SWE-bench 论文](https://arxiv.org/abs/2310.06770)
- [AgentBench 论文](https://arxiv.org/abs/2308.03688)
- [GAIA 论文](https://arxiv.org/abs/2311.12983)

**练习**
- 给自己的 ReAct Agent 写 10 个测试用例，覆盖：成功路径、失败重试、注入攻击
- 用 RAGAS 给 RAG 系统打 3 个维度的分（faithfulness / answer relevancy / context precision）

**笔记/资料**：`notes/concept-notes/U12-Agent评估标准.md` · 面试题：`notes/interview-qa/agent-eval.md`

#### U18 · Agent 安全与红队

**核心问题**：如何攻击和防御 Agent 系统？

**必学概念**
- Prompt 注入：直接注入、间接注入（通过文档/网页）
- 工具滥用：让 Agent 调危险 Tool、参数注入
- 权限模型：人类确认 / 沙箱 / 白名单
- 数据外泄：上下文泄露、训练数据回显
- 越权与提权：Agent 越权访问其他用户数据
- 防御设计：输入校验、最小权限、审计日志

**必读材料**
- [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Anthropic Sleeper Agents 论文](https://arxiv.org/abs/2401.05566)

**练习**
- 设计 5 个针对自己 Agent 的注入攻击用例，记录泄露情况
- 给 Agent 加白名单 + 沙箱 + 人工确认三道防线

**笔记/资料**：`notes/concept-notes/U18-Agent安全与红队.md` · 面试题：`notes/interview-qa/security-red-team.md` + `notes/interview-qa/Agent-安全面试题.md`


#### U34 · Guardrails 与 Output 防御（护栏工程）

**核心问题**：怎么在 Agent 系统的"输入 + 输出 + 流程"多层设防？

**必学概念**
+- 5 层 Guardrail 架构：Input / Process / Output / Action / Audit
+- **NeMo Guardrails**（NVIDIA）：工业级 + Colang 流程语言
+- **Guardrails AI**：Pydantic 风格 + Hub validators（ToxicLanguage / PII / BannedWords）
+- **PII 防护**：输入 anonymize + 输出 detect/remove
+- **Prompt injection 防御**：规则检测 + 小 LLM 分类 + 双 LLM 模式
+- 5 级风险分级（与 U33 HITL 联动）
+- 性能 vs 安全权衡（每层 10-500ms）

**必读材料**
+- [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) — 工业级
+- [Guardrails AI Hub](https://hub.guardrailsai.com/) — Validators 库
+- [Presidio](https://github.com/microsoft/presidio) — PII 检测
+- [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

**练习**
+- 用 Guardrails AI 验证 PII + ToxicLanguage
+- 写 Colang 文件禁止讨论特定话题
+- 实现 5 层 guardrail pipeline

**笔记/资料**：`notes/concept-notes/U34-Guardrails.md`

---

#### U35 · AgentOps / Eval 工具链（Agent 工程化评估与运维）

**核心问题**：怎么用 Promptfoo / DeepEval / RAGAS / LangSmith 把 Agent 的质量回归做成工程化流水线？

**必学概念**
+- AgentOps = LLM 时代的 DevOps（probabilistic vs deterministic）
+- **Promptfoo**：声明式 LLM 评估 + Red Team + CI 友好
+- **DeepEval**：pytest 风格，14+ metrics（Faithfulness / Hallucination / Bias）
+- **RAGAS**：RAG 评估事实标准，4 大指标
+- **LangSmith / Langfuse**：trace + eval + monitoring
+- 4 层评估：通用能力 / 任务能力 / 业务能力 / 用户反馈
+- CI/CD for Agent：每次改 prompt 都跑 eval
+- Drift Detection：监控线上 quality regression
+- Auto-augment 测试集：从 prod failures 自动入库

**必读材料**
+- [Promptfoo 文档](https://promptfoo.dev/docs/intro)
+- [DeepEval 文档](https://docs.confident-ai.com/)
+- [RAGAS 文档](https://docs.ragas.io/)
+- [LangSmith 文档](https://docs.smith.langchain.com/)
- [Langfuse 文档](https://langfuse.com/docs) — 开源可自部署

**练习**
+- 用 Promptfoo 写 10 条 prompt eval + CI 集成
+- 用 DeepEval 跑 14+ metrics 评估 agent
+- 用 RAGAS 评估 RAG 系统
+- 实现 online sampling + drift detection

**笔记/资料**：`notes/concept-notes/U35-AgentOps.md`

---

### C8 · 多模态与编程 Agent

#### U17 · 多模态 Agent

**核心问题**：Agent 如何感知和理解图像、音频、视频等多模态信息？

**必学概念**
- 多模态模型：GPT-4V / Claude 3 Vision / Gemini / LLaVA
- 视觉理解架构：Vision Encoder → Projector → LLM
- 文档理解：LayoutLMv3 / DocFormer
- 视觉 Agent：browser-use / 计算机使用（CUA）
- 工具调用中的图像：截图理解、UI 操作
- 多模态 RAG：图像 + 文本联合检索

**必读材料**
- [LLaVA 论文](https://arxiv.org/abs/2304.08485)
- [GPT-4V 系统卡](https://openai.com/research/gpt-4v-system-card)

**练习**
- 用 GPT-4V 识别一张截图中的 UI 元素并生成 click 指令
- 实现一个截图 + 文本联合检索的多模态 RAG

**笔记/资料**：`notes/concept-notes/U17-多模态Agent.md` · 面试题：`notes/interview-qa/multi-modal-agent.md`

#### U20 · 编程 Agent 内部原理

**核心问题**：编程 Agent 是如何工作的？它内部的决策循环、工具调用、上下文管理是怎么实现的？

**必学概念**
- Harness：循环驱动 + 工具调度 + 上下文管理
- 权限模型：文件读 / 写 / 命令执行的沙箱
- 上下文压缩：滑动窗口、摘要、lazy load
- 工具集：文件编辑、Shell、Browser、Search
- Sub-Agent 机制：上下文隔离 + 任务拆分
- Hooks：拦截 + 校验 + 扩展
- 评测：SWE-bench Verified / HumanEval / LiveCodeBench

**必读材料**
- [Claude Code Overview](https://code.claude.com/docs/en/overview)
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Dive into Claude Code 论文](https://arxiv.org/abs/2604.14228)
- [AI Harness Engineering 论文](https://arxiv.org/abs/2605.13357)

**练习**
- 阅读 Claude Code / Codex / OpenHands 任一开源 Harness 源码，画出主循环
- 用 LangGraph 复刻一个 mini coding agent：read_file / write_file / bash 三个 tool

**笔记/资料**：`notes/concept-notes/U20-编程Agent内部原理.md` · 面试题：`notes/interview-qa/coding-agent-internals.md`


#### U25 · Computer Use / GUI Agent（计算机使用）

**核心问题**：Agent 怎么"像人一样"操作电脑——点击、拖拽、输入、读屏？

**必学概念**
+- Computer Use 范式：Screenshot + Vision Model + Action Loop
+- Anthropic Computer Use（Claude 内置 tool，13 个 action）
+- OpenAI Operator / CUA（o3 推理驱动）
+- browser-use：浏览器自动化（Playwright + LLM，比 Computer Use 快）
+- UI-TARS / Aguvis：端到端多模态 GUI Agent 模型
+- 5 大挑战：屏幕理解 / 动作 grounding / 长任务 memory / 错误恢复 / 安全
+- 评测基准：OSWorld / WebArena / ScreenSpot / Mind2Web

**必读材料**
+- [Anthropic: Computer Use Overview](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use) — 官方
+- [OpenAI Operator](https://openai.com/index/introducing-operator/) — OpenAI CUA
+- [OSWorld](https://osworld.dev/) / [WebArena](https://webarena.dev/) — 评测基准

**练习**
+- 用 browser-use 完成"在 google.com 搜索某词"任务
+- 用 Anthropic Computer Use 在 macOS 上跑一个 demo
+- 给自己的 GUI Agent 加 "pay/delete/send" 关键字的 human confirm

**笔记/资料**：`notes/concept-notes/U25-ComputerUse.md`

---
### C9 · Loop Engineering 与长跑 Agent

#### U21 · Loop Engineering（循环工程）

**核心问题**：怎么让 Agent 在跨小时、跨天、跨会话、跨人类的循环里持续工作，而不是每次重启都从零开始？

**必学概念**
- Loop Engineering 与 Harness / Context / Goal Engineering 的层次关系
- 5 个基本块 + Memory：Automations / Worktrees / Skills / Plugins / Sub-agents / State
- Loop 的解剖：Schedule → Triage → State → Worktree → Implementer → Verifier → MCP → Human Gate
- 成熟度等级：L1（report-only）→ L2（assisted fix）→ L3（unattended），必须渐进
- 7 个生产 pattern：Daily Triage / PR Babysitter / CI Sweeper / Dependency Sweeper / Changelog Drafter / Post-Merge Cleanup / Issue Triage
- 失败模式：partial completion / goal drift / self-check bias / token waste / handoff gap / 失控循环
- 配额机制：`should-run` 决策（run / wait / ask / self-repair / quiet-stop）
- 证据写回：spend 只在 validated writeback 之后，不在 quiet skip

**必读材料**
- [Cobus Greyling: Loop Engineering（Substack）](https://cobusgreyling.substack.com/p/loop-engineering) — 概念入门必读
- [Addy Osmani: Loop Engineering](https://addyosmani.com/blog/loop-engineering/) — 工程视角解读
- [LoopX 文档：principles and pitfalls](https://github.com/huangruiteng/loopx/blob/main/docs/product/loop-engineering-principles-and-pitfalls.md) — 控制面视角的 7 原则 + 8 坑
- [Loom Technical Report](https://zonodqioyxil6r3k.public.blob.vercel-storage.com/Loomline-v0.pdf) — 交付层白皮书

**练习**
- 用 `npx @cobusgreyling/loop-init . --pattern daily-triage` 初始化一个 loop，看 Loop Ready 分数
- 手画一个 daily-triage loop 的 mermaid 图（Schedule → Triage → State → ...）
- 把现有 ReAct Agent 拆成 implementer + verifier 两个 sub-agent，对比准确率
- 为一个多日项目写 `STATE.md` + `LOOP.md`，模拟"agent 重启后能恢复"

**主流项目（按角色）**

| 项目 | 角色 |
|------|------|
| [LoopX](https://github.com/huangruiteng/loopx) | 本地控制面（Codex / Claude Code / Cursor 通用） |
| [Loom](https://github.com/valkor-ai/loom) | 编程 Agent 的开源自适应交付层 |
| [paper_agent](https://github.com/whut09/paper_agent) | 论文精读的 Loop Engineering 案例 |
| [oh-my-hermes](https://github.com/rlaope/oh-my-hermes) | 多 agent 编排 + loop skills |
| [cobusgreyling/loop-engineering](https://github.com/cobusgreyling/loop-engineering) | pattern 库 + CLI 工具（loop-init/loop-audit/loop-cost） |
| [Nexent](https://github.com/ModelEngine-Group/nexent) | 零代码 Harness Engineering 平台 |

**笔记/资料**：`notes/concept-notes/U21-LoopEngineering.md`


#### U27 · Goal Engineering（目标工程）

**核心问题**：怎么把"用户的模糊意图"转化成 Agent 可执行的"明确目标"？为什么目标定义的质量决定了 Agent 90% 的表现？

**必学概念**
+- 90% Agent 失败源于目标定义不清（不是模型不够强）
+- SMART 原则 + 4 层目标结构（lifetime → strategic → milestone → task）
+- Acceptance Criteria 必须是机器可验证的（test / E2E / command）
+- 显式 Out of Scope
+- Failure Plan：失败时怎么办
+- 与 Loop Engineering 互补：Goal 定"做什么"，Loop 定"怎么持续做"

**必读材料**
+- [Cobus Greyling: Goal Engineering](https://github.com/cobusgreyling/goal-engineering) — 概念 + 工具
+- [oh-my-hermes ultragoal skill](https://github.com/rlaope/oh-my-hermes) — 长跑目标管理
+- [LoopX lifetime-goal invariant](https://github.com/huangruiteng/loopx/blob/main/docs/architecture.md)

**练习**
+- 为当前学习项目写一份 GOAL.md（含 acceptance criteria + out of scope）
+- 把一个 lifetime goal 拆到 task 层
+- 故意写一个模糊 goal，让 Agent 跑，再 re-engineer

**笔记/资料**：`notes/concept-notes/U27-GoalEngineering.md`

### C10 · Agent 运行时工程（Context + Harness）

> 这一层是 Agent 真正的"能力来源"——模型本身的能力有限，所有超出模型原生能力的工程都属于 Harness / Context 的范畴。这一层是 2025-2026 年最活跃的研究方向。

#### U22 · Context Engineering（上下文工程）

**核心问题**：怎么把"对的 token"放进模型有限的注意力预算里，让 Agent 在多轮、长时、复杂任务中稳定表现？

**必学概念**
+- Context vs Prompt：Context 是每个 turn 都要做的实时策划，Prompt 是一次性
+- Context Rot：context 越长，信息检索精度越下降（attention budget）
+- 5 大 Context 组件：system prompts / tools / examples / message history / external data
+- Just-in-time retrieval：让 agent 用工具自主探索（CLAUDE.md 预加载 + glob/grep 即时）
+- Long-horizon 三大技术：**Compaction**（历史压缩）、**Structured Note-taking**（NOTES.md / memory tool）、**Sub-agent Architectures**（每个 sub-agent 干净 context window）

**必读材料**
+- [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)（**官方权威定义，必读**）
+- [Anthropic: Writing tools for AI agents – with AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
+- [Anthropic: How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
+- [Chroma: Context Rot Research](https://research.trychroma.com/context-rot)

**练习**
+- 跑 50 轮对话，看 context 在第几轮开始"丢东西"
+- 实现 `compact_history(messages, max_tokens)` 函数
+- 设计 5-7 个工具，确保 agent 能清楚判断每个工具的"何时用"
+- 用 NOTES.md 模式：让 agent 在长任务中维护外部状态，重启后能续上

**笔记/资料**：`notes/concept-notes/U22-ContextEngineering.md`

#### U23 · Harness Engineering（Harness 工程）

**核心问题**：怎么把模型 + runtime 编排成一个**可审计、可验证、可分级暴露运行时支持**的 Agent 系统？

**必学概念**
+- Harness = 模型周围的 runtime substrate（不是 prompt 也不是 model 本身）
+- **11 个组件职责**：Task Spec / Context Selection / Tool Access / Project Memory / Task State / Observability / Failure Attribution / Verification / Permissions / Entropy Auditing / Intervention Recording
+- **H0-H3 阶梯**：H0 裸模型 → H1 只读 → H2 写+沙箱 → H3 全权+human gate
+- Trace-based eval：把每次 agent run 打包成可审计 episode package
+- 4 大设计原则：让模型"看到该看到的"、"做该做的"、"知道自己做到了"、"被人类看见"

**必读材料**
+- [arXiv 2605.13357: AI Harness Engineering](https://arxiv.org/abs/2605.13357)（**论文原文，11 组件 + H0-H3 的正式定义**）
+- [Dive into Claude Code 论文](https://arxiv.org/abs/2604.14228)（Harness 最佳实战案例）
+- [Claude Code Overview](https://code.claude.com/docs/en/overview)
+- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)
+- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)

**练习**
+- H0 vs H1 对比：同一任务用裸 prompt vs 给 read-only 工具
+- 给自己 agent 写 11 组件 audit，标注哪些实现了、哪些缺失
+- trace-based eval：跑 5 个任务，每个打包成 episode package
+- entropy audit：跑 50 turn 的 agent，监控 context token 数、loop 模式

**笔记/资料**：`notes/concept-notes/U23-HarnessEngineering.md`

#### 三层运行时工程对照（Prompt / Context / Harness / Loop）

| 层次 | 单元 | 关注 | 时机 | 度量 |
|------|------|------|------|------|
| 1（内） | Prompt (U3) | 怎么写 | 一次性 | 输出质量 |
| 2 | Context (U22) | 塞什么进 window | 每 turn | attention 利用 |
| 3 | Harness (U23) | runtime 怎么调度 | 每 turn | trace eval |
| 4（外） | Loop (U21) | 跨 session 持续 | 跨 session | Loop Ready score |

> 关键区分：**Context 关心信息**（塞什么）、**Harness 关心机制**（怎么塞、塞了之后怎么处理）、**Loop 关心持续**（跨 session 怎么续）。



#### U37 · Subagents 与任务隔离

**核心问题**：怎么用 subagent 模式隔离 context、并行处理大任务、做对抗式 review？

**必学概念**
+- Subagent = 主从层级，**独立 context + 独立 tool set + 独立 model**
+- 4 大使用场景：Codebase 探索 / Writer-Reviewer 对抗 / Verification / Specialized roles
+- Claude Code Subagents：`.claude/agents/*.md` frontmatter（name/description/tools/model）
+- LangGraph Subgraphs：独立 state schema
+- OpenAI `Agent.as_tool()`：含 HITL approval
+- 4 大设计原则：明确边界 / 独立 context / 结果摘要回主 / 失败传递
+- Subagent vs Skill：独立 context vs 共享 context
+- 并行 Subagent：`asyncio.gather` 跑多个 reviewer
+- Writer/Reviewer 循环：fresh context 避免 self-bias

**必读材料**
+- [Claude Code: Subagents](https://code.claude.com/docs/en/sub-agents) — 工业级 subagent 实践
+- [Claude Code: Best practices](https://code.claude.com/docs/en/best-practices) — "Use a subagent to investigate" 模式
+- [LangGraph: Subgraphs](https://langchain-ai.github.io/langgraph/concepts/subgraphs/) — 独立 state
+- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/agents/) — `Agent.as_tool()`

**练习**
+- 定义一个 security-reviewer subagent
+- 实现 Writer/Reviewer loop
+- 并行跑 security + performance + tests reviewer
+- 调试 subagent trace

**笔记/资料**：`notes/concept-notes/U37-Subagents.md`

---

#### U38 · Context 管理深入

**核心问题**：怎么管理 LLM 的有限 context window——什么时候 /clear、怎么 /compact、怎么 rewind、怎么 context budgeting？

**必学概念**
+- **核心约束**：Context 有限 + performance 随 context 衰减（"the most important resource to manage"）
+- 6 大工具：`/clear` / `/compact` / `Esc+Esc /rewind` / 自定义 compact / `/btw` / Context Budget
+- Compaction 4 种实现：简单 truncation / 滑动窗口 + summary / Hierarchical / Token-aware
+- 4 大优化策略：引用而非包含 / Progressive disclosure / JIT retrieval / Selective compaction
+- Context 性能曲线：0-30%（最佳）→ 30-60% → 60-80% → 80-95%（必须压缩）→ 95%+（必须 /clear）
+- 5 大反模式：一次性全读 / 从不压缩 / 错误累积 / 任务切换不清 / 长 conversation 不拆
+- Claude Code `/statusline` 监控 + CLAUDE.md 策略 + Plan mode 隔离

**必读材料**
+- [Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — 上下文腐烂 + 三大 long-horizon 技术
+- [Claude Code: Best practices](https://code.claude.com/docs/en/best-practices) — Context 是"最重要资源"
- [Chroma: Context Rot Research](https://research.trychroma.com/context-rot) — context 越多性能越差的实验

**练习**
+- 跑 /statusline 观察 context 增长
+- 触发 /compact，对比压缩前后输出
+- /clear 测试任务切换
+- 写 CLAUDE.md "When compacting, always preserve X"
+- 手工写 context budget manager

**笔记/资料**：`notes/concept-notes/U38-ContextManagement.md`

---

#### U40 · Verification / Adversarial Review

**核心问题**：怎么让 agent 自己验证自己的输出？怎么用 adversarial review 抓出 LLM 的 self-bias？

**必学概念**
+- **核心问题**：LLM 有 self-bias——"I've completed" 经常只完成 70%
+- 4 大 verification 模式：① 同 prompt 跑 check ② /goal 持续 condition ③ Stop hook 强制 ④ Adversarial review
+- 5 大工具：Test suite / Lint / Type check / Visual diff / Build verification
+- Writer/Reviewer loop（fresh context 避免 self-bias）
+- Verification subagent：`.claude/agents/code-reviewer.md`
+- Self-assertion vs **Self-evidence**：要求"贴 test output"而不是"我完成了"
+- Adversarial review 反模式：over-engineering / 找 nits
+- Stop hook：`exit 0` 放行 / `exit 2` 拒绝（8 次后 override）

**必读材料**
+- [Claude Code: Best practices](https://code.claude.com/docs/en/best-practices) — "Give Claude a way to verify" + "Adversarial review"
- [Claude Code: Hooks](https://code.claude.com/docs/en/hooks) — Stop hook 实战
- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — "Evaluator-optimizer" 模式

**练习**
+- 配 Stop hook 跑 `npm test`，故意让测试失败看 agent 怎么修
+- 跑 Writer/Reviewer loop 3 个 iteration
+- 让 agent 跑 task 后必须贴 test output
+- 跑 multi-stage review（security + performance + tests 并行）

**笔记/资料**：`notes/concept-notes/U40-Verification.md`

#### U31 · Structured Output & Validation（结构化输出）

**核心问题**：怎么让 LLM 输出**可靠、可解析、符合 schema** 的结构化数据？

**必学概念**
+- 4 大技术层级：Prompt 引导 / 库辅助 / Provider 原生 / 自定义 Grammar
+- **OpenAI Structured Outputs**（`strict: true`）：token-level 约束，100% schema 遵循
+- **Anthropic Tool Use**：用 input_schema 约束
+- **Instructor**：Pydantic + LLM，自动 retry
+- **Outlines**：logit masking（本地模型首选）
+- **Pydantic** validators + 自动 retry + fallback chain
+- 多 tool Union（互斥）/ Parallel tool calls（并行）/ 嵌套 schema

**必读材料**
+- [OpenAI Structured Outputs 指南](https://platform.openai.com/docs/guides/structured-outputs)
+- [Instructor 文档](https://github.com/jxnel/instructor)
+- [Outlines 文档](https://outlines-dev.github.io/outlines/)
+- [Pydantic 文档](https://docs.pydantic.dev/)

**练习**
+- 用 Instructor + Pydantic 抽取结构化数据
+- 用 OpenAI strict mode 实现 100% schema 遵循
+- 写自定义 Pydantic validator + retry + fallback

**笔记/资料**：`notes/concept-notes/U31-StructuredOutput.md`

---

#### U33 · Human-in-the-loop 设计（人在环设计）

**核心问题**：怎么在 Agent 执行链中合理嵌入人类决策点？4 大框架的 HITL 模式分别怎么实现？

**必学概念**
- 4 大 HITL 模式：Approval / Interrupt / Oversight / Confidence-based
- 5 级风险分级（L0-L5）：从查天气到法律医疗
- **LangGraph** `interrupt()` + `Command(resume)` + 6 大铁律（不要 try/except、每节点一次、节点重跑、可重入、JSON-serializable、thread_id 稳定）
- **OpenAI Agents SDK** 声明式 `needs_approval` + `RunState` 持久化 + sticky decision
- **Microsoft AutoGen** 3 种 `human_input_mode`（NEVER / TERMINATE / ALWAYS）
- **Claude Code** Hooks（PreToolUse 拦截 + bash 脚本，工业级 shell-based HITL）
- **Confidence-based delegation**：self-consistency / logit-based / verifier model / calibration
- **Quorum 协议**：unanimous / majority / threshold / weighted / hierarchical
- **HITL × RLHF / DPO**：把人类决策转成训练数据，retrain agent
- **HITL × Loop Engineering**：跨 session 持久化 HITL 决策
- Async HITL queue（Redis）+ 多渠道通知
- 反馈学习：把人类决策写入 memory / 训练集

**必读教程（4 大框架官方）**
- [LangGraph: Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) — **最权威**的 interrupt() + Command(resume) 模式 + 6 大铁律
- [OpenAI Agents SDK: HITL](https://openai.github.io/openai-agents-python/human_in_the_loop/) — 声明式 needs_approval + RunState 持久化
- [Microsoft AutoGen: Human Feedback in Agents](https://microsoft.github.io/autogen/0.2/docs/tutorial/human-in-the-loop) — 3 种 human_input_mode
- [Anthropic: Claude Code Hooks](https://code.claude.com/docs/en/hooks) — PreToolUse 拦截工业实践
- [EU AI Act Article 14: Human Oversight](https://artificialintelligenceact.eu/article/14/) — 监管视角

**必读论文（精选 10+ 篇）**
- [Human-In-The-Loop Software Development Agents: Challenges and Future Directions](https://arxiv.org/abs/2506.11009) — Atlassian 实战 + challenges
- [CentaurEval](https://arxiv.org/abs/2512.04111) — 怎么测 HITL 的"价值"
- [Handoff Debt: The Rediscovery Cost](https://arxiv.org/abs/2606.02875) — Coding agent 接手被中断任务的成本
- [When Users Change Their Mind: Interruptible Agents in Long-Horizon Web Navigation](https://arxiv.org/abs/2604.00892) — 用户中途改主意的场景
- [Enabling Self-Improving Agents to Learn at Test Time With HITL Guidance](https://arxiv.org/abs/2507.17131) — Test-time learning

**练习**
- 用 LangGraph `interrupt()` 实现需要人类确认的 agent
- 用 OpenAI `needs_approval` + `RunState` 写一个跨进程可恢复的 approval
- 用 AutoGen 3 种 mode 跑同一对话，对比人类介入次数
- 用 Claude Code Hooks 拦截 `rm -rf` 和 `git push --force`
- 实现 risk classifier + Redis HITL queue + quorum 协议
- 用 self-consistency 实现 LLM 置信度估算 + calibration
- 把 HITL 决策转成 DPO 训练数据，跑一次 retrain

**笔记/资料**：`notes/concept-notes/U33-HumanInTheLoop.md`

### C11 · Agent 能力工程化（Skills + AGENTS.md）

> 把"项目知识 / 工具说明 / 最佳实践"以**可发现、可加载、可组合**的方式提供给 Agent，而不是堆在 system prompt 里。这是 2025-2026 工业界新共识。

#### U26 · Agent Skills & AGENTS.md

**核心问题**：怎么把"项目知识"以可发现、可加载、可组合的方式提供给 Agent？

**必学概念**
+- 三大规范对比：AGENTS.md（GitHub 开放标准）/ CLAUDE.md（Claude Code）/ Skills（Anthropic）
+- 何时用 Skill vs System Prompt vs Sub-agent
+- 多级 AGENTS.md：根目录 + 子目录分层，模块特定规则
+- Skill 设计原则：单一职责 / 明确 when_to_use / 可执行 steps / 明确 output format / 含示例
+- 工具映射：Claude Code / Cursor / Cline / Aider / Codex 各自的规则系统

**必读材料**
+- [GitHub: AGENTS.md 规范](https://agents.md/) — 开放标准
+- [Anthropic: Skills 文档](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/overview) — 最成熟实现
+- [Matt Pocock: Skills](https://github.com/mattpocock/skills) — 高质量示例
+- [Cursor: .cursorrules 文档](https://docs.cursor.com/context/rules)

**练习**
+- 给自己项目写 50 行内的 AGENTS.md
+- 写一个 code-review Skill（SKILL.md + scripts/ + references/）
+- 多级 AGENTS.md：root + 子目录两层

**笔记/资料**：`notes/concept-notes/U26-AgentSkills.md`

#### U41 · Agent 全栈项目学习路线（跨单元整合）

**核心问题**：学完所有概念单元后，怎么"按项目学"——把分散的概念整合成可展示的端到端项目？哪些项目既能深度整合多个单元，又能在简历/面试中展示？

**必学概念**
+- 6 个项目推荐（按难度递增）：ReAct 原型 → RAG 知识库 → Multi-Agent 协作 → Computer Use → Reasoning + Tool Use → Long-Running + Loop Engineering
+- 单元覆盖矩阵：每个项目对应一组核心单元
+- 项目驱动的学习循环：选项目 → 单元复习 → MVP → 逐功能 → eval → 复盘
+- 简历转化：README + 博客 + demo 视频 + 量化指标
+- datawhale 教程集成：hello-agents (63k⭐) / all-in-rag (9k⭐) / learn-claude-code (62k⭐) / hugging-multi-agent

**6 个项目核心**
| 项目 | 难度 | 时间 | 覆盖单元 | 简历价值 |
|------|------|------|----------|----------|
| **P1 ReAct 原型** | ⭐ | 20-30h | U1/U3/U5/U6/U-DL | ★★★ |
| **P2 RAG 知识库** | ⭐⭐ | 30-40h | U2/U4/U8/U15/U16/U19/U22/U31 | ★★★★ |
| **P3 Multi-Agent 协作** | ⭐⭐⭐ | 40-50h | U9/U11/U13/U22/U31/U36 | ★★★★★ |
| **P4 Computer Use / GUI Agent** | ⭐⭐⭐ | 40-50h | U17/U20/U25/U29/U34 | ★★★★★ |
| **P5 Reasoning + Tool Use** | ⭐⭐⭐ | 30-40h | U24/U6/U7/U9/U28 | ★★★★★ |
| **P6 Long-Running + Loop Engineering** | ⭐⭐⭐⭐ | 60-80h | U21/U22/U23/U27/U29/U33/U37/U40 | ★★★★★ |

**校招推荐组合**（3 个月，2.5h/天）：
+- **M1**：P1 ReAct 原型
+- **M2**：P2 RAG 知识库
+- **M3**：P3 Multi-Agent 或 P5 Reasoning

**必读资料**
+- [datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents) — 63k⭐ 中文最系统 Agent 教程
+- [datawhalechina/all-in-rag](https://github.com/datawhalechina/all-in-rag) — 9k⭐ RAG 全栈
+- [jjyaoao/HelloAgents](https://github.com/jjyaoao/helloagents) — OpenAI 原生自研 multi-agent 框架
+- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) — Claude Code 复刻

**练习**
+- 选 1 个项目（推荐 P1 起步）
+- MVP：50 行 Python 跑通最小流程
+- 加 eval：U12 / U35 工具测质量
+- 写 README + 量化指标

**笔记/资料**：`notes/concept-notes/U41-AgentProjectPath.md`

#### U42 · 面试真题与学习顺序（校招向）

**核心问题**：校招 Agent 工程岗怎么高效准备？41 单元的学习顺序怎么排？真实面试会问什么、怎么答？

**必学概念**
+- **校招画像**：主流 Agent 团队（OpenAI / Anthropic / Manus / 智谱 / 字节 Coze / Cursor 等）
+- **面试流程**：简历 → 初面（LeetCode + 概念 + 项目）→ 终面（系统设计 + 行为）
+- **41 单元学习顺序**：4 阶段（基础 / 核心 / 差异化 / 深度）+ 3 个月时间预算
+- **真实面试真题 Top 29**：按主题（LLM 基础 / Prompt / RAG / Agent / 推理 / 后训练 / 生产 / 行为）
+- **Top 30 必读论文**：按主题整理（LLM / Agent 范式 / RAG / Reasoning / Memory / Context / Multi-Agent / Coding / Safety / Regulation）
+- **简历策略**：3 大模块 + STAR 模板 + 量化指标
+- **阶段自测 Checklist**：每阶段完成检验
+- **3 个月时间表**：W1-12 学习 + 投递 + mock 节奏

**校招 3 个月时间表**
| 周 | 内容 | 关键产出 |
|----|------|----------|
| W1-2 | 阶段 1 全部（U1/U3/U5/U6/U7 + RAG 基础） | 跑通 ReAct + 基础 RAG |
| W3-4 | 阶段 2 前半（U8/U9/U11/U15/U18） | 1 个 Memory + Multi-Agent 项目 |
| W5-6 | 阶段 2 后半 + 阶段 3 选 1 | 1 个差异化亮点（Reasoning / 后训练 / Guardrails） |
| W7-8 | 阶段 4 + U41 项目 | 2-3 个端到端项目 |
| W9-10 | 模拟面试 + 刷题 | 至少 5 场 mock interview |
| W11-12 | 投递 + 真面试 | 拿到 offer |

**必读资料**
+- [datawhale/hello-agents 面试题总结](https://github.com/datawhalechina/hello-agents/blob/main/Extra-Chapter/Extra01-面试问题总结.md) — 中文 Agent 面试最全
+- [Anthropic Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
+- [OpenAI Agents SDK HITL](https://openai.github.io/openai-agents-python/human_in_the_loop/)
+- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
+- [EU AI Act Article 14](https://artificialintelligenceact.eu/article/14/) — 监管要求

**笔记/资料**：`notes/concept-notes/U42-InterviewGuide.md`

---

### C12 · 生产工程（Inference + Deploy）

> 把 Agent 从"我笔记本上能跑"变成"生产环境 7×24 小时稳定服务"——Inference 性能 + 部署沙箱 + 可靠性。

#### U28 · LLM Inference & Cost/Latency 工程

**核心问题**：怎么让 Agent 调用 LLM 时**更快、更便宜、更稳定**？

**必学概念**
+- 5 大成本来源：input / output / cache miss / retry / reasoning overhead
+- 6 大优化：Prompt Caching / Context Compression / Model Routing / Batching & Concurrency / Streaming TTFT / Local Serving
+- KV Cache / PagedAttention / Continuous Batching / Speculative Decoding / Quantization
+- vLLM / SGLang / TGI / TensorRT-LLM 对比
+- 关键指标：TTFT / TPS / latency P95 / cache hit rate / cost per 1k queries
+- Reliability：限流 / 重试（指数退避） / Circuit Breaker / Fallback Chain

**必读材料**
+- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
+- [vLLM Paper](https://arxiv.org/abs/2309.06180) — PagedAttention
+- [SGLang Paper](https://arxiv.org/abs/2312.07104) — RadixAttention
+- [Karpathy: LLM inference](https://www.youtube.com/watch?v=zjkBMFhNj_g) — 通俗讲解

**练习**
+- 跑 1000 个 query 测 P50/P95/P99 延迟 + 成本
+- 加 Anthropic prompt cache，对比 cache 命中前后 cost
+- 用 vLLM 本地部署一个 7B 模型
+- 实现 circuit breaker + fallback chain

**笔记/资料**：`notes/concept-notes/U28-LLMInference.md`

#### U29 · Agent 部署 & Sandbox DevOps

**核心问题**：怎么把 Agent 从"我笔记本上能跑"变成"生产环境 7×24 小时稳定服务"？

**必学概念**
+- 4 层部署架构：用户接入 / Gateway / Orchestrator / Sandbox
+- Sandbox 隔离方案：Process < Docker < gVisor < Firecracker < VM
+- Coding Agent 沙箱实践（Docker 限制文件 + 网络 + 资源）
+- Computer Use 强隔离（远程 VM snapshot）
+- Session / State 持久化（Postgres + Redis + checkpoint）
- LLM Gateway：Helicone / OpenRouter / 自建 FastAPI
- CI/CD for Agent：prompt regression test + tool mock + eval
- 生产 checklist：sandbox / reliability / observability / security / DR

**必读材料**
- [Docker 文档](https://docs.docker.com/) / [gVisor](https://gvisor.dev/) / [Firecracker](https://github.com/firecracker-microvm/firecracker)
- [LangGraph Checkpointing](https://langchain-ai.github.io/langgraph/concepts/persistence/)
- [Langfuse](https://langfuse.com/) / [Helicone](https://www.helicone.ai/) / [OpenLLMetry](https://opentelemetry.io/)
- [Anthropic: Computer Use safety](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use)
- [12-Factor App](https://12factor.net/)

**练习**
| [EU AI Act Article 14: Human Oversight](https://artificialintelligenceact.eu/article/14/) | 欧盟 AI 法案对 human oversight 的法律要求 | U33 |
- 用 LangGraph + Postgres 做一个能 resume 的 agent
- 跑通 Helicone 监控 1000 个请求的 cost / latency

**笔记/资料**：`notes/concept-notes/U29-AgentDeploy.md`


#### U39 · Permission & Auto Mode（权限与自动模式）

**核心问题**：怎么让 agent 安全地**自主执行**有副作用的操作？怎么在"自主"和"安全"之间找平衡？

**必学概念**
+- **核心问题**：LLM agent 能做不可逆操作（写文件 / 跑命令 / push 代码）= 必须有 permission
+- Claude Code 4 个 permission mode：default / accept-edits / plan / **auto**
+- **Auto mode**：用 classifier model 自动判断哪些 action 安全——长跑任务的杀手特性
+- 4 大 Permission 策略：永久 allowlist / 永久 denylist / 询问模式 / Pattern-based hook
+- 6 种 Hook 类型：PreToolUse / PostToolUse / Stop / SubagentStart / SubagentStop / UserPromptSubmit
+- Hook 规则：`exit 0` 放行 / `exit 2` 拒绝（stderr 作为 reason）
+- Sandbox 隔离（OS-level）：Process < Docker < gVisor < Firecracker < VM
+- 5 大 Permission 反模式：完全 auto 无 allowlist / 完全 deny / denylist 不更新 / hook 静默失败 / sandbox 配置错

**必读材料**
+- [Claude Code: Permissions](https://code.claude.com/docs/en/permissions) — 官方 permission 文档
- [Claude Code: Permission modes](https://code.claude.com/docs/en/permission-modes) — 4 种 mode 详解
- [Claude Code: Sandboxing](https://code.claude.com/docs/en/sandboxing) — OS-level 隔离
- [Claude Code: Hooks](https://code.claude.com/docs/en/hooks) — Hook 系统

**练习**
+- 配置个人 Claude Code Permission（allowlist / denylist / hooks）
+- 写一个 hook 拒绝 `rm -rf` 和 `curl * | bash`
+- 跑 auto mode 跑长任务
- 配置 sandbox 限制 filesystem
- 测试 hook 失败行为

**笔记/资料**：`notes/concept-notes/U39-Permission.md`

---

## 单元完成标准

每个单元完成后，需要达到：

1. 能**画概念图**（Mermaid 或手画）
2. 能**口述核心机制**（2 分钟内讲清楚）
3. 有**可运行的最小代码**（30 行以内）
4. 能回答对应的 **Gate 问题**（见各单元末尾）

---

## 深度学习 / RL / 微调：要不要学？

**简短答案：不是重心，有余力再补。**

Agent 工程岗的核心竞争力是**系统设计 + Prompt + API 调用**，不需要自己训练模型。

| 方向 | 是否必修 | 理由 | 推荐策略 |
|------|---------|------|---------|
| **深度学习基础** | 轻量扫盲 | 理解 LLM 原理的必要基础 | 学到"能直观解释 Attention"即可，不需要会训练 |
| **强化学习 (RL)** | 选学 | RLHF 是 Agent 自我改进的理论基础 | 了解概念（Reward / Policy / RLHF），不硬磕推导 |
| **模型微调 (Fine-tuning)** | 选学 | 多数 Agent 产品用预训练模型即可 | 有余力用 unsloth/llamafactory 跑过 LoRA 即可 |

详见 `notes/concept-notes/UD-DL深度学习扫盲.md`。

---

## 官方必读（入门级）

| 文章 | 为什么重要 |
|------|-----------|
| [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | Agent 设计入门必读，讲清 workflow 和 agent 的边界 |
| [OpenAI: A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/) | 面向产品和工程团队的 agent 落地指南 |
| [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling) | 掌握 tool-calling 的标准 API 格式 |
| [Claude Tool Use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) | Claude 工具调用机制 |

---

## 开源学习路线与项目（精选）

> 以下项目均为有完整代码、清晰文档、可直接学习的工程导向资源，按推荐程度排序。

### 第一梯队：从零构建（必做）

| Repo | Stars | 特点 | 适合单元 |
|------|-------|------|----------|
| [hello-agents](https://github.com/datawhalechina/hello-agents) | 53.6k | 中文智能体系统教程，795 commits，code+docs 配套 | C1~C5 |
| [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | 62.7k | 从 0 到 1 复刻 Claude Code-like agent harness | U20 |
| [claw0](https://github.com/shareAI-lab/claw0) | 2.7k | 从 0 到 1 构建 OpenClaw-like agent gateway | C6 |
| [GenAI_Agents](https://github.com/NirDiamant/GenAI_Agents) | 22.2k | 50+ 教程，涵盖从简单对话到复杂多 agent 系统 | 全部 |
| [agents-towards-production](https://github.com/NirDiamant/agents-towards-production) | 20.5k | Production-grade GenAI agent 端到端教程 | C7 |

### 第二梯队：框架与产品级参考

| Repo | Stars | 特点 | 适合单元 |
|------|-------|------|----------|
| [HelloAgents](https://github.com/jjyaoao/HelloAgents) | 1.8k | 基于 OpenAI 原生 API 的生产级多智能体框架 | C5, C6 |
| [smolagents](https://github.com/huggingface/smolagents) | 5.9k | Hugging Face 轻量 agent 框架，CodeAgent 风格 | C4 |
| [DeerFlow](https://github.com/bytedance/deer-flow) | 13.4k | 字节开源 long-horizon SuperAgent harness | C6, C8 |
| [OpenHands](https://github.com/All-Hands-AI/OpenHands) | 48.6k | 开源 coding agent | C8 (U20) |
| [SWE-agent](https://github.com/SWE-agent/SWE-agent) | 12.3k | 软件工程 agent | C8 (U20) |
| [browser-use](https://github.com/browser-use/browser-use) | 27.7k | 浏览器 agent 实战 | C8 (U17) |

### 第三梯队：专题深入

| Repo | Stars | 特点 | 适合单元 |
|------|-------|------|----------|
| [khoj](https://github.com/khoj-ai/khoj) | 19.4k | 个人 second brain | C5 (U11) |
| [mem0](https://github.com/mem0ai/mem0) | 11.9k | Agent 记忆层组件 | C5 (U11) |
| [Letta](https://github.com/letta-ai/letta) | 11.7k | Stateful agents memory/context 平台 | C5 (U11) |
| [GPT Researcher](https://github.com/assafelovic/gpt-researcher) | 27.1k | 资料研究助手 | C3 (U15) |
| [RAGFlow](https://github.com/infiniflow/ragflow) | 26.5k | 文档理解型 RAG 引擎 | C3 (U15) |

### 第四梯队：新兴方向（2025-2026 emerging）

> 2025-2026 年新出现的方向，概念还在演化但项目已经可用。

| Repo | Stars | 角色 | 适合单元 |
|------|-------|------|----------|
| [DeepSeek R1](https://github.com/deepseek-ai/DeepSeek-R1) | — | 开源推理模型代表（MIT 许可） | C1 (U24) |
| [OpenAI o1/o3](https://openai.com/index/openai-o1-system-card/) | — | 闭源推理模型代表 | C1 (U24) |
| [browser-use](https://github.com/browser-use/browser-use) | 27.7k | Playwright + LLM 浏览器 Agent | C8 (U25) |
| [UI-TARS](https://github.com/bytedance/UI-TARS) | — | 字节开源端到端多模态 GUI Agent | C8 (U25) |
| [Matt Pocock Skills](https://github.com/mattpocock/skills) | — | 高质量 skills 示例库 | C11 (U26) |
| [cobusgreyling/goal-engineering](https://github.com/cobusgreyling/goal-engineering) | — | Goal Engineering 概念 + 工具 | C9 (U27) |
| [vLLM](https://github.com/vllm-project/vllm) | — | 高吞吐 LLM serving | C12 (U28) |
| [SGLang](https://github.com/sgl-project/sglang) | — | 结构化输出 + RadixAttention | C12 (U28) |
| [Helicone](https://www.helicone.ai/) | — | LLM 专用 gateway | C12 (U28) |
| [Langfuse](https://langfuse.com/) | — | LLM observability | C12 (U28) |
| [LoopX](https://github.com/huangruiteng/loopx) | — | 本地控制面，跨 Agent 运行时 | C9 (U21) |
| [Loom](https://github.com/valkor-ai/loom) | 443 | 编程 Agent 的开源自适应交付层 | C8, C9 (U20, U21) |
| [cobusgreyling/loop-engineering](https://github.com/cobusgreyling/loop-engineering) | 4.3k | pattern 库 + CLI 工具 | C9 (U21) |
| [oh-my-hermes](https://github.com/rlaope/oh-my-hermes) | 39 | 多 agent 编排 + loop skills | C6, C9 (U9, U21) |
| [ModelEngine-Group/nexent](https://github.com/ModelEngine-Group/nexent) | 5.4k | 零代码 Harness Engineering 平台 | C10 (U23) |
| [Claude Code](https://code.claude.com/) | — | Harness 11 组件的完整工业实现 | C10 (U23) |
| [Codex CLI](https://github.com/openai/codex) | — | OpenAI 的 harness 参考实现 | C10 (U23) |
| [MCP 官方](https://modelcontextprotocol.io/) | — | Anthropic 推出的统一工具协议（USB-C for LLM）| C6 (U36) |
| [FastMCP](https://github.com/jlowin/fastmcp) | — | Python 快速写 MCP server | C6 (U36) |
| [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) | 3.5k+ | LangChain 接入 MCP | C6 (U36) |
| [MCP Inspector](https://github.com/modelcontextprotocol/inspector) | — | MCP 调试器（类似 Postman）| C6 (U36) |
| [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) | — | subagent + as_tool + HITL 模式 | C10 (U33, U37) |
| [LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) | — | 中文友好的全功能训练框架（500+ 模型） | C1 (U32) |
| [unsloth](https://github.com/unslothai/unsloth) | — | 2-5x 加速 + 4-bit 量化训练 | C1 (U32) |
| [trl](https://github.com/huggingface/trl) | — | HuggingFace 官方 SFT/DPO/GRPO/PPO | C1 (U30, U32) |
| [Instructor](https://github.com/jxnel/instructor) | — | Pydantic + LLM，结构化输出首选 | C10 (U31) |
| [Outlines](https://github.com/outlines-dev/outlines) | — | Logit masking 约束生成（本地模型） | C10 (U31) |
| [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) | — | 工业级全栈护栏（Colang） | C7 (U34) |
| [Guardrails AI](https://github.com/guardrails-ai/guardrails) | — | Pydantic 风格 output 验证 + Hub validators | C7 (U34) |
| [Promptfoo](https://github.com/promptfoo/promptfoo) | — | 声明式 LLM 评估 + Red Team + CI 友好 | C7 (U35) |
| [DeepEval](https://github.com/confident-ai/deepeval) | — | pytest 风格 LLM 评估（14+ metrics） | C7 (U35) |
| [RAGAS](https://github.com/explodinggradients/ragas) | — | RAG 评估事实标准（4 大指标） | C3, C7 (U15, U35) |
| [LangGraph](https://github.com/langchain-ai/langgraph) | 15k+ | interrupt() + Command(resume) canonical HITL 模式 | C6, C10 (U9, U33) |
| [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) | — | 声明式 needs_approval + RunState 持久化 | C10 (U33) |
| [Microsoft AutoGen](https://github.com/microsoft/autogen) | 28k+ | 3 种 human_input_mode（NEVER / TERMINATE / ALWAYS） | C6, C10 (U9, U33) |
| [datawhale/hello-agents](https://github.com/datawhalechina/hello-agents) | 63k+ | 中文最系统 Agent 教程（16 章 + 13 Extra）| U41 |
| [datawhale/all-in-rag](https://github.com/datawhalechina/all-in-rag) | 9k+ | RAG 技术全栈（10 章 + 2 项目）| U41 |
| [jjyaoao/HelloAgents](https://github.com/jjyaoao/helloagents) | — | OpenAI 原生 API 自研 multi-agent 框架 | C6, U41 |
| [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | 69k+ | Claude Code 复刻教程（s01-s20）| C10, U41 |
| [datawhale/agent-skills-anthropic](https://github.com/datawhalechina/agent-skills-with-anthropic) | 1.3k+ | 吴恩达 Agent Skills 课程中文版 | C11 |

---

## 学习参考来源

本路线参考自 [Agent Learning Hub](https://github.com/datawhalechina/Agent-Learning-Hub)（Datawhale 维护，1.6k⭐），整合了以下核心资源：

### 必读官方文档

| Resource | Why | Unit |
|----------|-----|------|
| [Claude Code Overview](https://code.claude.com/docs/en/overview) | 当前最值得拆解的 coding agent 产品 | U20 |
| [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents) | 学任务拆分、上下文隔离、专用子代理 | U9, U20 |
| [Claude Code Hooks](https://code.claude.com/docs/en/hooks) | 学如何拦截、校验和扩展 agent 行为 | U20 |
| [Model Context Protocol](https://modelcontextprotocol.io/) | Agent 连接外部工具和数据源的重要协议 | U6 |
| [OpenAI Agents SDK](https://platform.openai.com/docs/guides/agents-sdk/) | OpenAI 原生 agent 开发入口 | U6, U9 |
| [Google Agent Development Kit](https://google.github.io/adk-docs/) | Google 的多 agent 开发框架 | U9, U13 |
| [LangGraph](https://langchain-ai.github.io/langgraph/) | 状态图和可控 agent 编排 | U7, U9 |
| [Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Context Engineering 的官方权威定义（compaction / note-taking / sub-agent） | U22 |
| [LangGraph: Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) | 权威的 interrupt() + Command(resume) 模式 + 6 大铁律 | U33 |
| [OpenAI Agents SDK: HITL](https://openai.github.io/openai-agents-python/human_in_the_loop/) | 声明式 needs_approval + RunState 持久化 | U33 |
| [Microsoft AutoGen: Human Feedback in Agents](https://microsoft.github.io/autogen/0.2/docs/tutorial/human-in-the-loop) | 3 种 human_input_mode（NEVER / TERMINATE / ALWAYS） | U33 |
| [Anthropic: Claude Code Hooks](https://code.claude.com/docs/en/hooks) | PreToolUse 拦截，shell-based HITL 工业实践 | U33 |
| [EU AI Act Article 14: Human Oversight](https://artificialintelligenceact.eu/article/14/) | 欧盟 AI 法案对 human oversight 的法律要求 | U33 |
| [Claude Code: Best Practices](https://code.claude.com/docs/en/best-practices) | Context 是最重要资源 + Verification + Adversarial Review + Subagents | U37, U38, U40 |
| [Claude Code: Permissions](https://code.claude.com/docs/en/permissions) | 4 种 permission mode + allowlist / denylist / hooks | U39 |
| [Claude Code: Checkpointing](https://code.claude.com/docs/en/checkpointing) | rewind / /compact / checkpoint 机制 | U38 |
| [Claude Code: Subagents](https://code.claude.com/docs/en/sub-agents) | `.claude/agents/*.md` + independent context | U37 |
| [Claude Code: Hooks](https://code.claude.com/docs/en/hooks) | 6 种 hook 类型（Pre/Post/Stop/SubagentStart 等）| U40 |

### 从零构建参考项目

| Project | Learn | Unit |
|---------|-------|------|
| [hello-agents](https://github.com/datawhalechina/hello-agents) | 中文教程，从零构建智能体 | C1~C5 |
| [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | 从 0 到 1 复刻 Claude Code-like agent harness | U20 |
| [claw0](https://github.com/shareAI-lab/claw0) | 从 0 到 1 学 OpenClaw-like gateway | C6 |
| [smolagents](https://github.com/huggingface/smolagents) | Hugging Face 轻量 agent 框架，CodeAgent 思路 | C4 |
| [jjyaoao/HelloAgents](https://github.com/jjyaoao/HelloAgents) | 基于 OpenAI 原生 API 的生产级多智能体框架 | C5, C6 |

### 经典论文

| Paper | Topic | Unit |
|-------|-------|------|
| [Attention is All You Need](https://arxiv.org/abs/1706.03762) | Transformer 基础 | U-DL |
| [InstructGPT](https://arxiv.org/abs/2203.02155) | RLHF | U-DL, U1 |
| [ReAct](https://arxiv.org/abs/2210.03629) | Reasoning + acting | U5 |
| [Toolformer](https://arxiv.org/abs/2302.04761) | 模型学习何时调用工具 | U6 |
| [Reflexion](https://arxiv.org/abs/2303.11366) | 语言反馈和自我改进 | U8 |
| [Generative Agents](https://arxiv.org/abs/2304.03442) | 记忆、反思、规划 | U8 |
| [Voyager](https://arxiv.org/abs/2305.16291) | 开放世界中的长期学习 | U8 |
| [SWE-bench](https://arxiv.org/abs/2310.06770) | 真实 GitHub issue 修复评测 | U10, U12, U20 |
| [DSPy](https://arxiv.org/abs/2310.03714) | 声明式 LLM 编程 | U14 |
| [Tree of Thoughts](https://arxiv.org/abs/2305.10601) | Deliberate Problem Solving | U7 |
| [Dive into Claude Code](https://arxiv.org/abs/2604.14228) | 从系统设计角度分析 Claude Code | U20 |
| [AI Harness Engineering](https://arxiv.org/abs/2605.13357) | 把 harness 作为 agent 能力来源（11 组件 + H0-H3 阶梯） | U23 |
| [DeepSeek R1](https://arxiv.org/abs/2501.12948) | 开源推理模型代表（test-time compute scaling 实证） | U24 |
| [DPO](https://arxiv.org/abs/2305.18290) | Direct Preference Optimization（替代 RLHF） | U30 |
| [Human-In-The-Loop Software Development Agents: Challenges and Future Directions](https://arxiv.org/abs/2506.11009) | HITL 在多 agent 软件开发中的挑战与未来（Atlassian 实战） | U33 |
| [CentaurEval](https://arxiv.org/abs/2512.04111) | Benchmarking HITL value in agentic coding | U33 |

---

## Agent 全景图（市面全收录）

> 覆盖：商业产品 · 开源框架 · 学术研究 · 工具平台

### 一、商业级 Agent 产品

| 产品 | 公司 | 类型 | 核心能力 |
|------|------|------|---------|
| [Claude Code](https://code.claude.com/) | Anthropic | Coding Agent | 代码编写、文件编辑、git、shell、subagent、hooks |
| [Devin](https://devin.ai/) | Cognition | Coding Agent | 端到端工程任务，自主推理，SWEBench 领先 |
| [Cursor](https://cursor.com/) | Cursor | AI IDE | Copilot++，规则文件（.cursorrules），多文件编辑 |
| [GitHub Copilot](https://github.com/features/copilot) | Microsoft | Coding Assistant | 代码补全、多文件推理、IDE 深度集成 |
| [Copilot Workspace](https://github.com/features/copilot) | Microsoft | Coding Agent | 自然语言驱动，任务规划，代码修改 |
| [Amazon Bedrock Agent](https://aws.amazon.com/bedrock/agents/) | AWS | Cloud Agent | 云上部署，多 Tool，企业级安全 |
| [Azure AI Agent](https://azure.microsoft.com/en-us/products/ai-services/agent-service) | Microsoft | Cloud Agent | 多 Agent 编排，RAG，数据连接 |
| [Gemini Agent](https://deepmind.google/gemini) | Google | 多模态 Agent | 文本/代码/视觉/音频，统一 Agent 框架 |
| [Windsurf](https://windsurf.com/) | Windsurf | AI IDE | Agent 模式，cascade AI，多文件编辑 |

---

### 二、开源 Agent 框架

#### 2.1 通用 Agent 框架

| 框架 | Stars | 语言 | 核心特点 | 适用场景 |
|------|-------|------|---------|----------|
| [LangChain](https://github.com/langchain-ai/langchain) | 35k+ | Python/JS | 全生态：LLM/Tool/Memory/RAG/Graph | 快速原型，生产级 |
| [LangGraph](https://github.com/langchain-ai/langgraph) | 15k+ | Python | 状态机 + Cycles + 人机交互 | 复杂 Agent 编排 |
| [LlamaIndex](https://github.com/run-llama/llama-index) | 25k+ | Python | 专注 RAG + 数据检索，索引优化 | RAG 专家，私有数据 |
| [CrewAI](https://github.com/crewAIInc/crewAI) | 22k+ | Python | 角色驱动（role）+ 任务流，简单直观 | Multi-Agent 快速搭建 |
| [AutoGen](https://github.com/microsoft/autogen) | 28k+ | Python | 微软出品，多 Agent 对话，代码执行 | Multi-Agent 研究 |
| [MetaGPT](https://github.com/geekan/MetaGPT) | 35k+ | Python | 角色扮演（PM/DEV/QA），SOP 驱动 | Software Agent |
| [Swarm](https://github.com/openai/swarm) | 25k+ | Python | OpenAI 出品，轻量，多 Agent 协调，handoff 机制 | 实验性轻量框架 |
| [ChatDev](https://github.com/open-compass/ChatDev) | 30k+ | Python | 虚拟软件公司，多 Agent 角色扮演 | 教育/演示 |
| [dspy](https://github.com/stanfordnlp/dspy) | 22k+ | Python | 声明式编程，auto-optimize prompts | Prompt 优化研究 |
| [smolagents](https://github.com/huggingface/smolagents) | 6k+ | Python | HuggingFace 轻量框架，CodeAgent 风格 | 轻量/嵌入式 |

#### 2.2 Coding Agent（垂直领域）

| 框架 | Stars | 核心特点 |
|------|-------|---------|
| [OpenHands](https://github.com/All-Hands-AI/OpenHands) | 49k+ | Browser/CodeEditing/Bash 工具集，模拟 Claude Code |
| [SWE-agent](https://github.com/SWE-agent/SWE-agent) | 12k+ | 专注 GitHub Issue 修复，RCF 工具 |
| [Devika](https://github.com/stitionai/devika) | 12k+ | 类似 Devin，开源，支持多语言 |
| [browser-use](https://github.com/browser-use/browser-use) | 28k+ | Playwright 页面操作，视觉识别 |
| [agno](https://github.com/agno-ai/agno) | 11k+ | 轻量，Agent 优先，多模态 |

#### 2.3 Memory / Knowledge Agent

| 框架 | Stars | 核心特点 |
|------|-------|---------|
| [mem0](https://github.com/mem0ai/mem0) | 12k+ | Agent 记忆层，多层记忆（短期/长期/用户偏好） |
| [Letta](https://github.com/letta-ai/letta) | 12k+ | 有状态 Agent，memory/context 平台，REST API |
| [khoj](https://github.com/khoj-ai/khoj) | 19k+ | 个人知识库，本地文档 + 语义搜索 + Agent |
| [GPT Researcher](https://github.com/assafelovic/gpt-researcher) | 27k+ | 研究助手，搜索+抓取+筛选+引用+报告生成 |

| U29 | Agent 部署 & Sandbox DevOps | `notes/concept-notes/U29-AgentDeploy.md` | — | — |

### 三、Agent 技术栈全景图

```
┌─────────────────────────────────────────────────────┐
│                    用户界面层                         │
│  CLI / IDE 插件 / 浏览器 / API / 机器人 / 语音        │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                 Agent 编排层                          │
│  LangGraph / CrewAI / AutoGen / MetaGPT / Swarm      │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  Agent Core 层                        │
│  Planning / Tool Use / Memory / Reflection / Eval   │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   LLM 接口层                          │
│  OpenAI / Anthropic / DeepSeek / Google / 开源模型   │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   工具层                              │
│  Search / Browser / Code / File / Database / API     │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   记忆层                              │
│  VectorDB / KG / KV / Summarization / mem0           │
└─────────────────────────────────────────────────────┘
```

---

### 四、必选主流框架详解

#### 4.1 LangChain / LangGraph（最全生态）

**推荐指数**：⭐⭐⭐⭐⭐
**语言**：Python + JS
**学习曲线**：中等
**生产使用率**：极高

LangChain = LLM 包装 + Tool 接口 + Memory + RAG + Indexing
LangGraph = 添加 Cycles + 状态机 + 人机交互 → 真正可编程 Agent

#### 4.2 CrewAI（Multi-Agent 最简单）

**推荐指数**：⭐⭐⭐⭐
**语言**：Python
**学习曲线**：低
**核心概念**：Agent = role + goal + backstory，Task = description + expected_output + agent

#### 4.3 AutoGen（微软 Multi-Agent）

**推荐指数**：⭐⭐⭐⭐
**语言**：Python
**核心概念**：AssistantAgent（代码执行 + LLM）+ UserProxyAgent（人类输入触发）+ GroupChat

#### 4.4 smolagents（HuggingFace 轻量）

**推荐指数**：⭐⭐⭐
**核心概念**：CodeAgent = 模型直接写 Python 代码执行（不用 Tool Definition）

#### 4.5 Dify（可视化编排平台）

**推荐指数**：⭐⭐⭐⭐
**类型**：开源 LLM App 平台，有 UI，不需要写代码

---

### 五、框架选型决策树

```
需要写代码实现 Agent？
└── 否 → 需要写代码实现 Agent？
    ├── 是 → 需要 Multi-Agent？
    │   ├── 是 → 角色明确分工清晰 → CrewAI
    │   ├── 是 → 需要复杂状态机/Cycles → LangGraph
    │   ├── 是 → 微软生态、代码执行 → AutoGen
    │   └── 否 → 需要 RAG + Tool + Memory 全家桶 → LangChain
    └── 否 → 需要可视化搭建 → Dify / Coze
```

---

### 六、各框架快速启动

```python
# ======== CrewAI（最快）=======
from crewai import Agent, Task, Crew

writer = Agent(role="写作助手", goal="写文章", backstory="资深编辑", tools=[])
reviewer = Agent(role="审核专家", goal="审核文章", backstory="严格审稿", tools=[])

crew = Crew(agents=[writer, reviewer], tasks=[], process="sequential")
result = crew.kickoff()

# ======== smolagents（最简）=======
from smolagents import CodeAgent, LocalAgent

agent = CodeAgent(model="gpt-4o-mini", tools=[])
result = agent.run("帮我写一个排序算法")

# ======== LangGraph（最全）=======
from langgraph.graph import StateGraph, END
from typing import TypedDict

class State(TypedDict):
    input: str
    output: str

graph = StateGraph(State)
graph.add_node("process", lambda state: {"output": f"处理: {state['input']}"})
graph.set_entry_point("process")
graph.add_edge("process", END)
app = graph.compile()
result = app.invoke({"input": "hello"})

# ======== AutoGen（微软）=======

assistant = AssistantAgent("assistant", llm_config={"model": "gpt-4o-mini"})
user = UserProxyAgent("user")
user.initiate_chat(assistant, message="帮我写一个快速排序")
```

---

## 单元学习资源总索引

> 按单元对应到现有笔记 / 工具速查 / 面试题的位置，避免重复造轮子。

| 单元 | 主题 | 概念笔记 | 工具速查 | 面试题 |
|------|------|----------|----------|--------|
| U1 | LLM 基础 | `notes/concept-notes/U1-LLM基础.md` | — | `notes/interview-qa/agent-core.md` |
| U2 | Embedding 与 VectorDB | `notes/concept-notes/U2-Embedding与VectorDB.md` | — | `notes/interview-qa/agent-core.md` |
| U3 | Prompt 工程 | `notes/concept-notes/U3-Prompt工程.md` | — | `notes/interview-qa/agent-core.md` |
| U4 | RAG 基础 | `notes/concept-notes/U4-RAG基础.md` | — | `notes/interview-qa/agent-core.md` |
| U5 | ReAct 范式 | `notes/concept-notes/U5-ReAct范式.md` | `tools/langchain-guide.md` | `notes/interview-qa/agent-core.md` |
| U6 | Tool Use 与 Function Calling | `notes/concept-notes/U6-ToolUse与FunctionCalling.md` | `tools/langchain-guide.md`, `tools/smolagents-guide.md` | `notes/interview-qa/agent-core.md` |
| U7 | Planning 与 Reasoning | `notes/concept-notes/U7-Planning与Reasoning.md` | `tools/langchain-guide.md` | `notes/interview-qa/agent-core.md` |
| U41 | Agent 全栈项目学习路线 | `notes/concept-notes/U41-AgentProjectPath.md` | — | — |
| U9 | Multi-Agent 系统 | `notes/concept-notes/U9-MultiAgent系统.md` | `tools/langchain-guide.md`, `tools/crewai-guide.md`, `tools/autogen-guide.md` | `notes/interview-qa/frameworks.md`, `notes/interview-qa/coding-agent.md` |
| U10 | Agent 评估与安全 | `notes/concept-notes/U10-Agent评估与安全.md` | — | `notes/interview-qa/agent-core.md`, `notes/interview-qa/coding-agent.md` |
| U11 | Memory 框架对比 | `notes/concept-notes/U11-Memory框架对比.md` | `tools/langchain-guide.md`, `tools/llamaindex-guide.md` | `notes/interview-qa/memory-frameworks.md` |
| U12 | Agent 评估标准 | `notes/concept-notes/U12-Agent评估标准.md` | — | `notes/interview-qa/agent-eval.md` |
| U13 | A2A 协议 | `notes/concept-notes/U13-A2A协议.md` | `tools/a2a-guide.md` | — |
| U14 | DSPy | `notes/concept-notes/U14-DSPy.md` | `tools/dspy-guide.md` | `notes/interview-qa/dspy-qa.md` |
| U15 | RAG 进阶 | `notes/concept-notes/U15-RAG进阶.md` | `tools/llamaindex-guide.md` | `notes/interview-qa/advanced-rag.md` |
| U16 | Embedding 进阶 | `notes/concept-notes/U16-Embedding进阶.md` | `tools/llamaindex-guide.md` | — |
| U17 | 多模态 Agent | `notes/concept-notes/U17-多模态Agent.md` | `tools/llamaindex-guide.md` | `notes/interview-qa/multi-modal-agent.md` |
| U18 | Agent 安全与红队 | `notes/concept-notes/U18-Agent安全与红队.md` | — | `notes/interview-qa/security-red-team.md`, `notes/interview-qa/Agent-安全面试题.md` |
| U19 | 向量数据库深度对比 | `notes/concept-notes/U19-向量数据库深度对比.md` | — | `notes/interview-qa/frameworks.md` |
| U20 | 编程 Agent 内部原理 | `notes/concept-notes/U20-编程Agent内部原理.md` | — | `notes/interview-qa/coding-agent-internals.md` |
| U21 | Loop Engineering | `notes/concept-notes/U21-LoopEngineering.md` | — | — |
| U22 | Context Engineering | `notes/concept-notes/U22-ContextEngineering.md` | — | — |
| U23 | Harness Engineering | `notes/concept-notes/U23-HarnessEngineering.md` | — | — |
| U24 | Reasoning Model | `notes/concept-notes/U24-ReasoningModel.md` | — | — |
| U25 | Computer Use / GUI Agent | `notes/concept-notes/U25-ComputerUse.md` | — | — |
| U26 | Agent Skills & AGENTS.md | `notes/concept-notes/U26-AgentSkills.md` | — | — |
| U27 | Goal Engineering | `notes/concept-notes/U27-GoalEngineering.md` | — | — |
| U28 | LLM Inference & Cost/Latency | `notes/concept-notes/U28-LLMInference.md` | — | — |
| U29 | Agent 部署 & Sandbox DevOps | `notes/concept-notes/U29-AgentDeploy.md` | — | — |
| U30 | Post-training 与 Alignment | `notes/concept-notes/U30-PostTraining.md` | — | — |
| U31 | Structured Output & Validation | `notes/concept-notes/U31-StructuredOutput.md` | — | — |
| U32 | Fine-tuning 实践 | `notes/concept-notes/U32-FineTuning.md` | — | — |
| U33 | Human-in-the-loop 设计 | `notes/concept-notes/U33-HumanInTheLoop.md` | — | — |
| U34 | Guardrails 与 Output 防御 | `notes/concept-notes/U34-Guardrails.md` | — | — |
| U35 | AgentOps / Eval 工具链 | `notes/concept-notes/U35-AgentOps.md` | — | — |
| U36 | MCP 深入 | `notes/concept-notes/U36-MCPDeep.md` | `tools/mcp-guide.md` | — |
| U37 | Subagents 与任务隔离 | `notes/concept-notes/U37-Subagents.md` | — | — |
| U38 | Context 管理深入 | `notes/concept-notes/U38-ContextManagement.md` | — | — |
| U39 | Permission & Auto Mode | `notes/concept-notes/U39-Permission.md` | — | — |
| U40 | Verification / Adversarial Review | `notes/concept-notes/U40-Verification.md` | — | — |
| U41 | Agent 全栈项目学习路线 | `notes/concept-notes/U41-AgentProjectPath.md` | — | — |
| U-DL | 深度学习扫盲 | `notes/concept-notes/UD-DL深度学习扫盲.md` | — | — |
