# U24 · Reasoning Model / Test-time Compute（推理模型与测试时计算）

> **核心问题**：为什么 OpenAI o1/o3、DeepSeek R1、Gemini 2.0 Thinking 这类"推理模型"突然改变了 Agent 设计的范式？Agent 工程师该怎么用它们？
> **预计学时**：5h
> **对应 Phase**：Phase 2 / Phase 3
> **状态**：核心前沿（2024-09 OpenAI o1 发布后行业范式转折）

---

## 1. 什么是 Reasoning Model

**Reasoning Model**（推理模型，也叫 Thinking Model）是 2024-09 OpenAI 发布 o1 时正式命名的一类新型 LLM：

> "We trained [o1] to think before it answers. It produces a long internal chain of thought before responding to the user."
> —— OpenAI o1 System Card

**核心特征**：
- 在回答用户之前，先生成一段**长的内部 chain of thought**（推理链）
- 推理时**使用大量 test-time compute**（更多 token，更多思考步骤）
- 在数学、代码、科学、规划等任务上**显著超越 GPT-4o 类模型**
- 用户**看不到**内部思考链（或者只能看到摘要）

代表模型：
- **OpenAI o1 / o3 / o3-mini / o4-mini**
- **DeepSeek R1**（开源，2025-01）
- **Gemini 2.0 Flash Thinking / 2.5 Pro Thinking**
- **Claude 3.7 Sonnet with extended thinking**（Anthropic）
- **Qwen QwQ / Qwen3 系列**
- **Kimi K1.5**

---

## 2. 为什么这是 2025-2026 的范式转折

### 2.1 Test-Time Compute Scaling Law

传统 LLM 优化的两个杠杆：
- **Pre-training compute**（更多训练数据 + 更多参数）
- **Training-time compute**（更多 RLHF / post-training）

Reasoning Model 引入了**第三个杠杆**：
- **Test-time compute**（推理时用更多 token 思考）

> "The more the model thinks, the better it answers." 

OpenAI o1 内部已经验证：在数学竞赛 AIME、博士级科学问答 GPQA、Codeforces 编程等任务上，**o1 思考越久，准确率越高**。这是 LLM 领域第一次有"inference-time scaling" 的明确证据。

### 2.2 对 Agent 设计的影响

**改写 Agent 工作流**：

| 旧范式（GPT-4o 时代） | 新范式（Reasoning Model 时代） |
|------------------------|-------------------------------|
| Agent 自己做 planning | 让 Reasoning Model 内部做深度推理 |
| 多个 ReAct step 串起来 | 一个 deep think 直接出 plan |
| Prompt 复杂 + Few-shot | Prompt 简洁（模型自带思考能力） |
| 实时性优先 | 质量优先（允许秒级-分钟级等待） |
| Function calling 频繁 | 思考完一次，action 步骤更少更准 |

**关键洞察**：Reasoning Model 改变了"Agent Loop"的形状——**ReAct 的循环次数变少了，但每次循环的"思考"成本变高了**。

### 2.3 实际影响

- **SWE-bench Verified 排行榜**：2024 末到 2025 初，Reasoning Model 把 coding agent 准确率从 ~30% 拉到 ~70%
- **数学 / 科学问答**：从 GPT-4o 的 ~50% 拉到 o1 的 ~90%+
- **多步规划任务**：Reasoning Model 几乎不需要外部 planner，自己就能拆解

---

## 3. Reasoning Model 怎么用

### 3.1 OpenAI o1/o3 API 用法

```python
from openai import OpenAI
client = OpenAI()

# 关闭 reasoning effort（适合低延迟场景）
response = client.chat.completions.create(
    model="o3-mini",
    messages=[{"role": "user", "content": "解这道数学题：..."}],
    reasoning_effort="low",  # low / medium / high
)

# 高 reasoning effort（更准但更慢）
response = client.chat.completions.create(
    model="o1",
    messages=[{"role": "user", "content": "..."}],
    reasoning_effort="high",
)
```

`reasoning_effort` 参数控制思考预算（low / medium / high），等价于控制 test-time compute 投入。

### 3.2 DeepSeek R1 实战

```python
# DeepSeek R1 是开源的，可以自部署
import requests

response = requests.post(
    "http://localhost:8000/v1/chat/completions",
    json={
        "model": "deepseek-reasoner",
        "messages": [
            {"role": "user", "content": "用动态规划解这道题..."}
        ],
    },
)
# 返回包含 reasoning_content（思考过程）和 content（最终答案）
result = response.json()
print("思考:", result["choices"][0]["message"]["reasoning_content"])
print("答案:", result["choices"][0]["message"]["content"])
```

### 3.3 Claude 3.7 Extended Thinking

```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-3-7-sonnet-20250219",
    max_tokens=20000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000,  # 思考 token 上限
    },
    messages=[{"role": "user", "content": "..."}],
)
# 返回包含 thinking（思考）和 text（答案）
```

### 3.4 Anthropic Prompt 缓存 + Extended Thinking

可以组合使用：**prompt caching 节省 input token，extended thinking 用更多 output token 思考**——这是生产推理 Agent 的最佳实践之一。

---

## 4. Reasoning Model 的 5 大设计模式

### 4.1 何时用 Reasoning Model vs 标准 LLM

| 任务 | 推荐 | 理由 |
|------|------|------|
| 闲聊 / 简单 QA | 标准 LLM（GPT-4o / Claude 3.5） | Reasoning Model 慢且贵 |
| 数学 / 科学 / 编程 | Reasoning Model | 显著优势 |
| 多步规划 / 决策 | Reasoning Model | 能内部拆解 |
| 实时对话（< 1s 响应） | 标准 LLM | 延迟太大 |
| Agent 中的"思考者"角色 | Reasoning Model | 见下文 |
| Agent 中的"快执行"角色 | 标准 LLM | 见下文 |

### 4.2 双模型协作：Thinker + Doer

生产级 Agent 系统的常见架构：
- **Thinker（Reasoning Model）**：负责规划、决策、复杂推理、code review
- **Doer（标准 LLM）**：负责简单执行、工具调用结果解析

```python
def agent_loop(task):
    # Thinker 阶段：深度推理 + 拆解
    plan = reasoning_llm.invoke(f"为以下任务制定计划：{task}")
    steps = parse_plan(plan)

    # Doer 阶段：标准 LLM 执行每个 step
    results = []
    for step in steps:
        result = standard_llm.invoke(
            f"执行：{step}\n可用工具：..."
        )
        results.append(result)
    return aggregate(results)
```

### 4.3 Cascade：先快后慢

```python
def smart_solve(question):
    # 1. 先用标准 LLM 快速试解
    quick_answer = standard_llm.invoke(question)

    # 2. 如果置信度低，升级到 Reasoning Model
    if needs_deep_thinking(quick_answer):
        deep_answer = reasoning_llm.invoke(
            f"问题：{question}\n之前的快速答案：{quick_answer}\n"
            f"请深度思考后给出更准确的答案。"
        )
        return deep_answer
    return quick_answer
```

节省 token 成本：90% 的简单问题用快模型，10% 的难题升级到推理模型。

### 4.4 推理模型 + RAG

Reasoning Model 配合 RAG：
- **检索阶段用快模型**（embed + ANN 检索）
- **生成阶段用推理模型**（深度合成 + 引用核对）

Reasoning Model 擅长**多跳推理 + 引用验证**，特别适合复杂 RAG。

### 4.5 推理模型 + Tool Use

OpenAI o1 / o3 / DeepSeek R1 都支持 tool use，但要注意：
- **Reasoning Model 的 tool call 之前会有内部思考**（看不到）
- **不要在 Reasoning Model 上做频繁的小 tool call**（每次都要重新思考）
- **适合"少而精"的 tool use**——一次深度推理后做关键 tool 调用

---

## 5. Reasoning Model 的局限

| 局限 | 说明 |
|------|------|
| **慢** | 思考一次可能 10s-60s+，不适合实时场景 |
| **贵** | o1 的 API 价格是 GPT-4o 的 5-10 倍，R1 因为开源便宜很多 |
| **不可控** | 你看不到内部思考，只能调 `reasoning_effort` |
| **不能保证正确** | 还是可能犯错，只是比标准 LLM 错得少 |
| **streaming 弱** | 思考过程往往不能流式（一次性返回），影响 UX |
| **对 prompt 敏感度低** | 复杂 prompt engineering 收益变小，反而简化 |

> **关键洞察**：Reasoning Model 不是"更好的 LLM"，是**不同范式的 LLM**。它替代不了一切，需要根据任务选型。

---

## 6. 面试视角

- **Q：什么是 Reasoning Model？它和标准 LLM 有什么区别？**
  → Reasoning Model 在回答前会内部生成长的 chain of thought，使用更多 test-time compute 换取准确率。代表：o1/o3/R1/Gemini Thinking。标准 LLM 直接采样 token，Reasoning Model 内部"思考"。

- **Q：Test-time compute scaling law 是什么意思？**
  → 与 pre-training scaling law 不同，test-time compute scaling 指**推理时用更多计算**（更多思考 token、更多搜索路径、更多 self-consistency）能换来更高准确率。o1 在 AIME、GPQA、Codeforces 等任务上验证了这一点。

- **Q：Agent 怎么用 Reasoning Model？**
  → 三种模式：① 双模型协作（Thinker + Doer）② Cascade（先快后慢）③ 推理模型 + RAG/Tool（深度合成 + 关键工具调用）。**不要**用 Reasoning Model 做频繁的小 tool call。

- **Q：Reasoning Model 有什么局限？**
  → 慢（10s-60s+）、贵（API 价格 5-10x）、不可控（看不到内部）、对 prompt engineering 收益小、streaming 弱。

- **Q：o1 / o3 / R1 / Claude 3.7 Thinking 怎么选？**
  → 闭源：OpenAI o1/o3（最强）、Claude 3.7 Extended Thinking（agent 友好）、Gemini 2.5 Thinking（多模态）。开源：DeepSeek R1（性价比）、Qwen QwQ/Qwen3（中文友好）。

- **Q：ReAct 还需要吗？Reasoning Model 是不是替代了 ReAct？**
  → 不替代，但改变了 ReAct 的形状。Reasoning Model 让单次 step 更准，所以 ReAct 的循环次数变少。**Hybrid**：Reasoning Model 内部推理 + 标准 LLM/ReAct 做执行。

---

## 7. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [OpenAI o1 System Card](https://openai.com/index/openai-o1-system-card/) | 官方权威定义 test-time compute |
| [OpenAI o3 Blog](https://openai.com/index/openai-o3/) | o3 在 AIME / GPQA / Codeforces 上的成绩 |
| [OpenAI Learning to Reason with LLMs](https://openai.com/index/learning-to-reason-with-llms/) | o1 的训练方法解释（RL + process reward） |
| [DeepSeek R1 Paper](https://arxiv.org/abs/2501.12948) | 开源推理模型的代表，方法完全公开 |
| [Anthropic: Claude 3.7 Sonnet and Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet) | Extended thinking 模式的设计 |
| [Karpathy on o1](https://x.com/karpathy/status/1836503993638502597) | "We are now in the test-time compute scaling era" |
| [Simon Willison: o1 and the test-time compute inflection point](https://simonwillison.net/2024/Sep/12/openai-o1/) | 通俗解释 |
| [Latent Space: Reasoning Models](https://www.latent.space/) | 行业访谈系列 |

---

## 8. 练习

1. **对比实验**：用同一道 AIME 数学题，分别用 GPT-4o / o1 / DeepSeek R1 解，记录正确率和延迟
2. **reasoning_effort 调参**：用 o3-mini 跑同一题，分别设 `reasoning_effort="low"` 和 `"high"`，对比准确率/成本/延迟
3. **Cascade 实现**：实现一个"先 GPT-4o 试，置信度低升级 o1" 的 Cascade Agent
4. **Thinker + Doer**：用 o1 做架构设计，用 GPT-4o-mini 做后续代码生成，对比纯 GPT-4o 的方案
5. **RAG + Reasoning**：用 RAG 检索 5 段文档，让 R1 深度合成并核对引用，验证 R1 在多跳 RAG 上的优势
6. **观察内部思考**：用 DeepSeek R1 的 `reasoning_content` 字段观察模型实际思考过程，理解它什么时候会"卡住"

---

## 9. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U1 LLM 基础 | Reasoning Model 是 LLM 发展的新范式，补充 next-token prediction 视角 |
| U3 Prompt Engineering | Reasoning Model 让 prompt 简化（模型自带思考），但 system prompt 仍重要 |
| U5 ReAct 范式 | **Reasoning Model 改变了 ReAct 的形状**——单步更深，循环更少 |
| U7 Planning & Reasoning | Reasoning Model 内部自带 planning 能力，可部分替代外部 planner |
| U9 Multi-Agent | 双模型协作（Thinker + Doer）是一种 Multi-Agent 模式 |
| U20 编程 Agent 内部原理 | SWE-bench 排行榜上 Reasoning Model 已经把 coding agent 推到新高度 |
| U22 Context Engineering | Reasoning Model + Extended Thinking + Prompt Caching 是 Context Engineering 的新维度 |
| U28 LLM Inference & Cost（U28） | Reasoning Model 改变了 token economics，详见 U28 |

### 在分层中的位置

```
            ┌───────────────────────────────┐
            │   应用：Agent Loop / RAG      │
            ├───────────────────────────────┤
            │   Reasoning Model (U24) ★     │  ← 新增维度
            │   + Standard LLM              │
            ├───────────────────────────────┤
            │   Context (U22) + Harness (U23)│
            ├───────────────────────────────┤
            │   Inference (U28)             │  ← 详见 U28
            │   KV cache / batching / cost  │
            ├───────────────────────────────┤
            │   Model (U1)                  │
            └───────────────────────────────┘
```

---

*最后更新：2026-06-30 新增 U24（Reasoning Model / Test-time Compute）*
*来源：OpenAI o1/o3 官方文档、DeepSeek R1 论文、Anthropic Claude 3.7 博客、Karpathy 评论*
