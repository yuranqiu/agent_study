# U22 · Context Engineering（上下文工程）

> **核心问题**：怎么把"对的 token"放进模型有限的注意力预算里，让 Agent 在多轮、长时、复杂任务中稳定表现？
> **预计学时**：5h
> **对应 Phase**：Phase 5 / Phase 3（项目实战阶段必学）
> **状态**：概念已标准化（Anthropic 2025-09 官方背书）

---

## 1. 什么是 Context Engineering

**Context Engineering** 是 Anthropic 团队 2025 年 9 月正式命名的概念：

> "Context refers to the set of tokens included when sampling from a large-language model (LLM). The **engineering** problem at hand is optimizing the utility of those tokens against the inherent constraints of LLMs in order to consistently achieve a desired outcome."
> —— [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

简单说：**Context = 任何会进 LLM context window 的 token**，包括但不限于：
- System prompt
- Tools（MCP server 定义、function schema）
- Few-shot examples
- 消息历史（user / assistant / tool messages）
- 检索结果（RAG、文件读、网页内容）
- 外部状态（scratchpad、NOTES.md、todo list）
- 记忆（短期 / 长期）

Context Engineering 就是**对这一坨 token 做精心策划和持续维护**，让它每一轮都是"最小高信号集"。

---

## 2. 与 Prompt Engineering 的关系

Anthropic 的官方表述：

> "We view context engineering as the natural progression of prompt engineering."

| 维度 | Prompt Engineering | Context Engineering |
|------|--------------------|---------------------|
| 目标 | 写好一段 prompt | 维护整个 context state |
| 范围 | System + user 消息 | System + tools + history + memory + RAG + ... |
| 时机 | 一次性 | 每个 turn 都做 curation |
| 关注点 | 措辞、示例、结构 | 哪些 token 该进、该出、该压缩 |
| 比喻 | 写一封好邮件 | 维护一整个收件箱 |

**Prompt Engineering 是 Context Engineering 的子集**——但两者在工程上关注点完全不同。Prompt 是一次性决策，Context 是**每个 turn 都要做的实时决策**。

---

## 3. 为什么 Context Engineering 是 Agent 必修课

### 3.1 Context Rot（上下文腐烂）

> "As the number of tokens in the context window increases, the model's ability to accurately recall information from that context decreases."
> —— [Chroma research: context rot](https://research.trychroma.com/context-rot)

**Context 不是越多越好**。每个新 token 都会消耗一部分"注意力预算"：
- LLM 是 transformer 架构，n 个 token → n² pairwise relationships
- 模型在短序列训练数据上更熟练
- 长 context 会出现"位置编码插值"的精度损失

### 3.2 注意力的有限性

人脑有 working memory 容量上限（7±2 items），LLM 也有"attention budget"：
- **不是 hard cliff**，是 performance gradient
- 长 context 仍能工作，但信息检索和长程推理的精度会下降
- 高 context 长度 ≠ 高 context 质量

> 关键洞察：**"什么该进 context"和"什么该出 context"是同等重要的问题**。

---

## 4. 5 大 Context 组件

Anthropic 官方列出的 5 大组件（每个都是优化对象）：

| 组件 | 优化目标 | 常见反模式 |
|------|----------|------------|
| **System prompts** | 给出"对的海拔"——既不过度具体、也不过度模糊 | 把所有 if-else 都硬编码进 prompt；或写得太泛 |
| **Tools** | 工具集小而清晰、token 效率高、行为明确 | 工具集臃肿、功能重叠、agent 不知道该选哪个 |
| **Examples** | 一组多样、有代表性的"教科书例子" | 把所有 edge case 都塞进 prompt |
| **Message history** | 早期消息压缩、工具结果清理 | 原样保留所有 tool 输出 |
| **External data** | 检索 vs 即时加载的平衡 | 全部预先塞进 context |

### 4.1 System prompts 的"对的海拔"

```
过高（brittle）              对的海拔                  过低（vague）
─────────────────     ─────────────────────     ─────────────────
"如果 X 且 Y 则 Z，      "你是 X 角色，            "你是一个智能助手"
否则如果 X 且 Z 则..."   按以下步骤处理..."         "请按用户要求做"
                                                  （假设模型知道任何事）
```

**方法**：从最小 prompt + 最好的模型开始，根据失败模式加明确指令和示例。

### 4.2 Tools 的设计原则

> "If a human engineer can't definitively say which tool should be used in a given situation, an AI agent can't be expected to do better."

- 工具应该 **self-contained / robust to error / extremely clear**
- 输入参数要 **descriptive, unambiguous, play to model's strengths**
- 工具应该**鼓励 agent 高效行为**（不是鼓励它绕路）

### 4.3 Examples 的克制

不要堆 edge case。要给一组 **diverse, canonical examples**——它们是"千言万语的图片"。

---

## 5. Context Retrieval：Just-in-Time 策略

### 5.1 两种策略

| 策略 | 做法 | 优劣 |
|------|------|------|
| **Pre-inference retrieval** | 启动前把所有相关数据 embedding 检索出来塞进 context | 快，但索引可能 stale、syntax tree 复杂 |
| **Just-in-time retrieval** | 给 agent **轻量级标识符**（file path / 存储的 query / URL），让 agent 在 runtime 用 tools 动态加载 | 慢，但更灵活、能处理动态内容 |

### 5.2 Hybrid 策略（Claude Code 的做法）

> "CLAUDE.md files are naively dropped into context up front, while primitives like glob and grep allow it to navigate its environment and retrieve files just-in-time."

- **预加载**：CLAUDE.md、AGENTS.md 这类"项目级元信息"
- **即取即用**：glob、grep、head、tail 让 agent 自主探索
- **决策边界**：根据任务动态决定（pre-loaded vs just-in-time）

### 5.3 Progressive Disclosure

让 agent 增量发现 context：
- file sizes → 提示复杂度
- naming conventions → 提示用途
- timestamps → 提示时效
- 每次 tool 调用 → 提示下一轮决策

> 关键洞察：context 不应该是"一次性塞好"，应该是**每一轮都重新 curate**。

---

## 6. Long-Horizon 三大技术

Anthropic 明确列出 3 个针对"token 数超过 context window 的任务"的核心技术：

### 6.1 Compaction（压缩）

> "Compaction is the practice of taking a conversation nearing the context window limit, summarizing its contents, and reinitiating a new context window with the summary."

**最小粒度的 compaction**：tool result clearing（已经在 Claude Developer Platform 上线为正式 feature）——历史工具结果可以清掉，因为 agent 不需要看到原始数据。

**Claude Code 的做法**：
- 把历史消息传给模型 → 总结 → 压缩
- 保留：架构决策、未解决的 bug、实现细节
- 丢弃：冗余的 tool 输出、消息
- 续接：压缩后的 context + 最近访问的 5 个文件

**Compaction 的艺术**：决定"保留 vs 丢弃"。太激进的压缩会丢失微妙但关键的 context。

### 6.2 Structured Note-Taking（结构化笔记）

> "The agent regularly writes notes persisted to memory outside of the context window."

实现形式：
- Claude Code 的 todo list
- 用户的 `NOTES.md` / `STATE.md` / `LOOP.md`
- Claude Plays Pokémon：精确记录"过去 1234 步 Pikachu 升了 8 级到目标 10"

Anthropic 2025-09 在 Claude Developer Platform 上发布了 **memory tool（公测）**——一个文件系统接口，让 agent 在 context 外存储和查询信息。

**模式：NOTES.md + 续接时读自己的笔记**

### 6.3 Sub-Agent Architectures

> "Specialized sub-agents can handle focused tasks with clean context windows."

- 主 agent 维护高层 plan
- 子 agent 用**各自干净的 context window**做深度技术工作
- 子 agent 返回 1000-2000 token 的精炼摘要
- 主 agent 只做综合分析

> "This pattern, discussed in [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system), showed a substantial improvement over single-agent systems on complex research tasks."

### 6.4 三大技术的选择

| 任务特征 | 推荐技术 |
|----------|----------|
| 大量来回对话、需要对话连贯性 | Compaction |
| 迭代开发、清晰里程碑 | Note-taking |
| 复杂研究、并行探索收益大 | Multi-agent |

---

## 7. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U3 Prompt Engineering | Context Engineering 是 Prompt Engineering 的超集 |
| U8 Memory & Reflection | 长期记忆 = Note-taking 的一种实现 |
| U9 Multi-Agent | Sub-agent 是 Context Engineering 的三大 long-horizon 技术之一 |
| U20 编程 Agent 内部原理 | Claude Code 的 compaction / CLAUDE.md / sub-agent 模式是 Context Engineering 的经典案例 |
| U21 Loop Engineering | 跨 session 持续工作的 context 持久化 = Loop Engineering 的核心 |
| U23 Harness Engineering | Harness 是 **Context Engineering 的 runtime substrate**——把 context 管理变成可工程化的 runtime |

### 层次关系

```
┌─────────────────────────────────────────────┐
│ Loop Engineering（跨 session 持续运转）      │  U21
├─────────────────────────────────────────────┤
│ Harness Engineering（runtime substrate）     │  U23
├─────────────────────────────────────────────┤
│ Context Engineering（塞什么进 context）      │  U22 ← 本单元
├─────────────────────────────────────────────┤
│ Prompt Engineering（怎么写 prompt）          │  U3
└─────────────────────────────────────────────┘
```

> **关键区分**：
> - **Context Engineering**：要管理**什么信息**进 context
> - **Harness Engineering**：管理 context 的**runtime 机制**（tool、permission、memory、verification）
> - **Loop Engineering**：harness 之上**跨 session 的持续运转**框架

---

## 8. 面试视角

- **Q：Context Engineering 和 Prompt Engineering 的区别？**
  → Prompt 是一次性写好一段 instruction；Context 是每个 turn 都要做的实时策划（包括 tools / history / memory / RAG / examples / external data）。Context 包含 Prompt。

- **Q：Context 越多越好吗？**
  → 不是。存在 context rot：随着 context token 数增加，模型的信息检索和长程推理精度下降。Context 是有限资源，attention budget 是 fixed pool。

- **Q：Long-horizon 任务怎么处理？**
  → 三大技术：Compaction（历史压缩）、Structured Note-taking（外部 NOTES.md / memory tool）、Sub-agent Architectures（每个 sub-agent 干净 context window）。选哪个看任务特征。

- **Q：Just-in-time retrieval vs pre-loaded context 怎么选？**
  → 动态内容多的（代码、文档）用 just-in-time，让 agent 用 glob/grep 自主探索；静态元信息（CLAUDE.md、AGENTS.md）预加载。Claude Code 用 hybrid 策略。

- **Q：Context window 越来越大，是不是就不用 compaction 了？**
  → 不是。即使 1M context 也存在 context rot 和 attention budget 问题。Compaction 是工程策略，不只是 token 限制的 workaround。

---

## 9. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | **官方权威定义**，必读。Anthropic Applied AI 团队 2025-09 发布 |
| [Anthropic: Writing tools for AI agents – with AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents) | Tools 设计原则，是 Context Engineering 的一部分 |
| [Anthropic: How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) | Sub-agent 架构的实战案例 |
| [Anthropic Memory & Context Management cookbook](https://platform.claude.com/cookbook/tool-use-memory-cookbook) | 实战代码 |
| [Chroma: Context Rot Research](https://research.trychroma.com/context-rot) | "context 不是越多越好"的实验证据 |
| [Karpathy on context engineering](https://x.com/karpathy/status/1937902205765607626) | "Context engineering is the art and science of curating what will go into the limited context window" |

---

## 10. 练习

1. **观察自己的 Agent**：跑一个 50 轮对话的 agent，看 context 在第几轮开始"丢东西"（比如忘了之前用户的偏好）
2. **实现 compaction**：写一个 `compact_history(messages, max_tokens)` 函数，能总结对话历史并保留关键决策
3. **设计 tool set**：为自己的 agent 设计 5-7 个工具，确保每个工具的"何时用"agent 都能清楚判断
4. **NOTES.md 模式**：用一个长任务，让 agent 在 NOTES.md 写中间状态，验证重启后能否续上
5. **Just-in-time retrieval**：让 agent 拿到一个 git repo URL，先只传 README，再用 glob/grep 自主探索完成一个任务，对比"全部预加载"的 token 消耗

---

## 11. 与 Loop / Harness 的对照表

| 维度 | Prompt | Context | Harness | Loop |
|------|--------|---------|---------|------|
| 提出方 | 行业通用 | Anthropic (2025-09) | arXiv 2605.13357 | cobusgreyling (2026) |
| 核心问题 | 怎么写 | 塞什么进 window | runtime 怎么调度 | 怎么跨 session 持续 |
| 范围 | system msg | 整个 context state | 11 个 runtime 组件 | 5 个 primitives + state |
| 时机 | 一次性 | 每 turn 实时 | 每 turn 实时 | 跨 turn 跨 session |
| 度量 | 输出质量 | 注意力预算利用 | trace-based eval | Loop Ready score |
| 必读 | Anthropic prompt guide | Anthropic 上下文文章 | arXiv 2605.13357 | cobusgreyling blog |

---

*最后更新：2026-06-30 新增 U22（Context Engineering）*
*来源：Anthropic 官方博客 2025-09、arXiv 2605.13357、Chroma research、Claude Code 实践*
