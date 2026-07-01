# U37 · Subagents 与任务隔离（Sub-agents 模式）

> **核心问题**：怎么用 subagent 模式**隔离 context、并行处理大任务、做对抗式 review**，让单个 agent 不被长任务撑爆？
> **预计学时**：3h
> **对应 Phase**：Phase 3（项目实战）/ Phase 4
> **状态**：2025-2026 工业级标准（Claude Code / Codex / OpenHands / Cursor 全部使用）

---

## 1. 什么是 Subagent

**Subagent** = 一个**独立 context + 独立 tool set + 独立 model**的子 agent，由主 agent 委派任务。

**vs 普通 Multi-Agent**：
- **Multi-Agent**（U9）：多个**对等**的 agent 协作（如 CrewAI / AutoGen）
- **Subagent**：一个**主从**的层级关系，主 agent 把任务"外包"给子 agent

**Subagent 的核心价值**：
- **Context 隔离**：子 agent 读 100 个文件不会污染主 agent 的 context
- **专业化**：子 agent 可以用不同的 model（如 security reviewer 用 Opus，普通 agent 用 Sonnet）
- **可观测性**：子 agent 的 trace 独立，调试更容易
- **权限隔离**：子 agent 可以限制 tool（只 Read，不 Write）

---

## 2. 4 大使用场景

### 2.1 Codebase 探索（最常见）

```python
# 主 agent 不知道要不要读 50 个相关文件
# → 用 subagent 探索，结果摘要回主 agent

def main_agent(task):
    if needs_codebase_exploration(task):
        # 委派给探索型 subagent
        findings = exploration_subagent(task)
        return main_agent_with_context(task, findings)
```

**优势**：主 agent 的 context 不被 50 个文件塞满。

### 2.2 Writer / Reviewer 对抗模式

```python
# Writer 写代码
writer_output = writer_subagent(task)

# Reviewer 在新 context 里审查（不被 writer 的"自我感觉良好"影响）
review = reviewer_subagent(
    f"Review this code:\n{writer_output}\nLook for: edge cases, security, performance"
)

if review.has_issues:
    writer_output = writer_subagent(f"Fix these issues:\n{review.issues}")

return writer_output
```

**核心优势**：Reviewer 是**fresh context**——不会因为看到 writer 的推理过程就"同意"它。

### 2.3 Verification / 测试

```python
# 主 agent 写完代码
code = main_agent(task)

# Verification subagent 跑测试
verdict = verifier_subagent(f"Run tests on:\n{code}")
if not verdict.passed:
    code = main_agent(f"Fix failing tests:\n{verdict.failures}")
```

### 2.4 Specialized Roles

```python
# 定义多个 subagent，每个有专门的 system prompt + tool set
agents = {
    "security_reviewer": SubAgent(
        system="You are a senior security engineer. Look for: SQL injection, XSS, auth flaws, secrets.",
        tools=[Read, Grep, Glob, Bash],
        model="opus",
    ),
    "test_writer": SubAgent(
        system="You are a test engineer. Write comprehensive tests for the given code.",
        tools=[Read, Write, Bash],
        model="sonnet",
    ),
    "doc_writer": SubAgent(
        system="You are a technical writer. Update README based on code changes.",
        tools=[Read, Write],
        model="haiku",
    ),
}
```

---

## 3. 主流框架实现

### 3.1 Claude Code Subagents

**Claude Code 是 subagent 模式的最佳实践**。

定义 subagent（`.claude/agents/security-reviewer.md`）：

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior security engineer. Review code for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure data handling

Provide specific line references and suggested fixes.
```

**使用方式**：
- **自动委派**：Claude 看到任务匹配 subagent description 时自动用
- **显式委派**：`Use a subagent to review this code for security issues.`

**关键设计**：
- **独立的 tools**：security-reviewer 只有 Read/Grep/Glob/Bash，不能 Write（防止误改）
- **独立的 model**：可以用更强的 Opus 处理 security 任务
- **独立的 system prompt**：专门的视角

### 3.2 LangGraph Subgraphs

LangGraph 的 subgraph = subagent：

```python
from langgraph.graph import StateGraph, START, END

# 主 graph
main_builder = StateGraph(MainState)
main_builder.add_node("orchestrator", orchestrator)
main_builder.add_node("research", research_subgraph)  # ← subgraph
main_builder.add_edge(START, "orchestrator")
main_builder.add_edge("orchestrator", "research")
main_builder.add_edge("research", END)

# Subgraph：独立的 state 和 nodes
def research_subgraph(state):
    sub_builder = StateGraph(ResearchState)
    sub_builder.add_node("search", search_node)
    sub_builder.add_node("summarize", summarize_node)
    sub_builder.add_edge(START, "search")
    sub_builder.add_edge("search", "summarize")
    sub_builder.add_edge("summarize", END)
    return sub_builder.compile().invoke({"query": state["query"]})
```

**关键**：subgraph 有**独立的 state schema**，不会污染主 graph 的 state。

### 3.3 OpenAI Agents SDK

```python
from agents import Agent, Runner

specialist = Agent(
    name="researcher",
    instructions="Research the given topic thoroughly and report findings.",
    tools=[web_search_tool, read_url_tool],
)

# 主 agent 用 specialist 作为 tool
orchestrator = Agent(
    name="orchestrator",
    instructions="Use the researcher subagent for research tasks.",
    tools=[
        specialist.as_tool(
            tool_name="research",
            tool_description="Research a topic in depth",
        )
    ],
)
```

**亮点**：`Agent.as_tool()` 自动支持 HITL approval（U33 杀手特性）。

### 3.4 OpenAI Swarm

Swarm 用 **handoff** 而不是 subagent：

```python
from swarm import Swarm, Agent

triage_agent = Agent(
    name="Triage",
    instructions="Determine which specialist should handle the request",
)
sales_agent = Agent(name="Sales", instructions="Handle sales questions")
support_agent = Agent(name="Support", instructions="Handle support questions")

triage_agent.functions.append({
    "type": "handoff",
    "destination": sales_agent,  # 或 support_agent
})
```

---

## 4. Subagent 的 4 大设计原则

### 4.1 明确边界

**原则**：subagent 应该有**明确的 scope**（不能"什么都做"）。

```python
# ❌ 错误：subagent 啥都做
general_subagent = SubAgent(
    system="You are a helpful assistant",
    tools=[...all_tools],
)

# ✓ 正确：subagent 专门化
security_subagent = SubAgent(
    system="You ONLY review for security. Don't write code.",
    tools=[Read, Grep, Glob],  # 没有 Write
)
```

### 4.2 独立 context

**原则**：subagent 不应该**看到**主 agent 的所有 context（避免污染）。

```python
# ✓ 主 agent 只传必要的 context 给 subagent
subagent_input = {
    "task": "review this code",
    "code": main_agent.selected_files,  # 只传相关 files
    # 不传主 agent 的完整 conversation history
}
```

### 4.3 结果摘要回主

**原则**：subagent 完成后返回**摘要**，不是 raw output。

```python
result = subagent.run(task)
summary = summarize(result)  # 100 tokens
# 而不是返回 10000 tokens 的 raw output
return main_agent.update({"context": summary})
```

### 6.4 失败传递

**原则**：subagent 失败要明确传递给主 agent。

```python
result = subagent.run(task)
if not result.success:
    return main_agent.update({
        "error": f"subagent {subagent.name} failed: {result.error}",
        "retry_suggestion": "Try different approach or escalate to human",
    })
```

---

## 5. Subagent vs Skill

| 维度 | Subagent | Skill（U26）|
|------|----------|------------|
| **Context** | **独立** context | 共享主 agent context |
| **Tool set** | 可独立 | 共享 |
| **Model** | 可独立（opus / sonnet / haiku）| 共享 |
| **调用方式** | 委派（主从）| 加载（同一 context）|
| **适用** | 大任务、隔离、专业化 | 知识加载、流程规范 |
| **典型场景** | 探索 50 个文件 | 写 code 时遵守公司 style |

**实战组合**：
- Subagent = "派一个实习生做完整任务"
- Skill = "派一个实习生前，给他一份操作手册"

---

## 6. 并行 Subagent 模式

```python
import asyncio

async def parallel_research(topics: list[str]) -> dict:
    """并行跑多个 subagent 加速"""
    tasks = [research_subagent.async_run(topic) for topic in topics]
    results = await asyncio.gather(*tasks)
    return dict(zip(topics, results))

# 主 agent 用法
results = await parallel_research(["Python async", "Rust ownership", "Go goroutines"])
```

**适用**：
- 多个独立研究任务
- 大规模 file analysis
- 多角度 review（security / performance / maintainability 并行）

---

## 7. 实战：Writer/Reviewer 模式

```python
class WriterReviewerLoop:
    def __init__(self, max_iterations=3):
        self.writer = writer_subagent
        self.reviewer = reviewer_subagent
        self.max_iterations = max_iterations
    
    def run(self, task):
        for i in range(self.max_iterations):
            # Writer 工作
            code = self.writer.run(task)
            
            # Reviewer 审查（fresh context）
            review = self.reviewer.run(
                f"Review this code for correctness, security, performance:\n{code}"
            )
            
            if review.approved:
                return code
            
            # 把 review 反馈给 writer
            task = f"{task}\n\nPrevious attempt had these issues:\n{review.feedback}\nPlease fix and try again."
        
        raise Exception("Writer couldn't satisfy reviewer after max iterations")
```

**关键**：Reviewer 每次都看到**新的 context**（不带前一次 review 的偏见）。

---

## 8. 实战：Security Reviewer Subagent

```markdown
# .claude/agents/security-reviewer.md
---
name: security-reviewer
description: Reviews code for security vulnerabilities. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior security engineer reviewing code for a production system.

## Focus Areas
1. **Injection**: SQL injection, XSS, command injection, LDAP injection
2. **Auth/Authz**: Missing authentication, broken access control, privilege escalation
3. **Secrets**: Hardcoded credentials, API keys in code, .env in git
4. **Data exposure**: Logging sensitive data, returning PII in errors
5. **Cryptography**: Weak algorithms (MD5, SHA1), weak key sizes, hardcoded IVs
6. **Dependencies**: Known vulnerable packages, supply chain risks

## Workflow
1. Read changed files via `git diff main..HEAD` or list of touched files
2. Grep for common vulnerability patterns:
   - `eval(`, `exec(`, `os.system(`
   - `subprocess.*shell=True`
   - Raw SQL with user input
3. Check for hardcoded secrets: `password = "..."`, `api_key = "..."`
4. Review auth/authz logic carefully
5. Output findings as:
   - **Critical**: must fix before merge
   - **Warning**: should fix
   - **Nit**: nice to fix

For each finding, provide:
- File:line reference
- Description of vulnerability
- Suggested fix (code snippet)

If you find no issues, say so explicitly. Don't fabricate findings.
```

**使用**：
```
"Use a subagent to review this code for security issues."
```

---

## 9. 实战：并行 Multi-Subagent Review

```python
# 用 subagent 并行做多角度 review
async def comprehensive_review(diff: str) -> ReviewResult:
    reviewers = {
        "security": security_subagent,
        "performance": performance_subagent,
        "maintainability": maintainability_subagent,
        "tests": test_coverage_subagent,
    }
    
    # 并行跑
    tasks = [agent.run(diff) for agent in reviewers.values()]
    results = await asyncio.gather(*tasks)
    
    return ReviewResult(
        security=results[0],
        performance=results[1],
        maintainability=results[2],
        tests=results[3],
    )
```

**好处**：4 个 subagent 并行，4x 加速；每个 reviewer 是 fresh context，偏见少。

---

## 10. 面试视角

- **Q：什么是 subagent？和 multi-agent 有什么区别？**
  → Subagent 是主从层级，主 agent 委派任务给子 agent，子 agent 有**独立 context + tool set + model**。Multi-agent（U9）是对等协作。CrewAI / AutoGen 是 multi-agent；Claude Code subagents / LangGraph subgraphs 是 subagent。

- **Q：Subagent 的 4 大使用场景？**
  → ① Codebase 探索（context 隔离）② Writer/Reviewer 对抗（fresh context）③ Verification（独立测试）④ Specialized roles（不同 model 不同 tool）。

- **Q：Subagent vs Skill 怎么选？**
  → Subagent = **独立 context + 独立 tool + 独立 model**——大任务、隔离、并行。Skill = **共享 context**——知识加载、流程规范。**实战组合**：先 Skill 加载手册，再 Subagent 派任务。

- **Q：怎么用 Claude Code 定义 subagent？**
  → 在 `.claude/agents/` 创建 `name.md`，frontmatter 写 `name / description / tools / model`，body 写 system prompt。Claude 看到匹配 description 的任务时自动用，或显式 `Use a subagent to ...`。

- **Q：Writer/Reviewer 模式为什么有效？**
  → 因为 Reviewer 是 **fresh context**——不会因为看到 Writer 的推理过程就"同意"它。Self-bias 是 LLM agent 的常见失败模式，subagent 隔离是有效对策。

- **Q：Subagent 的设计原则？**
  → ① 明确边界（专门化，不要"什么都做"）② 独立 context（只传必要信息）③ 结果摘要回主（不要 raw 10000 tokens）④ 失败传递（明确告知主 agent）。

- **Q：Subagent 失败了怎么办？**
  → 三层处理：① Retry with different approach（同 subagent 不同 prompt）② 委派给其他 subagent（fallback）③ 上报主 agent，由主 agent 决定（回退 / 升级 / 终止）。

- **Q：怎么 debug subagent？**
  → Claude Code：subagent 的 trace 独立显示，能看每一步。LangGraph：subgraph state 单独 checkpoint。OpenAI Agents SDK：trace 包含 subagent 的工具调用。

---

## 11. 必读材料

| 材料 | 链接 | 为什么读 |
|------|------|----------|
| **Claude Code: Subagents** | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents) | 工业级 subagent 实践 |
| **Claude Code: Best practices** | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices) | "Use a subagent to investigate" 模式 |
| **LangGraph: Subgraphs** | [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/concepts/subgraphs/) | LangGraph 官方 subgraph 文档 |
| **OpenAI Agents SDK: Agents as tools** | [openai.github.io/openai-agents-python/agents](https://openai.github.io/openai-agents-python/agents/) | `Agent.as_tool()` 模式 |
| **Anthropic Building effective agents** | [anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents) | "Orchestrator-workers" 模式 |

---

## 12. 练习

1. **定义一个 security-reviewer subagent**：参考 §8 模板，创建 `.claude/agents/security-reviewer.md`
2. **Writer/Reviewer loop**：用 §7 的代码跑一个实际任务，观察 reviewer 怎么改进 writer
3. **并行 review**：用 §9 的代码并行跑 security + performance + tests
4. **Subagent trace 调试**：在 Claude Code 跑 subagent 时，看 trace 怎么显示子 agent 的工作
5. **LangGraph subgraph**：把现有 LangGraph agent 的一个 node 拆成 subgraph，验证 context 隔离
6. **Permission 隔离**：给 subagent 只 Read 权限，验证它不能 Write

---

## 13. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U9 Multi-Agent | Subagent 是 multi-agent 的特例（主从层级）|
| U22 Context Engineering | Subagent 是 Context 隔离的核心机制 |
| U23 Harness Engineering | Subagent 是 Harness 组件 4（Task State）的隔离实现 |
| U26 Agent Skills | Subagent 和 Skill 是互补的（独立 vs 共享）|
| U31 Structured Output | Subagent 输入输出应该结构化（Instructor）|
| U33 Human-in-the-loop | Subagent 输出可走 HitL approval（U33 §4.2 OpenAI nested agent）|
| U35 AgentOps | Subagent 的 trace 独立收集（U35 trace-based eval）|
| U40 Verification / Adversarial Review | Subagent 是 verification 的核心模式（Writer/Reviewer）|

---

*最后更新：2026-06-30 新增 U37（Subagents 与任务隔离）*
*来源：[Claude Code Subagents](https://code.claude.com/docs/en/sub-agents) · [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices) · [LangGraph Subgraphs](https://langchain-ai.github.io/langgraph/concepts/subgraphs/) · [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/agents/) · [Anthropic Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)*
