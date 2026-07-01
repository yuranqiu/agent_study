# U35 · AgentOps / Eval 工具链（Agent 工程化评估与运维）

> **核心问题**：怎么用 **Promptfoo / DeepEval / RAGAS / LangSmith** 等工具，把 Agent 的"质量回归"和"持续监控"做成**工程化流水线**？
> **预计学时**：4h
> **对应 Phase**：Phase 3 / 项目实战
> **状态**：2025-2026 新趋势（AgentOps = LLM 时代的 DevOps）

---

## 1. 什么是 AgentOps

**AgentOps** = 把传统 DevOps（CI/CD / 监控 / 可观测）的思路应用到 **LLM 应用 + Agent 系统**。

```
传统 DevOps            AgentOps
─────────────────────────────────────
Unit test           →  Prompt unit test
Integration test    →  Trajectory test
E2E test            →  Agent E2E + Eval
Metrics             →  Quality + Cost + Latency
Logs                →  Trace + LLM call logs
Deploy              →  Prompt + Model + Agent version
Alerting            →  Quality regression alert
```

**核心差异**：传统软件 deterministic（输入→输出固定），LLM 应用**概率性**（同样输入可能不同输出）—— 所以传统 CI/CD 不够用，需要**eval-driven** 的方式。

---

## 2. 主流工具全景

### 2.1 评估框架（Eval Framework）

| 工具 | 定位 | 优势 |
|------|------|------|
| **Promptfoo** | LLM 评估 + red team | 简单 / 声明式 / CI 友好 |
| **DeepEval** | LLM 评估（pytest 风格） | 测试驱动 / 多种 metrics |
| **RAGAS** | RAG 评估 | RAG 专用 / 4 大指标 |
| **LangSmith** | LangChain 生态 trace + eval | 一体化 |
| **Langfuse** | 开源 LLM 观测 + eval | 开源 / 可自部署 |
| **Braintrust** | 团队协作 + eval | 团队友好 |
| **Arize Phoenix** | 可观测 + drift detection | ML 出身 |
| **Patronus** | Production monitoring | 监控导向 |

### 2.2 观测（Observability）

| 工具 | 定位 |
|------|------|
| **LangSmith** | LangChain 官方 |
| **Langfuse** | 开源 + 兼容 OpenTelemetry |
| **Helicone** | LLM gateway + 监控 |
| **Arize Phoenix** | ML 出身 + 可观测 |
| **OpenLLMetry** | OpenTelemetry for LLM |

### 2.3 评测数据集 / 基准

| Benchmark | 场景 |
|-----------|------|
| **SWE-bench** | 编程 Agent |
| **AgentBench** | 综合 Agent |
| **GAIA** | 真实任务 |
| **WebArena** | 浏览器 Agent |
| **OSWorld** | Computer Use |
| **MMLU** | 通用知识 |
| **GSM8K** | 数学 |
| **HumanEval** | 代码 |
| **MT-Bench** | 多轮对话 |

### 2.4 Red Team

| 工具 | 定位 |
|------|------|
| **Promptfoo Red Team** | 自动生成 adversarial prompts |
| **PyRIT（Microsoft）** | 风险识别 toolkit |
| **Garak** | LLM 漏洞扫描 |
| **DeepEval Safety** | 安全 metrics |

---

## 3. Promptfoo（最实用入门）

[Promptfoo](https://github.com/promptfoo/promptfoo) = 声明式 LLM 评估框架，CI 友好。

### 3.1 安装

```bash
npm install -g promptfoo
# 或
pip install promptfoo
```

### 3.2 写第一个 prompt eval

```yaml
# promptfooconfig.yaml
prompts:
  - "你是一个 helpful 助手。\n问题：{{question}}\n回答："
  - |
    你是 helpful 助手。请用 Markdown 格式回答。
    
    问题：{{question}}
    
    回答：

providers:
  - openai:gpt-4o
  - openai:gpt-4o-mini
  - anthropic:claude-3-5-sonnet-20241022

tests:
  - vars:
      question: "什么是 RAG？"
    assert:
      - type: contains
        value: "检索"
      - type: contains
        value: "生成"
      - type: llm-rubric
        value: "回答应该解释 RAG = Retrieval + Augmentation + Generation"
  
  - vars:
      question: "法国的首都是什么？"
    assert:
      - type: equals
        value: "巴黎"
  
  - vars:
      question: "请解释 Transformer"
    assert:
      - type: javascript
        value: "output.length > 100"  # 至少 100 字
      - type: llm-rubric
        value: "回答应该提到 self-attention"
```

### 3.3 运行

```bash
promptfoo eval
# 浏览器看结果
promptfoo view
```

输出 HTML 报告：每个 prompt × provider × test case 的结果。

### 3.4 CI 集成

```yaml
# .github/workflows/eval.yml
name: Prompt Eval
on: [push, pull_request]

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm install -g promptfoo
      - run: promptfoo eval
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      - uses: actions/upload-artifact@v3
        with:
          name: eval-report
          path: promptfoo-report.html
```

### 3.5 Red Team

```bash
promptfoo redteam init
promptfoo redteam run
# 自动生成 100+ adversarial prompts 测试你的 agent
```

---

## 4. DeepEval（pytest 风格）

[DeepEval](https://github.com/confident-ai/deepeval) = 单元测试风格的 LLM 评估。

### 4.1 安装

```bash
pip install deepeval
```

### 4.2 写测试

```python
import pytest
from deepeval import assert_test
from deepeval.metrics import (
    AnswerRelevancyMetric,
    FaithfulnessMetric,
    ContextualRelevancyMetric,
    HallucinationMetric,
)
from deepeval.test_case import LLMTestCase

def test_rag_quality():
    test_case = LLMTestCase(
        input="什么是 RAG？",
        actual_output="RAG = 检索 + 增强 + 生成。",
        expected_output="RAG 通过检索相关文档增强 LLM 的回答。",
        retrieval_context=["RAG（Retrieval-Augmented Generation）..."],
    )
    
    assert_test(test_case, [
        AnswerRelevancyMetric(threshold=0.7),
        FaithfulnessMetric(threshold=0.7),  # 不能有幻觉
        ContextualRelevancyMetric(threshold=0.6),
    ])

def test_no_hallucination():
    test_case = LLMTestCase(
        input="OpenAI 哪年成立？",
        actual_output="OpenAI 成立于 2015 年 12 月。",
        context=["OpenAI was founded in December 2015."],
    )
    assert_test(test_case, [
        HallucinationMetric(threshold=0.5),
    ])

def test_agent_tool_use():
    test_case = LLMTestCase(
        input="北京天气",
        actual_output="北京今天 25°C 晴。",
        tools_called=[{"name": "get_weather", "args": {"city": "北京"}}],
    )
    # 自定义 metric：必须调了 get_weather
    assert "get_weather" in [t["name"] for t in test_case.tools_called]
```

### 4.3 14+ 内置 Metrics

| Metric | 测什么 |
|--------|--------|
| **AnswerRelevancy** | 回答是否切题 |
| **Faithfulness** | 回答是否忠于 context（无幻觉） |
| **ContextualRelevancy** | 检索的 context 是否相关 |
| **ContextualRecall** | 检索是否覆盖了所有必要信息 |
| **ContextualPrecision** | 检索的 context 排名质量 |
| **Hallucination** | 是否有幻觉 |
| **Bias** | 是否有偏见 |
| **Toxicity** | 是否有毒内容 |
| **GEval** | 自定义 LLM-as-judge |
| **Summarization** | 摘要质量 |
| **JsonCorrectness** | JSON 格式正确性 |
| **ToolUse** | 工具调用正确性 |
| **TaskCompletion** | 任务完成度 |
| **ExactMatch** | 完全匹配 |

### 4.4 自定义 Metric

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams

custom_metric = GEval(
    name="Brand Voice",
    criteria="回答应该友好、专业、避免营销话术。",
    evaluation_params=[
        LLMTestCaseParams.INPUT,
        LLMTestCaseParams.ACTUAL_OUTPUT,
    ],
    threshold=0.7,
)

assert_test(test_case, [custom_metric])
```

### 4.5 CI 集成

```bash
deepeval test run test_agent.py
# 返回 exit code 0/1（可被 CI 拦截）
```

---

## 5. RAGAS（RAG 专用评估）

[RAGAS](https://github.com/explodinggradients/ragas) = RAG 评估的事实标准。

### 5.1 4 大核心指标

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# 准备数据
data = {
    "question": ["什么是 RAG？", "..."],
    "answer": ["RAG = 检索增强生成。", "..."],
    "contexts": [["RAG 是..."], ["..."]],
    "ground_truth": ["RAG 通过检索..."],  # 可选
}
dataset = Dataset.from_dict(data)

# 评估
result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)
print(result)
# {'faithfulness': 0.89, 'answer_relevancy': 0.92, ...}
```

### 5.2 在 RAG pipeline 中使用

```python
# 评估自己的 RAG 系统
from my_rag import answer_question

test_cases = [
    {
        "question": "...",
        "ground_truth": "...",
    },
    # ...
]

results = []
for case in test_cases:
    answer, contexts = answer_question(case["question"])
    results.append({
        "question": case["question"],
        "answer": answer,
        "contexts": contexts,
        "ground_truth": case["ground_truth"],
    })

dataset = Dataset.from_list(results)
scores = evaluate(dataset, metrics=[faithfulness, answer_relevancy])
print(scores)
```

---

## 6. LangSmith（LangChain 生态一体化）

[LangSmith](https://www.langchain.com/langsmith) = LangChain 官方的 trace + eval 平台。

### 6.1 核心功能

- **Trace**：每个 LLM call / tool call / chain step 的完整 trace
- **Eval**：在线 / 离线评估
- **Dataset**：管理测试集
- **Annotation**：人工标注
- **Monitoring**：生产监控

### 6.2 用法

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls__..."
os.environ["LANGCHAIN_PROJECT"] = "my-agent"

from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o")

# 自动 trace
response = llm.invoke("hello")
```

所有 LangChain 操作自动上传到 LangSmith。

### 6.3 离线 Eval

```python
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

# 创建测试集
dataset = client.create_dataset("my-test")
client.create_examples(
    inputs=[{"q": "..."}],
    outputs=[{"a": "..."}],
    dataset_id=dataset.id,
)

# 评估
def my_agent(input):
    return llm.invoke(input["q"]).content

results = evaluate(
    my_agent,
    data=dataset.name,
    evaluators=[...],  # 自定义 evaluator
)
```

---

## 7. 实战：完整的 Agent CI/CD Pipeline

```yaml
# .github/workflows/agent-ci.yml
name: Agent CI

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: pip install -e .
      - run: pytest tests/unit/

  prompt-eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm install -g promptfoo
      - run: promptfoo eval
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      - uses: actions/upload-artifact@v3
        with:
          name: promptfoo-report
          path: promptloo-report.html

  deepeval-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: pip install deepeval
      - run: deepeval test run tests/eval/
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

  ragas-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
      - run: pip install ragas
      - run: python tests/eval_rag.py

  redteam:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm install -g promptfoo
      - run: promptfoo redteam run
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

**关键**：每改 prompt / model / agent code，都自动跑：
- Unit test（工具 / 解析器）
- Prompt eval（输出质量）
- DeepEval（trajectory 质量）
- RAGAS（如果改了 RAG）
- Red team（PR 时跑）

---

## 8. 生产监控（Online Monitoring）

### 8.1 在线 sampling eval

```python
# 5% 流量做在线 eval
import random

def process_with_sampling(user_id, input, output):
    if random.random() < 0.05:  # 5% sampling
        # 跑 eval
        score = llm_as_judge(
            f"评估这个 agent 的回答质量（0-1）。\n"
            f"问题：{input}\n回答：{output}\n分数："
        )
        # 上报 metrics
        metrics.histogram("online_quality_score", score)
        
        # 如果分数低，触发告警 + 收集
        if score < 0.5:
            alerts.send("low_quality_response", user_id, input, output)
            collect_to_review_queue(user_id, input, output, score)
```

### 8.2 Drift Detection

```python
# 监控 RAGAS 分数随时间变化
class DriftMonitor:
    def __init__(self):
        self.baseline = {"faithfulness": 0.9, "answer_relevancy": 0.85}
        self.threshold = 0.05  # 5% 下降就告警
    
    def check_drift(self, current_scores):
        for metric, baseline in self.baseline.items():
            if current_scores[metric] < baseline - self.threshold:
                alert(f"Drift detected in {metric}: {current_scores[metric]:.2f} < {baseline - self.threshold:.2f}")
```

### 8.3 Cost / Latency 监控

```python
# U28 的 metrics + U35 的 quality metrics 整合
@dataclass
class AgentMetrics:
    # Performance (U28)
    latency_p50: float
    latency_p95: float
    cost_per_query: float
    error_rate: float
    
    # Quality (U35)
    faithfulness: float  # RAGAS
    answer_relevancy: float
    tool_call_accuracy: float
    task_completion_rate: float
    
    # Safety (U34)
    injection_blocked_rate: float
    pii_removed_rate: float
    action_denied_rate: float
```

---

## 9. 评测集管理（重要但常被忽略）

### 9.1 测试集的版本管理

```
tests/
├── golden/
│   ├── v1.0/
│   │   ├── basic_qa.jsonl         # 50 条
│   │   ├── tool_use.jsonl          # 30 条
│   │   ├── multi_turn.jsonl        # 20 条
│   │   └── edge_cases.jsonl        # 20 条
│   ├── v1.1/
│   │   └── ...                     # 改进后
```

### 9.2 自动扩充测试集

```python
# 用 prod failure 自动生成测试 case
def auto_generate_tests_from_failures():
    # 1. 拉取 prod 失败 case
    failures = db.query("SELECT * FROM agent_runs WHERE user_rating < 3")
    
    # 2. 标"regression case"（避免再犯）
    for f in failures:
        test_case = {
            "input": f.input,
            "expected_output": f.expected_output or None,
            "actual_output": f.actual_output,
            "rating": f.user_rating,
            "category": f.failure_category,
        }
        add_to_golden(test_case)
```

### 9.3 Eval-Driven Development

```
写新 prompt → 跑 prompt eval → 优化 → 跑 deepeval → 优化 → 跑 ragas → ...
                                                            ↓
                                                     达标 → commit
```

**核心**：**没有 eval 集就不要改 prompt**。改完不跑 eval = 赌博。

---

## 10. 工具选型决策

| 场景 | 推荐 |
|------|------|
| **快速 prototype** | Promptfoo（一行 yaml） |
| **CI/CD 集成** | Promptfoo + DeepEval |
| **RAG 专项** | RAGAS |
| **LangChain 项目** | LangSmith（一体化） |
| **开源 / 自部署** | Langfuse + RAGAS |
| **团队协作** | Braintrust |
| **生产监控 + drift** | Arize Phoenix / Langfuse |
| **Red Team / 安全** | Promptfoo red team + PyRIT |

---

## 11. 实战：从 0 到 1 搭建 AgentOps

### Step 1：建评测集

```python
# tests/golden_v1/basic_qa.jsonl
{"input": "什么是 RAG？", "expected": "包含 '检索' 和 '生成'"}
{"input": "退款流程", "expected": "包含 '三步'"}
```

### Step 2：写 Prompt eval

```yaml
# promptfooconfig.yaml
prompts:
  - "..."
providers: [openai:gpt-4o]
tests: tests/golden_v1/
```

### Step 3：写 DeepEval 测试

```python
# tests/test_quality.py
def test_quality():
    assert_test(test_case, [FaithfulnessMetric(threshold=0.8)])
```

### Step 4：CI 集成

```yaml
# .github/workflows/ci.yml
- run: promptfoo eval
- run: deepeval test run
```

### Step 5：生产监控

```python
# 5% sampling + 监控 + 告警
```

### Step 6：持续改进

```
collect failures → augment golden set → re-eval → improve prompt → re-eval
```

---

## 12. 面试视角

- **Q：什么是 AgentOps？和传统 DevOps 区别？**
  → AgentOps = 把 DevOps 思路应用到 LLM 应用。区别：LLM 输出是概率性，传统 CI/CD 不够。**必须 eval-driven**——每个 prompt / model 改动都要跑 eval 集。

- **Q：怎么评估 Agent？**
  → 4 层：① 通用能力（lm-eval-harness）② 任务能力（SWE-bench / AgentBench）③ 业务能力（自建 golden set）④ 用户反馈（A/B test + 评分）。

- **Q：Promptfoo / DeepEval / RAGAS 怎么选？**
  → Promptfoo：声明式、CI 友好、最容易上手。DeepEval：pytest 风格、14+ metrics、可自定义。RAGAS：RAG 专用、4 大指标是事实标准。**生产通常用 Promptfoo + DeepEval + RAGAS 组合**。

- **Q：怎么自动化 Agent 的 regression test？**
  → ① 建 golden test set（50-500 条） ② CI 里跑 promptfoo + deepeval ③ 每次改 prompt/model 都触发 ④ 不达标不允许 merge。

- **Q：怎么监控生产 Agent 的 quality drift？**
  → ① 5% sampling + LLM-as-judge 在线评分 ② RAGAS / DeepEval 离线跑 sampled data ③ 监控 faithfulness / answer_relevancy 趋势 ④ 下降 5% 触发告警。

- **Q：怎么自动扩充测试集？**
  → ① 收集 prod 失败 case（user rating < 3）② 标 regression case 入库 ③ 定期 review ④ 失败 case 多了就 retrain / re-prompt。

- **Q：RAG 怎么评估？**
  → RAGAS 4 大指标：① Faithfulness（回答忠于 context）② Answer Relevancy（回答切题）③ Context Precision（检索排名质量）④ Context Recall（检索覆盖率）。**Faithfulness 是最重要的——直接防幻觉**。

---

## 13. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Promptfoo 文档](https://promptfoo.dev/docs/intro) | 入门首选 |
| [DeepEval 文档](https://docs.confident-ai.com/) | pytest 风格 |
| [RAGAS 文档](https://docs.ragas.io/) | RAG 评估 |
| [LangSmith 文档](https://docs.smith.langchain.com/) | LangChain 生态 |
| [Langfuse 文档](https://langfuse.com/docs) | 开源可自部署 |
| [Arize Phoenix](https://docs.arize.com/phoenix) | ML 出身的可观测 |
| [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | eval 思想 |
| [RAGAS Paper](https://arxiv.org/abs/2309.15217) | RAG 评估论文 |

---

## 14. 练习

1. **Promptfoo 入门**：写 10 条 prompt eval，跑通
2. **DeepEval 入门**：写 pytest 风格的 5 个测试，覆盖 faithfulness / hallucination
3. **RAGAS 实战**：评估自己的 RAG 系统，对比不同 prompt
4. **CI 集成**：把 promptfoo + deepeval 接到 GitHub Actions
5. **Red Team**：用 promptfoo 自动生成 adversarial prompts
6. **Drift Detection**：跑 RAGAS 100 次，观察分数变化
7. **Auto-augment**：从 prod log 提取失败 case 入库

---

## 15. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U10 Agent 评估与安全 | U10 是概念，U35 是工程实现 |
| U12 Agent 评估标准 | U12 是 SWE-bench 等标准，U35 是工具链 |
| U18 Agent 安全与红队 | Red Team 工具是 U18 的工程实现 |
| U23 Harness Engineering | AgentOps = Harness 的"运行"部分 |
| U28 LLM Inference | Cost / Latency 监控是 U28 的延伸 |
| U31 Structured Output | JSON correctness metric 是 U31 的验证 |
| U34 Guardrails | Safety metrics 来自 U34 |

---

*最后更新：2026-06-30 新增 U35（AgentOps / Eval 工具链）*
*来源：Promptfoo / DeepEval / RAGAS / LangSmith / Langfuse 官方文档*
