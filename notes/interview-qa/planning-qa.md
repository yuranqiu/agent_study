# Interview QA · Planning 与推理（U7）

> 来源：U7-Planning与Reasoning + 100-Agentic-AI-Interview-Questions
> 覆盖：GSM-SF / ToT / PoT / Plan-and-Solve / LLM Compiler / ADaPT / 子目标缓存 / 状态回溯

---

## 一、基础概念

### Q: 为什么 Agent 需要 Planning？

**Answer:**
复杂任务（如"帮我写一个完整的博客"）不是一步能完成的，需要：
- 拆解成可执行的子任务
- 识别任务间的依赖关系
- 处理失败和回退

没有 Planning 的 Agent 只能做简单指令，遇到复杂任务要么直接失败，要么走偏。

---

### Q: GSM-SF 是什么？适用场景？

**Answer:**

**G**o **S**tep by **S**tep **F**orward = 最基础的任务分解。

```
问题：计算 (3 + 5) * 2 - 4

GSM-SF 分解：
Step 1: 计算 3 + 5 = 8
Step 2: 计算 8 * 2 = 16
Step 3: 计算 16 - 4 = 12
Answer: 12
```

**适用场景：** 数学多步计算、明确的流程性任务（填表、申请流程）、步骤之间有明确顺序。

---

## 二、规划方法对比

### Q: GSM-SF / ToT / PoT 的核心区别？

**Answer:**

| 方法 | 探索方式 | 适用场景 | 代价 |
|------|---------|---------|------|
| **GSM-SF** | 单链顺序 | 步骤清晰、顺序确定 | 低 |
| **ToT** | 多叉树搜索 | 需要探索、比较、决策 | 高（多 token） |
| **PoT** | 生成代码执行 | 需要精确计算 | 中（代码执行成本） |

**面试追问：** ToT 和 ReAct 的区别？
→ ToT 是模型内部的树搜索（多条候选路径），ReAct 是 Thought-Action 交替循环（外部工具调用）。ToT 更重，ReAct 更灵活。

---

### Q: Plan-and-Solve 范式？

**Answer:**

**Plan-and-Solve** = 把 ReAct 的"边想边做"改成"先规划后执行"。

```
1. Planner：把问题拆成步骤
2. Executor：按步骤执行
3. Re-planner：如果失败 → 重新规划
```

**与 ReAct 对比：**

| 维度 | ReAct | Plan-and-Solve |
|------|-------|----------------|
| 思考节奏 | Thought-Action 交替 | Plan once, then Execute |
| 灵活性 | 高（可中途改） | 中（plan 后改成本高） |
| 稳定性 | 中（容易走偏） | 高（先想清楚） |
| 适用 | 探索性任务 | 明确目标的任务 |

---

### Q: LLM Compiler 并行规划？

**Answer:**

**LLM Compiler** = 把规划拆成 **Planning + Task Fetching Unit (TFU) + Executor** 三层。

**核心洞察：** 很多任务没有依赖关系，可以并行执行。
```
传统 ReAct：
Step 1 (3s) → Step 2 (3s) → Step 3 (3s) = 9s

LLM Compiler（并行）：
Step 1 (3s) → Step 2, 3 并行 (3s) = 6s（节省 33%）
```

**4 步流程：**
1. Planner 生成任务依赖图
2. TFU 把任务分发给 Executor
3. Executor 并行执行无依赖任务
4. Aggregator 汇总结果

---

### Q: ADaPT 动态规划？

**Answer:**

**ADaPT** = **A**daptive **D**ecomposition for **A**gent **P**lanning and **T**ask execution.

核心思想：**边执行边规划**，不是一开始就定死计划。

**vs Plan-and-Solve：**
- Plan-and-Solve：一开始就规划好全部步骤
- ADaPT：执行一步 → 评估 → 决定下一步怎么规划

**适用场景：** 不确定深度、需要自适应调整的任务。

---

## 三、失败恢复

### Q: 子目标缓存与状态回溯？

**Answer:**

**子目标缓存：** 把已经分解好的子目标存起来，避免重复分解。
```python
sub_goals = {
    "写博客": ["确定主题", "列提纲", "写正文", "检查校对"],
    "数据分析": ["加载数据", "清洗数据", "特征工程", "建模", "评估"],
}
```

**状态回溯（Backtracking）：** 当某个路径失败时，回到上一个分支点，尝试其他路径。

| 策略 | 适用场景 |
|------|---------|
| **重试（Retry）** | 临时性错误（网络超时等） |
| **回退（Backtrack）** | 路径选择错误 |
| **降级（Fallback）** | 子任务完全失败，用简化方案 |

---

## 四、实战代码

### Q: 实现一个简单的 Planning Agent？

**Answer:**

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

## 五、面试速查

### 高频题 TOP 10

1. **GSM-SF 和 ToT 的核心区别？**
   → GSM-SF 是单链顺序，ToT 是多叉树搜索。ToT 适合需要探索、比较、决策的复杂任务。

2. **PoT 相比 CoT 的优势？**
   → PoT 生成可执行代码，避免 LLM 数字计算错误。适合数学/结构化数据。

3. **Plan-and-Solve vs ReAct？**
   → Plan-and-Solve 先规划后执行（稳定性高），ReAct 边想边做（灵活性高）。

4. **LLM Compiler 的核心思想？**
   → 用依赖图调度并行执行，无依赖的任务并行跑，速度比 ReAct 快 3.5x。

5. **ADaPT 和 Plan-and-Solve 的区别？**
   → ADaPT 边执行边规划（自适应），Plan-and-Solve 一开始就定死计划。

6. **子目标缓存为什么有用？**
   → 避免重复分解，节省 token 和时间。

7. **Backtracking 和 Retry 的区别？**
   → Backtracking 是路径选择错误（换条路），Retry 是临时性错误（同一路再试一次）。

8. **Planning Agent 的核心组件？**
   → Planner（拆解） + Executor（执行） + Re-planner（失败时重新规划）。

9. **什么时候用 ToT vs ReAct？**
   → ToT：需要多方案比较（如算法设计）。ReAct：需要外部知识/工具调用。

10. **Planning 的常见失败模式？**
    → 目标模糊、依赖识别错误、回溯策略缺失、无限循环。对应：Goal Engineering + 显式依赖图 + 回退策略 + max_iterations。

---

### 面试系统设计题

**Q：怎么设计一个"帮我准备一场面试"的 Planning Agent？**

**Answer 框架：**
1. **Goal 定义**：用户需要准备技术面试（明确 scope）
2. **分解**：
   - Step 1: 了解目标公司/岗位
   - Step 2: 梳理知识盲区
   - Step 3: 生成练习计划
   - Step 4:  mock interview
3. **依赖**：Step 2 依赖 Step 1 的输出
4. **验证**：每个 step 有 acceptance criteria
5. **回退**：如果 mock interview 失败，回到 Step 3 调整计划

---

*最后更新：2026-07-01*
