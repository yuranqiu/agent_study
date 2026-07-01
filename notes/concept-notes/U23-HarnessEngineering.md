# U23 · Harness Engineering（Harness 工程）

> **核心问题**：怎么把模型 + runtime 编排成一个**可审计、可验证、可分级暴露运行时支持**的 Agent 系统？
> **预计学时**：6h
> **对应 Phase**：Phase 5 / Phase 3（项目实战阶段必学）
> **状态**：学术概念 + 工业实践（arXiv 2605.13357 + Claude Code / Codex / LoopX 实战）

---

## 1. 什么是 Harness Engineering

**Harness Engineering** 由 arXiv 论文 [AI Harness Engineering: A Runtime Substrate for Foundation-Model Software Agents](https://arxiv.org/abs/2605.13357) 正式定义：

> "Software-engineering capability emerges from a model-harness-environment system, in which a runtime substrate — the harness — mediates how a foundation-model agent observes a project, acts on it, receives feedback, and establishes that a change is complete."

> "We formalize this substrate as an **AI Harness Engineering** and identify **eleven component responsibilities**."

简单说：**Harness = 模型周围的 runtime substrate**——所有"让模型能做事"的工程化机制。模型本身不会自动有这些能力，需要被 harness 包裹。

### 1.1 与 Context Engineering 的关系

- **Context Engineering**（Anthropic）关心**"塞什么进 context window"**——是 information-level
- **Harness Engineering** 关心**"怎么用 runtime 机制管理 context"**——是 mechanism-level

Harness 是 Context Engineering 的**工程化实现层**。没有 harness，context engineering 只是原则；有了 harness，context engineering 变成可重复、可审计的运行时。

```
            ┌─────────────────────────────────┐
            │  Prompt Engineering (U3)        │  写什么
            ├─────────────────────────────────┤
            │  Context Engineering (U22)      │  塞什么进 context
            ├─────────────────────────────────┤
   关注点 → │  Harness Engineering (U23) ★    │  怎么用 runtime 管 context
            ├─────────────────────────────────┤
            │  Loop Engineering (U21)         │  跨 session 持续运转
            └─────────────────────────────────┘
```

---

## 2. 11 个组件职责（The 11 Component Responsibilities）

论文明确列出 harness 的 11 个职责：

| # | Component | 职责 | 典型实现 |
|---|-----------|------|----------|
| 1 | **Task specification** | 怎么向 agent 表达任务 | prompt template、issue template、work item schema |
| 2 | **Context selection** | 该把哪些 token 放进 context window | RAG 检索、token 预算管理、compaction |
| 3 | **Tool access** | 怎么让 agent 触达工具 | function calling、JSON schema、MCP |
| 4 | **Project memory** | 项目级持久化记忆 | `STATE.md`、CLAUDE.md、NOTES.md、向量库 |
| 5 | **Task state** | 当前任务的状态机 | todo list、claim/lease、in-progress 标记 |
| 6 | **Observability** | 怎么看到 agent 在做什么 | trace、log、metrics、UI dashboard |
| 7 | **Failure attribution** | 失败时定位是模型问题还是环境问题 | error code、retry、context diff |
| 8 | **Verification** | 怎么确认 agent 真的做对了 | tests、playwright、assertion、eval |
| 9 | **Permissions** | agent 能做什么、不能做什么 | tool 白名单、沙箱、human-in-the-loop gate |
| 10 | **Entropy auditing** | 检测 context / 状态的退化 | context rot 检测、loop 检测、dead-end 警报 |
| 11 | **Intervention recording** | 人类介入时记录为什么介入 | intervention log、override reason、audit trail |

> **记忆口诀**：Task + Context + Tools + Memory + State + Observe + Attribute + Verify + Permission + Entropy + Intervention

### 2.1 这 11 个组件的关系

```
                         ┌──────────────────────┐
                         │  1. Task Spec        │
                         │  2. Context Select   │  ← 信息进 harness
                         │  3. Tool Access      │
                         └──────────┬───────────┘
                                    ↓
                         ┌──────────────────────┐
                         │  4. Project Memory   │
                         │  5. Task State       │  ← 状态在 harness 内
                         └──────────┬───────────┘
                                    ↓
   ┌──────────────┐  ┌────────────────────────────────┐
   │ Agent Loop   │→ │  6. Observability              │  ← 运行时可见
   │              │  │  7. Failure Attribution         │
   │              │  │  8. Verification               │
   └──────────────┘  └────────────┬───────────────────┘
                                   ↓
                         ┌──────────────────────┐
                         │  9. Permissions      │
                         │ 10. Entropy Auditing │  ← 守护与质量
                         │ 11. Intervention Rec │
                         └──────────────────────┘
```

---

## 3. 四级阶梯：H0 → H3

论文提出 harness 的**渐进暴露**模型，从最小支持到完整支持：

| Level | 描述 | 能力 |
|-------|------|------|
| **H0** | **Bare model** | 纯模型，无 harness 支持——只看到 prompt，返回文本 |
| **H1** | **Read-only harness** | 能读文件、能看代码，但**不能写**——agent 只能建议 |
| **H2** | **Write harness** | 能写文件、能跑测试、能 commit——但每次操作都在沙箱里 |
| **H3** | **Production harness** | 全权执行——能开 PR、能部署、但每次关键操作有 human-in-loop gate |

> **为什么分级**：让 harness 的能力**渐进暴露**给 agent，避免一上来给太多权限导致灾难。

### 3.1 各级产出的 trace 不同

| Level | trace 包含 | 典型输出 |
|-------|------------|----------|
| H0 | 只有 prompt + 输出 | 一次 patch |
| H1 | + 读取的文件、观察到的内容 | patch + 引用 |
| H2 | + 写入的 diff、测试结果 | patch + reproduction log |
| H3 | + 失败归因、验证命令、permission 决策 | 完整 episode package |

> 论文的 trace-based 评测协议：把每次 agent run 打包成可审计的 episode package。

---

## 4. 与"工程化 Agent"的对应

把 11 个组件映射到现实中的工程实践：

| Component | Claude Code | OpenAI Codex | LangGraph | smolagents |
|-----------|-------------|--------------|-----------|------------|
| Task spec | `--print` / `CLAUDE.md` | prompt template | State schema | task prompt |
| Context select | `@file` / `@folder` | `@filename` | `MessagesState` | variable injection |
| Tool access | Bash/Read/Edit/Grep | function calling | `ToolNode` | `@tool` decorator |
| Project memory | CLAUDE.md | `AGENTS.md` | checkpointer | — |
| Task state | todo list | worktree | `State` object | step counter |
| Observability | trace view | log stream | LangSmith | `step_callbacks` |
| Failure attribution | exit code / trace | error event | retry policy | exception handling |
| Verification | bash test | `pytest` | evaluator | custom check |
| Permissions | permission prompt | approval flow | node-level guard | tool whitelist |
| Entropy auditing | context warning | token counter | cycle detection | max_steps |
| Intervention recording | override log | — | interrupt | — |

---

## 5. Harness 设计原则（实战提炼）

基于 Claude Code / Codex / LoopX / paper_agent 等项目的实战经验，Harness Engineering 有 4 条核心原则：

### 5.1 让模型"看到该看到的"

> **Harness = context provider**。模型的输出质量上限 = 它能看到的信息质量。

- 选对 file path（不要 all-files 一次性塞）
- 设计 tool 返回值（返回精炼结果，不返回冗长日志）
- 维护好的 project memory（CLAUDE.md 要小而精）

### 5.2 让模型"做该做的"

> **Harness = action surface**。权限粒度决定 agent 能做什么。

- 工具集要小而清晰（避免歧义）
- 危险操作要 gate（删文件、push、deploy）
- 失败要可重试、可恢复

### 5.3 让模型"知道自己做到了"

> **Harness = verification layer**。没有 verification，agent 不知道自己成功没。

- 每个 task 都要有 acceptance criteria
- 测试、assertion、playwright check
- 不要让模型自己说"做完了"——让外部 signal 说

### 5.4 让人类"看到模型在做什么"

> **Harness = observability layer**。否则就是黑盒。

- 完整的 trace（每个 tool call、每个 model response）
- observability 还要包含：失败归因、permission 决策、人类干预
- entropy auditing：context 是不是烂了？agent 是不是在 loop 里？

---

## 6. Harness vs Loop vs Context

三者经常被混用，但**层次清楚**：

| 层次 | 概念 | 关注 | 持续时间 |
|------|------|------|----------|
| 5（外） | **Loop Engineering** | 跨 session 的持续运转 | 几天到几周 |
| 4 | **Harness Engineering** | 一次 session 内的 runtime 机制 | 一次 session |
| 3 | **Context Engineering** | 塞什么进 context | 每个 turn |
| 2 | **Prompt Engineering** | 怎么写好一段 prompt | 一次性 |
| 1（内） | **Model** | 模型的 attention / 能力 | n/a |

> **关键**：Harness 是 Context 的**实现层**。Context 决定"什么该进"，Harness 决定"怎么进、进了之后怎么处理"。

---

## 7. 面试视角

- **Q：Harness Engineering 解决什么问题？**
  → 软件工程 Agent 的不可靠性不来自模型能力，而来自 runtime 缺位。Harness 是模型周围的 runtime substrate，负责 observe / act / feedback / verify。论文用 11 个组件 + H0-H3 阶梯形式化了这个概念。

- **Q：Harness Engineering 和 Context Engineering 的区别？**
  → Context 关心**信息**——塞什么进 window；Harness 关心**机制**——用 runtime 怎么管理 context。Harness 是 Context 的工程化实现层。Context 是原则，Harness 是机制。

- **Q：H0-H3 阶梯的实战价值？**
  → 渐进暴露 runtime 支持给 agent，避免一上来给太多权限。H0（裸模型）→ H1（只读）→ H2（写+沙箱）→ H3（全权+human gate）。每个 level 产出的 trace 不同，可用于 trace-based eval。

- **Q：Harness 的 11 个组件哪些最重要？**
  → 实战优先级：**Permissions > Verification > Observability > Failure Attribution**。前两个是"别让 agent 搞砸"，后两个是"agent 搞砸了我能看到"。

- **Q：为什么 Claude Code 比裸 prompt 强这么多？**
  → 因为它把 harness 的 11 个组件都实现了：sub-agent / hooks / permission / compaction / verification / trace / memory / state。Harness 是 Claude Code 能力的来源。

---

## 8. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [arXiv 2605.13357: AI Harness Engineering](https://arxiv.org/abs/2605.13357) | **论文原文**，11 个组件 + H0-H3 阶梯的正式定义 |
| [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Harness 是 Context 的实现层，Context 的官方权威定义必读 |
| [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | 早期 agent 设计原则（2024），harness 的雏形 |
| [Claude Code Overview](https://code.claude.com/docs/en/overview) | Harness 的最佳实战案例 |
| [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents) | Harness 组件 4（sub-agent）的实现 |
| [Claude Code Hooks](https://code.claude.com/docs/en/hooks) | Harness 组件 11（intervention / override）的实现 |
| [Codex CLI 文档](https://github.com/openai/codex) | 另一个完整的 harness 实现 |
| [Dive into Claude Code 论文](https://arxiv.org/abs/2604.14228) | 从系统设计角度分析 Claude Code 的 harness |
| [LoopX principles and pitfalls](https://github.com/huangruiteng/loopx/blob/main/docs/product/loop-engineering-principles-and-pitfalls.md) | Loop Engineering 视角下的 harness 限制 |

---

## 9. 练习

1. **H0 vs H1 对比**：用同一个任务，先用裸 prompt（H0），再给 read-only 工具（H1），对比输出质量
2. **实现 11 组件检查表**：为自己的 agent 写一份 11 组件 audit，标注哪些实现了、哪些缺失
3. **设计 permission 策略**：列出你 agent 需要的所有 tool，给每个 tool 标"safe / need-approval / dangerous" 三档
4. **trace-based eval**：跑 5 个任务，把每个的 trace 打包成 episode package（model + context + tool calls + verification result + permission decisions）
5. **entropy audit**：跑一个 50 turn 的 agent，监控 context token 数、重复 tool call、loop 模式，验证 entropy auditing 组件

---

## 10. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U3 Prompt Engineering | Harness 的"入口"是 prompt；harness 是 prompt 的执行环境 |
| U22 Context Engineering | **Harness 是 Context 的实现层**——概念相邻，机制互补 |
| U6 Tool Use & Function Calling | Harness 组件 3（Tool access）的具体实现 |
| U8 Memory & Reflection | Harness 组件 4（Project memory） |
| U9 Multi-Agent | Harness 组件 5（Task state）支持多 agent 协作 |
| U12 Agent 评估标准 | Trace-based eval 是 harness 的核心评估方法 |
| U18 Agent 安全与红队 | Harness 组件 9（Permissions）和组件 10（Entropy auditing）的安全角度 |
| U20 编程 Agent 内部原理 | Claude Code 是 Harness 11 组件的完整实现案例 |
| U21 Loop Engineering | Harness 是 Loop 的"runtime 单元"；Loop 是跨 session 的 harness 编排 |

---

*最后更新：2026-06-30 新增 U23（Harness Engineering）*
*来源：arXiv 2605.13357 论文、Anthropic 官方文档、Claude Code / Codex 实践*
