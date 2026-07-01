# U38 · Context 管理深入（Context Window 实战工程）

> **核心问题**：怎么管理 LLM 的有限 context window——什么时候 `/clear`、怎么 `compaction`、怎么 `rewind`、怎么 context budgeting？LLM performance 随 context 怎么衰减？
> **预计学时**：4h
> **对应 Phase**：Phase 2 / 项目实战
> **状态**：2025 工业级共识（Anthropic、OpenAI、Anthropic Claude Code 都有专门文档）

---

## 1. 核心约束：Context 是有限资源

**Claude Code 官方文档最重要的论断**：

> "Most best practices are based on one constraint: **Claude's context window fills up fast, and performance degrades as it fills**."

**Context 装的是什么**：
- 整个 conversation history（每条 message）
- 每个 file read 的内容
- 每个 command output
- system prompt + tools schema

**Context 怎么快速填满**：
- 一个 debugging session：几万 tokens
- 一个 codebase exploration：可能 50+ 文件 = 100k+ tokens
- 一个 50 轮对话：轻松 200k+ tokens

**Performance 怎么衰减**：
- 0-30% 占用：最佳
- 30-60%：略下降
- 60-90%：明显下降（开始"忘"早期指令）
- 90%+：容易 confused

---

## 2. 6 大 Context 管理工具（按使用频率）

### 2.1 /clear：最简单

```bash
# 清除 conversation history，重置 context
/clear
```

**使用场景**：
- 任务切换（无关任务不要共享 context）
- 修过 2+ 次同一个问题（context 已被错误尝试污染）
- 长时间 session 后想重置

**反模式**：
- 任务**相关**时不要 clear（会丢失重要信息）
- 在长任务中段不要 clear（会丢失推理过程）

### 2.2 /compact：保留关键信息

```bash
# 压缩 context，保留关键信息
/compact

# 自定义压缩指令
/compact "保留所有修改过的文件列表和测试命令"
```

**工作原理**：
1. LLM 总结当前 context
2. 用 summary 替换原 context
3. 保留：file paths、key decisions、test commands、未完成的工作
4. 丢弃：失败的尝试、debug 输出

**vs /clear**：
- /clear = 全部重置
- /compact = 压缩但保留关键信息

### 2.3 Esc + Esc / /rewind：回滚到 checkpoint

```bash
# 在 Claude Code 中按 Esc Esc
/rewind
```

**功能**：
- 弹出回滚菜单
- 可选：恢复 conversation / 恢复 code / 两者都恢复
- 可选：summarize from a selected message

**使用场景**：
- 尝试风险大的修改，发现方向错了
- 想换一种实现方式
- 保留了"试试看"的灵活性

### 2.4 自定义 Compaction

```markdown
# CLAUDE.md
When compacting, always preserve:
- 完整修改过的文件列表
- 测试命令
- 未解决的 bug
- 关键设计决策
```

**OpenAI 也有类似**：`compact` 在 response 中用 `<system>...</system>` 注入总结。

### 2.5 /btw：side questions

```bash
# 问 quick question 不进 conversation history
/btw "这个项目的 license 是什么？"
```

**用途**：快速查 info 不污染主 context。

### 2.6 Context Budget

```python
def build_prompt_with_budget(messages, max_tokens=100_000):
    """手动管理 context budget"""
    
    # 1. 固定部分
    system = messages[0]  # system prompt
    base_tokens = count_tokens(system)
    
    # 2. 剩余预算
    remaining = max_tokens - base_tokens - 500  # 500 token buffer for output
    
    # 3. 分配
    recent_budget = int(remaining * 0.5)  # 最近对话 50%
    rag_budget = int(remaining * 0.3)  # RAG 30%
    summary_budget = int(remaining * 0.2)  # 早期 summary 20%
    
    return assemble(
        system=system,
        recent=truncate(messages[-N:], recent_budget),
        rag=select_top_k(query, rag_budget),
        summary=summarize(messages[:-N], summary_budget),
    )
```

---

## 3. Compaction 的 4 种实现方式

### 3.1 简单 Truncation

```python
# 保留 system + 最近 N 轮
def simple_compact(messages, keep_last_n=10):
    return [messages[0]] + messages[-keep_last_n:]
```

**优点**：快、可预测
**缺点**：丢中间所有信息

### 3.2 滑动窗口 + Summary

```python
def sliding_window_with_summary(messages, window_size=20, summary_budget=2000):
    if len(messages) <= window_size:
        return messages
    
    # 早期：summarize
    early_summary = summarize(messages[:-window_size], budget=summary_budget)
    
    # 最近：原样保留
    recent = messages[-window_size:]
    
    return [
        {"role": "system", "content": f"Earlier summary: {early_summary}"},
        *recent,
    ]
```

### 3.3 Hierarchical Summary（Anthropic 推荐）

```python
def hierarchical_compact(messages, token_budget):
    """多层压缩"""
    if count_tokens(messages) <= token_budget:
        return messages
    
    # 第一层：粗粒度 summary（保留主题）
    coarse = coarse_summarize(messages, target=token_budget // 2)
    
    # 第二层：细粒度（保留关键细节）
    if count_tokens(coarse) > token_budget:
        fine = fine_summarize(messages, focus=extract_key_points(messages))
        return fine
    
    return coarse
```

### 3.4 Token-aware Compaction（Anthropic Claude Code）

```python
# 模仿 Claude Code 的 compaction
def claude_code_compact(messages):
    """保留：file states / decisions / test commands
       丢弃：raw outputs / 失败的尝试"""
    
    # 提取关键信息
    key_info = {
        "modified_files": extract_modified_files(messages),
        "test_commands": extract_test_commands(messages),
        "decisions": extract_decisions(messages),
        "unresolved_bugs": extract_unresolved(messages),
    }
    
    # 重建 messages
    summary = f"""
    Modified files: {key_info['modified_files']}
    Test commands: {key_info['test_commands']}
    Key decisions: {key_info['decisions']}
    Unresolved: {key_info['unresolved_bugs']}
    """
    
    return [
        messages[0],  # system
        {"role": "user", "content": f"Earlier work summary:\n{summary}"},
        *messages[-10:],  # 最近 10 轮
    ]
```

---

## 4. 4 大 Context 优化策略

### 4.1 引用而非包含

**原则**：传 file path 让 LLM 自己读，不要把整个 file 内容塞 context。

```python
# ❌ 差：把整个文件塞 context
context = f"""
File content (5000 lines):
{open("main.py").read()}
"""

# ✓ 好：让 LLM 用工具读
context = "Read the file @main.py using the Read tool"
```

### 4.2 Progressive Disclosure

**原则**：让 LLM 按需读，不是一次性读所有。

```python
# System prompt 设计
system = """
When exploring a codebase:
1. Start with README.md and high-level structure
2. Use Glob to find relevant files
3. Read only the files you need for the current task
4. Don't read all files upfront — be surgical
"""
```

### 4.3 Just-in-Time Retrieval（详见 U22）

```python
# CLAUDE.md 的方式
# 只在 LLM 询问特定文件时才让 tool 读
@tool
def read_file(path: str):
    return open(path).read()  # LLM 主动调用
```

### 4.4 Selective Compaction

```python
# 保留工具调用 + 结果，丢弃推理过程
def selective_compact(messages):
    compacted = []
    for msg in messages:
        if msg.role == "assistant":
            # 保留 tool calls（关键的） + response（短的）
            keep = {"role": "assistant", "tool_calls": msg.tool_calls}
            if msg.content and len(msg.content) < 200:
                keep["content"] = msg.content
            compacted.append(keep)
        else:
            compacted.append(msg)
    return compacted
```

---

## 5. 实战：Context Budget Manager

```python
class ContextBudgetManager:
    def __init__(self, max_tokens=100_000):
        self.max = max_tokens
        self.used = 0
    
    def add(self, content: str, priority: int = 5):
        """priority 1 (highest) to 10 (lowest)"""
        tokens = count_tokens(content)
        if self.used + tokens > self.max:
            # 触发 compaction
            self.compact()
        
        if self.used + tokens > self.max:
            raise ContextOverflow(
                f"Can't fit {tokens} tokens, only {self.max - self.used} available"
            )
        
        self.used += tokens
        return content
    
    def compact(self):
        # 调用 LLM 压缩
        summary = llm.summarize(self.contents)
        # 替换
        self.contents = [summary]
        self.used = count_tokens(summary)
```

---

## 6. Context Window 性能曲线

**经验数据**（来自社区测试）：

| Context 占用 | 性能 | 建议 |
|------------|------|------|
| 0-30% | 100% 性能 | 理想区 |
| 30-60% | 95-100% | 正常区 |
| 60-80% | 85-95% | 警惕 / 开始压缩 |
| 80-95% | 70-85% | 必须压缩 |
| 95%+ | <70% | 清空 + 重新开始 |

**关键指标**：
- **TTFT**（Time To First Token）会随 context 增加（attention 计算 O(n²)）
- **Cost** 线性增加
- **Accuracy** 衰减（特别是需要 recall 早期信息的任务）

---

## 7. Claude Code 的 Context Window 实战技巧

### 7.1 监控 Context

```bash
# Claude Code 自带 statusline 显示 context 使用
/statusline "context: {used}/{max} tokens"

# 输出会显示在每条消息底部
# "context: 45230/200000 tokens (22.6%)"
```

### 7.2 用 CLAUDE.md 控制 Context 增长

```markdown
# CLAUDE.md

## Context 策略
- 不要一次性读所有文件，先 Glob 后 Read
- 大型任务用 /clear 拆分成多个独立 session
- 长 conversation 每 30 分钟 /compact 一次
- 保留最近 5 个修改的文件路径
```

### 7.3 Plan Mode 隔离 Context

```bash
# Plan mode: Claude 只读不写
# 适合：exploration / design / 大规划
# 优势：不污染主 context 的"实现"部分
```

---

## 8. 5 大 Context 反模式

| 反模式 | 后果 | 修复 |
|--------|------|------|
| **一次性全读** | context 爆 | 用 Glob + 按需 Read |
| **从不压缩** | 后半段性能差 | 定期 /compact |
| **错误累积** | 失败尝试堆 context | 失败 2+ 次就 /clear |
| **任务切换不清** | 任务间 context 污染 | 切换任务 /clear |
| **长 conversation 不拆** | 后半段完全失忆 | 大任务拆成多个 session |

---

## 9. 面试视角

- **Q：为什么 context window 是 Agent 工程的核心约束？**
  → LLM attention 是 O(n²)，context 越长延迟越高、成本越高。**更糟的是性能会衰减**——context 填到 80%+ 时 LLM 开始"忘"早期指令、做更多错误。这是 Claude Code 文档明确强调的"the most important resource to manage"。

- **Q：/clear vs /compact 怎么选？**
  → /clear = 全部清空（任务切换时用）。/compact = 压缩但保留关键信息（长 conversation 中段用）。**经验**：2 次失败 /clear；正常长 session /compact。

- **Q：怎么判断 context 满了？**
  → Claude Code statusline 显示 `used/max` 比例。**经验阈值**：60% 开始警惕，80% 必须压缩，95% 必须清空。

- **Q：Context 满了不处理会怎样？**
  → 性能下降、cost 上升、容易 confused。**最严重**：LLM 开始"忘"早期 system prompt，导致违反指令。

- **Q：compaction 的 4 种实现？**
  → ① 简单 truncation（保留最近 N 轮）② 滑动窗口 + summary（早期压缩 + 最近原样）③ Hierarchical（粗 + 细两层）④ Token-aware（保留 file states / decisions / test commands）。

- **Q：怎么写 CLAUDE.md 优化 context？**
  → 明确"context 策略"：让 LLM 用 Glob 后 Read，不要一次全读；定期 /compact；大任务用 /clear 拆分 session。

- **Q：Context Engineering 和 Context Management 的区别？**
  → Context Engineering（U22）= **初始**塞什么进 context（"选什么"）。Context Management（U38）= **运行中**怎么管 context（"压缩 / 清理 / 拆分"）。两者结合：U22 决定起点，U38 决定过程中。

- **Q：生产 agent 怎么监控 context？**
  → ① Track tokens per request（U28 metrics）② 设阈值告警（80% warning）③ 强制 auto-compact（OpenAI / Anthropic SDK 都支持）④ 长 session 自动拆（每 1 小时 /clear + summary 到 handoff doc）。

---

## 10. 必读材料

| 材料 | 链接 | 为什么读 |
|------|------|----------|
| **Anthropic: Effective context engineering** | [anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 上下文腐烂 + 三大 long-horizon 技术 |
| **Claude Code: Best practices** | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices) | Context 是"最重要资源" |
| **Claude Code: Statusline** | [code.claude.com/docs/en/statusline](https://code.claude.com/docs/en/statusline) | 监控 context |
| **Claude Code: Checkpointing** | [code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing) | rewind + summary 机制 |
| **Chroma: Context Rot Research** | [research.trychroma.com/context-rot](https://research.trychroma.com/context-rot) | "context 越多性能越差"的实验数据 |

---

## 11. 练习

1. **监控自己的 context**：在 Claude Code 跑 /statusline，观察 context 增长
2. **触发 compaction**：跑 50 轮对话，触发 /compact，对比压缩前后的输出
3. **/clear 测试**：连续跑 3 个不相关任务，每个用 /clear，对比不 /clear 的 context 残留
4. **自定义 compaction prompt**：写 CLAUDE.md "When compacting, always preserve X"，观察 compaction 结果
5. **手工 context budget**：写一个脚本分配 100k context 到 system / rag / recent / summary
6. **观察 context rot**：用同一个 agent 在 30% / 60% / 90% context 跑同样任务，对比 accuracy

---

## 12. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U22 Context Engineering | U22 决定"塞什么"，U38 决定"怎么管理" |
| U28 LLM Inference | Context 影响 latency / cost（O(n²)）|
| U29 Agent Deploy | 生产 context 监控 + 自动 compaction |
| U35 AgentOps | Context metrics 是 U35 的关键指标 |
| U37 Subagents | Subagent 是 context 隔离的核心机制 |

---

*最后更新：2026-06-30 新增 U38（Context 管理深入）*
*来源：[Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) · [Claude Code: Best practices](https://code.claude.com/docs/en/best-practices) · [Claude Code: Checkpointing](https://code.claude.com/docs/en/checkpointing) · [Chroma: Context Rot Research](https://research.trychroma.com/context-rot)*
