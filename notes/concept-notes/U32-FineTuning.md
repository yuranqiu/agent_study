# U32 · Fine-tuning 实践（微调实战）

> **核心问题**：怎么用 SFT / DPO / GRPO 把开源 base model 训练成**专属 agent 模型**？工具、流程、坑点全在这。
> **预计学时**：5h
> **对应 Phase**：Phase 4 / 进阶
> **状态**：实战必备（Agent 工程师能微调专属模型是 2024-2026 加分项）

---

## 1. 什么时候需要 Fine-tuning

**Fine-tuning 适用场景**：
- 公司有**大量专有数据**（10k+ 标注样本）
- 模型需要**特定格式输出**（schema / tone / style）
- 需要**降本**（用 7B 蒸馏替代 GPT-4o）
- 需要**离线/私有化**（数据不能出公司）

**不适用**：
- 只有几百条数据——用 prompt engineering
- 通用知识问题——用 RAG
- 实时变化的知识——用 RAG

**决策树**：

```
你的目标是什么？
├─ 改输出格式 / tone → Prompt Engineering
├─ 加私有知识 → RAG
├─ 学新能力（agent / reasoning） → Fine-tuning
└─ 降本 → Fine-tuning（小模型 + 蒸馏）
```

---

## 2. 3 大微调范式

### 2.1 SFT（监督微调）

**SFT** = 用 (prompt, response) 数据对继续训练 base model。

- **数据**：10k-100k 条 (prompt, response)
- **训练**：全参数 or LoRA
- **目的**：让模型学会"按某种模式回答"

```python
# 训练数据格式
data = [
    {
        "prompt": "用户：退款流程是什么？\n助手：",
        "response": "退款流程分三步：1) 提交申请 2) 客服审核 3) 原路退回。需要我详细说吗？",
    },
    # ...
]
```

### 2.2 LoRA / QLoRA（参数高效微调）

**LoRA** = Low-Rank Adaptation，只训练低秩矩阵，**冻结原模型权重**。

- **参数量**：原模型的 0.1%-1%
- **显存**：原模型的 1/4 - 1/2
- **效果**：在大多数任务上接近全参数微调
- **可合并**：训练后可以 merge 到原模型，无推理开销

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,                    # 低秩矩阵的 rank
    lora_alpha=32,           # scaling factor
    target_modules=["q_proj", "v_proj"],  # 哪些层
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)

model = get_peft_model(base_model, lora_config)
model.print_trainable_parameters()
# trainable params: 4,194,304 || all params: 7,242,601,600 || trainable%: 0.058
```

**QLoRA** = LoRA + 4-bit quantization：
- 7B 模型训练只需 **6GB 显存**（单张消费级 GPU）
- 13B 模型训练只需 **10GB 显存**

### 2.3 DPO / GRPO（偏好 / 推理微调）

详见 U30 Post-training。

---

## 3. 工具栈对比

| 工具 | 框架 | 优势 | 推荐场景 |
|------|------|------|----------|
| **LLaMA-Factory** | PyTorch | 中文友好、500+ 模型支持、CLI + WebUI | 国内首选 |
| **trl（HuggingFace）** | PyTorch | 官方、DPO/PPO/GRPO 全支持 | 国际首选 |
| **unsloth** | PyTorch | 2-5x 加速、4-bit 优化 | 资源有限时 |
| **axolotl** | PyTorch | 高度可定制、YAML 配置 | 复杂需求 |
| **swift（ModelScope）** | PyTorch | 阿里出品、多模态 | 中文多模态 |
| **llama.cpp + QLoRA** | C++ | CPU 训练 / 边缘 | 极低资源 |
| **OpenRLHF** | PyTorch | 字节、高性能 PPO/GRPO | 大规模 RLHF |

---

## 4. 实战：用 LLaMA-Factory 做 SFT

### 4.1 安装

```bash
git clone --depth 1 https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
pip install -e ".[torch,metrics]"
```

### 4.2 准备数据

LLaMA-Factory 用 JSONL 格式：

```jsonl
{"instruction": "退款流程是什么？", "input": "", "output": "退款流程分三步：1) 提交申请 2) 客服审核 3) 原路退回。"}
{"instruction": "查询订单状态", "input": "订单号 12345", "output": "您的订单 12345 正在配送中，预计明天到达。"}
```

存为 `data/my_agent.jsonl`，并在 `data/dataset_info.json` 注册：

```json
{
  "my_agent": {
    "file_name": "my_agent.jsonl",
    "columns": {
      "prompt": "instruction",
      "query": "input",
      "response": "output"
    }
  }
}
```

### 4.3 启动训练

```bash
llamafactory-cli train \
    --model_name_or_path Qwen/Qwen2.5-7B-Instruct \
    --template qwen \
    --dataset my_agent \
    --finetuning_type lora \
    --lora_rank 16 \
    --lora_alpha 32 \
    --output_dir ./sft-agent \
    --num_train_epochs 3.0 \
    --per_device_train_batch_size 2 \
    --gradient_accumulation_steps 4 \
    --learning_rate 1e-4 \
    --fp16 \
    --logging_steps 20 \
    --save_steps 500
```

### 4.4 用 WebUI（更友好）

```bash
llamafactory-cli webui
# 打开 http://localhost:7860
```

WebUI 支持点选配置 + 实时看 loss 曲线。

### 4.5 推理 + 评估

```bash
llamafactory-cli chat \
    --model_name_or_path Qwen/Qwen2.5-7B-Instruct \
    --adapter_name_or_path ./sft-agent \
    --template qwen \
    --finetuning_type lora
```

---

## 5. 实战：用 unsloth 做 QLoRA（资源友好）

### 5.1 安装

```bash
pip install "unsloth[colab-new] @ git+https://github.com/unslothai/unsloth.git"
pip install --no-deps trl peft accelerate bitsandbytes
```

### 5.2 训练

```python
from unsloth import FastLanguageModel
from trl import SFTTrainer
from transformers import TrainingArguments

# 1. 加载 4-bit 模型
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Qwen2.5-7B-Instruct-bnb-4bit",
    max_seq_length=2048,
    load_in_4bit=True,
)

# 2. 加 LoRA adapter
model = FastLanguageModel.get_peft_model(
    model,
    r=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_alpha=16,
    use_gradient_checkpointing="unsloth",
)

# 3. 训练
trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=instruction_dataset,
    args=TrainingArguments(
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        warmup_steps=10,
        num_train_epochs=3,
        learning_rate=2e-4,
        fp16=not is_bfloat16_supported(),
        bf16=is_bfloat16_supported(),
        logging_steps=20,
        output_dir="outputs",
    ),
)

trainer.train()
```

### 5.3 导出

```python
# 合并 LoRA 到 base model（可选）
model.save_pretrained_merged("merged_model", tokenizer, save_method="merged_16bit")

# 或只保存 LoRA adapter
model.save_pretrained("lora_adapter")
```

**unsloth 优势**：
- 2-5x 加速（vs 普通 transformers）
- 4-bit 量化（6GB 显存训 7B）
- 兼容 trl / peft 生态

---

## 6. 实战：用 trl 做 DPO

```python
from trl import DPOTrainer, DPOConfig
from datasets import load_dataset

# 1. 准备偏好数据 {"prompt", "chosen", "rejected"}
dataset = load_dataset("Anthropic/hh-rlhf", split="train")

# 2. 训练
trainer = DPOTrainer(
    model=sft_model,
    ref_model=ref_model,  # 同一 model 冻结 reference
    train_dataset=dataset,
    args=DPOConfig(
        output_dir="./dpo-agent",
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        learning_rate=5e-7,
        beta=0.1,  # ← KL 系数
        num_train_epochs=1,
    ),
)

trainer.train()
```

---

## 7. 实战：用 trl 做 GRPO（reasoning 微调）

```python
from trl import GRPOTrainer, GRPOConfig

# 1. 定义 reward function（RLVR 风格）
def accuracy_reward(completions, **kwargs):
    """答对得 1 分，答错得 0 分"""
    rewards = []
    for completion, answer in zip(completions, kwargs["answer"]):
        if extract_answer(completion) == answer:
            rewards.append(1.0)
        else:
            rewards.append(0.0)
    return rewards

def format_reward(completions, **kwargs):
    """格式正确（<answer>X</answer>）得 0.5 分"""
    rewards = []
    for completion in completions:
        if has_answer_tag(completion):
            rewards.append(0.5)
        else:
            rewards.append(0.0)
    return rewards

# 2. 训练
trainer = GRPOTrainer(
    model=base_model,
    reward_funcs=[accuracy_reward, format_reward],
    train_dataset=math_problems,
    args=GRPOConfig(
        output_dir="./grpo-reasoning",
        num_generations=8,  # ← 同一 prompt 采样 8 个
        per_device_train_batch_size=4,
        learning_rate=1e-6,
    ),
)

trainer.train()
```

---

## 8. 数据准备（最关键）

**数据质量 > 数量**：
- 100 条高质量 > 10k 低质量
- 标注者要少而精（2-3 个领域专家 > 100 个众包工人）

**数据来源**：
- **人工标注**：用 Label Studio / Argilla
- **GPT-4 生成**：蒸馏 GPT-4 的回答，再用人类 review
- **用户反馈**：从生产 agent 收集失败 case → 标正例
- **开源数据集**：Alpaca / Dolly / ShareGPT / OpenAssistant

**数据清洗**：
- 去重（用 MinHash / SimHash）
- 去敏感信息（regex + LLM 检查）
- 平衡（每个类别样本数）
- 校验（schema / 格式）

### 8.1 用 Argilla 做数据标注

```python
import argilla as rg

dataset = rg.Dataset(
    name="agent_sft",
    settings=rg.Settings(
        fields=[rg.TextField(name="prompt")],
        questions=[rg.TextQuestion(name="response", required=True)],
    ),
)

# 标注者填入 response
# 导出为 DataFrame
df = dataset.records.to_pandas()
```

### 8.2 用 GPT-4 蒸馏数据

```python
def generate_synthetic_data(prompts):
    for prompt in prompts:
        # 1. GPT-4 生成
        response = openai_client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
        )
        # 2. 人类 review（可采样 10% 验证质量）
        # 3. 写入训练集
        yield {"prompt": prompt, "response": response.choices[0].message.content}
```

---

## 9. 训练监控与评估

### 9.1 训练监控

```python
from transformers import TrainingArguments

args = TrainingArguments(
    output_dir="./output",
    logging_steps=10,            # 每 10 步 log
    eval_steps=100,             # 每 100 步 eval
    save_steps=500,             # 每 500 步 checkpoint
    eval_strategy="steps",
    report_to="wandb",          # 推到 W&B
    load_best_model_at_end=True,
)
```

**关键 metrics**：
- **Training loss**：应稳定下降
- **Eval loss**：应同步下降（不上升 = 没过拟合）
- **Reward**（RLHF/GRPO）：应上升
- **KL divergence**：不应增长太快（防止偏离太远）

### 9.2 模型评估

| 评估类型 | 工具 |
|----------|------|
| **通用能力** | lm-eval-harness, OpenCompass |
| **对话质量** | AlpacaEval, MT-Bench, Chatbot Arena |
| **Reasoning** | GSM8K, MATH, HumanEval |
| **Agent 能力** | SWE-bench, AgentBench |
| **业务能力** | 自建 eval 集（必须） |

**业务评估示例**：
```python
test_cases = [
    {"input": "退款流程是什么？", "expected_keywords": ["三步", "审核", "原路退回"]},
    {"input": "我的订单什么时候到？", "expected_keywords": ["订单", "时间"]},
]

for case in test_cases:
    response = model.generate(case["input"])
    score = check_keywords(response, case["expected_keywords"])
    print(f"Score: {score}")
```

---

## 10. 部署 Fine-tuned 模型

### 10.1 vLLM 部署

```bash
vllm serve ./merged_model \
    --tensor-parallel-size 1 \
    --gpu-memory-utilization 0.9 \
    --max-model-len 4096
```

### 10.2 用 OpenAI 兼容 API

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

response = client.chat.completions.create(
    model="./merged_model",
    messages=[{"role": "user", "content": "..."}],
)
```

### 10.3 监控线上表现

- 收集线上 query + response + 用户反馈
- 离线跑 eval 集
- 定期 retrain（用新数据）

---

## 11. Fine-tuning 常见坑

| 坑 | 表现 | 修复 |
|------|------|------|
| **数据太少** | loss 不下降 / 学不到 | 至少 1k 条；用 GPT-4 蒸馏 |
| **数据质量差** | 学会错误模式 | 人工 review；过滤低质样本 |
| **学习率太高** | loss 爆炸 / 模式崩溃 | 降到 1e-5 ~ 5e-5（SFT）/ 5e-7（DPO） |
| **学习率太低** | 学不到东西 | 升到 1e-4（LoRA） |
| **训练 epochs 太多** | 过拟合（train loss 0 / eval loss 上升） | 早停；3-5 epochs 通常够 |
| **OOM** | CUDA OOM | 用 QLoRA / gradient checkpointing / 减小 batch |
| **灾难性遗忘** | 通用能力下降 | 混合原始数据；用 KL 约束 |
| **复读问题** | 输出重复 | 降低 temperature；检查数据多样性 |

---

## 12. 完整流程（生产级）

```
1. 需求分析
   ↓
2. 数据收集
   - 内部数据 1k-10k 条
   - 标注 / review
   ↓
3. 数据清洗
   - 去重 / 去敏感
   - 平衡 / 校验
   ↓
4. Baseline 评估
   - 用 GPT-4o 跑测试集
   - 记录准确率 / 成本
   ↓
5. 选 base model
   - Qwen2.5 / Llama-3.1 / DeepSeek
   - 7B 通常够用
   ↓
6. SFT 训练
   - LoRA / QLoRA
   - 3 epochs
   ↓
7. 评估 + 调参
   - 看 loss 曲线
   - 跑测试集
   ↓
8. DPO（可选）
   - 收集偏好数据
   - DPO 训练 1 epoch
   ↓
9. 部署
   - vLLM serve
   - OpenAI 兼容 API
   ↓
10. 监控 + 持续改进
   - 收集线上数据
   - 定期 retrain
```

---

## 13. 面试视角

- **Q：什么时候该 fine-tuning vs prompt engineering？**
  → Fine-tuning：10k+ 标注数据 / 专属能力 / 降本 / 私有化。Prompt：几百条 / 通用 / 快速迭代。**RAG** 又是另一个维度（加知识）。

- **Q：LoRA / QLoRA / 全参数微调怎么选？**
  → 全参数：充足算力 + 极致效果。LoRA：算力有限 + 效果接近全参数。QLoRA：消费级 GPU 训 7B-13B。**80% 场景 LoRA 够用**。

- **Q：怎么准备 SFT 数据？**
  → 质量 > 数量。100 条高质量 > 10k 低质量。来源：人工标注 / GPT-4 蒸馏 + 人类 review / 收集线上数据。**必须做数据清洗**（去重 / 去敏感 / 平衡）。

- **Q：学习率怎么设？**
  → SFT：1e-5 ~ 5e-5（全参数）/ 1e-4 ~ 2e-4（LoRA）。DPO：5e-7 ~ 1e-6。**LoRA 学习率比全参数高 10x**。

- **Q：灾难性遗忘是什么？怎么避免？**
  → Fine-tuning 后模型丧失通用能力。原因：训练数据单一 + 训练太久。修复：① 混合原始通用数据 ② KL 约束（DPO 天然）③ 短训练（1-3 epochs）④ LoRA 而非全参数。

- **Q：怎么评估 fine-tuning 效果？**
  → 三层：① 训练曲线（loss / reward）② 通用 benchmark（lm-eval-harness）③ 业务 eval 集（自建）。**业务 eval 集最重要**。

- **Q：unsloth 是什么？优势？**
  → 第三方训练加速库。2-5x 加速 + 4-bit 量化 + 兼容 trl/peft。适合单卡消费级 GPU 训练。

---

## 14. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [HuggingFace TRL 文档](https://huggingface.co/docs/trl) | 官方训练库 |
| [LLaMA-Factory 文档](https://github.com/hiyouga/LLaMA-Factory) | 中文友好 |
| [unsloth 文档](https://docs.unsloth.ai/) | 资源友好训练 |
| [PEFT 文档](https://huggingface.co/docs/peft) | LoRA / QLoRA |
| [QLoRA 论文](https://arxiv.org/abs/2305.14314) | 4-bit 量化微调 |
| [LoRA 论文](https://arxiv.org/abs/2106.09685) | 低秩适配 |
| [LLaMA-Factory 500+ 模型支持](https://github.com/hiyouga/LLaMA-Factory) | 模型选择参考 |

---

## 15. 练习

1. **SFT 最小例子**：用 LLaMA-Factory 在 100 条 Alpaca 数据上 SFT Qwen2.5-0.5B
2. **LoRA 训练**：用 unsloth 在 7B 模型上 LoRA 微调，2 epochs
3. **QLoRA 训练**：用 unsloth 在 4-bit 量化下训练 13B 模型
4. **DPO 训练**：用 trl DPO trainer 在偏好数据上训练
5. **GRPO 训练**：用 trl GRPO 在数学题上训练（rule-based reward）
6. **数据合成**：用 GPT-4 蒸馏 1000 条 agent 数据
7. **业务 eval**：为自己的项目写 50 条测试集 + 自动化评估脚本
8. **部署**：用 vLLM 部署 fine-tuned 模型，OpenAI 兼容 API 调用

---

## 16. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U-DL 深度学习扫盲 | U32 是 U-DL 之后的进阶实战 |
| U1 LLM 基础 | 理解 base model 才能 fine-tune |
| U3 Prompt 工程 | Fine-tuning 和 prompt 是"改输出"的两种方式 |
| U14 DSPy | DSPy 自动 prompt 优化 = "soft fine-tuning" |
| U24 Reasoning Model | R1 训练全靠 GRPO，详见 U30 |
| U28 LLM Inference | Fine-tuned 模型用 vLLM 部署 |
| U30 Post-training | U30 是理论 + 算法，U32 是工程实战 |

---

*最后更新：2026-06-30 新增 U32（Fine-tuning 实践）*
*来源：LLaMA-Factory / trl / unsloth / PEFT 官方文档、LoRA / QLoRA 论文*
