# Agent 评估面试题

> 来源：U10-Agent评估与安全 + U12-Agent评估标准 + 主流评测框架
> 覆盖：评估指标 / 评测方法 / 工具实践 / CI/CD集成

---

## 一、评估基础

### Q1: Agent 评估为什么比传统软件测试更难？

**Answer:**

| 维度 | 传统软件 | Agent |
|------|---------|-------|
| 输出确定性 | 确定输出 | 非确定输出（同一问题多次回答不同） |
| 执行链路 | 短且确定 | 长链路，多步执行中途可能失败 |
| 正确答案 | 有明确定义 | 开放式任务难以定义"正确答案" |
| 评测方式 | 单元测试 | 需要任务完成率、效率、质量多维度 |

Agent 评估需要新的方法论，不能简单用传统测试思路。

---

### Q2: Agent 评估的三层模型是什么？

**Answer:**

| 层级 | 名称 | 说明 |
|------|------|------|
| L1 | 基础指标 | 原始测量值（准确率、步数、耗时） |
| L2 | 能力维度 | 归一化能力打分（0-100 或 A-E） |
| L3 | 业务价值 | 最终业务收益（成本降低、转化提升） |

单一指标无法全面反映 Agent 能力——只管结果不管过程、实验室指标≠用户感知。

---

### Q3: 核心评估指标有哪些？如何计算？

**Answer:**

| 类型 | 指标 | 公式 |
|------|------|------|
| 任务完成 | Task Completion Rate | `成功数 / 总数 × 100%` |
| 任务完成 | Task Failure Rate | `超时数 / 总数 × 100%` |
| 效率 | Step Efficiency | `实际步数 / 基准步数` |
| 效率 | Time Efficiency | `实际耗时 / 基准耗时` |
| 质量 | Answer Correctness | `精确匹配 / 模糊匹配` |
| 质量 | Answer Relevance | Embedding 余弦相似度 |

**效率比判断**：
- = 1.0：最优
- > 1.0：有冗余步
- < 1.0：使用了捷径（需验证是否正确）

---

### Q4: 错误恢复率怎么计算？

**Answer:**

```
错误恢复率 = 错误后成功完成数 / 总错误数
```

例如：Agent 执行了 10 个任务，中间遇到 3 次错误，最终成功完成 2 个，则恢复率 = 2/3 ≈ 67%

---

## 二、分场景评估标准

### Q5: 对话型 Agent 评估标准？

**Answer:**

| 指标 | 优秀 | 良好 | 及格 | 不及格 |
|------|------|------|------|--------|
| 响应正确率 | ≥95% | ≥85% | ≥70% | <70% |
| 平均响应时间 | <1s | <2s | <5s | ≥5s |
| 对话轮数 | 3-5轮 | 5-8轮 | 8-12轮 | >12轮 |
| 用户满意度 | ≥4.5 | ≥4.0 | ≥3.0 | <3.0 |

---

### Q6: 任务型 Agent 评估标准？

**Answer:**

| 指标 | 优秀 | 良好 | 及格 | 不及格 |
|------|------|------|------|--------|
| 任务完成率 | ≥90% | ≥75% | ≥60% | <60% |
| 步数效率 | ≤1.2 | ≤1.5 | ≤2.0 | >2.0 |
| 错误恢复率 | ≥80% | ≥60% | ≥40% | <40% |
| 工具使用准确率 | ≥95% | ≥85% | ≥70% | <70% |

---

### Q7: 代码型 Agent 评估标准？

**Answer:**

| 指标 | 优秀 | 良好 | 及格 | 不及格 |
|------|------|------|------|--------|
| 代码通过率 | ≥80% | ≥60% | ≥40% | <40% |
| 一次通过率 | ≥60% | ≥40% | ≥20% | <20% |
| 平均修复轮次 | ≤2 | ≤4 | ≤6 | >6 |
| 代码质量评分 | ≥90 | ≥75 | ≥60 | <60 |

---

## 三、主流 Benchmark

### Q8: SWE-bench 的设计思路是什么？为什么它比人工评测更可靠？

**Answer:**

**设计思路**：
1. 收集真实 GitHub Issue + PR patches
2. Agent 需要理解 Issue → 定位代码 → 修改 → 提交
3. 评判标准：修改后的代码能否通过测试

**为什么可靠**：
- 任务真实：不是模拟的，是真实软件工程问题
- 可自动评判：有测试用例，能自动判断对错
- 发现能力边界：多数 Agent 完成率 < 30%

---

### Q9: 常用评测 Benchmark 有哪些？

**Answer:**

| 数据集 | 任务类型 | 特点 |
|--------|---------|------|
| SWE-bench | GitHub Issue 修复 | 真实代码任务，客观可判 |
| AgentGym | 多任务（web/code/tool） | 统一环境，统一评测 |
| WebArena | Web 交互 | 真实网站，多步任务 |
| MiniWob++ | UI 操作 | 简单但可控的 web 任务 |
| ALFWorld | 具身智能（文本） | 家庭任务，交互式 |

---

### Q10: AgentGym 解决什么问题？

**Answer:**
- 之前各 Agent评测环境不统一，无法公平对比
- AgentGym 提供统一环境，同时评测 Web、Code、Tool 等多种任务
- 支持多个 Agent 同台竞技，发现通用能力最强的 Agent

---

## 四、Trace Log 与可观测性

### Q11: 为什么 Agent 需要 Trace Log？

**Answer:**
Agent 决策链路长，出问题后需要能"回放"每一步在做什么。

Trace Log 记录：
- 每一步的 Thought（思考）
- Action（执行的动作）
- Observation（观察结果）

---

### Q12: 如何实现基础的 Trace Log？

**Answer:**

```python
import json
import time

class AgentTrace:
    def __init__(self, task_id: str):
        self.task_id = task_id
        self.steps = []

    def log(self, step_type: str, content: str, metadata: dict = None):
        self.steps.append({
            "step_type": step_type,  # thought / action / observation
            "content": content,
            "timestamp": time.time(),
            "metadata": metadata or {},
        })

    def summary(self):
        for i, step in enumerate(self.steps, 1):
            print(f"[{i}] {step['step_type'].upper()}: {step['content'][:200]}")

# 使用
trace = AgentTrace("task_001")
trace.log("thought", "用户想知道北京天气，需要调用天气工具")
trace.log("action", "get_weather", {"city": "北京"})
trace.log("observation", "晴，22°C")
trace.log("final", "北京今天天气晴，气温 22°C")
trace.summary()
```

---

## 五、评估流程与工具

### Q13: 标准评估流程是什么？

**Answer:**

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 准备评估 │ → │ 执行评估 │ → │ 数据采集 │ → │ 报告生成 │
│   数据   │    │   任务   │    │   指标   │    │   结果   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

### Q14: 常用评估工具有哪些？各自特点？

**Answer:**

| 工具 | 适用场景 | 特点 |
|------|---------|------|
| LangSmith | 全场景 | LangChain 官方，集成方便 |
| Weights & Biases | 实验追踪 | 强大可视化，适合研究 |
| LangFuse | 自托管 | 开源，可私有部署 |
| Phoenix (Arize) | LLM 观测 | 实时追踪，Prompt 追踪 |
| HeliXon | 对比评测 | 多 Agent 对比，自动化 |

---

### Q15: 如何实现 A/B 测试？

**Answer:**

```python
def ab_test(agent_a, agent_b, tasks, n_runs=3):
    results_a = []
    results_b = []
    
    for task in tasks:
        for run in range(n_runs):
            result_a = run_once(agent_a, task)
            result_b = run_once(agent_b, task)
            results_a.append(result_a)
            results_b.append(result_b)
    
    return {
        'agent_a': aggregate(results_a),
        'agent_b': aggregate(results_b),
        'winner': statistical_significance(results_a, results_b)
    }
```

---

## 六、CI/CD 集成

### Q16: 如何将评估集成到 CI/CD？

**Answer:**

```yaml
# .github/workflows/agent-eval.yml
name: Agent Evaluation

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Evaluation
        run: |
          python -m agent_eval \
            --agent ./agent.py \
            --dataset ./eval_data.json \
            --output ./reports/eval_$(date +%Y%m%d).json
      - name: Check Thresholds
        run: |
          python -m agent_eval.check_threshold \
            --report ./reports/eval_*.json \
            --config ./eval_config.yaml
```

---

### Q17: 评估配置示例？

**Answer:**

```yaml
# eval_config.yaml
evaluation:
  name: "production-agent-v1"
  version: "1.2.0"
 
thresholds:
  task_completion_rate: 0.85  # 85%
  step_efficiency: 1.5        # 最多 1.5 倍基准步数
  error_recovery_rate: 0.60   # 60%
  avg_response_time: 2.0      # 秒
 
safety:
  min_security_score: 0.95    # 95%
  block_prompt_injection: true
  block_data_exfiltration: true
```

---

## 七、Human Evaluation

### Q18: 什么时候需要人工评估？

**Answer:**

对于开放式输出（写作、回答），自动化评测困难，用人工评估：

- **正确性**：回答是否准确
- **完整性**：是否覆盖所有要点
- **可读性**：语言是否通顺

---

## 八、关联资源

- [SWE-bench 评测框架](https://arxiv.org/abs/2310.06770)
- [AgentGym 统一评测](https://arxiv.org/abs/2406.12023)
- [LangSmith 官方文档](https://docs.smith.langchain.com/)
- [Helicone 开源可观测性](https://github.com/Helicone/helicone)

---

*最后更新：2026-05-27*
