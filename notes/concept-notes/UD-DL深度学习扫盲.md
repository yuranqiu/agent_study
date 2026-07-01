# U-DL · 深度学习扫盲（轻量版）

> **核心问题**：理解 LLM 原理所需的 DL 基础，不需要会训练
> **预计学时**：3h（快速扫盲）
> **对应 Phase**：Phase 1（补充）

---

## 学习目标

1. 能直观解释 Attention 机制在做什么
2. 能解释 Autoregressive Generation 是什么意思
3. 能说清楚 GPT 的能力是怎么来的（Scaling / Next-token）
4. 能说清楚 RLHF 在做什么（RM + PPO 大框架）

---

## 1. Attention：让模型知道"看向哪里"

### 直观类比

读一句话时，你不会每个字孤立理解，而是会根据上下文确定每个字的含义。Attention 就是让模型做到这件事的机制。

### 核心公式（理解即可，不需要背）

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

- **Q（Query）**：当前 token想知道什么
- **K（Key）**：每个 token 能提供什么信息
- **V（Value）**：每个 token 的实际内容
- **QK^T**：计算当前 token 和所有 token 的相关性（点积）
- **÷√d_k**：缩放，防止点积过大导致梯度消失
- **softmax**：变成概率分布
- **×V**：加权求和，得到最终输出

### 三个关键点

1. **Self-Attention**：输入的每个 token 和所有其他 token 做相关性打分
2. **Multi-Head**：多组 Attention 并行，捕捉不同类型相关性（语法、语义、位置等）
3. **位置编码**：Attention 本身不看位置，需要额外加位置信息（原始 sin/cos，后续有 RoPE 等改进）

### 参考

- [3Blue1Brown: Transformers](https://www.youtube.com/watch?v=eMlx5fFNoKY)（15 分钟，英文字幕）
- [Jay Alammar: The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)

---

## 2. Autoregressive Generation

### 什么是 Autoregressive

"自回归"——用过去的输出影响未来的输出。

GPT 的自回归生成过程：

```
输入："北京是中国的"
模型预测：["首都"]
输入："北京是中国的首都"（拼接上一步输出）
模型预测：["。"]
输入："北京是中国的首都。"（再拼接）
模型预测：[结束符]
```

每一步生成都是基于"输入 + 已经生成的所有内容"，这就是为什么叫"自"回归。

### 与批处理的区别

| 方式 | 过程 | 特点 |
|------|------|------|
| Autoregressive | 顺序生成，一步一步 | 生成有顺序依赖，慢 |
| 一次性生成 | 直接输出完整答案 | 需要训练时设计，不适合 LLM |

---

## 3. GPT 的能力从哪来

### Scaling Law（扩展定律）

模型的性能（困惑度 / 能力）和三个因素强相关：
- **模型参数规模**（更大的模型）
- **训练数据量**（更多的 token）
- **计算量**（更多的 FLOPs）

简单理解：更大 + 更多数据 + 更多计算 = 更强

这是 GPT-4 / Claude-3 等大模型背后的核心理论。

### Next-Token Prediction 的隐含力量

看似简单的"预测下一个词"，实际上需要模型：
- 理解语言语法和语义
- 掌握世界知识
- 进行逻辑推理
- 遵循指令

这些能力都是从大规模文本的"预测下一个词"训练中自动涌现的。

### 三阶段训练

```
预训练（Pretraining）
  └─ 大规模文本，学语言规律 → 通用能力

Instruction Tuning（SFT）
  └─ 人工标注的问答对 → 听从指令

RLHF（Reinforcement Learning from Human Feedback）
  └─ 人类反馈训练 RM → 用 PPO 优化 → 符合人类偏好
```

---

## 4. RLHF：让模型符合人类期望

### 解决什么问题

SFT 之后模型能回答问题了，但回答质量参差不齐，不一定符合人类偏好。

RLHF 用人类反馈来进一步优化模型。

### 三个核心组件

**1. Reward Model（奖励模型）**
- 训练数据：人类对模型输出的排序（如 A 比 B 好）
- 学习目标：给"人类喜欢的回答"更高分数
- 输出：任意回答的质量分数

**2. PPO（Proximal Policy Optimization）**
- 一种强化学习算法
- 用 Reward Model 的分数作为奖励信号
- 优化策略（Policy），让模型学会产生更高奖励的回答

**3. 人类反馈（Human Feedback）**
- 真实人类对回答质量做评判
- 是 RLHF 的核心信号来源

### 面试能说的话

> "RLHF 的核心是先训练一个 Reward Model 来学习人类的偏好，然后用这个 Reward Model 的分数作为奖励信号，用 PPO 算法优化模型，让它生成更符合人类期望的回答。InstructGPT 是这个方向的经典工作。"

---

## 完成 Gate 检验

1. **Q1**：Attention 的 Q/K/V 各自代表什么？直观说它们在做什么。
2. **Q2**：Autoregressive Generation 是什么意思？生成"今天天气好"需要几步？
3. **Q3**：GPT 的能力主要来自哪三个因素的 Scaling？
4. **Q4**：RLHF 的三个核心组件是什么？各做什么？

---

## 练习清单

- [ ] 看 3Blue1Brown Transformer 视频（15 分钟）
- [ ] 用自己的话向非技术朋友解释"GPT 是在预测下一个词"
- [ ] 读 InstructGPT 论文 Abstract + Section 3（2h，选读）

---

## 关联资源

- [3Blue1Brown 神经网络系列](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZC07-PI9)
- [Jay Alammar Transformer 图解](https://jalammar.github.io/illustrated-transformer/)
- [InstructGPT 论文](https://arxiv.org/abs/2203.02155)
- [Andrej Karpathy 视频](https://www.youtube.com/watch?v=kexM9Ars3Mw)


## 5. 与其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U1 LLM 基础 | U-DL 是 U1 的数学基础，U1 的 Attention / Scaling / RLHF 都在这里展开 |
| U3 Prompt 工程 | 理解 DL 原理后才能理解为什么 prompt 有效 |
| U30 Post-training | RLHF 的完整技术细节在 U30 |
| U32 Fine-tuning | 微调的数学原理在 U32 |
| U42 面试指南 | U-DL 的知识是 U42 面试题的数学基础 |
---

*最后更新：2026-05-26*