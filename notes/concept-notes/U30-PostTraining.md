# U30 · Post-training 与 Alignment（后训练与对齐）

> **核心问题**：预训练后的 LLM 怎么变成能用的 chat model / reasoning model / agent？后训练（post-training）的核心技术有哪些？为什么 RLHF 之后是 DPO / GRPO / RLVR？
> **预计学时**：6h
> **对应 Phase**：Phase 2 / Phase 5
> **状态**：核心前沿（2024-2026 后训练范式革命，Reasoning Model 全部基于此）

---

## 1. 为什么 Post-training 重要

**Post-training**（后训练）= 预训练完成之后的所有训练步骤，是 LLM 从"续写机器"变成"有用助手"的关键。

```
┌─────────────────────────────────────┐
│ Pre-training（自监督）              │  ← 学会"续写"
│   - Next-token prediction           │
│   - 万亿 token 训练                  │
│   - GPT / Llama / Qwen 基础         │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Post-training（后训练）              │  ← 学会"有用"
│   - SFT / DPO / RLHF / GRPO         │  ← 本单元
│   - Instruction following           │
│   - Reasoning / Tool use            │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Deployment                          │
└─────────────────────────────────────┘
```

**为什么 Agent 工程师要懂 post-training**：
- **理解 reasoning model 怎么训练**（U24 的 DeepSeek R1 怎么来的）
- **用 unsloth/llama-factory 训练专属 agent**（U32）
- **理解模型能力的边界**（为什么 GPT-4 比 GPT-3.5 更"听话"）
- **面试必问**（RLHF / DPO / GRPO）

---

## 2. Post-training 的 5 大阶段

### 2.1 SFT（Supervised Fine-Tuning）

**SFT** = 用人工标注的 (prompt, response) 数据对继续训练模型。

- **来源**：人工标注 / GPT-4 生成 / 开源数据集（Alpaca / Dolly / ShareGPT）
- **格式**：SFT trainer（trl / swift / LLaMA-Factory）
- **目的**：让模型学会"按指令回答"

```python
from trl import SFTTrainer
from transformers import TrainingArguments

trainer = SFTTrainer(
    model=base_model,
    train_dataset=instruction_dataset,  # [{"prompt": ..., "response": ...}]
    args=TrainingArguments(
        output_dir="./sft-model",
        num_train_epochs=3,
        per_device_train_batch_size=4,
        learning_rate=2e-5,
    ),
)
trainer.train()
```

**SFT 的局限**：
- 数据贵（人工标注 1k 数据 = 数天 / 数万美元）
- 只能学到"模仿"，不能学到"更好"
- 容易过拟合到标注者风格

### 2.2 RM（Reward Model）训练

**Reward Model** = 一个单独训练的模型，给 (prompt, response) 打分（0-1 或 -∞ 到 +∞），模拟人类偏好。

- **数据**：人工对同一个 prompt 的两个 response 排序（"A 更好" / "B 更好"）
- **训练**：把 RM 当二分类器，预测人类偏好
- **代表**：InstructGPT 的 RM 用 ~33k 人类偏好对训练

```python
# RM 训练伪代码
def train_reward_model(prompt, response_a, response_b, human_preference):
    score_a = reward_model(prompt, response_a)  # 标量
    score_b = reward_model(prompt, response_b)
    loss = -log_sigmoid(score_preferred - score_other)
    return loss
```

**RM 的问题**：
- **Reward hacking**：模型学会"骗 RM"（生成 RM 喜欢但实际不好的回答）
- **Goodhart's law**：当 RM 成为优化目标，它就不再是好指标
- **分布外泛化差**：RM 只在训练数据分布附近准

### 2.3 RLHF（Reinforcement Learning from Human Feedback）

**RLHF** = 用 RM 的分数作为 reward signal，用 PPO 算法优化 LLM。

**InstructGPT 的 RLHF 流程**：
1. SFT 训练一个初始模型
2. 训练 RM
3. **PPO 阶段**：用 RM 的 reward 训练 SFT 模型
   - prompt → LLM 生成 response
   - RM 给 response 打分
   - PPO 更新 LLM 权重，maximize RM score
   - 加 KL 散度惩罚（防止模型偏离太远）

```python
# RLHF/PPO 伪代码
def ppo_step(prompt):
    response = policy_llm.generate(prompt)
    reward = reward_model(prompt, response)
    kl_penalty = kl_divergence(policy_llm, ref_llm)
    advantage = reward - baseline - kl_coef * kl_penalty
    loss = -advantage * log_prob(response | prompt)
    optimizer.update(loss)
```

**RLHF 的问题**：
- **训练不稳定**：PPO 超参敏感
- **成本高**：要同时跑 policy / ref / reward / value 4 个模型
- **Reward hacking**：RM 被 game
- **价值函数难训**：value model 训练不充分

### 2.4 DPO（Direct Preference Optimization）

**DPO** = 直接用偏好数据优化 LLM，**不需要训练 RM，也不需要 PPO**。

> "DPO is the simplest realization of the RLHF objective, requiring only a supervised dataset of preferences."
> —— Stanford / CZ Biohub 2023

**核心洞察**：把 RLHF 的 reward 隐式表达为 LLM 自身 log-probability 的函数，从而**直接优化偏好损失**。

```python
# DPO 训练伪代码
def dpo_loss(prompt, chosen, rejected, ref_model):
    # chosen: 人类偏好的 response
    # rejected: 人类不偏好的 response
    log_pi_chosen = policy_llm.log_prob(prompt, chosen)
    log_pi_rejected = policy_llm.log_prob(prompt, rejected)
    log_ref_chosen = ref_model.log_prob(prompt, chosen)
    log_ref_rejected = ref_model.log_prob(prompt, rejected)

    # DPO 损失
    logits = beta * ((log_pi_chosen - log_ref_chosen) - (log_pi_rejected - log_ref_rejected))
    loss = -F.logsigmoid(logits).mean()
    return loss
```

**DPO 优势**：
- **不需要训练 RM**（省 1 个模型训练）
- **不需要 PPO**（训练更稳定）
- **只用偏好数据**（不需要 reward 标量）
- **超参少**（主要是 β）

**DPO 局限**：
- 偏好数据要"质量高 + 多样"——难收集
- 不能直接做 online RL（要用 IPO / KTO 变体）

### 2.5 GRPO（Group Relative Policy Optimization）

**GRPO** = DeepSeek R1 用的算法，2025 年最火。

> "GRPO is a variant of PPO that eliminates the critic (value model) by using a group of sampled responses to estimate the baseline."
> —— DeepSeek R1 Paper

**核心**：
- 同一个 prompt 采样 N 个 response
- 用 RM 给每个 response 打分
- 用**组内相对排名**作为 advantage（不要 value model）
- PPO 更新但用 group-relative advantage

```python
# GRPO 伪代码
def grpo_step(prompt):
    # 1. 采样一组 response
    responses = policy_llm.sample(prompt, n=8, temperature=0.8)

    # 2. RM 打分
    rewards = [reward_model(prompt, r) for r in responses]

    # 3. 组内归一化
    advantages = (rewards - mean(rewards)) / std(rewards)

    # 4. PPO 风格更新
    for response, advantage in zip(responses, advantages):
        loss = -advantage * log_prob(response | prompt)
        optimizer.update(loss)
```

**GRPO 优势**：
- **不需要 value model**（省 1 个模型）
- **训练更稳定**（组内 baseline 比学习的 baseline 准）
- **推理模型特别有效**（R1 用的就是这个）

### 2.6 RLVR（Reinforcement Learning with Verifiable Rewards）

**RLVR** = 2025 年的新趋势，用**可验证的 rule-based reward** 替代 learned RM。

- **数学题**：答案对错自动判断
- **代码题**：单元测试通过/失败
- **逻辑题**：规则匹配

**优势**：
- **没有 reward hacking**（reward 不会被 game）
- **训练更稳定**（reward 信号精确）
- **便宜**（不需要标注 reward）

**代表**：
- DeepSeek R1 用 RLVR 训练数学/代码能力
- OpenAI o 系列内部用 RLVR

---

## 3. Post-training 全景对照

| 方法 | 数据 | 训练成本 | 稳定性 | 适用 |
|------|------|----------|--------|------|
| **SFT** | (prompt, response) | 低 | 高 | 所有模型的起点 |
| **RLHF (PPO)** | 偏好对 + RM | 高 | 低 | 经典 ChatGPT 路线 |
| **DPO** | 偏好对 | 中 | 高 | RLHF 替代品 |
| **IPO / KTO / SimPO** | 偏好对 | 中 | 高 | DPO 变体 |
| **GRPO** | 偏好对 + RM | 中 | 高 | Reasoning Model 训练 |
| **RLVR** | (prompt, verifiable answer) | 中 | 很高 | 数学/代码/逻辑 |
| **Online RL (RLOO / ReMax)** | 在线采样 | 高 | 中 | 探索性研究 |
| **Constitutional AI** | 原则 + 自我批评 | 中 | 中 | Anthropic 路线 |

---

## 4. Reasoning Model 的训练（DeepSeek R1 案例）

**R1 的训练流程（被广泛模仿）**：

```
Stage 1: SFT（冷启动）
   - 用几千条高质量 reasoning 数据做 SFT
   - 让模型学会"思考格式"
   ↓
Stage 2: GRPO + RM（推理能力）
   - 用 GRPO + rule-based RM 训练
   - reward = 答案对错 + 格式分
   - 训练出 R1-Zero（纯 RL）
   ↓
Stage 3: SFT（拒采 + 语言一致性）
   - 拒采低质量 reasoning 数据
   - 加语言一致性奖励
   - 训练出 R1
   ↓
Stage 4: 通用 RLHF（helpful + harmless）
   - 用人类偏好做 DPO/RLHF
   - 提升通用能力
```

**关键洞察**：**R1 的核心不是架构创新，而是 post-training pipeline**。任何 base model 都能用这套流程训练出 reasoning 能力。

---

## 5. 后训练实战工具栈

### 5.1 SFT / DPO 工具

| 工具 | 特点 | 推荐场景 |
|------|------|----------|
| **LLaMA-Factory** | 中文友好，全功能 | 国内首选 |
| **trl（HuggingFace）** | 官方，DPO/PPO/SFT 都有 | 国际首选 |
| **unsloth** | 2-5x 加速，省显存 | 资源有限时 |
| **axolotl** | 高度可定制 | 复杂需求 |
| **swift（ModelScope）** | 阿里出品，多模态 | 中文多模态 |

### 5.2 GRPO / RLVR 工具

| 工具 | 特点 |
|------|------|
| **trl (GRPO trainer)** | HuggingFace 官方支持 |
| **OpenRLHF** | 字节开源，高性能 |
| **verl** | 字节开源，Volcengine RL |
| **Tunix** | Google JAX 生态 |
| **Prime** | RL with implicit PRM |

### 5.3 评估工具

| 工具 | 场景 |
|------|------|
| **lm-eval-harness** | 标准 benchmark |
| **alpaca-eval** | Chat 质量 |
| **MT-Bench** | 多轮对话 |
| **MMLU / GSM8K / HumanEval** | Reasoning / Code |
| **Reward Bench** | RM 评估 |

---

## 6. 实战：训练一个专属 Agent 模型

### 6.1 数据准备

```python
# 1000 条 (prompt, response) 数据
data = [
    {
        "prompt": "用户：查一下北京天气\n助手：",
        "response": "我来用工具查一下。Action: get_weather\nAction Input: {\"city\": \"北京\"}\nObservation: 25°C 晴\nFinal Answer: 北京今天 25°C，晴朗。",
    },
    # ... 999 more
]
```

### 6.2 SFT 训练（用 LLaMA-Factory）

```bash
llamafactory-cli train \
    --model_name_or_path Qwen/Qwen2.5-7B-Instruct \
    --dataset my_agent_data \
    --template qwen \
    --finetuning_type lora \
    --lora_rank 16 \
    --output_dir ./sft-agent \
    --num_train_epochs 3 \
    --per_device_train_batch_size 2
```

### 6.3 DPO 训练（偏好对齐）

```bash
# 准备偏好数据 {"prompt", "chosen", "rejected"}
llamafactory-cli train \
    --model_name_or_path ./sft-agent \
    --dataset my_agent_dpo_data \
    --finetuning_type lora \
    --training_stage dpo \
    --output_dir ./dpo-agent
```

### 6.4 GRPO 训练（reasoning 能力）

```python
# 用 trl GRPO trainer
from trl import GRPOTrainer

trainer = GRPOTrainer(
    model=base_model,
    reward_funcs=[accuracy_reward, format_reward],  # RLVR 风格
    train_dataset=math_problems,
    args=GRPOConfig(num_generations=8, ...),
)
trainer.train()
```

---

## 7. Agent 工程师用 post-training 的 3 大场景

### 7.1 场景 1：训练专属 agent 模型

- 用公司内部数据 SFT
- 用人类反馈 DPO
- 部署到生产替换 GPT-4o

### 7.2 场景 2：Reasoning 微调

- 业务问题用 R1 / o1 蒸馏到小模型
- 用 GRPO 训练数学/逻辑能力
- 降本 90%

### 7.3 场景 3：Tool use 能力增强

- 收集 agent 真实调用数据
- SFT 训练 tool-calling 能力
- 比 prompt 工程更稳定

---

## 8. 面试视角

- **Q：post-training 包含哪些阶段？**
  → SFT（指令微调）→ RM 训练（可选）→ RLHF / DPO / GRPO（偏好对齐）→ RLVR（可验证 reward）。每个阶段解决不同问题。

- **Q：RLHF 的核心问题是什么？DPO 怎么解决？**
  → RLHF 要训 RM + PPO，训练不稳定、reward hacking、成本高。DPO 直接用偏好数据优化 LLM 自身，不需要 RM 也不需要 PPO。DPO 把 RLHF 的 reward 隐式表达为 log-probability 差。

- **Q：GRPO 是什么？为什么 R1 用它？**
  → GRPO 是 DeepSeek R1 用的算法，是 PPO 的变体——用同一 prompt 采样的 N 个 response 的相对排名作为 advantage，不需要 value model。比 PPO 训练更稳定，比 DPO 适合 reasoning（可以 RLVR 风格的 reward）。

- **Q：RLHF 的 reward hacking 是什么？怎么解决？**
  → RM 被训练后，policy LLM 会学"骗 RM"（生成 RM 高分但实际不好的回答）。解决方法：KL 散度惩罚、RLVR（rule-based reward）、Constitutional AI（原则 + 自我批评）、Reward ensemble。

- **Q：SFT 数据要多少？质量 vs 数量？**
  → 数量上 1k-100k 都见过，**质量 >> 数量**。Llama-2 用 27k SFT 数据，质量极高。Alpaca 用 52k 但质量参差，效果不如前者。**100 条高质量 > 10k 低质量**。

- **Q：R1 的训练流程？**
  → Stage 1 SFT 冷启动（几千条 reasoning 数据）→ Stage 2 GRPO + rule-based RM（R1-Zero）→ Stage 3 SFT 拒采 + 语言一致性 → Stage 4 通用 RLHF。**核心不是架构，是 post-training pipeline**。

- **Q：为什么 Agent 工程师要懂 post-training？**
  → ① 理解 reasoning model 怎么来 ② 训练专属 agent 模型 ③ 理解模型能力边界 ④ 降本（蒸馏小模型）⑤ 面试必问。

---

## 9. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [InstructGPT 论文](https://arxiv.org/abs/2203.02155) | RLHF 经典论文 |
| [DPO 论文](https://arxiv.org/abs/2305.18290) | DPO 原始论文 |
| [DeepSeek R1 论文](https://arxiv.org/abs/2501.12948) | GRPO + R1 训练流程 |
| [HuggingFace TRL 文档](https://huggingface.co/docs/trl) | trl 库（SFT/DPO/GRPO/PPO） |
| [LLaMA-Factory 文档](https://github.com/hiyouga/LLaMA-Factory) | 中文友好的训练工具 |
| [Alignment Handbook](https://github.com/huggingface/alignment-handbook) | HuggingFace 官方 alignment 教程 |
| [Karpathy: RLHF 讲解](https://www.youtube.com/watch?v=2MBJOuVq340) | 通俗讲解 |

---

## 10. 练习

1. **SFT 实战**：用 LLaMA-Factory 在 Alpaca 数据上 SFT 一个 7B 模型
2. **DPO 实战**：用 trl DPO trainer 训练偏好对齐
3. **GRPO 实战**：用 trl GRPO trainer 训练一个数学模型
4. **数据实验**：对比 100 条高质量 vs 1000 条低质量 SFT 数据效果
5. **复现 R1 流程**：用 Qwen2.5 + GRPO 训练一个 mini reasoning model
6. **RM 评估**：用 Reward Bench 评估自己训练的 RM

---

## 11. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U-DL 深度学习扫盲 | U-DL 提了 RLHF 大框架，U30 深入展开 |
| U1 LLM 基础 | U1 提了 Instruction Tuning，U30 展开 SFT/DPO/GRPO |
| U3 Prompt 工程 | Prompt 是 SFT 数据的一部分 |
| U14 DSPy | DSPy 用 teleprompter 自动做 prompt 优化，是 prompt 层的 SFT 替代 |
| U24 Reasoning Model | R1 / o1 是 post-training 的产物，U30 解释怎么训 |
| U25 Computer Use | Computer Use 模型也用 post-training 训练 |
| U32 Fine-tuning 实践 | U30 是理论，U32 是实战（unsloth/llama-factory） |

---

*最后更新：2026-06-30 新增 U30（Post-training 与 Alignment）*
*来源：InstructGPT / DPO / DeepSeek R1 论文、trl / LLaMA-Factory 文档、Karpathy 教学*
