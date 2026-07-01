# U1 · LLM 基础

> **核心问题**：GPT 是怎么生成下一个词的？
> **预计学时**：5h
> **对应 Phase**：Phase 1

---

## 学习目标（Gate 检验）

1. 能用一句话解释 "next-token prediction" 是什么
2. 能解释 Temperature 和 Top-p 采样的作用和区别
3. 能解释 Context Window 对 Agent 设计的限制
4. 能解释 Instruction Tuning 和 RLHF 在做什么

---

## 1. Next-Token Prediction：本质的理解

### 直观类比

GPT 不是在"回答问题"，而是在"续写文本"。

就像你用手机输入法打字的**联想提示**——它根据前文预测下一个最可能的字。区别只是 GPT 用了几千亿参数来预测下一个 token（词片段）。

### 核心要点

- 模型输入：一段文本（context）
- 模型输出：下一个 token 的**概率分布**
- "回答"只是 context 恰好以问句结尾时，续写出来的内容恰好像答案

### 常见误解

| 误解 | 真相 |
|------|------|
| GPT 知道正确答案 | GPT 生成的是统计上最可能的续写，不保证事实正确 |
| GPT 有记忆 | 没有，每次调用都是用 context window 内的所有 token |
| GPT 理解了问题 | GPT 是模式匹配，看到"中国的首都是"会续写"北京" |

### 参考

- [Jay Alammar: The Illustrated ChatGPT](https://jalammar.github.io/illustrated-chatgpt/)（图解，30 分钟）
- [Andrej Karpathy: GPTs are predict-the-next-token](https://www.youtube.com/watch?v=kexM9Ars3Mw)（视频 1h，需要翻墙）

---

## 2. Transformer 和 Attention：直观理解

### 不需要会训练，但需要能说清楚

**Attention 机制在做什么：**

当模型生成每个 token 时，会"看一眼"输入序列中所有其他 token，根据相关性加权。

类比：你在读一句话时，不会每个字孤立理解，而是会根据上下文确定每个字的含义。Attention 就是让模型做到这件事的机制。

### 三个关键点（面试能说出来即可）

1. **Self-Attention**：输入的每个 token 和所有其他 token 做相关性打分
2. **Multi-Head Attention**：多组 Attention 并行，捕捉不同类型的相关性（语法、语义、位置等）
3. **位置编码**：Attention 本身不看位置，需要额外加位置信息（原始用 sin/cos，后续有 RoPE 等改进）

### 参考

- [3Blue1Brown: Transformers](https://www.youtube.com/watch?v=eMlx5fFNoKY)（15 分钟，最好看中英字幕）
- [Jay Alammar: The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)

---

## 3. Temperature / Top-p 采样

### 为什么同一个问题两次回答不一样？

因为生成是**概率采样**，不是取最大值。

### Temperature

控制输出的随机性：
- `temperature = 0`：每次都取概率最高的 token（确定性输出）
- `temperature = 1.0`：按原始概率分布采样
- `temperature > 1.0`：增加随机性
- `temperature < 1.0`：减少随机性，更保守

公式直觉：Temperature 是在 softmax 前对 logits 做缩放。T 高 → 分布更平滑（各种结果都有机会）；T 低 → 分布更尖锐（更确定的结果占据主导）。

### Top-p (Nucleus Sampling)

另一种控制方式：选累积概率达到 p 的最小 token 集合，在这个集合里采样。

例：`top_p = 0.9`，意思是选概率之和达到 90% 的最小 token 集合，在这个集合里按概率采样。

### 实战建议

| 场景 | Temperature | Top-p | 原因 |
|------|-----------|-------|------|
| 代码生成 / 结构化输出 | 0 ~ 0.3 | 0.9 | 需要稳定可复现的结果 |
| 创意写作 / 头脑风暴 | 0.7 ~ 1.0 | 0.95 | 需要多样性 |
| Agent 工具调用 | 0 ~ 0.3 | 0.9 | Tool 选择要稳定可靠 |
| RAG 生成答案 | 0.3 ~ 0.6 | 0.9 | 平衡准确性和多样性 |

---

## 4. Context Window

### 本质

Context Window 是模型一次能处理的最大 token 数（输入 + 输出共享）。

常见模型：
- GPT-4 Turbo：128k tokens
- Claude 3.5：200k tokens
- DeepSeek Chat：64k tokens
- GPT-4o Mini：128k tokens

### 对 Agent 设计的影响

1. **历史对话不能无限累积**：超过 context window 的历史会被截断
2. **RAG 是解决方案之一**：把长历史的关键信息提炼到向量库，需要时检索回来
3. **Context Compaction**：长 session 需要主动压缩历史（如 Claude Code 的处理方式）

### Token 计算工具

- [OpenAI Tokenizer](https://platform.openai.com/tokenizer)：自己动手查
- 中文粗略估算：1 个中文字 ≈ 1~2 tokens（GPT-4 的话）

---

## 5. Instruction Tuning / RLHF：怎么来的

### 三个阶段（简化版）

**阶段 1：预训练（Pre-training）**
- 数据：互联网大规模文本
- 目标：预测下一个 token
- 结果：学会了语言规律，但不会问答

**阶段 2：Instruction Tuning（SFT）**
- 数据：人工标注的问答对
- 目标：给定 instruction，生成正确 response
- 结果：能回答问题了，但回答质量参差不齐

**阶段 3：RLHF（Reinforcement Learning from Human Feedback）**
- 用 Human Feedback 训练一个 Reward Model
- 用 Reward Model 指导模型进一步优化
- 结果：回答更符合人类偏好，更安全，更有帮助

### 面试能说的话

> "GPT 预训练是学语言规律，SFT 是教它听从指令，RLHF 是用人类反馈让它回答得更符合期望。"

### 参考

- InstructGPT 论文（[arXiv:2212.09760](https://arxiv.org/abs/2212.09760)）：读 Abstract + Section 3 即可，2h

---

## 6. 主流模型对比

| 模型 | 上下文 | 擅长 | 价格 |
|------|--------|------|------|
| GPT-4o | 128k | 全能，工具调用稳定 | 中高 |
| GPT-4o Mini | 128k | 性价比，日常任务 | 低 |
| Claude 3.5 Sonnet | 200k | 长上下文、安全、代码 | 中高 |
| DeepSeek V3 | 64k | 性价比、中文 | 低 |
| Qwen 2.5 | 128k | 开源、中文、工具调用 | 低 |

---

## 练习（必须动手）

### 练习 1：Token 计算（10 分钟）

```python
# 用 tiktoken 计算 token 数
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")  # GPT-4 的编码器

texts = [
    "你好",
    "北京是中国的首都",
    "Please help me write a Python function that calculates the factorial of a number.",
]

for text in texts:
    tokens = enc.encode(text)
    print(f"文本: {text}")
    print(f"Token数: {len(tokens)}")
    print()
```

### 练习 2：Temperature 对比（15 分钟）

```python
import openai
import os

client = openai.OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

question = "给我一个健康的早餐建议，要包含碳水、蛋白质和蔬菜。"

responses = []
for temp in [0.0, 0.5, 1.0, 1.5]:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": question}],
        temperature=temp,
        max_tokens=200,
    )
    print(f"=== Temperature={temp} ===")
    print(response.choices[0].message.content)
    print()
```

观察：temperature 越高，输出越多样/越不稳定。

### 练习 3：Context Window 限制（10 分钟）

```python
# 构造一个超过 context 的超长文本，看模型怎么处理
long_text = "这是一个测试句子。" * 2000  # 约 10000 tokens

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": f"请总结这段文字：{long_text}"}],
    max_tokens=50,
)
# 会报错或自动截断
```

---

## 完成 Gate 检验

回答以下问题，不需要背，理解后能说出来即可：

1. **Q1**：为什么 GPT 有时候会"胡说八道"（幻觉）？
   - 答案提示：它是在"续写"，不是"查数据库"，续写内容是统计上最可能的，不是事实正确的。

2. **Q2**：Temperature = 0 和 Temperature = 1.0 的核心区别是什么？
   - 答案提示：前者取最确定的输出（贪婪），后者按原始概率分布采样（有随机性）。

3. **Q3**：Context Window 满了怎么办？Agent 系统有哪些应对方案？
   - 答案提示：①截断（丢失早期信息）②压缩（Summarization）③RAG（外部向量库）④滑动窗口（只保留最近 N 条）。

4. **Q4**：预训练、SFT、RLHF 分别在学什么？
   - 答案提示：预训练=学语言规律；SFT=学听从指令；RLHF=学符合人类偏好。

---

## 关联资源

- [OpenAI Tokenizer](https://platform.openai.com/tokenizer)
- [tiktoken](https://github.com/openai/tiktoken)
- Jay Alammar Blog（所有图解文章，质量高）
- [3Blue1Brown 神经网络系列](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZC07-PI9)

---


## 7. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U3 Prompt 工程|LLM 是 prompt 的底座，U3 的所有技巧建立在 U1 的模型能力之上|
|U24 Reasoning Model|Reasoning Model 是 LLM 的新范式，扩展了 U1 的 next-token prediction 视角|
|U28 LLM Inference|U1 讲模型是什么，U28 讲怎么高效调用模型|
|U30 Post-training|U1 的 Instruction Tuning / RLHF 在 U30 深度展开|
|U42 面试指南|U1 的基础知识是 U42 面试题的底层|

## 8. Attention 变体与现代优化（面试加分）

### 8.1 RoPE（Rotary Position Embedding）

RoPE 把位置信息编码到 attention 计算中，让模型能**外推**到比训练时更长的序列：
- 原始 Transformer 用 sin/cos 绝对位置编码，超过训练长度效果骤降
- RoPE 用相对位置编码，推理时能处理 2x-4x 长度的输入
- 代表模型：LLaMA / Qwen / DeepSeek 全部用 RoPE

### 8.2 GQA / MQA / MLA

|变体|说明|优势|
|---|---|---|
|**MHA** (Multi-Head Attention)|原始多头，每个 head 独立 Q/K/V|质量最高，但 KV cache 最大|
|**MQA** (Multi-Query Attention)|所有 head 共享一个 K/V|KV cache 省 8x，质量略降|
|**GQA** (Grouped-Query Attention)|若干 head 共享一个 K/V|平衡方案（LLaMA-2 用）|
|**MLA** (Multi-head Latent Attention)|DeepSeek 提出，低秩压缩 KV|KV cache 省 93%，质量无损|

**面试能说的话**：'GQA 是 MHA 和 MQA 的折中，LLaMA-2 用它来平衡质量和推理速度。MLA 是 DeepSeek V3 的发明，用低秩联合压缩 KV cache，让 671B 模型也能在单节点推理。'

### 8.3 KV Cache 优化

KV Cache 是推理时的内存瓶颈：
- **问题**：每个 token 的 K/V 都要缓存，batch size * seq_len * hidden_dim * 2 个矩阵
- **优化**：MQA/GQA 减少 KV head 数 / MLA 压缩 / PagedAttention（vLLM）动态分配

### 8.4 2025-2026 新模型速查

|模型|上下文|特点|
|---|---|---|
|DeepSeek V3|128k|MoE 671B 激活 37B，MLA 压缩|
|Qwen 2.5|128k|开源最强之一，支持 29 语言|
|LLaMA 3.1|128k|405B 参数，Meta 开源|
|GPT-4o|128k|多模态原生|
|Claude 3.5 Sonnet|200k|代码 + 安全最强|

*最后更新：2026-05-26*