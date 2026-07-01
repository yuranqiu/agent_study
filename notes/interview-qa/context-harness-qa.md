# Interview QA · Context Engineering & Harness Engineering

> 覆盖：U22 Context Engineering + U23 Harness Engineering
> 来源：Anthropic 官方博客、arXiv 2605.13357、Claude Code / Codex 实践

---

## 一、Context Engineering 基础

### Q1: 什么是 Context Engineering？Anthropic 的官方定义是什么？

**Answer:**

**Context Engineering** 是 Anthropic 团队 2025 年 9 月正式命名的学科：

> "Context refers to the set of tokens included when sampling from a large-language model (LLM). The **engineering** problem at hand is optimizing the utility of those tokens against the inherent constraints of LLMs in order to consistently achieve a desired outcome."

**核心含义：**
- **Context** = 任何会进入 LLM context window 的 token
- **Engineering** = 对这些 token 做精心策划和持续维护
- **目标** = 让每一轮都是"最小高信号集"

**Context 包含的内容：**
- System prompt、Tools（function schema）、Few-shot examples
- 消息历史（user / assistant / tool messages）
- 检索结果（RAG、文件读、网页内容）
- 外部状态（scratchpad、NOTES.md、todo list）
- 记忆（短期 / 长期）

---

### Q2: Context Engineering 和 Prompt Engineering 有什么区别？

**Answer:**

| 维度 | Prompt Engineering | Context Engineering |
|------|--------------------|---------------------|
| 目标 | 写好一段 prompt | 维护整个 context state |
| 范围 | System + user 消息 | System + tools + history + memory + RAG + ... |
| 时机 | 一次性 | 每个 turn 都做 curation |
| 关注点 | 措辞、示例、结构 | 哪些 token 该进、该出、该压缩 |
| 比喻 | 写一封好邮件 | 维护一整个收件箱 |

**关键区分：**
- Prompt Engineering 是 Context Engineering 的子集
- Prompt 是一次性决策，Context 是**每个 turn 都要做的实时决策**
- Anthropic 官方表述："We view context engineering as the natural progression of prompt engineering."

```python
# Context Engineering（每轮实时策划）
def curate_context(messages, tools, memory, max_tokens=8000):
    return {
        "system_prompt": get_system_prompt(),
        "tools": select_relevant_tools(current_task, max_tokens),
        "history": compact_message_history(messages, max_tokens),
        "memory": retrieve_relevant_memory(current_task, top_k=3)
    }
```

---

### Q3: 什么是 Context Rot（上下文腐烂）？

**Answer:**

**Context Rot** = 随着 context token 数增加，模型的信息检索和长程推理精度下降的现象。

**原因：**
- LLM 是 transformer 架构，n 个 token → n² pairwise relationships
- 模型在短序列训练数据上更熟练
- 长 context 会出现"位置编码插值"的精度损失

**核心特征：**
- **不是 hard cliff**，是 performance gradient
- 长 context 仍能工作，但精度会下降
- 高 context 长度 ≠ 高 context 质量

**面试追问：**
> Q: Context window 越来越大，是不是就不用 compaction 了？
> A: 不是。即使 1M context 也存在 context rot 和 attention budget 问题。Compaction 是工程策略，不只是 token 限制的 workaround。

---

### Q4: LLM 的"注意力预算"是什么？

**Answer:**

**Attention Budget** = 模型在有限 context window 内有效处理信息的容量上限。

**类比人类 Working Memory：**
- 人脑：7±2 items（Miller's Law）
- LLM：也有类似的"注意力容量"限制

**数学理解：**
```
Attention Cost ≈ O(n²)  # n 个 token 的 pairwise relationships
n = 1000  → 1,000,000 对关系
n = 4000  → 16,000,000 对关系（16 倍）
```

**关键洞察：** "什么该进 context"和"什么该出 context"是同等重要的问题。

---

## 二、5 大 Context 组件

### Q5: Anthropic 官方列出的 5 大 Context 组件是什么？

**Answer:**

| 组件 | 优化目标 | 常见反模式 |
|------|----------|------------|
| **System prompts** | 给出"对的海拔"——既不过度具体、也不过度模糊 | 把所有 if-else 都硬编码进 prompt；或写得太泛 |
| **Tools** | 工具集小而清晰、token 效率高、行为明确 | 工具集臃肿、功能重叠、agent 不知道该选哪个 |
| **Examples** | 一组多样、有代表性的"教科书例子" | 把所有 edge case 都塞进 prompt |
| **Message history** | 早期消息压缩、工具结果清理 | 原样保留所有 tool 输出 |
| **External data** | 检索 vs 即时加载的平衡 | 全部预先塞进 context |

**System Prompt 的"对的海拔"：**

```
过高（brittle）              对的海拔                  过低（vague）
─────────────────     ─────────────────────     ─────────────────
"如果 X 且 Y 则 Z，      "你是 X 角色，            "你是一个智能助手"
否则如果 X 且 Z 则..."   按以下步骤处理..."         "请按用户要求做"
```

**方法：** 从最小 prompt + 最好的模型开始，根据失败模式加明确指令和示例。

---

### Q6: Tools 设计的最佳实践是什么？

**Answer:**

**核心原则：**
> "If a human engineer can't definitively say which tool should be used in a given situation, an AI agent can't be expected to do better."

**设计要点：**
1. **Self-contained**：工具应该独立、健壮、错误 tolerant
2. **Descriptive names**：输入参数要 descriptive, unambiguous
3. **Clear boundaries**：工具应该鼓励 agent 高效行为

```python
# 反模式：工具定义模糊
@tool("do_something")
def do_something(x):
    """Does something with x"""
    pass

# 好实践：工具定义清晰
@tool("search_code")
def search_code(query: str, file_pattern: str = "*.py", case_sensitive: bool = False) -> str:
    """在代码库中搜索匹配 query 的代码片段。"""
    pass
```

**面试追问：**
> Q: 工具集多大合适？
> A: 没有固定数字，但要确保 agent 能清楚判断"何时用哪个"。通常 5-10 个核心工具比 50 个重叠工具更有效。

---

## 三、Context Retrieval 策略

### Q7: Just-in-time Retrieval vs Pre-loaded Context 有什么区别？

**Answer:**

| 策略 | 做法 | 优劣 |
|------|------|------|
| **Pre-inference retrieval** | 启动前把所有相关数据 embedding 检索出来塞进 context | 快，但索引可能 stale |
| **Just-in-time retrieval** | 给 agent **轻量级标识符**（file path / query / URL），让 agent 在 runtime 用 tools 动态加载 | 慢，但更灵活、能处理动态内容 |

**Claude Code 的 Hybrid 策略：**
> "CLAUDE.md files are naively dropped into context up front, while primitives like glob and grep allow it to navigate its environment and retrieve files just-in-time."

```python
# Claude Code 的 hybrid 策略
def build_context(task: str, repo_path: str):
    return {
        # 1. 预加载：项目级元信息（小而精）
        "system": read_file("CLAUDE.md"),      # ~2K tokens
        "readme": read_file("README.md"),       # ~3K tokens

        # 2. 给 agent 探索工具
        "tools": ["glob", "grep", "read"],

        # 3. 任务描述
        "task": task
    }
    # Agent 会自己决定：用 glob 找文件、用 grep 搜索、用 read 读取
```

---

### Q8: 什么是 Progressive Disclosure（渐进披露）？

**Answer:**

**Progressive Disclosure** = 让 agent 增量发现 context，而不是一次性塞满。

| 信号 | 含义 | Agent 的决策 |
|------|------|--------------|
| File sizes | 提示复杂度 | 大文件先看目录结构 |
| Naming conventions | 提示用途 | 文件名暗示功能 |
| Timestamps | 提示时效 | 最新文件可能最重要 |
| Tool call results | 提示下一轮决策 | 根据结果调整探索方向 |

**关键洞察：**
> context 不应该是"一次性塞好"，应该是**每一轮都重新 curate**。

```python
# 反模式：一次性塞满
context = read_all_files("src/")  # 100K tokens

# 好实践：Progressive Disclosure
def progressive_disclosure(repo_path: str):
    # Level 1: 目录结构
    structure = glob("**/*.py", root_dir=repo_path)

    # Level 2: 文件大小（判断哪些值得深入）
    file_sizes = {f: get_file_size(f) for f in structure}

    # Level 3: Agent 自主决定读哪些
    return {
        "structure": structure[:20],
        "file_sizes": file_sizes,
        "tools": ["read", "grep", "glob"]
    }
```

---

## 四、Long-Horizon 三大技术

### Q9: 什么是 Compaction（压缩）？

**Answer:**

**Compaction** = 当对话接近 context window 上限时，总结历史内容，然后用摘要重新初始化 context window。

> "Compaction is the practice of taking a conversation nearing the context window limit, summarizing its contents, and reinitiating a new context window with the summary."

**Claude Code 的做法：**
- 把历史消息传给模型 → 总结 → 压缩
- 保留：架构决策、未解决的 bug、实现细节
- 丢弃：冗余的 tool 输出、消息
- 续接：压缩后的 context + 最近访问的 5 个文件

```python
def compact_history(messages: list, max_tokens: int = 4000) -> list:
    """压缩对话历史"""
    if count_tokens(messages) <= max_tokens:
        return messages

    recent_messages = messages[-10:]
    early_messages = messages[:-10]
    summary = summarize_messages(early_messages)  # 用 LLM 总结

    return [
        {"role": "system", "content": f"压缩摘要：{summary}"},
        *recent_messages
    ]
```

---

### Q10: 什么是 Structured Note-Taking（结构化笔记）？

**Answer:**

**Structured Note-Taking** = Agent 定期把笔记持久化到 context window 外部的存储。

> "The agent regularly writes notes persisted to memory outside of the context window."

**实现形式：**
- Claude Code 的 todo list
- 用户的 `NOTES.md` / `STATE.md` / `LOOP.md`
- Anthropic 2025-09 发布的 **memory tool（公测）**——文件系统接口

**模式：NOTES.md + 续接时读自己的笔记**

```markdown
# Agent Notes

## Current Task
Implement user authentication

## Progress
- [x] Design database schema
- [x] Create User model
- [ ] Implement login endpoint
- [ ] Add password hashing

## Key Decisions
- Using JWT for authentication
- Password hashing with bcrypt (cost factor 12)

## Blockers
- Need to decide on token expiration time
```

---

### Q11: 什么是 Sub-Agent Architecture（子 Agent 架构）？

**Answer:**

**Sub-Agent Architecture** = 主 agent 维护高层 plan，子 agent 用各自干净的 context window 做深度技术工作。

**工作方式：**
- 主 agent 维护高层 plan
- 子 agent 用**各自干净的 context window**做深度技术工作
- 子 agent 返回 1000-2000 token 的精炼摘要
- 主 agent 只做综合分析

**三大技术选择：**

| 任务特征 | 推荐技术 |
|----------|----------|
| 大量来回对话、需要对话连贯性 | Compaction |
| 迭代开发、清晰里程碑 | Note-taking |
| 复杂研究、并行探索收益大 | Multi-agent |

---

## 五、Harness Engineering 基础

### Q12: 什么是 Harness Engineering？

**Answer:**

**Harness Engineering** 由 arXiv 论文 [AI Harness Engineering](https://arxiv.org/abs/2605.13357) 正式定义：

> "Software-engineering capability emerges from a model-harness-environment system, in which a runtime substrate — the harness — mediates how a foundation-model agent observes a project, acts on it, receives feedback, and establishes that a change is complete."

**核心含义：**
- **Harness** = 模型周围的 runtime substrate
- 所有"让模型能做事"的工程化机制
- 模型本身不会自动有这些能力，需要被 harness 包裹

**与 Context Engineering 的关系：**
- **Context Engineering** 关心**"塞什么进 context window"** —— information-level
- **Harness Engineering** 关心**"怎么用 runtime 机制管理 context"** —— mechanism-level

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

### Q13: Harness 的 11 个组件职责是什么？

**Answer:**

| # | Component | 职责 | 典型实现 |
|---|-----------|------|----------|
| 1 | **Task specification** | 怎么向 agent 表达任务 | prompt template、issue template |
| 2 | **Context selection** | 该把哪些 token 放进 context window | RAG 检索、token 预算管理、compaction |
| 3 | **Tool access** | 怎么让 agent 触达工具 | function calling、JSON schema、MCP |
| 4 | **Project memory** | 项目级持久化记忆 | `STATE.md`、CLAUDE.md、NOTES.md |
| 5 | **Task state** | 当前任务的状态机 | todo list、claim/lease、in-progress 标记 |
| 6 | **Observability** | 怎么看到 agent 在做什么 | trace、log、metrics、UI dashboard |
| 7 | **Failure attribution** | 失败时定位原因 | error code、retry、context diff |
| 8 | **Verification** | 怎么确认 agent 真的做对了 | tests、playwright、assertion、eval |
| 9 | **Permissions** | agent 能做什么、不能做什么 | tool 白名单、沙箱、human-in-the-loop gate |
| 10 | **Entropy auditing** | 检测 context / 状态的退化 | context rot 检测、loop 检测、dead-end 警报 |
| 11 | **Intervention recording** | 人类介入时记录为什么介入 | intervention log、override reason、audit trail |

**记忆口诀：**
```
Task + Context + Tools + Memory + State + Observe + Attribute + Verify + Permission + Entropy + Intervention
```

---

### Q14: H0-H3 四级阶梯是什么？

**Answer:**

| Level | 描述 | 能力 |
|-------|------|------|
| **H0** | **Bare model** | 纯模型，无 harness 支持——只看到 prompt，返回文本 |
| **H1** | **Read-only harness** | 能读文件、能看代码，但**不能写**——agent 只能建议 |
| **H2** | **Write harness** | 能写文件、能跑测试、能 commit——但每次操作都在沙箱里 |
| **H3** | **Production harness** | 全权执行——能开 PR、能部署、但每次关键操作有 human-in-loop gate |

**为什么分级：**
让 harness 的能力**渐进暴露**给 agent，避免一上来给太多权限导致灾难。

**各级产出的 trace：**

| Level | trace 包含 | 典型输出 |
|-------|------------|----------|
| H0 | 只有 prompt + 输出 | 一次 patch |
| H1 | + 读取的文件、观察到的内容 | patch + 引用 |
| H2 | + 写入的 diff、测试结果 | patch + reproduction log |
| H3 | + 失败归因、验证命令、permission 决策 | 完整 episode package |

**面试追问：**
> Q: H0 和 H1 的区别在哪里？
> A: H0 是裸模型，只能根据 prompt 生成文本。H1 给它 read-only 工具（glob/grep/read），让它能观察代码库但只能建议修改。这个渐进暴露很重要——先让 agent 学会"看"，再让它学会"做"。

---

### Q15: Claude Code 的 Harness 实现有哪些特点？

**Answer:**

Claude Code 是 Harness Engineering 的**完整实战案例**，实现了 11 个组件的绝大多数：

| Component | Claude Code 实现 |
|-----------|------------------|
| Task spec | `--print` / `CLAUDE.md` |
| Context select | `@file` / `@folder` 语法 |
| Tool access | Bash/Read/Edit/Grep 四大工具 |
| Project memory | CLAUDE.md（项目级元信息） |
| Task state | todo list（内建） |
| Observability | trace view（UI 展示执行过程） |
| Failure attribution | exit code / trace（失败可追溯） |
| Verification | bash test（自动运行测试） |
| Permissions | permission prompt（危险操作需确认） |
| Entropy auditing | context warning（context 过大时提示） |
| Intervention recording | override log（人类覆盖时记录） |

**关键特性：**

1. **Hybrid Context Strategy**
   - CLAUDE.md 预加载
   - glob/grep/read 即时探索

2. **Compaction 实现**
   - 自动总结历史对话
   - 保留最近访问的 5 个文件
   - 丢弃冗余 tool 输出

**面试追问：**
> Q: Claude Code 为什么比裸 prompt 强这么多？
> A: 因为它把 harness 的 11 个组件都实现了。模型本身只是"大脑"，harness 提供了"眼睛（context selection）、手（tool access）、记忆（memory）、判断力（verification）"。

---

## 六、面试专题

### Q16: Context Engineering vs Prompt Engineering 怎么区分？

**Answer:**

| 维度 | Prompt Engineering | Context Engineering |
|------|--------------------|---------------------|
| 定义来源 | 行业通用 | Anthropic (2025-09 官方) |
| 核心问题 | 怎么写 | 塞什么进 window |
| 范围 | system msg | 整个 context state |
| 时机 | 一次性 | 每 turn 实时 |
| 度量 | 输出质量 | 注意力预算利用 |

**一句话区分：**
- **Prompt Engineering**：写好**一段** instruction
- **Context Engineering**：维护**整个** context state（每轮都要决定什么进、什么出）

**面试回答模板：**
> "Prompt Engineering 是 Context Engineering 的子集。Prompt 关注怎么写好一段 instruction，Context Engineering 关注怎么管理整个 context state。Context Engineering 的提出是因为我们发现，即使 prompt 写得再好，如果 context 管理不好，模型仍然会表现不佳。"

---

### Q17: Harness 的 11 个组件哪些最重要？

**Answer:**

**实战优先级排序：**

| 优先级 | 组件 | 理由 |
|--------|------|------|
| **P0** | **Permissions** | 别让 agent 搞砸（删文件、泄露数据） |
| **P0** | **Verification** | 别让 agent 以为自己做对了 |
| **P1** | **Observability** | Agent 搞砸了我能看到 |
| **P1** | **Failure Attribution** | 知道失败原因才能修复 |

**核心逻辑：**
> **Permissions > Verification > Observability > Failure Attribution**
> - 前两个是"别让 agent 搞砸"
> - 后两个是"agent 搞砸了我能看到"

```python
# 最小可行 Harness（MVP）
class MinimalHarness:
    def __init__(self):
        self.permissions = PermissionManager()  # P0
        self.verification = VerificationLayer()  # P0
        self.observability = ObservabilityLayer()  # P1

    def execute(self, task: str):
        if not self.permissions.can_execute(task):
            raise PermissionError("Task not allowed")

        result = self.run_agent(task)

        if not self.verification.check(result):
            raise VerificationError("Task failed verification")

        self.observability.log(task, result)
        return result
```

**面试追问：**
> Q: 如果只有 1 个组件的时间，你选哪个？
> A: Permissions。因为安全问题是不可逆的——agent 删了文件、泄露了数据，后果很严重。其他问题都是可以迭代改进的。

---

### Q18: 如何设计一个 Permission 系统？

**Answer:**

**三档权限模型：**

| 档位 | 工具类型 | 示例 |
|------|----------|------|
| **Safe** | 可自动执行 | 读文件、搜索代码、运行测试 |
| **Need-Approval** | 需人类确认 | 修改文件、运行构建、打开 PR |
| **Dangerous** | 禁止或严格限制 | 删除文件、推送 main、部署生产 |

```python
class PermissionManager:
    def __init__(self):
        self.permission_matrix = {
            "read": "safe",
            "grep": "safe",
            "glob": "safe",
            "edit": "need-approval",
            "bash": "need-approval",
            "delete": "dangerous",
            "push": "dangerous",
            "deploy": "dangerous"
        }

    def check_permission(self, tool: str, args: dict) -> str:
        level = self.permission_matrix.get(tool, "need-approval")
        if level == "safe":
            return "allow"
        elif level == "need-approval":
            return self.request_approval(tool, args)
        else:
            return "deny"
```

---

### Q19: 什么是 Trace-based Eval？

**Answer:**

**Trace-based Eval** = 把每次 agent run 打包成可审计的 episode package，用于评估和改进。

**Episode Package 包含：**

| 组件 | 内容 |
|------|------|
| Model | 使用的模型及配置 |
| Context | 完整的 context state |
| Tool Calls | 所有工具调用及结果 |
| Verification | 验证结果 |
| Permission Decisions | 权限决策记录 |

```python
class TraceBasedEval:
    def run_episode(self, task: str) -> dict:
        episode = {
            "task": task,
            "model": get_model_config(),
            "context": get_context_state(),
            "tool_calls": [],
            "verification": None,
            "permission_decisions": []
        }

        result = self.harness.execute(task)
        episode["tool_calls"] = self.harness.tool_history
        episode["permission_decisions"] = self.harness.permission_log
        episode["verification"] = self.verify(result)

        return episode

    def evaluate(self) -> dict:
        return {
            "success_rate": sum(1 for e in self.episodes if e["verification"]["passed"]),
            "avg_tool_calls": statistics.mean(len(e["tool_calls"]) for e in self.episodes),
            "permission_violations": sum(1 for e in self.episodes for d in e["permission_decisions"] if d["violated"])
        }
```

---

### Q20: Loop Engineering、Harness Engineering、Context Engineering 三者关系？

**Answer:**

| 层次 | 概念 | 关注 | 持续时间 |
|------|------|------|----------|
| 5（外） | **Loop Engineering** | 跨 session 的持续运转 | 几天到几周 |
| 4 | **Harness Engineering** | 一次 session 内的 runtime 机制 | 一次 session |
| 3 | **Context Engineering** | 塞什么进 context | 每个 turn |
| 2 | **Prompt Engineering** | 怎么写好一段 prompt | 一次性 |
| 1（内） | **Model** | 模型的 attention / 能力 | n/a |

**关键区分：**
- **Context Engineering**：要管理**什么信息**进 context
- **Harness Engineering**：管理 context 的**runtime 机制**（tool、permission、memory、verification）
- **Loop Engineering**：harness 之上**跨 session 的持续运转**框架

```python
# Loop Engineering（跨 session）
loop = Loop(agent=agent, harness=harness, max_iterations=1000)
loop.run()  # 可以持续几天

# Harness Engineering（一次 session）
harness = Harness(permissions=permissions, verification=verification)
result = harness.execute(task)  # 一次 session

# Context Engineering（每个 turn）
def curate_context(messages, tools, memory):
    return {
        "system_prompt": get_system_prompt(),
        "tools": select_tools(current_task),
        "history": compact_history(messages),
        "memory": retrieve_memory(current_task)
    }
```

---

## 七、综合面试题

### Q21: 描述一个你设计过的 Context Engineering 方案

**Answer:**

**回答框架：**

1. **问题定义**：任务类型、Context 挑战
2. **组件设计**：5 大组件分别怎么设计、Token 预算分配
3. **Long-horizon 策略**：用 Compaction / Note-taking / Sub-agent 哪个？为什么？
4. **Just-in-time vs Pre-loaded**：哪些信息预加载？哪些即时检索？
5. **验证方法**：怎么评估 context 质量？怎么检测 context rot？

**示例回答：**
> "在设计代码审查 agent 时，我遇到的问题是：代码库很大，不可能把所有代码都塞进 context。我的方案是：
> 1. **Pre-loaded**：README、CLAUDE.md（项目级元信息）
> 2. **Just-in-time**：用 glob/grep 找到相关文件，用 read 读取
> 3. **Note-taking**：把审查结果写入 REVIEW.md，跨 session 累积
> 4. **Compaction**：每 20 轮对话压缩一次历史
> 5. **验证**：对比 agent 的审查结果和人工审查的覆盖率"

---

## 八、速查表

### Context Engineering 速查

| 概念 | 一句话 |
|------|--------|
| Context Engineering | 维护整个 context state，每轮都做 curation |
| Context Rot | 随着 context 增加，模型的信息检索精度下降 |
| Attention Budget | 模型在有限 context 内有效处理信息的容量 |
| Right Altitude | System prompt 的抽象程度恰到好处 |
| Just-in-time | 给 agent 标识符，让它 runtime 动态加载 |
| Pre-loaded | 启动前把所有相关数据塞进 context |
| Compaction | 总结历史，重新初始化 context window |
| Note-taking | 定期把笔记持久化到 context 外 |
| Sub-agent | 子 agent 用干净的 context window 做深度工作 |

### Harness Engineering 速查

| 概念 | 一句话 |
|------|--------|
| Harness | 模型周围的 runtime substrate |
| 11 组件 | Task + Context + Tools + Memory + State + Observe + Attribute + Verify + Permission + Entropy + Intervention |
| H0 | 裸模型，只有 prompt + 输出 |
| H1 | 只读，能看不能改 |
| H2 | 可写，但在沙箱里 |
| H3 | 全权执行，有 human-in-the-loop gate |
| Trace-based Eval | 把每次 run 打包成可审计的 episode package |

---

## 九、参考资料

| 材料 | 为什么读 |
|------|----------|
| [Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | **官方权威定义**，必读 |
| [arXiv 2605.13357](https://arxiv.org/abs/2605.13357) | Harness Engineering 论文，11 组件 + H0-H3 |
| [Claude Code Overview](https://code.claude.com/docs/en/overview) | Harness 最佳实战案例 |
| [Chroma: Context Rot](https://research.trychroma.com/context-rot) | Context Rot 实验证据 |

---

*最后更新：2026-07-01*
*来源：U22 Context Engineering + U23 Harness Engineering*
