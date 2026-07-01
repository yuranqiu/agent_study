# U42 · 面试真题与学习顺序（校招向 Agent 工程岗指南）

> **核心问题**：校招 Agent 工程岗怎么高效准备？41 个单元的学习顺序怎么排？真实面试会问什么、怎么答？
> **预计学时**：贯穿整个学习周期
> **对应 Phase**：Phase 2-5（所有阶段都需要）
> **状态**：跨单元整合（综合所有 41 单元 + interview-qa 资料）

---

## 1. Agent 工程岗校招画像

### 1.1 主流公司 Agent 团队（2025-2026）

| 类别 | 公司 | 核心产品 |
|------|------|----------|
| **模型厂商** | OpenAI / Anthropic / Google / DeepSeek | Operator / Claude Code / Gemini |
| **C 端 Agent** | Manus / Devin (Cognition) / Sierra | 通用 assistant |
| **B 端 Agent** | Decagon / Intercom (Fin) / Glean | 客服 / 知识库 |
| **国内大厂** | 字节 (扣子/Coze) / 阿里 (通义) / 百度 (文心) | Agent 平台 |
| **国内创业** | 智谱 / 月之暗面 / MiniMax / 深度求索 | 模型 + Agent |
| **垂直 Agent** | Harvey (法律) / Cursor (代码) | 行业 Agent |

### 1.2 典型面试流程

```
1. 简历筛选（HR 看关键词：Agent / LLM / RAG / ReAct）
2. 技术初面（60-90 min）：1-2 个 LeetCode + 1 个 LLM 概念题 + 1 个项目深挖
3. 技术终面（90-120 min）：项目深挖 + 系统设计 + 行为面试
4. 主管面 / HR 面
```

**Agent 工程岗特点**：
- LeetCode：Medium 难度（不需要 Hard）
- 概念题：**LLM/Agent 知识**比算法多
- 系统设计：**RAG / Agent 架构设计**（最常见）
- 项目：**必须有 1-2 个端到端 Agent 项目**（U41 P1-P3 必做）

---

## 2. 41 单元学习顺序（Dependency Graph）

**核心原则**：**基础先于高级，依赖先于应用**。

### 2.1 学习阶段划分

```
阶段 1（必学，3-4 周）：C1 + C2 + C3 + C4 基础
├── U1 LLM 基础
├── U3 Prompt 工程
├── U-DL 深度学习扫盲
├── U2 Embedding 与 VectorDB
├── U4 RAG 基础
├── U5 ReAct 范式
├── U6 Tool Use
└── U7 Planning

阶段 2（核心，4-6 周）：C5-C9 进阶
├── U8 Memory
├── U9 Multi-Agent
├── U11 Memory 框架对比
├── U15 RAG 进阶
├── U16 Embedding 进阶
├── U18 Agent 安全
├── U19 向量库对比
└── U22 Context Engineering

阶段 3（差异化，4-6 周）：选 2-3 个 cluster
├── A. 前沿话题：U24 Reasoning / U25 Computer Use / U26 Skills / U27 Goal
├── B. 生产工程：U28 Inference / U29 Deploy / U30 Post-training / U32 Fine-tuning
└── C. 评估与安全：U10/U12/U18/U34/U35

阶段 4（深度，4-6 周）：跨单元 + 项目
├── U20 编程 Agent 内部
├── U21 Loop Engineering
├── U23 Harness Engineering
├── U31 Structured Output
├── U33 Human-in-the-loop
├── U36 MCP / U37 Subagents / U38 Context / U39 Permission / U40 Verification
└── U41 全栈项目（P1-P3）
```

### 2.2 学习顺序图（Dependency）

```
U1 / U-DL ──┬── U3 ──┬── U5 ──┬── U8 ──┬── U22 ──┬── U31
            │        │        │        │        │
            │        ├── U6 ──┤        │        ├── U33
            │        │        │        │        │
            │        ├── U7 ──┤        │        │
            │        │        │        │        ├── U40
            │        │        │        │        │
U2 ────────┴── U4 ──┴── U9 ──┴── U11 ─┴── U36 ─┴── U37
│           │        │        │        │
│           ├── U15 ─┤        │        ├── U38
│           │        │        │        │
│           ├── U16 ─┤        │        ├── U39
│           │        │        │        │
│           ├── U19 ─┘        │        │
│                            │        │
U18 ──────────────────────────┘        │
U10/U12/U34/U35 ───────────────────────┘
U28/U29/U30/U32 ───────────────────────
U17/U20/U25/U26/U27 ──────────────────
U21/U23/U41 ───────────────────────────
```

### 2.3 时间预算（校招 3 个月）

| 阶段 | 时间 | 重点 |
|------|------|------|
| M1 | 4 周 | 阶段 1 全部 + 阶段 2 前半 |
| M2 | 4 周 | 阶段 2 后半 + 阶段 3 选 1 个方向 |
| M3 | 4 周 | U41 项目 1-2 个 + 面试刷题 |

---

## 3. 真实面试真题（按主题分类）

### 3.1 LLM 基础（5-10 题）

**Q1：为什么 LLM 用 Transformer 而不是 RNN？**
- **答**：并行计算（O(1) 而非 O(n) seq dep）+ 长距离依赖（self-attention 直接连接任意位置）+ 训练稳定性（无梯度消失/爆炸）
- **加分**：对比 attention O(n²) 复杂度 vs RNN O(n)

**Q2：什么是 tokenization？BPE 怎么工作？**
- **答**：把文本切成 token（subword）。BPE = Byte Pair Encoding：迭代合并最高频的字符对，最终得到 subword 词表（如 GPT 的 cl100k_base 100k tokens）
- **加分**：对比 BPE / WordPiece / SentencePiece / tiktoken 的差异

**Q3：解释 Self-Attention 的 Q/K/V**
- **答**：Q = query（"我在找什么"）/ K = key（"我代表什么"）/ V = value（"如果匹配到，我的实际内容"）。`Attention(Q,K,V) = softmax(QK^T / √d) V`
- **加分**：解释 multi-head（多个 Q/K/V 子空间拼接）和 KV cache（推理优化）

**Q4：RLHF 三步流程？有什么问题？**
- **答**：1) SFT（监督微调） 2) RM 训练（奖励模型） 3) PPO（用 RM 做 RL）
- **问题**：训练不稳定 / reward hacking / 成本高
- **加分**：DPO / GRPO / RLVR 如何解决

### 3.2 Prompt 工程（3-5 题）

**Q5：Zero-shot / Few-shot / CoT 的适用场景？**
- **答**：Zero-shot：模型强 + 任务简单。Few-shot：需要特定格式。CoT：复杂推理（数学 / 逻辑）。**CoT 的代价**：token 多 5-10x，延迟也大
- **加分**：Self-consistency（采样 N 次投票）vs CoT 的对比

**Q6：如何防御 Prompt Injection？**
- **答**：三层：1) 强 system prompt（明确边界）2) 输入 sanitization（检测 injection 模式）3) 多层 guardrails（U34）。**永不信任用户输入**，把它当 untrusted data
- **加分**：双 LLM 模式（小 LLM 预审 + 大 LLM 主答）

### 3.3 RAG（10-15 题，高频）

**Q7：RAG vs Fine-tuning vs Long Context 怎么选？**
- **答**：
  - **RAG**：知识更新频繁 / 知识库大 / 需要引用来源
  - **Fine-tuning**：固定模式 / 改变输出格式 / 专业化
  - **Long Context**：context < 100k / 偶尔 / 预算足
- **经验数据**：100k+ context 质量下降 + cost 爆炸

**Q8：Chunking 怎么选 size？**
- **答**：256 起 / 中文 +128 / overlap 64-128。**过大**：召回噪声多。**过小**：语义不完整
- **实战**：必跑 chunk size 评估（128/256/512/1024 对比 faithfulness）

**Q9：Hybrid 检索为什么有效？**
- **答**：向量检索（语义）+ BM25（字面）互补。**向量找"意思对"，BM25 找"词对"**。生产标准做法
- **加分**：Reciprocal Rank Fusion（RRF）合并两路结果

**Q10：Reranker 为什么必要？**
- **答**：Bi-encoder（向量检索）速度快但精度有限。Cross-encoder（reranker）慢但精度高。**生产做法**：retriever 召回 Top-100 → reranker 重排 Top-10 → LLM 用 Top-10
- **加分**：bge-reranker / Cohere Rerank 实战

**Q11：RAGAS 4 大指标？**
- **答**：Faithfulness（忠于 context） / Answer Relevancy（切题） / Context Precision（top-k 质量） / Context Recall（覆盖率）
- **加分**：Faithfulness 是最关键的（防幻觉）

### 3.4 Agent（15-20 题，最高频）

**Q12：ReAct 四步循环？**
- **答**：Thought → Action → Observation → Next Thought
- **加分**：vs Chain-of-Thought（纯推理无行动）vs Plan-and-Execute（先规划后执行）

**Q13：Tool Use 怎么实现？**
- **答**：OpenAI Function Calling / Anthropic Tool Use：1) 工具定义（JSON Schema）2) LLM 决定调哪个 + 参数 3) 执行 + 返回 observation 4) LLM 继续
- **加分**：Tool Choice 模式（auto / required / none）

**Q14：Agent Memory 4 层架构？**
- **答**（详见 U8）：L1 Working（in-context） / L2 Short-term（session） / L3 Long-term（user profile） / L4 Procedural（skills）
- **加分**：mem0 / Letta / Zep 选型

**Q15：Multi-Agent 通信协议有哪些？**
- **答**：MCP（Agent ↔ 工具，纵向）/ A2A（Agent ↔ Agent，横向）/ ANP（更大范围网络）
- **加分**：MCP Host/Client/Server 三层架构

**Q16：HITL 4 大模式？**
- **答**（详见 U33）：Approval / Interrupt / Oversight / Confidence-based
- **加分**：LangGraph interrupt() + Command(resume) 的 6 大铁律

**Q17：Context window 满了怎么办？**
- **答**（详见 U38）：/clear 切换任务 / /compact 压缩 / rewind 回滚 / Context Budget
- **加分**：Subagent 隔离 context 是 U37 的核心

**Q18：Subagent 模式为什么有效？**
- **答**（详见 U37）：Fresh context 避免 self-bias。Writer/Reviewer loop 是 SOTA 模式
- **加分**：Claude Code 的 `.claude/agents/*.md` 实现

### 3.5 推理模型与后训练（5-10 题，新热点）

**Q19：Reasoning Model（o1/R1）和标准 LLM 区别？**
- **答**：Reasoning model 内部生成长 chain of thought，test-time compute 越多效果越好。**范式变化**：ReAct 循环减少（一次 deep think 替代多步推理）
- **加分**：Test-time compute scaling law

**Q20：DPO vs RLHF 优缺点？**
- **答**：DPO 不需要训练 RM 和 PPO，训练更稳定。但 DPO 只能用 offline preference，不能 online
- **加分**：GRPO 用组内相对排名做 advantage，不需要 value model

**Q21：Fine-tuning 什么时候值得做？**
- **答**：标准决策：数据量 > 1k + 需要特定行为/格式 + 长期高频调用。**否则用 prompt + RAG**
- **加分**：QLoRA（4-bit 量化）让 7B 模型在单 GPU 上微调

### 3.6 生产工程（5-10 题）

**Q22：Context Engineering vs Prompt Engineering？**
- **答**：Prompt 是 system prompt + user input。Context 是 system + user + history + RAG + memory + tool results。**Context 是更大的概念**
- **加分**：Compaction / Note-taking / Sub-agent（U22 / U37）

**Q23：Harness 11 组件？**
- **答**（详见 U23）：Task Spec / Context Selection / Tool Access / Project Memory / Task State / Observability / Failure Attribution / Verification / Permissions / Entropy Auditing / Intervention Recording

**Q24：Verification 4 大模式？**
- **答**（详见 U40）：1) 同 prompt 跑 check 2) /goal 持续 3) Stop hook 强制 4) Adversarial review
- **加分**：Writer/Reviewer loop 避免 self-bias

**Q25：MCP vs Function Calling？**
- **答**：Function Calling 是模型厂商定义。MCP 是开放协议（Anthropic 2024-11），一份 Server 跨多个客户端用。**USB-C for LLM**
- **加分**：FastMCP 装饰器 + langchain-mcp-adapters

### 3.7 行为面试（10 题）

**Q26：为什么做 Agent / 为什么来我们公司？**
- **框架**：兴趣起源（"我用过 ChatGPT 解决 XXX"）+ 行业判断（"Agent 是 LLM 下半场"）+ 公司匹配（"贵公司 XX 产品吸引我"）+ 个人规划
- **避免**：泛泛说"AI 是未来"（所有人都会说）

**Q27：项目深挖的 STAR 框架？**
- **S**ituation：背景 + 约束
- **T**ask：你的具体目标
- **A**ction：做了什么 + 怎么选 + trade-off
- **R**esult：量化指标 + 影响

**Q28：最大技术失败 + 学到什么？**
- **诚实**：RAG 跑出来 faithfulness 只有 0.6 / Agent 跑 10 步就乱了
- **学到了什么**：chunking + reranker / 加 verifier subagent
- **避免**：编造失败或怪别人

**Q29：你怎么看 Agent 的未来？**
- **3 个角度**：
  1. **技术**：Reasoning + Tool Use + Multi-Agent 融合
  2. **产品**：从 Chatbot 到 Action Agent（Operator / Devin）
  3. **生态**：MCP / A2A 标准化 + Skill / Subagent 复用

---

## 4. 简历策略

### 4.1 简历 3 大模块

**模块 1：教育背景**
- 学校 / 专业 / GPA（3.5+ 必写）
- 核心课程：机器学习 / NLP / 数据结构

**模块 2：项目经验**（2-3 个 Agent 项目）
- **P1 ReAct 原型**（U41）：ReAct + 3 个 tool + trace
- **P2 RAG 知识库**（U41）：PDF / RAGAS 评估 0.85
- **P3 Multi-Agent / P5 Reasoning**（U41）：差异化亮点

**模块 3：技能 / 竞赛 / 论文**
- 技能：Python / PyTorch / LangChain / RAG / ReAct / MCP
- 竞赛：Kaggle / ACM / 数学建模（加分项）
- 论文：NeurIPS / ACL / EMNLP（强加分）

### 4.2 STAR 模板（项目描述）

```
项目：基于 RAG 的企业知识助手
时间：2025.06 - 2025.08
S：团队内部知识分散在 200+ PDF 文档中，搜索效率低
T：构建一个 RAG 助手，能基于文档回答问题
A：
- 选 LangChain + Chroma + GPT-4o-mini（成本 vs 质量权衡）
- Chunking 256 + overlap 64，RecursiveCharacterTextSplitter
- Hybrid 检索：向量 + BM25 + Reranker（bge-reranker-base）
- 评估：RAGAS faithfulness 0.87，answer relevancy 0.83
R：内部使用率 80% / 平均查询时间 < 2s / 月省 40 小时
```

**关键**：**A 必须有 trade-off**，**R 必须有量化**。

---

## 5. 阶段自测 Checklist

### 5.1 阶段 1 完成检验（基础）

- [ ] 能解释 Transformer attention 机制
- [ ] 能列出 5 种 LLM 采样参数（temperature / top_p / top_k / max_tokens / stop）
- [ ] 能手写 ReAct 循环
- [ ] 能用 Pydantic 验证 LLM 输出
- [ ] 能用 LangChain 跑通基础 RAG

### 5.2 阶段 2 完成检验（核心）

- [ ] 能解释 Hybrid 检索为什么比单一检索好
- [ ] 能列出 5 大 RAG 优化技术
- [ ] 能用 LangGraph / CrewAI 实现 Multi-Agent
- [ ] 能解释 4 种 Memory 模式
- [ ] 能识别 5 大 Agent 安全威胁

### 5.3 阶段 3 完成检验（差异化）

- [ ] 能解释 Reasoning Model 与标准 LLM 的差异
- [ ] 能跑 SFT / DPO / GRPO 训练
- [ ] 能用 Langfuse / Helicone 做监控
- [ ] 能写 Stop hook 强制 verification
- [ ] 能用 NeMo Guardrails / Guardrails AI

### 5.4 阶段 4 完成检验（深度）

- [ ] 能设计 Harness 11 组件的 Agent 系统
- [ ] 能实现 LangGraph interrupt + Command(resume) 模式
- [ ] 能写 OpenAI needs_approval + RunState 持久化
- [ ] 能用 Claude Code Hooks 实现 HITL
- [ ] 能跑 Writer/Reviewer 循环
- [ ] 能实现跨 session 的 long-running agent
- [ ] **有 1-2 个端到端 Agent 项目可 demo**

---

## 6. 必读论文 Top 30（按主题）

> 完整论文列表分散在 42 个单元里。这里集中整理"面试/生产最常被引用"的 30 篇。

### 6.1 LLM 基础（5 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **Attention is All You Need** | [arxiv.org/abs/1706.03762](https://arxiv.org/abs/1706.03762) | Transformer 原始论文 |
| **InstructGPT (RLHF)** | [arxiv.org/abs/2203.02155](https://arxiv.org/abs/2203.02155) | RLHF 奠基 |
| **LLaMA 1/2/3** | [arxiv.org/abs/2302.13971](https://arxiv.org/abs/2302.13971) | 开源 LLM 范式 |
| **Scaling Laws for Neural LMs** | [arxiv.org/abs/2001.08361](https://arxiv.org/abs/2001.08361) | Scaling Law |
| **GPT-4 Technical Report** | [openai.com/research/gpt-4](https://openai.com/research/gpt-4) | 多模态 + RLHF + 评估 |

### 6.2 Agent 范式（8 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **ReAct** | [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629) | Reason + Act 循环 |
| **Toolformer** | [arxiv.org/abs/2302.04761](https://arxiv.org/abs/2302.04761) | LLM 学会调工具 |
| **Reflexion** | [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366) | 自我反思 |
| **Generative Agents** | [arxiv.org/abs/2304.03442](https://arxiv.org/abs/2304.03442) | Memory + Reflection + Planning |
| **Voyager** | [arxiv.org/abs/2305.16291](https://arxiv.org/abs/2305.16291) | 长期学习 agent |
| **Tree of Thoughts** | [arxiv.org/abs/2305.10601](https://arxiv.org/abs/2305.10601) | 树搜索推理 |
| **AutoGPT** | [github.com/Significant-Gravitas/AutoGPT](https://github.com/Significant-Gravitas/AutoGPT) | 自主 agent 实践 |
| **DSPy** | [arxiv.org/abs/2310.03714](https://arxiv.org/abs/2310.03714) | 声明式 prompt 优化 |

### 6.3 RAG（4 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **RAG Original (Lewis 2020)** | [arxiv.org/abs/2005.11401](https://arxiv.org/abs/2005.11401) | RAG 原始论文 |
| **Self-RAG** | [arxiv.org/abs/2310.11511](https://arxiv.org/abs/2310.11511) | Self-reflective RAG |
| **GraphRAG (Microsoft)** | [arxiv.org/abs/2404.16130](https://arxiv.org/abs/2404.16130) | 知识图谱 RAG |
| **Corrective RAG (CRAG)** | [arxiv.org/abs/2401.15884](https://arxiv.org/abs/2401.15884) | 自校正 RAG |

### 6.4 Reasoning（3 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **Chain-of-Thought** | [arxiv.org/abs/2201.11903](https://arxiv.org/abs/2201.11903) | CoT 原始 |
| **DeepSeek R1** | [arxiv.org/abs/2501.12948](https://arxiv.org/abs/2501.12948) | GRPO + RLVR |
| **Self-Consistency** | [arxiv.org/abs/2203.11171](https://arxiv.org/abs/2203.11171) | 采样 N 次投票 |

### 6.5 Memory & Reflection（3 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **MemGPT** | [arxiv.org/abs/2310.06860](https://arxiv.org/abs/2310.06860) | 分层 memory + 工具 |
| **Voyager** | [arxiv.org/abs/2305.16291](https://arxiv.org/abs/2305.16291) | 长期 skill library |
| **Reflexion** | [arxiv.org/abs/2303.11366](https://arxiv.org/abs/2303.11366) | 语言反思 |

### 6.6 Context & Harness（2 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **Anthropic: Effective context engineering** | [anthropic.com/.../effective-context-engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Context 实战 |
| **AI Harness Engineering** | [arxiv.org/abs/2605.13357](https://arxiv.org/abs/2605.13357) | 11 组件 + H0-H3 阶梯 |

### 6.7 Multi-Agent & Protocol（2 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **MCP 规范** | [modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-03-26/basic) | MCP 协议细节 |
| **AutoGen** | [arxiv.org/abs/2308.08155](https://arxiv.org/abs/2308.08155) | Multi-Agent 范式 |

### 6.8 Coding / Computer Use（2 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **SWE-bench** | [arxiv.org/abs/2310.06770](https://arxiv.org/abs/2310.06770) | Coding Agent 评测 |
| **Dive into Claude Code** | [arxiv.org/abs/2604.14228](https://arxiv.org/abs/2604.14228) | Claude Code 系统设计 |

### 6.9 Safety（1 篇）

| 论文 | 链接 | 为什么读 |
|------|------|----------|
| **Sleeper Agents** | [arxiv.org/abs/2401.05566](https://arxiv.org/abs/2401.05566) | 后门攻击 |

### 6.10 Regulation（必读）

| 文档 | 链接 | 为什么读 |
|------|------|----------|
| **EU AI Act Article 14** | [artificialintelligenceact.eu/article/14](https://artificialintelligenceact.eu/article/14/) | 欧盟法律要求 human oversight |
| **OWASP LLM Top 10** | [owasp.org/.../top-10-for-large-language-model-applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | 行业安全标准 |
| **NIST AI RMF** | [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) | 美国国家标准 |

---

## 7. 行为面试高频题（10 题）

### 7.1 为什么做 Agent？
- **3 层结构**：1) 我用过 Agent 解决了 X 2) Agent 是 LLM 下半场（从 Chatbot 到 Action）3) 我的能力匹配（ReAct / RAG / Multi-Agent）

### 7.2 最大失败 + 学到什么？
- **诚实 + STAR**：失败 + 原因 + 学到的工程化教训
- **避免**：编造失败 / 怪别人

### 7.3 5 年后你想做什么？
- **Agent 工程师 → Agent 架构师 → Agent 产品负责人**（或者创业）
- **不**：要转 PM / 不想写代码 / 想躺平

### 7.4 你有什么问题问我？
- **3 类好问题**：
  1. **技术深度**：贵团队 Agent 系统用什么架构？最大的技术挑战是什么？
  2. **业务**：贵产品的核心 user scenario 是什么？
  3. **团队**：Agent 团队几个人？怎么分工？code review 怎么做？
- **避免**：薪资 / 福利 / 几号入职

### 7.5-7.10 略（其他高频题）
- 你的优缺点
- 最有成就感的项目
- 为什么来我们公司
- 期望薪资
- 何时能入职
- 你的长期职业规划

---

## 8. 模拟面试节奏

### 8.1 校招时间线（2026 届参考）

```
2026.06-08  暑期实习 / 提前批（投递 + 准备）
2026.09-10  秋招正式批（投递 + 笔试 + 多轮面试）
2026.11-12  Offer 收割期
2027.01-03  春招补录
```

### 8.2 准备节奏（3 个月）

| 周 | 内容 | 关键产出 |
|----|------|----------|
| W1-2 | 阶段 1 全部 | 跑通 ReAct 原型 + 基础 RAG |
| W3-4 | 阶段 2 前半 | Memory + Multi-Agent 项目 |
| W5-6 | 阶段 2 后半 + 阶段 3 选 1 | 1 个差异化亮点 |
| W7-8 | 阶段 4 + U41 项目 | 2-3 个端到端项目 |
| W9-10 | 模拟面试 + 刷题 | 至少 5 场 mock interview |
| W11-12 | 投递 + 真面试 | 拿到 offer |

### 8.3 Mock Interview 推荐

- **找朋友 mock**：每 2 周一次，互相问项目
- **GitHub Copilot Chat**：用 AI 模拟面试官
- **牛客 / LeetCode 模拟面试平台**：找 Agent 岗的模拟题
- **录音 + 回放**：每次 mock 录音，听自己的表达问题

---

## 9. 校招 Agent 工程岗 简历模板

```markdown
# 你的名字
邮箱 | GitHub | LinkedIn

## 教育
- 学校 / 专业 / 学位 (20XX - 20XX)
- GPA: 3.X/4.0 (Top 10%)
- 核心课程：机器学习 / NLP / 数据结构

## 项目经验

### 1. 企业 RAG 知识助手 (P2)
**时间**：2026.05 - 2026.07
**技术栈**：LangChain + Chroma + GPT-4o-mini + bge-reranker

- 设计 Hybrid 检索（向量 + BM25）架构，平均查询延迟 < 2s
- 用 RAGAS 评估 faithfulness 0.87，answer relevancy 0.83
- 部署 200+ 员工使用，月省 40 小时

### 2. Multi-Agent 自动化研究系统 (P3)
**时间**：2026.07 - 2026.08
**技术栈**：LangGraph + Claude 3.5 + MCP

- 实现 Planner / Researcher / Writer / Reviewer 4 个 agent 协作
- 用 LangGraph interrupt + Command(resume) 实现 HITL
- 自动完成深度研究报告，平均 30 分钟 / 报告，质量超过单 agent baseline 35%

### 3. ReAct 智能体原型 (P1)
**时间**：2026.04 - 2026.05
- 手写 150 行 ReAct 循环，支持 3 个 tool（搜索 / 计算 / 文件）
- 加 trace log + streaming 输出 + unit test

## 技能
- **语言**：Python / TypeScript / SQL
- **ML**：PyTorch / Transformers / LangChain
- **Agent**：ReAct / RAG / Multi-Agent / HITL / MCP
- **工具**：LangGraph / Langfuse / vLLM / Claude Code

## 竞赛 / 论文
- [ACM 区域赛 / Kaggle 银牌 / NeurIPS workshop paper 等]
```

---

## 10. 与本仓库其他单元的关系

U42 是**跨单元整合**单元，覆盖所有 41 个单元 + interview-qa 资料库。

详见每个章节对应的单元：
- §2 学习顺序 → 所有 41 单元
- §3 真实面试题 → U1-U40 各单元
- §6 必读论文 Top 30 → 各单元的"必读材料"部分
- §7 行为面试 → U26 / U33 / U41

---

*最后更新：2026-06-30 新增 U42（面试真题与学习顺序）*
*来源：[datawhale/extra-chapter 面试题总结](https://github.com/datawhalechina/hello-agents/blob/main/Extra-Chapter/Extra01-面试问题总结.md) + 41 单元核心内容综合 + [Anthropic Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) + [OpenAI Agents SDK HITL](https://openai.github.io/openai-agents-python/human_in_the_loop/) + [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices) + 30+ arxiv 论文*
