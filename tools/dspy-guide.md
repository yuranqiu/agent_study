# Tool Guide · DSPy

> 覆盖：核心概念 / Signature / Teleprompter / 快速上手 / 面试高频问法

---

## 1. 核心概念

**DSPy = Declarative Self-Programming (声明式自编程)**

**解决的问题**：
- Prompt 工程脆弱：改一点 prompt 可能破坏其他能力
- 手工写 prompt 难以优化
- DSPy 用编程代替 prompt，让模型自动学习最佳提示

**核心思想**：
```
传统：Human 写 Prompt → LM 执行
DSPy：Human 写 Signature → DSPy 自动生成 Prompt → LM 执行
```

**Signature = 声明式任务描述**
```python
class QA Signature:
    inputs: question (str)
    outputs: answer (str)
```
告诉 DSPy "输入是问题，输出是答案"，而不是 "用友好语气回答"

---

## 2. 核心组件

### 2.1 Signature

```python
import dspy

# 基本签名
qa = dspy.Signature("question -> answer")

# 带子字段的签名
analyze = dspy.Signature(
    "discussion_thread -> summary, sentiment, key_points"
)
```

### 2.2 Chain of Thought (CoT)

```python
class CoT(dspy.Module):
    def __init__(self):
        super().__init__()
        self.generate_answer = dspy.CoT(qa)
    
    def forward(self, question):
        return self.generate_answer(question=question)

cot = CoT()
result = cot(question="什么是量子计算？")
print(result.answer)
```

### 2.3 Program Modes

```python
# 内置的 program 类型
dspy.CoT(signature)      # Chain of Thought
dspy.ChainOfThought(signature, reason_config=...)
dspy.MLP(dspy.Signature) # 简单 MLP
dspy.ReAct(signature)    # ReAct 范式
```

---

## 3. Teleprompter（自动优化）

```python
import dspy
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

### 3.2 常用 Teleprompter

| Teleprompter | 用途 |
|--------------|------|
| `BootstrapFewShot` | 小样本引导自动优化 |
| `LabeledFewShot` | 用标注数据优化 |
| `Ensemble distillation` | 集成蒸馏 |

---

## 4. 快速上手

```bash
pip install dspy
```

### 4.1 最小例子

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

### 4.2 接入 LM

```python
# OpenAI
gpt3 = dspy.OpenAI(model='gpt-3.5-turbo')
dspy.settings.configure(lm=gpt3)

# 也可以用本地模型
ollama = dspy.OllamaLocal(model='llama2')
dspy.settings.configure(lm=ollama)
```

### 4.3 完整示例

```python
import dspy

# 定义任务
analyze_sig = dspy.Signature(
    "customer_feedback -> sentiment, category, action_items"
)

# 写程序
analyzer = dspy.ChainOfThought(analyze_sig)

# 运行
feedback = "这个产品很好，但配送太慢了，希望能改进"
result = analyzer(customer_feedback=feedback)

print(f"Sentiment: {result.sentiment}")
print(f"Category: {result.category}")
print(f"Action Items: {result.action_items}")
```

---

## 5. DSPy vs 传统 Prompt 工程

| 维度 | 传统 Prompt | DSPy |
|------|-------------|------|
| 定义方式 | 写字符串 | 写 Signature（声明式） |
| 优化方式 | 手工调优 | 自动优化（Teleprompter） |
| 可复用性 | 低，改了可能影响其他任务 | 高，模块化 |
| 推理过程 | 不透明 | 可选 CoT 可视化 |

---

## 6. 面试高频问法

### Q: DSPy 是什么？解决什么问题？

→ Declarative Self-Programming，声明式自编程框架
→ 解决手工 prompt 工程脆弱、难以优化的问题
→ 用 Signature 声明任务，让 DSPy 自动生成最优 prompt

### Q: Signature 和传统 Prompt 的区别？

→ 传统 Prompt：写死的字符串，改动可能影响其他能力
→ Signature：声明式描述输入输出，DSPy 自动生成 prompt
→ 更稳健、可优化、可复用

### Q: Teleprompter 的作用？

→ 自动优化 prompt 的模块
→ 输入：学生模型、教师模型（可选）
→ 输出：优化后的学生模型

### Q: DSPy 的执行流程？

→ 写 Signature → DSPy 生成 Prompt → 执行
→ 可选：用 Teleprompter 自动优化

---

*最后更新：2026-05-27*