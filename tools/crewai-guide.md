# Tool Guide · CrewAI

> 覆盖：安装 / 核心概念 / 最小可运行代码 / 常见坑点 / 面试高频问法

---

## 1. 安装

```bash
pip install crewai crewai-tools
```

---

## 2. 核心概念速查

| 概念 | 组成 | 说明 |
|------|------|------|
| **Agent** | role + goal + backstory + tools | 单个角色 |
| **Task** | description + expected_output + agent + context | 任务单元 |
| **Crew** | agents + tasks + process | 团队，定义协作方式 |
| **Process** | sequential / hierarchical | 任务执行流程 |

---

## 3. 最小可运行代码

### 3.1 最小 Crew（串行）

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="研究分析师",
    goal="深入研究 AI 最新趋势",
    backstory="10年市场研究经验",
    verbose=True
)

writer = Agent(
    role="内容编辑",
    goal="撰写清晰准确的文章",
    backstory="资深科技记者",
    verbose=True
)

research_task = Task(
    description="研究 2025 年 AI 发展趋势",
    agent=researcher,
    expected_output="5个关键趋势及分析"
)

write_task = Task(
    description="基于研究报告撰写文章",
    agent=writer,
    expected_output="完整文章",
    context=[research_task]  # 接收前一个任务输出
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential
)

result = crew.kickoff()
print(result)
```

### 3.2 Hierarchical Process（层级）

```python
from crewai import Agent, Crew, Task, Process, LLM

manager = Agent(
    role="项目经理",
    goal="协调团队高效完成任务",
    backstory="资深 PM"
)

developer = Agent(
    role="开发者",
    goal="编写高质量代码",
    backstory="全栈工程师"
)

tester = Agent(
    role="测试工程师",
    goal="确保代码质量",
    backstory="QA 专家"
)

crew = Crew(
    agents=[manager, developer, tester],
    tasks=[dev_task, test_task],
    process=Process.hierarchical,
    manager_llm=LLM(model="gpt-4o-mini")
)
```

### 3.3 自定义 Tool

```python
from crewai.tools import tool

@tool("数据库搜索")
def search_db(query: str) -> str:
    """搜索数据库。args: query - 搜索词"""
    return f"数据库结果: {query}"

analyst = Agent(
    role="数据分析师",
    goal="从数据库获取洞察",
    tools=[search_db]
)
```

### 3.4 带 Memory

```python
researcher = Agent(
    role="研究员",
    goal="研究主题",
    memory=True  # 开启记忆
)
```

---

## 4. Task context（任务依赖）

```python
research_task = Task(description="研究 AI", agent=researcher)
writing_task = Task(
    description="写文章",
    agent=writer,
    context=[research_task]  # 写入任务结果
)
# writer 会收到 research_task 的输出作为额外 context
```

---

## 5. 常见坑点

| 坑 | 说明 | 解法 |
|----|------|------|
| Task 不传 context | 默认 task 之间不共享输出 | 显式加 `context=[previous_task]` |
| Hierarchical 报错 | 没传 `manager_llm` | 必须指定 `manager_llm=LLM(...)` |
| Agent 不执行 Tool | `allow_delegation=True` 时可能外包 | 明确 `allow_delegation=False` |
| 顺序执行乱 | tasks 列表顺序不对 | 确保 `[task1, task2]` 顺序正确 |
| output 是空 | `expected_output` 描述不够具体 | 写清楚输出格式要求 |

---

## 6. 面试高频问法

### Q: CrewAI 核心概念？

→ Agent (role/goal/backstory/tools) + Task (description/expected_output/context) + Crew (agents/tasks/process)

### Q: Sequential 和 Hierarchical 区别？

→ Sequential：固定顺序，前一个 output 做下一个 input
→ Hierarchical：Manager Agent 动态分配任务给 Worker

### Q: Task context 怎么用？

→ 后置任务加 `context=[前置_task]`，会自动接收前置任务输出

### Q: 如何让 Agent 使用多个 Tool？

→ `Agent(..., tools=[tool1, tool2, tool3])`

### Q: CrewAI 的记忆机制？

→ `Agent(memory=True)` 或 `Crew(memory=True)`
→ 存储任务输出和交互上下文，跨任务记忆

---

*最后更新：2026-05-27*