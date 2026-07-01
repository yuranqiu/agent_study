# U7 · Planning 与 Reasoning

> **核心问题**：Agent 怎么拆解复杂任务？
> **预计学时**：5h
> **对应 Phase**：Phase 2

---

## 学习目标（Gate 检验）

1. 能解释 GSM-SF（Go Step by Step）任务分解的思路
2. 能比较 CoT / ToT / PoT 三种规划方法的适用场景
3. 能实现一个简单问题的子目标分解
4. 能说清楚状态回溯和路径恢复的区别

---

## 1. 为什么要 Planning

复杂任务（如"帮我写一个完整的博客」）不是一步能完成的，需要：
- 拆解成可执行的子任务
- 识别任务间的依赖关系
- 处理失败和回退

没有 Planning 的 Agent 只能做简单指令，遇到复杂任务要么直接失败，要么走偏。

---

## 2. GSM-SF：最基础的任务分解

### 全称

**G**o **S**tep **by **S**tep **F**orward

### 思路

把复杂问题拆成"第一步做什么，第二步做什么..."，顺序执行。

### 示例

```
问题：计算 (3 + 5) * 2 - 4

GSM-SF 分解：
Step 1: 计算 3 + 5 = 8
Step 2: 计算 8 * 2 = 16
Step 3: 计算 16 - 4 = 12
Answer: 12
```

### 适用场景

- 数学多步计算
- 明确的流程性任务（填表、申请流程）
- 步骤之间有明确顺序

---

## 3. ToT：Tree of Thoughts

### 思路

不是单一路径，而是探索多条可能的路径，取最优。

```
问题：需要优化一个产品的用户留存率

ToT 探索：
路径A：增加用户激励（折扣、积分）→ 短期有效，长期成本高
路径B：改进产品体验（UI、流程）→ 长期有效，短期慢
路径C：个性化推荐 → 需要数据积累

评估：路径B 最优 → 继续深挖 → 子问题：具体改哪些流程？
```

### 与 GSM-SF 的区别

| 方法 | 探索方式 | 适用场景 |
|------|---------|---------|
| GSM-SF | 单链顺序 | 步骤清晰、顺序确定 |
| ToT | 多叉树搜索 | 需要探索、比较、决策 |

---

## 4. PoT：Program of Thoughts

### 思路

让 LLM 生成可执行代码（而非自然语言推理步骤），通过代码执行得到结果。

```python
# 问题：某商品原价 100 元，打 8 折后再减 20 元，最终价格是多少？

# PoT 思路：让模型写出代码
def calculate():
    original = 100
    discount = original * 0.8  # 8折
    final = discount - 20
    return final

result = calculate()  # 执行代码得到结果
```

### 适用场景

- 需要精确计算的数学问题（避免 LLM 数字计算错误）
- 结构化数据处理（表格、JSON 操作）
- 需要确定性结果的场景

---

## 5. 子目标缓存与状态回溯

### 子目标缓存

把已经分解好的子目标存起来，避免重复分解。

```python
sub_goals = {
    "写博客": ["确定主题", "列提纲", "写正文", "检查校对"],
    "数据分析": ["加载数据", "清洗数据", "特征工程", "建模", "评估"],
}
```

### 状态回溯（Backtracking）

当某个路径失败时，回到上一个分支点，尝试其他路径。

```
ToT 树：
                    根节点
                   /      \
              路径A        路径B
             /    \         |
         A1失败   A2成功    B1成功
                    ↓
              继续深挖A2
```

### 失败恢复策略

| 策略 | 适用场景 |
|------|---------|
| 重试（Retry）| 临时性错误（网络超时等）|
| 回退（Backtrack）| 路径选择错误 |
| 降级（Fallback）| 子任务完全失败，用简化方案 |

---

## 6. 实现：简单 Planning Agent

```python
import json
import openai

client = openai.OpenAI()

PLANNER_PROMPT = """你是一个任务规划器。用户提出一个复杂任务，你需要：
1. 判断是否需要分解（简单任务直接回答）
2. 如果需要分解，把任务拆成 3~7 个可执行的子步骤
3. 用 JSON 格式输出子步骤

输出格式：
{
  "need_plan": true/false,
  "steps": ["步骤1", "步骤2", ...]
}

注意：
- 每个步骤要是可执行的具体动作，不能是模糊目标
- 步骤之间有顺序时标注依赖关系
"""

def plan_task(task: str) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": PLANNER_PROMPT},
            {"role": "user", "content": task}
        ],
        temperature=0,
    )
    return json.loads(resp.choices[0].message.content)

# 示例
result = plan_task("帮我写一篇关于 AI Agent 的博客")
print(json.dumps(result, ensure_ascii=False, indent=2))
```

---

## 完成 Gate 检验

1. **Q1**：GSM-SF 和 ToT 的核心区别是什么？各适合什么场景？
2. **Q2**：PoT 相比 CoT 在数学问题上的优势是什么？
3. **Q3**：什么是子目标缓存？为什么有用？
4. **Q4**：Backtracking 和 Retry 的区别是什么？各在什么场景用？

---

## 练习清单

- [ ] 对一个多步问题（如"帮我准备一场面试"）做 GSM-SF 分解
- [ ] 实现一个 ToT 风格的多路径探索（简单问题，人肉设定路径即可）
- [ ] 用 PoT 思路写代码解决一个数学问题，对比直接 CoT 的结果

---

## 关联资源

- [Tree of Thoughts 论文](https://arxiv.org/abs/2305.10601)
- [Self-Consistency 论文](https://arxiv.org/abs/2203.11171)
- [LangChain Planning](https://python.langchain.com/docs/concepts/planning/)

---


## 11. 与本仓库其他单元的关系

|关联单元|关系|
||---|---|
|U1 LLM 基础|Planning 依赖 LLM 的推理能力，U1 的 next-token prediction 是基础|
|U5 ReAct 范式|ReAct 是轻量 planning，U7 的 ToT 是更复杂的规划|
|U9 Multi-Agent|Multi-Agent 的 planner 角色依赖 U7 的规划技术|
|U21 Loop Engineering|Planning 是 Loop 中的决策阶段，U21 讲怎么把规划嵌入循环|
|U27 Goal Engineering|Goal 分解是 Planning 的第一步，U27 深入目标工程|
*最后更新：2026-05-26*
---

## 7. Plan-and-Solve 范式（[Wang 2023](https://arxiv.org/abs/2305.04091)）

**Plan-and-Solve** = 把 ReAct 的"边想边做"改成"先规划后执行"。

```
1. Planner：把问题拆成步骤
2. Executor：按步骤执行
3. Re-planner：如果失败 → 重新规划
```

### 7.1 ReAct vs Plan-and-Solve

| 维度 | ReAct | Plan-and-Solve |
|------|-------|----------------|
| **思考节奏** | Thought-Action 交替 | Plan once, then Execute |
| **灵活性** | 高（可中途改）| 中（plan 后改成本高）|
| **稳定性** | 中（容易走偏）| 高（先想清楚）|
| **计算成本** | 中（多步推理）| 低（plan 一次）|
| **适用** | 探索性任务 | 明确目标的任务 |

### 7.2 Plan-and-Solve 实战代码

```python
class PlanAndSolveAgent:
    def __init__(self, llm):
        self.llm = llm

    def plan(self, task):
        prompt = f"""任务：{task}

请制定一个详细的执行计划。每个步骤应该是可执行的、单原子操作。
格式：
步骤 1: [具体动作]
步骤 2: [具体动作]
...
最终答案

只输出计划，不要解释。"""
        response = self.llm.invoke(prompt)
        # Parse: 提取步骤列表
        return [line.strip() for line in response.split("\n") if line.strip().startswith("步骤")]

    def execute_step(self, task, plan, step_index, history):
        prompt = f"""任务：{task}
计划：{plan}
已完成：{history}
当前步骤：{plan[step_index]}

执行这个步骤并输出结果。"""
        return self.llm.invoke(prompt)

    def run(self, task):
        plan = self.plan(task)
        history = []
        for i, step in enumerate(plan):
            result = self.execute_step(task, plan, i, history)
            history.append(f"步骤 {i+1}: {result}")
        return "\n".join(history)
```

---

## 8. LLM Compiler（[ICLR 2024](https://arxiv.org/abs/2312.04511)）

**LLM Compiler** = 把 agent 规划拆成 **Planning + Task Fetching Unit (TFU) + Executor** 三层。

### 8.1 核心思想

**核心洞察**：很多 agent 任务**没有依赖关系**（可以并行）。LLM Compiler 用**依赖图**调度并行执行，速度比 ReAct 快 3.5x。

```
传统 ReAct：
Step 1 (3s) → Step 2 (3s) → Step 3 (3s) = 9s

LLM Compiler（并行）：
Step 1 (3s) → Step 2, 3 并行 (3s) = 6s（节省 33%）
```

### 8.2 4 步流程

```
1. Planner：把 task 拆成 DAG（有向无环图）
2. Task Fetching Unit (TFU)：调度 ready 节点
3. Executor：跑 ready 节点（可并行）
4. Rephraser：失败时重规划
```

### 8.3 实战代码

```python
import asyncio
from collections import defaultdict

class LLMCompilerAgent:
    def __init__(self, llm):
        self.llm = llm
        self.results = {}  # step_id → result

    def plan(self, task):
        """返回 DAG：step_id → (description, dependencies)"""
        prompt = f"""任务：{task}

拆成可并行执行的步骤图。格式 JSON：
{{
  "steps": [
    {{"id": 1, "desc": "...", "deps": []}},
    {{"id": 2, "desc": "...", "deps": [1]}},
    {{"id": 3, "desc": "...", "deps": [1]}}
  ]
}}"""
        response = self.llm.invoke(prompt)
        return json.loads(response)

    def is_ready(self, step, completed):
        return all(dep in completed for dep in step["deps"])

    async def execute(self, task, plan):
        dag = {s["id"]: s for s in plan["steps"]}
        completed = set()

        while len(completed) < len(dag):
            # 找出所有 ready 的 step（可并行的）
            ready = [dag[i] for i in dag if i not in completed and self.is_ready(dag[i], completed)]
            if not ready:
                break

            # 并行执行
            tasks = [self.execute_step(task, s) for s in ready]
            results = await asyncio.gather(*tasks)

            for s, r in zip(ready, results):
                self.results[s["id"]] = r
                completed.add(s["id"])

        return self.results
```

**实战价值**：工具调用密集的 agent（multi-step RAG / multi-source research）能省 50%+ 时间。

---

## 9. ADaPT（[ICML 2023](https://arxiv.org/abs/2211.00235)）

**ADaPT** = "As-Needed Decomposition and Planning"。

**核心思想**：**不预先规划所有步骤**——根据执行结果**动态决定**是否需要更多子任务。

```
Task: "找出 2024 年美国 GDP 最高的州"
  ↓ Step 1: search "2024 US state GDP"
  ↓ Result: 给 50 个州 GDP
  ↓ ADaPT: 还要按 GDP 排序吗？要排除掉？根据结果动态决定
  ↓ Step 2: 排序 + 排除（动态）
  ↓ Result: 加州第一
  ↓ Final answer
```

**vs ReAct**：ReAct 一步一步硬想；ADaPT 决定"还需要做什么"。

**vs Plan-and-Solve**：Plan-and-Solve 一开始就规划好；ADaPT 边执行边规划。

---

## 10. Planning Agent 实战选型

| 任务类型 | 推荐范式 | 理由 |
|---------|---------|------|
| **简单 1-3 步**（查天气）| ReAct | 灵活 |
| **明确多步**（SQL pipeline）| Plan-and-Solve | 稳定 |
| **并行可拆**（多源 research）| LLM Compiler | 速度 |
| **不确定深度**（多跳推理）| ADaPT | 自适应 |
| **代码任务** | ReAct + subagent | debug 友好 |

**实战**：先 ReAct 跑通 → 看哪步慢 → 换 LLM Compiler（并行）/ ADaPT（动态）。
