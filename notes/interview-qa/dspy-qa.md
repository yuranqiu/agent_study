# Interview QA · DSPy 篇

> 来源：DSPy 核心概念 / 面试高频题
> 覆盖：Signature / Teleprompter / 模块 / 优化 / 实战

---

## 一、基础概念篇

### Q: DSPy 是什么？解决什么问题？

**Answer:**
- **DSPy = Declarative Self-Programming（声明式自编程）**
- 解决手工 Prompt 工程脆弱、难以优化的问题
- 核心思想：传统是 Human 写 Prompt → LM 执行；DSPy 是 Human 写 Signature → DSPy 自动生成 Prompt → LM 执行

---

### Q: Signature 和传统 Prompt 的区别？

**Answer:**

| 维度 | 传统 Prompt | DSPy Signature |
|------|-------------|----------------|
| 定义方式 | 写死的字符串 | 声明式描述输入输出 |
| 优化方式 | 手工调优 | 自动优化（Teleprompter） |
| 可复用性 | 低，改了可能影响其他任务 | 高，模块化 |
| 推理过程 | 不透明 | 可选 CoT 可视化 |

**Signature 示例：**
```python
# 基本签名
qa = dspy.Signature("question -> answer")

# 带多输出
analyze = dspy.Signature(
    "discussion_thread -> summary, sentiment, key_points"
)
```

---

### Q: DSPy 的核心执行流程？

**Answer:**

```
写 Signature → DSPy 生成 Prompt → 执行
```

可选：用 Teleprompter 自动优化整个流程

---

## 二、核心组件篇

### Q: DSPy 有哪些内置 Module？

**Answer:**

| Module | 用途 |
|--------|------|
| `dspy.CoT` | Chain of Thought（默认带推理） |
| `dspy.ChainOfThought` | 显式推理链 |
| `dspy.MLP` | 简单 MLP 网络 |
| `dspy.ReAct` | ReAct 范式 |
| `dspy.BootstrapFewShot` | 小样本引导优化 |

---

### Q: 如何用 DSPy 实现一个 QA？

**Answer:**

```python
import dspy

# 1. 定义 Signature
qa = dspy.Signature("question -> answer")

# 2. 创建 Module
program = dspy.CoT(qa)

# 3. 运行
result = program(question="生命的意义是什么？")
print(result.answer)
```

---

### Q: DSPy 如何接入 LM？

**Answer:**

```python
# OpenAI
gpt3 = dspy.OpenAI(model='gpt-3.5-turbo')
dspy.settings.configure(lm=gpt3)

# 本地模型
ollama = dspy.OllamaLocal(model='llama2')
dspy.settings.configure(lm=ollama)
```

---

## 三、Teleprompter 篇

### Q: Teleprompter 是什么？解决什么问题？

**Answer:**
- 自动优化 Prompt 的模块
- 解决手工调优 prompt 费时费力、效果不稳定的问题
- 输入：学生模型、教师模型（可选）
- 输出：优化后的学生模型

---

### Q: BootstrapFewShot 的工作原理？

**Answer:**

```python
from dspy.teleprompt import BootstrapFewShot

# 定义教师和学生
student = dspy.CoT(qa)
teacher = dspy.ChainOfThought(qa)  # teacher 有完整推理过程

# 自动优化
teleprompter = BootstrapFewShot(
    teacher=teacher,
    student=student,
    num_trials=10
)

optimized_student = teleprompter.compile(
    student,
    trainset=train_set
)
```

**原理：**
1. 用 Teacher 生成带有推理过程的完整示例
2. 用这些示例引导 Student 学习
3. 多轮 trial 迭代优化

---

### Q: 常用 Teleprompter 有哪些？

**Answer:**

| Teleprompter | 用途 |
|--------------|------|
| `BootstrapFewShot` | 小样本引导自动优化 |
| `LabeledFewShot` | 用标注数据优化 |
| `Ensemble distillation` | 集成蒸馏 |

---

## 四、进阶用法篇

### Q: DSPy 如何实现多输入多输出？

**Answer:**

```python
# 多输入
multi_input = dspy.Signature("question, context -> answer")

# 多输出
analyze = dspy.Signature(
    "customer_feedback -> sentiment, category, action_items"
)

analyzer = dspy.ChainOfThought(analyze)
result = analyzer(customer_feedback="产品很好，但配送太慢")

print(result.sentiment)      # 正面
print(result.category)       # 服务
print(result.action_items)   # 改进配送
```

---

### Q: DSPy 和 LangChain Agent 的区别？

**Answer:**

| 维度 | LangChain Agent | DSPy |
|------|-----------------|------|
| 核心机制 | Tool Definition + Tool Selection | Signature + Program |
| 优化方式 | 手工调 Prompt | 自动优化（Teleprompter） |
| 灵活性 | 高（可自定义 Tool） | 中（模块化程度高） |
| 适用场景 | 复杂 Tool 调用 | 固定 Pipeline 优化 |

---

### Q: DSPy 适合什么场景？

**Answer:**

**适合：**
- Prompt 固定但需要持续优化的场景
- 多步骤 Pipeline（检索→排序→生成）
- 需要自动化优化 Prompt 的生产系统

**不适合：**
- 需要大量自定义 Tool 的复杂 Agent 场景
- 动态决策流程（这仍是 LangChain 的强项）

---

## 五、面试高频题

### Q: DSPy 的 Forward 流程是什么？

**Answer:**

```python
class MyProgram(dspy.Module):
    def __init__(self):
        self.generate_answer = dspy.CoT(qa)
    
    def forward(self, question):
        return self.generate_answer(question=question)
```

Forward 就是数据流经各个 Module 的过程。

---

### Q: 如何评估 DSPy 程序的效果？

**Answer:**

```python
# 定义评估指标
from dspy.evaluate import Evaluate

evaluate = Evaluate(
    devset=dev_set,
    metric=answer_exact_match,  # 或自定义 metric
    num_threads=4
)

results = evaluate(optimized_program)
```

---

### Q: DSPy 的优缺点？

**Answer:**

**优点：**
- 自动优化 Prompt，效果往往优于手工
- 模块化，易复用和调试
- 声明式，代码简洁

**缺点：**
- 学习曲线（Signature、Module、Teleprompter 概念多）
- 对复杂 Tool 调用支持不如 LangChain
- 调试困难（Prompt 是 DSPy 自动生成的）

---

*最后更新：2026-05-27*
