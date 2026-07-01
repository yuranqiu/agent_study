# U14 · DSPy

> **核心问题**：如何用声明式编程范式构建 LLM 应用？
> **预计学时**：4h
> **对应 Phase**：Phase 2

---

## 学习目标（Gate 检验）

1. 能解释 DSPy 的核心理念（声明式编程 vs 传统 prompt 编程）
2. 能使用 `dspy.ChainOfThought` 和 `dspy.Predict` 构建模块
3. 能用 `dspy.Optimize` 对 prompt 进行自动优化
4. 能区分 DSPy 的 Bootstrap、Metalearning 和 Teleprompter 机制

---

## 1. 为什么需要 DSPy

### 传统 LLM 应用的问题

```
用户：我想要一个能回答问题的 Agent
开发者：好，我写 prompt
用户：效果不好
开发者：调一调 prompt
用户：还是不行
开发者：加几个示例（few-shot）
用户：换个模型试试
开发者：...（崩溃）
```

**痛点**：prompt 是脆弱的字符串，依赖人工调参，难以迁移到新模型。

### DSPy 的核心理念

> "Separation of concerns: tell the model WHAT to do, not HOW to do it"

DSPy 把 LLM 应用开发从"写 prompt"变成"写程序逻辑"，prompt 被隐藏在工作流背后，通过优化器自动生成。

类比：
- 传统方式 ≈ 手工编写 SQL 字符串
- DSPy 方式 ≈ 用 ORM，SQL 由框架生成

---

## 2. 核心概念

### 2.1 Signature（签名）

Signature 定义模块的输入输出行为：

```python
class GenerateAnswer(dspy.Signature):
    """给定一个问题，生成一个准确的答案"""
    context = dspy.InputField(desc="可能的背景信息")
    question = dspy.InputField()
    answer = dspy.OutputField()
```

比 prompt 更结构化：
- 明确的输入/输出字段
- 每个字段有描述（desc）
- 可复用、可组合

### 2.2 Module（模块）

基于 Signature 的可组合单元：

| 模块 | 作用 |
|------|------|
| `dspy.Predict` | 最基础的预测模块 |
| `dspy.ChainOfThought` | 让 LLM 逐步思考 |
| `dspy.SimpleChain` | 串联多个模块 |
| `dspy.Parallel` | 并行执行多个模块 |

```python
# 基础用法
predict = dspy.Predict(GenerateAnswer)
result = predict(context="...", question="...")

# CoT 用法
cot = dspy.ChainOfThought(GenerateAnswer)
result = cot(context="...", question="...")
# result.reasoning 包含思考过程
# result.answer 包含最终答案
```

### 2.3 Teleprompter（优化器）

DSPy 的核心价值——自动优化 prompt：

| 优化器 | 用途 |
|--------|------|
| `BootstrapFewShot` | 用少量示例引导模型 |
| `LabeledFewShot` | 使用人工标注的示例 |
| `MIPRO` | 多任务prompt优化 |
| `Ensemble` | 多模块组合优化 |

```python
# 简单例子
teleprompter = dspy.BootstrapFewShot(metric=your_metric)
optimized_program = teleprompter.compile(
    student=your_program,
    trainset=training_examples
)
```

---

## 3. 实战代码

### 3.1 最小示例：RAG

```python
import dspy

# 1. 定义 Signature
class RAGSignature(dspy.Signature):
    """基于上下文回答问题"""
    context = dspy.InputField(desc="上下文信息")
    question = dspy.InputField()
    answer = dspy.OutputField()

# 2. 写程序逻辑
class RAG(dspy.Module):
    def __init__(self):
        super().__init__()
        self.generate = dspy.ChainOfThought(RAGSignature)
    
    def forward(self, question, retrieved_contexts):
        context = "\n".join(retrieved_contexts)
        return self.generate(context=context, question=question)

# 3. 使用
rag = RAG()
response = rag(
    question="中国的首都是哪里？",
    retrieved_contexts=["北京是中国的首都"]
)
print(response.answer)
```

### 3.2 自动优化 Prompt

```python
import dspy

# 假设有训练数据
trainset = [
    dspy.Example(
        question="1+1等于几？",
        answer="2"
    ).with_inputs("question", "answer"),
    # ... 更多数据
]

# 定义程序
program = dspy.ChainOfThought("question -> answer")

# 用 MIPRO 优化
teleprompter = dspy.MIPRO(metric=your_eval_metric)
optimized = teleprompter.compile(
    program,
    trainset=trainset,
    max_bootstrapped_demos=4,
    num_threads=4
)
```

---

## 4. 与传统方式的对比

| 维度 | 传统方式 | DSPy |
|------|----------|------|
| Prompt 管理 | 字符串，分散 | 结构化声明，集中 |
| 调优方式 | 人工实验 | 自动优化 |
| 模型迁移 | 需重写 prompt | 换模型即可，prompt 自动生成 |
| 调试难度 | 高（字符串黑箱） | 中（模块化可追踪） |
| 学习曲线 | 低 | 较高 |

---

## 5. 面试相关

### Q1: DSPy 和传统 prompt 工程的本质区别是什么？

**参考答案**：
传统 prompt 工程是"写字符串"，prompt 和业务逻辑耦合；DSPy 是声明式编程，定义输入输出签名，prompt 由优化器自动生成。核心解耦了"做什么"和"怎么做"。

### Q2: DSPy 的自动优化是怎么工作的？

**参考答案**：
通过Bootstrap机制，先用少量示例让模型生成候选 reasoning/prompt，然后用metric评估，选择表现好的作为下一轮样本，迭代优化。类似self-improvement的loop。

### Q3: 什么时候应该用 DSPy？

**参考答案**：
当你需要：
- 在多个模型间切换
- 有明确的评估指标
- 需要系统性优化而非手工调参
- 构建可复用的 LLM Pipeline

不适合场景：快速原型验证、一次性简单任务。

---

## 6. 疑问/思考

- [ ] DSPy 的优化过程本身消耗的 LLM 调用成本是否可接受？
- [ ] Signature 的 desc 字段和最终 prompt 的关系是什么？
- [ ] 多模型场景下，DSPy 的优化结果能否通用？

---

## 关联资源

- [DSPy 官方文档](https://dspy.ai/)
- [DSPy GitHub](https://github.com/stanfordnlp/dspy)
- [DSPy 论文：Compiling Declarative Language Models](https://arxiv.org/abs/2312.03788)
- [Stanford NLP Blog](https://stanfordnlp.github.io/dspy/)

---

## 9. 与本仓库其他单元的关系

|关联单元|关系|
||---|---|
|U3 Prompt 工程|DSPy 自动优化 prompt，是 U3 技巧的工程化升级|
|U30 Post-training|DSPy 的优化器类似于 post-training 的自动调优|
|U15 RAG 进阶|DSPy 常用于优化 RAG pipeline 的 prompt|
|U31 Structured Output|DSPy 的 output schema 依赖 U31 的 structured output|
|U35 AgentOps|DSPy 的 eval 模块与 AgentOps 的监控互补|

---
*最后更新：2026-05-27*
---

## 7. DSPy GEval：自定义 Metric（最强特性）

**GEval** = 用 LLM-as-judge 评估任意复杂标准。

```python
import dspy
from dspy.metrics import GEval

# 定义自定义 metric：评估"回答是否有清晰的结构"
structure_metric = GEval(
    name="Structure Quality",
    criteria="""
    评估回答的结构化程度：
    - 有清晰的开头（背景 / 问题陈述）
    - 有中间论证（理由 / 例子）
    - 有明确结论（答案 / 建议）
    - 使用了合适的格式（标题 / bullet / 段落）
    评分 0-1，1 表示完美结构化。
    """,
    # 传给 judge 的字段
    evaluation_params=[
        dspy.InputField(),
        dspy.PredictionField(),
    ],
    # judge model
    model=dspy.LM("openai/gpt-4o"),
)

# 编译 + 优化
evaluator = dspy.Evaluate(devset=dev_examples, metric=structure_metric)
evaluator(program)
```

**实战场景**：
- "回答是否礼貌"（subjective）
- "代码是否符合团队规范"（custom criteria）
- "回答是否包含所有必要信息"（checklist）

---

## 8. DSPy 优化算法全景

| 算法 | 描述 | 数据需求 | 适用 |
|------|------|----------|------|
| **BootstrapFewShot** | 随机采样示例，挑选 best | 50+ examples | 入门首选 |
| **BootstrapFewShotWithRandomSearch** | 多组 random search | 100+ | 标准选择 |
| **MIPRO** | Bayesian optimization over instructions + demos | 50+ | 高质量 |
| **MIPROv2** | MIPRO 升级版（2024）| 50+ | 当前 SOTA |
| **COPRO** | Coordinate ascent over instructions | 30+ | 旧版（不推荐）|
| **KNNFewShot** | KNN 选相似示例 | 100+ | Few-shot baseline |
| **BootstrapFinetune** | Bootstrap + fine-tune | 200+ | 终极方案 |

**实战选择**：
- 50 examples → BootstrapFewShot
- 100-200 examples → MIPROv2
- 500+ examples → BootstrapFinetune（用小模型蒸馏）

### 8.1 MIPROv2 实战

```python
import dspy
from dspy.teleprompt import MIPROv2

# 1. 准备数据
trainset = [...]  # 100+ examples
valset = [...]    # 50 examples

# 2. 定义 program
class RAGProgram(dspy.Module):
    def __init__(self):
        self.retrieve = dspy.Retrieve(k=3)
        self.generate = dspy.ChainOfThought("context, question -> answer")
    def forward(self, question):
        context = self.retrieve(question).passages
        return self.generate(context=context, question=question)

# 3. 定义 metric
def metric(example, pred, trace=None):
    return dspy.evaluate.answer_exact_match(example, pred)

# 4. 跑 MIPROv2
optimizer = MIPROv2(
    metric=metric,
    auto="medium",  # 优化强度：light/medium/heavy
    num_candidates=10,
)
optimized = optimizer.compile(RAGProgram(), trainset=trainset, valset=valset)

# 5. 用
optimized("什么是 RAG？")
```

**效果**：MIPROv2 通常比 BootstrapFewShot +5-15% 准确率。

---

## 9. DSPy 完整生产部署

### 9.1 保存 + 加载

```python
# 训练后保存
optimized.save("rag_model_v1.json")

# 加载
loaded = dspy.load("rag_model_v1.json")
loaded("用户问题")
```

### 9.2 模型版本管理

```python
# 用 MLflow / Weights & Biases 跟踪
import mlflow

mlflow.set_tracking_uri("http://mlflow:5000")
mlflow.set_experiment("rag-optimization")

with mlflow.start_run():
    mlflow.log_param("optimizer", "MIPROv2")
    mlflow.log_metric("accuracy", 0.89)
    mlflow.log_artifact("rag_model_v1.json")
```

### 9.3 A/B 测试

```python
import random

def serve(question):
    if random.random() < 0.5:
        result = baseline_program(question)
        model_version = "baseline"
    else:
        result = optimized_program(question)
        model_version = "optimized"

    # 上报指标
    langfuse.trace(name="RAG", metadata={"model": model_version})
    return result
```

---

## 10. DSPy vs 其他 Prompt 优化工具

| 工具 | 范式 | 优势 | 适用 |
|------|------|------|------|
| **DSPy** | 声明式 + 优化器 | 自动化 + 评测驱动 | 生产 RAG / Agent |
| **Promptfoo** | YAML + A/B 测试 | 简单 | 评估 + Red Team |
| **TextGrad** | 文本梯度 | 灵活 | 研究 |
| **Promptify** | 模板 + 评估 | 易上手 | 简单场景 |
| **SAMMO** | Structured | 研究导向 | 学术 |

**实战**：**DSPy + Promptfoo 组合**。DSPy 做生产 prompt 优化，Promptfoo 做 CI/CD 验证。
