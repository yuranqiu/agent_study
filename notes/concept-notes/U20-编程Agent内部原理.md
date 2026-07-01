# U20 · 编程 Agent 内部原理

> **核心问题**：编程 Agent 是如何工作的？它内部的决策循环、工具调用、上下文管理是怎么实现的？
> **预计学时**：6h
> **对应 Phase**：Phase 2

---

## 学习目标（Gate 检验）

1. 能画出编程 Agent 的完整架构图（从用户输入到代码输出的数据流）
2. 能解释 Planning、Memory、ToolUse 三大模块如何协作
3. 能说明子 Agent（Sub-agent）与主 Agent 的上下文隔离与通信机制
4. 能描述代码修改的原子性和回滚机制的实现思路
5. 能分析 ToolCall 失败时的降级与重试策略
6. 能对比星型与网状多 Agent 拓扑的优劣

---

## 1. 编程 Agent 的定义与架构

### 什么是编程 Agent

编程 Agent 是以 LLM 为核心控制器，能够自主理解任务、规划步骤、执行代码修改、验证结果并循环迭代的智能系统。它不同于简单的代码补全工具（如 GitHub Copilot），后者是被动响应式；而编程 Agent 是主动规划式——给定一个高层目标，它自主拆解为可执行步骤并逐步完成。

### 核心架构

```
┌──────────────────────────────────────────────────────────┐
│                      用户输入                            │
│           （任务描述 / GitHub Issue / Bug 描述）          │
└────────────────────────┬────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│                     Planning 模块                        │
│           （任务拆解 · ReAct 循环 · 步骤排序）            │
└────────────────────────┬────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│                     Memory 模块                          │
│    （短期上下文 · 长期记忆 · 项目知识 · Session State）   │
└────────────────────────┬────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│                     ToolUse 模块                         │
│         （文件读写 · Bash 执行 · Git · 浏览器 / MCP）     │
└────────────────────────┬────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│                    Verification 模块                     │
│           （测试执行 · Linter · 编译器错误捕获）          │
└────────────────────────┬────────────────────────────────┘
                         ↓
                         ↓ （循环直到任务完成 / 超时 / 放弃）
┌──────────────────────────────────────────────────────────┐
│                     代码输出                             │
│            （修改后的文件 / PR / 验证结果）               │
└──────────────────────────────────────────────────────────┘
```

### 与传统软件开发范式的对比

| 维度 | 传统开发（IDE） | 编程 Agent |
|------|----------------|------------|
| 决策者 | 程序员显式操作 | LLM 隐式决策 |
| 步骤执行 | 手动一步步做 | Agent 自动循环执行 |
| 上下文 | 靠人脑记忆 | Memory 模块持续维护 |
| 错误修正 | 人眼检查 + 手动修改 | Verification + 自动重试 |
| 并行性 | 串行思维 | 多 Agent 可并行探索 |

### 各模块职责概述

- **Planning**：负责任务分解、步骤排序、判断完成条件。它是 Agent 的"大脑"。
- **Memory**：维护历史上下文、项目知识、用户偏好，是 Agent 的"记忆"。
- **ToolUse**：封装操作系统、文件系统、Git、网络等能力，是 Agent 的"手脚"。
- **Verification**：运行测试、解析编译器输出，是 Agent 的"自我检验"。

---

## 2. Planning 模块——任务拆解与执行控制

### Planning 的本质

Planning 模块的本质是一个**受限的自然语言推理循环**。给定用户的高层目标，它输出一个结构化的步骤序列（Step 1, Step 2, ...），并在每步执行后根据结果更新下一步的计划。

这与传统的任务规划（如古典 AI 的 HTN 规划）的区别在于：LLM-based Planning 不需要一个形式化的世界模型，而是用自然语言描述状态和目标，靠 LLM 的常识推理能力生成动作序列。

### ReAct 循环在 Planning 中的位置

ReAct（Reasoning + Acting）是多数编程 Agent 采用的核心循环范式：

```
while not done:
    thought = llm.reason(goal, history, context)
    if thought.is_tool_call:
        result = tool.execute(thought.tool, thought.args)
        history.add(observation=result)
    elif thought.is_final_answer:
        done = True
```

### 任务拆解策略

#### 自顶向下拆解（Decomposition）

适用于明确知道最终目标，但路径不清晰的任务。

```
目标：实现用户登录功能
  → 子目标1：设计数据库 User 表
  → 子目标2：实现后端 Auth API
  → 子目标3：实现前端登录表单
  → 子目标4：串联测试
```

#### 渐进式揭示（Progressive Elaboration）

适用于目标本身不清晰、需要在执行中逐渐明确的任务。Agent 先做最小的探索，根据反馈逐步细化计划。

### 步骤排序与依赖图

Planning 模块会维护一个内部依赖图：

```
Step A（读代码） → Step B（理解结构） → Step C（写测试） → Step D（实现功能） → Step E（验证）
```

其中 A 必须先于 B 执行，但 C 和 D 可以并行（如果测试不依赖实现先完成）。

### 什么时候停止 Planning

停止条件的三种判断方式：

1. **显式完成信号**：Agent 输出了 "Task completed" 并通过了 Verification
2. **最大循环次数**：防止无限循环（如 max_iterations=50）
3. **资源上限**：token 预算或时间预算耗尽

---

## 3. Memory 模块——上下文的维护与演化

### 短期记忆（Session Context）

短期记忆是当前对话窗口内的全部上下文，包括：

- 用户的原始指令
- 从磁盘读取的文件内容
- Tool Call 的输入输出
- LLM 的思考过程（Thought）
- Verification 的执行结果

它是 Agent 做出决策的直接依据。上下文窗口大小（context window）直接限制了 Agent 能"记住"多少信息。

### 长期记忆（Long-term Memory）

长期记忆存储跨 Session 的信息，常见形式：

- **项目知识**：CLAUDE.md / AGENTS.md 中维护的项目规范
- **用户偏好**：编程风格偏好、框架选择、命名约定
- **执行历史**：历史任务的总结（用于避免重复犯错）
- **Auto Memory**：自动从代码库结构和文档中提取的知识

### Memory 的更新策略

#### Compact（压缩）

当上下文即将超出限制时，Agent 执行压缩：保留关键信息（系统 prompt、项目规范），丢弃中间过程（详细的 Tool Call 输出），用摘要替代。

```
压缩前（示例）：
- Read(/src/auth/login.py): 300 行代码全文
- Edit(/src/auth/login.py): 修改了 5 行
- Bash(test): 运行了 20 条测试输出

压缩后：
- Read(/src/auth/login.py): 摘要（登录逻辑在 authenticate() 函数）
- Edit: 修改了 authenticate() 的错误处理
- Bash(test): 18 passed, 2 failed（具体失败输出被摘要）
```

#### 分级加载（Tiered Loading）

不是所有记忆都同等重要。Tiered Loading 根据优先级决定哪些内容常驻上下文：

- **Tier 1（始终加载）**：System prompt、项目根 CLAUDE.md、未限定的 rules
- **Tier 2（按需加载）**：子目录的 CLAUDE.md、限定 path 的 rules
- **Tier 3（从不自动加载）**：旧 Session 总结、过时文档

### 上下文污染的来源与对策

**来源**：
1. 大量文件读取后内容残留在上下文
2. Tool Call 输出的长命令结果（如 git log、完整编译输出）
3. 重复失败的 Tool Call 记录
4. Sub-agent 返回的冗长 summary

**对策**：
- 用 head/grep/limit 收敛长输出
- Explore Sub-agent 只返回 summary，不回传详细输出
- 及时 Compact
- Sub-agent 严格控制返回内容的大小

---

## 4. ToolUse 模块——工具封装与调用机制

### Tool 的分类

编程 Agent 的 Tool 可分为三大类：

#### 第一类：基础设施工具（必选）

| Tool | 作用 | 风险等级 |
|------|------|----------|
| Read | 读取文件内容 | 低 |
| Edit | 修改文件 | 中（可回滚） |
| Bash | 执行 Shell 命令 | 高（可能有副作用） |
| Grep | 全文搜索 | 低 |
| Glob | 按模式查找文件 | 低 |

#### 第二类：代码操作工具

| Tool | 作用 |
|------|------|
| Write | 创建新文件 |
| Multi-Edit | 批量修改 |
| Git | 版本控制操作 |
| Search | Web 搜索 |

#### 第三类：MCP 工具（Model Context Protocol）

MCP 是标准化的工具发现与调用协议，允许 Agent 连接到外部服务（数据库、API、IDE 插件等）。MCP 工具的特点是按需加载（Deferred Tools）：首次只知道名字，调用时才获取完整 schema。

### Function Calling 的工作流

```
用户输入 → Planning 推理 → 决定调用 Tool → 生成 Tool Call
    ↓
Tool Executor 接收调用 → 执行工具 → 捕获输出
    ↓
Observation 格式化 → 返回给 LLM → 更新 Memory → 下一轮推理
```

### ToolCall 失败的处理策略

#### 失败分类

| 失败类型 | 示例 | 处理策略 |
|----------|------|----------|
| 语法/参数错误 | 调用 Tool 时参数格式不对 | LLM 自我修正，重新生成调用 |
| 执行失败 | 文件不存在、权限不足 | 尝试替代路径或告知用户 |
| 超时 | 命令执行超过限制 | 杀掉进程，尝试简化命令 |
| 部分失败 | 批量操作中部分成功 | 回滚全部，重新规划 |

#### 重试机制

典型的重试策略：

```
max_retries = 3
retry_delay = 1s (指数退避)
for attempt in range(max_retries):
    result = tool.execute(args)
    if result.success:
        break
    elif is_retryable(result.error):
        sleep(retry_delay * (2 ** attempt))
    else:
        raise result.error
```

### Bash 工具的特殊性

Bash 是最高风险的 Tool，因为它可以执行任意 Shell 命令。

**安全措施**：
1. **权限分级**（Permission Mode）：从「每次询问」到「全部放行」分多级
2. **白名单限制**：某些危险命令（rm -rf /）强制要求确认
3. **熔断机制**：单次操作 token 或时间超限直接中断
4. **Dry Run**：危险操作先输出要执行的命令，让用户确认

---

## 5. Sub-agent 与多 Agent 协作机制

### Sub-agent 的设计目的

Sub-agent 解决的核心问题是**上下文隔离**和**并行处理**。

- **上下文隔离**：Sub-agent 有自己独立的 context，历史 Tool Call 输出不会污染主对话
- **并行处理**：多个无依赖的子任务可以同时派发给不同的 Sub-agent

### Sub-agent 与主 Agent 的通信

```
主 Agent（Main Context）
    ├── 派发 Task A → Sub-agent A（独立 context）
    ├── 派发 Task B → Sub-agent B（独立 context）
    └── 派发 Task C → Sub-agent C（独立 context）
            ↓              ↓              ↓
        返回 Summary    返回 Summary    返回 Summary
            ↓              ↓              ↓
    主 Agent 汇总结果，继续主任务
```

Sub-agent 返回给主 Agent 的是**摘要**（Summary），不是完整输出。这避免了大量中间过程污染主上下文。

### 拓扑对比：星型 vs 网状

#### 星型拓扑（Star）

所有 Sub-agent 只与主 Agent 通信，Sub-agent 之间不直接通信。

**优点**：
- 上下文完全可控（信息汇聚到主 Agent）
- 易于调试（每个 Sub-agent 的输入输出清晰）
- 避免循环对话

**缺点**：
- 主 Agent 可能是瓶颈（所有信息都要经过它）
- 不适合需要大量实时协作的场景

#### 网状拓扑（Mesh）

Sub-agent 之间可以直接通信，形成对等网络。

**优点**：
- 接近真实团队协作模式
- 信息传播更自然
- 可以并行形成共识

**缺点**：
- 上下文可能爆炸（循环对话无限增长）
- 调试困难（消息链路不透明）
- 容易产生矛盾结论（不同 Agent 对同一问题给出不同答案）

### 什么时候用 Sub-agent

**应该用**：
- 读 10+ 文件做调研（输出会很噪）
- 独立的安全 review / code review
- 并行做多个无依赖的子任务
- 需要完全独立的推理过程

**不应该用**：
- 只需要 1-3 个 Tool Call（主 Agent 直接做更快）
- 需要主 Agent 保留思考过程用于后续决策
- 需要与用户交互式确认（Sub-agent 不能问用户）

---

## 6. 上下文污染防控与 Agent 优化实践

### 上下文污染的根源

上下文污染的本质是**信噪比下降**：关键指令（如"不要修改登录逻辑"）被大量无关信息稀释，导致 Agent 在无关的方向上推理或行动。

典型污染场景：

1. **文件读取污染**：Agent 读了 20 个文件，相关文件的内容还在上下文中占位
2. **长命令输出污染**：Bash 的完整输出（数千行）留在上下文
3. **失败历史污染**：多个失败的重试记录让 Agent 困在错误路径
4. **Sub-agent 输出污染**：Sub-agent 返回的详细日志直接回传主 Agent

### 主动防控策略

#### 精确化工具调用

不要让 Agent 自己判断"读哪些文件"，而是在 System prompt 中明确指导：

```
正确示范：
"需要修改 /src/auth/login.py 中的 authenticate() 函数，先读这个文件，
确认函数签名和返回值格式后再修改。"

错误示范：
"先了解一下项目结构，然后找到登录相关的代码并修改。"
```

#### 输出收敛

对于 Bash 等输出不可预测的工具，使用收敛手段：

```bash
# 不收敛：返回数千行
git log --all --oneline --graph

# 收敛：只返回相关行
git log --oneline -20 -- src/auth/

# 用 grep 二次收敛
git log --oneline | grep -i "login\|auth"
```

#### 及时 Compact

在每个里程碑后执行 Compact，丢弃中间过程的详细输出：

```
里程碑 checkpoint：
1. 任务理解 ✓ → Compact：丢弃原始任务描述的冗余表述
2. 代码探索 ✓ → Compact：丢弃已读文件的详细内容，只保留摘要
3. 实现完成 ✓ → Compact：丢弃中间失败的编辑尝试记录
```

### Verification 驱动的自我修正

没有 Verification，Agent 就是"盲人骑瞎马"。Verification 是 Agent 自我检验的唯一客观手段。

#### 测试驱动的 Agent 工作流（TDD for Agent）

```
1. Plan：先写验收测试（明确目标）
2. Implement：根据测试要求实现代码
3. Verify：运行测试，捕获失败信息
4. Fix：基于失败信息修正
5. Loop：回到步骤 3，直到通过
```

#### 关键原则：测试不由 Agent 独自写

Agent 自己写测试再自己实现，可能"为过测试而过测试"（测试和实现用同一份错误的假设）。关键测试应由人编写或经过独立 review。

### 代码修改的原子性与回滚

#### 原子性保障

单次 Edit 操作应该是原子的：
- 要么全部成功（写入文件）
- 要么全部失败（不修改文件）

实现上，通常是先写临时文件，确认格式正确后再 rename（原子替换）。

#### 回滚机制

常用的回滚策略：

1. **Git 工作树备份**：修改前 `git stash`，失败后 `git stash pop`
2. **影子文件**：修改前复制一份 `.bak`，失败后还原
3. **Edit 历史记录**：记录每次 Edit 的 before/after，用于 undo

### 性能与成本优化

#### Token 成本

每次 LLM 推理都按完整 context 的 token 数计费。优化方向：

- **控制上下文长度**：及时 Compact，用收敛工具
- **减少 Tool Call 次数**：批量操作，而非逐个调用
- **选择合适的模型**：简单任务用小模型，复杂推理用大模型

#### 时延优化

Tool Call 每次都有网络往返时延（LLM → Tool → LLM）。优化方向：

- **并行 Tool Call**：支持同时发起多个无依赖的 Tool Call
- **缓存**：重复读取同一文件时返回缓存内容（避免重复 IO）
- **预取**：根据推理上下文预测下一个 Tool Call 并预执行

---


## 7. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U1 LLM 基础|编程 Agent 的 LLM 推理能力来自 U1|
|U6 Tool Use|编程 Agent 的代码执行依赖 U6 的 Tool Calling|
|U9 Multi-Agent|编程 Agent 常作为 Multi-Agent 系统的一部分|
|U25 Computer Use|编程 Agent 和 Computer Use Agent 共享底层架构|
|U29 Agent Deploy|编程 Agent 的生产部署依赖 U29 的知识|
*最后更新：2026-05-27*