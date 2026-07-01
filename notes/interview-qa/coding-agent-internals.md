# Coding Agent Internals

> 编程 Agent 内部原理：架构、决策循环、上下文管理与协作机制
> 对应 Phase 2 · U20

---

## 1. 核心架构概览

编程 Agent 是以 LLM 为核心控制器的主动规划系统，区别于 GitHub Copilot 等被动补全工具——给定高层目标，自主拆解步骤并循环执行直到完成。

```
用户输入（任务描述 / Issue / Bug 描述）
    ↓
Planning 模块（任务拆解 · ReAct 循环 · 步骤排序）
    ↓
Memory 模块（短期上下文 · 长期记忆 · 项目知识 · Session State）
    ↓
ToolUse 模块（文件读写 · Bash · Git · MCP）
    ↓
Verification 模块（测试 · Linter · 编译器错误）
    ↓
循环直到完成 / 超时 / 放弃
    ↓
代码输出（修改后文件 / PR / 验证结果）
```

### 四大模块职责

| 模块 | 职责 | 类比 |
|------|------|------|
| Planning | 任务分解、步骤排序、判断完成条件 | 大脑 |
| Memory | 维护上下文、项目知识、用户偏好 | 记忆 |
| ToolUse | 封装 OS/ 文件系统 / Git / 网络等能力 | 手脚 |
| Verification | 运行测试、解析编译器输出 | 自我检验 |

---

## 2. Planning 模块——任务拆解与执行控制

### 2.1 Planning 的本质

Planning 本质是一个**受限的自然语言推理循环**。LLM 不需要形式化世界模型，直接用自然语言描述状态和目标，靠常识推理生成动作序列。

### 2.2 ReAct 循环

```
while not done:
    thought = llm.reason(goal, history, context)
    if thought.is_tool_call:
        result = tool.execute(thought.tool, thought.args)
        history.add(observation=result)
    elif thought.is_final_answer:
        done = True
```

### 2.3 任务拆解策略

**自顶向下拆解（Decomposition）**：目标明确但路径不清晰时适用。

```
目标：实现用户登录功能
  → 子目标1：设计数据库 User 表
  → 子目标2：实现后端 Auth API
  → 子目标3：实现前端登录表单
  → 子目标4：串联测试
```

**渐进式揭示（Progressive Elaboration）**：目标本身不清晰时，Agent 先做最小探索再逐步细化。

### 2.4 依赖图与并行

Planning 维护内部依赖图，部分步骤可并行：

```
Step A（读代码） → Step B（理解结构） → Step C（写测试） → Step D（实现功能） → Step E（验证）
                                                    ↑
                                            C 和 D 可并行（如果测试不依赖实现先完成）
```

### 2.5 停止条件

1. **显式完成信号**：输出 "Task completed" + 通过 Verification
2. **最大循环次数**：防止无限循环（如 max_iterations=50）
3. **资源上限**：token 预算或时间预算耗尽

---

## 3. Memory 模块——上下文维护与演化

### 3.1 短期记忆（Session Context）

当前对话窗口内的全部上下文：

- 用户原始指令
- 从磁盘读取的文件内容
- Tool Call 的输入输出
- LLM 的思考过程（Thought）
- Verification 执行结果

### 3.2 长期记忆（Long-term Memory）

跨 Session 存储的信息：

| 类型 | 示例 |
|------|------|
| 项目知识 | CLAUDE.md / AGENTS.md 中的项目规范 |
| 用户偏好 | 编程风格、框架选择、命名约定 |
| 执行历史 | 历史任务总结（避免重复犯错） |
| Auto Memory | 自动从代码库结构和文档中提取的知识 |

### 3.3 分级加载（Tiered Loading）

```
Tier 1（始终加载）  ：System prompt、项目根 CLAUDE.md、未限定的 rules
Tier 2（按需加载）  ：子目录的 CLAUDE.md、限定 path 的 rules
Tier 3（从不自动加载）：旧 Session 总结、过时文档
```

### 3.4 Compact（压缩）

当上下文即将超出限制时执行压缩：

```
压缩前：
- Read(/src/auth/login.py): 300 行代码全文
- Edit(/src/auth/login.py): 修改了 5 行
- Bash(test): 运行了 20 条测试输出

压缩后：
- Read(/src/auth/login.py): 摘要（登录逻辑在 authenticate() 函数）
- Edit: 修改了 authenticate() 的错误处理
- Bash(test): 18 passed, 2 failed（具体失败输出被摘要）
```

### 3.5 CLAUDE.md / AGENTS.md 加载优先级

```
优先级高 → 低：
1. Managed policy（企业 IT 管控，不可覆盖）
2. User：~/.claude/CLAUDE.md
3. Project：仓库根 ./CLAUDE.md
4. Local：CLAUDE.local.md（通常 gitignore）

实践原则：
- 每个文件 < 200 行
- 写"agent 必须遵守的规则"，不写"项目介绍"
- 避免信息腐烂 — 宁缺毋滥
```

---

## 4. ToolUse 模块——工具封装与调用机制

### 4.1 Tool 分类

#### 第一类：基础设施工具（必选）

| Tool | 作用 | 风险等级 |
|------|------|----------|
| Read | 读取文件内容 | 低 |
| Edit | 修改文件 | 中（可回滚） |
| Bash | 执行 Shell 命令 | 高 |
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

标准化的工具发现与调用协议，允许 Agent 连接外部服务（数据库、API、IDE 插件等）。

### 4.2 MCP vs 裸 API

| 维度 | 裸 API | MCP |
|------|-------|------|
| 描述方式 | 自然语言文档 | 标准 JSON Schema |
| 工具发现 | 需预先告知模型 | 连接时自动 list |
| 鉴权 | 各家自己定义 | 标准 OAuth 2.0 |
| 传输 | HTTP 各异 | stdio/HTTP/SSE |
| 复用性 | 每个平台重做 | 一份 server 全网通用 |

### 4.3 Deferred Tools / ToolSearch

**现象**：MCP 工具和部分内置工具的完整 JSON schema 不立刻加载，只把名字告知模型。

**原因**：
1. 大型 MCP server 可能几十上百工具，全部 schema 上万 token
2. 降低注意力噪声
3. 按需加载

**代价**：第一次使用某工具多一次 ToolSearch round trip。

### 4.4 Function Calling 工作流

```
用户输入 → Planning 推理 → 决定调用 Tool → 生成 Tool Call
    ↓
Tool Executor 接收调用 → 执行工具 → 捕获输出
    ↓
Observation 格式化 → 返回给 LLM → 更新 Memory → 下一轮推理
```

### 4.5 ToolCall 失败处理

| 失败类型 | 示例 | 处理策略 |
|----------|------|----------|
| 语法/参数错误 | 参数格式不对 | LLM 自我修正，重新生成 |
| 执行失败 | 文件不存在、权限不足 | 尝试替代路径或告知用户 |
| 超时 | 命令执行超限 | 杀掉进程，尝试简化命令 |
| 部分失败 | 批量操作中部分成功 | 回滚全部，重新规划 |

**重试机制（指数退避）**：
```
max_retries = 3
retry_delay = 1s
for attempt in range(max_retries):
    result = tool.execute(args)
    if result.success: break
    elif is_retryable(result.error):
        sleep(retry_delay * (2 ** attempt))
    else: raise result.error
```

### 4.6 Permission Mode

| 模式 | 行为 | 适合场景 |
|------|------|---------|
| default | 每工具首次询问 | 不熟悉新项目 |
| acceptEdits | 自动放行文件编辑/fs | 熟悉项目加速迭代 |
| plan | 只读，禁止写操作 | code review / 方案设计 |
| auto | 后台分类器判断 | 半自动信任模式 |
| dontAsk | 白名单外一律拒绝 | 严格生产场景 |
| bypassPermissions | 全部放行（保留熔断） | 沙箱/容器内 |

### 4.7 Bash 工具特殊安全措施

1. **权限分级**：从「每次询问」到「全部放行」分多级
2. **白名单限制**：危险命令（rm -rf /）强制要求确认
3. **熔断机制**：单次操作 token 或时间超限直接中断
4. **Dry Run**：危险操作先输出要执行的命令，让用户确认

---

## 5. Sub-agent 与多 Agent 协作

### 5.1 Sub-agent 设计目的

**上下文隔离**：Sub-agent 有独立 context，历史 Tool Call 输出不污染主对话。

**并行处理**：多个无依赖子任务可同时派发给不同 Sub-agent。

### 5.2 Sub-agent 与主 Agent 通信

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

Sub-agent 返回的是**摘要**，不是完整输出。

### 5.3 Sub-agent 关键限制

- **max_depth=1**：Sub-agent 不能嵌套衍生 Sub-agent（防止无限递归）
- **默认不能互相通信**：星型拓扑，所有子 Agent 只与主对话通信
- **Agent Team 实验特性**：开启后可用 `SendMessage(to=<agent_id>, message=...)` 直接互通

### 5.4 星型 vs 网状拓扑

#### 星型拓扑

所有 Sub-agent 只与主 Agent 通信。

**优点**：上下文完全可控、易调试、避免循环对话。

**缺点**：主 Agent 是瓶颈、不适合大量实时协作。

#### 网状拓扑

Sub-agent 之间可直接通信，形成对等网络。

**优点**：接近真实团队协作模式、可以并行形成共识。

**缺点**：上下文可能爆炸、调试困难、容易产生矛盾结论。

**现实答案**：大多数任务星型够用，极少数复杂多角色才上网状。

### 5.5 什么时候用 Sub-agent

**应该用**：
- 读 10+ 文件做调研
- 大量 grep/find/日志分析，输出会很噪
- 独立 code review / 安全 review
- 并行做多个无依赖子任务

**不应该用**：
- 只需要 1-3 个 Tool Call → 主对话直接做更快
- 需要主对话保留思考过程用于后续决策
- 需要多次交互式确认（Sub-agent 不能问用户）
- 任务就是"读一个已知路径" → 直接 Read

### 5.6 Skill vs Sub-agent

| 维度 | Skill | Sub-agent |
|------|-------|-----------|
| 上下文 | 共享主上下文 | 独立上下文 |
| 调用代价 | 低（只是文本注入） | 高（完整推理回合） |
| 适合场景 | "怎么做"的指导、规范 | 大量工具调用、可能产生大量噪声 |
| 结果 | 改变主对话后续行为 | 返回一段总结 |
| 内部子 Agent | 不开新上下文 | 无法再开 sub-agent |

---

## 6. 上下文污染防控

### 6.1 污染来源

| 来源 | 示例 |
|------|------|
| 文件读取污染 | Agent 读了 20 个文件，相关内容还残留在上下文 |
| 长命令输出污染 | Bash 完整输出（数千行）留在上下文 |
| 失败历史污染 | 多个失败的重试记录让 Agent 困在错误路径 |
| Sub-agent 输出污染 | Sub-agent 返回的详细日志直接回传主 Agent |

### 6.2 主动防控策略

#### 输出收敛

```bash
# 不收敛：返回数千行
git log --all --oneline --graph

# 收敛：只返回相关行
git log --oneline -20 -- src/auth/

# 用 grep 二次收敛
git log --oneline | grep -i "login\|auth"
```

#### 精确化工具调用

```bash
# 正确示范：明确指定路径和目标
"需要修改 /src/auth/login.py 中的 authenticate() 函数，
先读这个文件，确认函数签名和返回值格式后再修改。"

# 错误示范：模糊指令
"先了解一下项目结构，然后找到登录相关的代码并修改。"
```

#### 及时 Compact

```
里程碑 checkpoint：
1. 任务理解 ✓ → Compact：丢弃原始任务描述的冗余表述
2. 代码探索 ✓ → Compact：丢弃已读文件的详细内容，只保留摘要
3. 实现完成 ✓ → Compact：丢弃中间失败的编辑尝试记录
```

---

## 7. Verification 与自我修正

### 7.1 为什么 Verification 重要

没有 Verification，Agent 就是"盲人骑瞎马"。Agent 容易"自信地写错"——会编 API/字段/返回值，测试是唯一客观裁判。

### 7.2 TDD for Agent 工作流

```
1. Plan：先写验收测试（明确目标）
2. Implement：根据测试要求实现代码
3. Verify：运行测试，捕获失败信息
4. Fix：基于失败信息修正
5. Loop：回到步骤 3，直到通过
```

### 7.3 关键原则：测试不由 Agent 独自写

Agent 自己写测试再自己实现，可能"为过测试而过测试"。关键测试应由人编写或经过独立 review。

### 7.4 代码修改的原子性与回滚

**原子性**：单次 Edit 要么全部成功（写入文件）要么全部失败（不修改文件）。实现上通常先写临时文件，确认格式正确后再 rename。

**回滚机制**：
1. **Git 工作树备份**：修改前 `git stash`，失败后 `git stash pop`
2. **影子文件**：修改前复制一份 `.bak`，失败后还原
3. **Edit 历史记录**：记录每次 Edit 的 before/after，用于 undo

---

## 8. Hooks 机制

### 8.1 Hook 类型

| 类型 | 触发时机 | 典型用途 |
|------|----------|----------|
| command | Shell 命令执行前后 | 注入环境变量、格式化输出 |
| http | HTTP 请求前后 | 鉴权、脱敏、记录 |
| prompt | 单轮 LLM 调用前后 | 修改 prompt、过滤输出 |
| agent | 多轮 Agent 对话前后 | 注入上下文、总结对话 |
| mcp_tool | MCP 工具调用前后 | 转换参数、处理结果 |

### 8.2 常用场景

1. `PostToolUse` 命中 Edit → 自动跑 prettier/eslint
2. `PreToolUse` 命中 Edit(/.env*) → 阻止误改密钥（exit 2）
3. `Stop` → 发桌面通知
4. `SessionStart` → 输出 git 状态、环境变量
5. `PreToolUse` 命中 Bash(rm -rf *) → ask 确认

---

## 9. 性能与成本优化

### 9.1 Token 成本

每次 LLM 推理按完整 context 的 token 数计费。优化方向：

- **控制上下文长度**：及时 Compact，用收敛工具
- **减少 Tool Call 次数**：批量操作，而非逐个调用
- **选择合适的模型**：简单任务用小模型，复杂推理用大模型

### 9.2 时延优化

Tool Call 每次都有网络往返时延。优化方向：

- **并行 Tool Call**：支持同时发起多个无依赖的 Tool Call
- **缓存**：重复读取同一文件时返回缓存内容
- **预取**：根据推理上下文预测下一个 Tool Call 并预执行

---

## 10. 概念哲学

### 10.1 保持 Context 简单

- **注意力是有限资源**：过多无关信息稀释关键指令权重
- **Token 成本**：每次推理按全部 context 计费
- **可调试性**：短 context 容易定位冲突
- **可演化性**：context 越简单越容易迭代/替换

### 10.2 主动 vs 被动的边界

- **可逆性**：可逆的（本地改文件）自决；不可逆的（push/delete/send）必问
- **影响半径**：只影响本机自决；影响共享系统必问
- **不确定度**：心里没底时应该问，不赌

### 10.3 Sub-agent 反复改不对 bug 时

**信号识别**：Agent 在循环尝试同一类错误解 → 上下文已被污染，继续多半无效。

**沉没成本**：已经烧了多少 token 不重要，重要的是"再烧 X token 能否解决"。

**元问题**：Agent 卡住通常意味着信息缺失而非推理失败 — 应人工补信息（贴日志/文档），不是让它继续猜。

---

*最后更新：2026-05-27*