# Reasoning Model 面试题集

> **范围**：U24 Reasoning Model / Test-time Compute
> **来源**：U24-ReasoningModel.md + OpenAI / DeepSeek / Anthropic 官方文档

---

## 目录

1. [基础概念（Q1–Q4）](#1-基础概念q1q4)
2. [Test-Time Compute Scaling Law（Q5–Q7）](#2-test-time-compute-scaling-lawq5q7)
3. [模型选型与对比（Q8–Q11）](#3-模型选型与对比q8q11)
4. [工程实践：Reasoning Effort / Thinker+Doer / Cascade（Q12–Q15）](#4-工程实践reasoning-effort--thinkerdoer--cascadeq12q15)
5. [RAG / Tool Use / Agent 集成（Q16–Q18）](#5-rag--tool-use--agent-集成q16q18)
6. [局限与权衡（Q19）](#6-局限与权衡q19)
7. [面试综合题（Q20）](#7-面试综合题q20)

---

## 1. 基础概念（Q1–Q4）

---

### Q1：什么是 Reasoning Model？和标准 LLM 有什么区别？

**答案**：

Reasoning Model（又称 Thinking Model）是 2024-09 OpenAI 发布 o1 时正式命名的一类新型 LLM。核心特征：**在给出最终答案前，先内部生成长链的 chain-of-thought（CoT），并主动使用更多的 test-time compute**。

| 维度 | 标准 LLM | Reasoning Model |
|------|----------|-----------------|
| 输出方式 | 直接采样下一个 token | 先生成内部推理链，再输出答案 |
| 思考过程 | 对用户可见（或无显式思考） | 内部 CoT（通常不暴露） |
| 计算投入 | 固定 | 可根据难度动态投入更多计算 |
| 擅长任务 | 开放对话、创意写作 | 数学、代码、科学、多步规划 |
| 延迟 | 秒级 | 10s–60s+ |
| 价格 | 基准 | o1 API ≈ GPT-4o 的 5–10 倍 |

代表模型：**OpenAI o1/o3/o3-mini/o4-mini**、**DeepSeek R1**、**Gemini 2.0/2.5 Thinking**、**Claude 3.7 Sonnet + Extended Thinking**、**Qwen QwQ / Qwen3**。

> **面试追问**：为什么叫 "Test-time Compute Scaling"？
> **追问答案**：传统 LLM 的 scaling 发生在 pre-training 或 post-training 阶段。Reasoning Model 引入了第三个 scaling 杠杆——**推理时投入更多 compute 也能换准确率提升**。

```python
# 标准 LLM：直接返回答案
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "解这道微积分题"}],
)

# Reasoning Model：内部先推理再回答
response = openai.chat.completions.create(
    model="o1",
    messages=[{"role": "user", "content": "解这道微积分题"}],
    reasoning_effort="high",
)
```

**面试追问**：Reasoning Model 写诗好吗？
**追问答案**：不一定好。创意生成任务可能因为过度推理而显得生硬。**选模型要看任务匹配度**。

---

### Q2：Test-Time Compute Scaling Law 是什么？和 Pre-training Scaling Law 有什么区别？

**答案**：

- **Pre-training Scaling Law**：更多训练数据 + 更多参数 → 模型更聪明（训练阶段 compute 换性能）。
- **Test-Time Compute Scaling Law**：推理时投入更多 compute → 单次回答更准（推理阶段 compute 换性能）。

OpenAI o1 在 **AIME**（美国数学竞赛）、**GPQA**（博士级科学问答）、**Codeforces**（编程竞赛）上验证了 test-time scaling：思考时间越长，准确率越高。

> **面试追问**：这个 law 有没有边界？
> **追问答案**：有。闲聊、事实性 QA 多想想收益很小甚至为负。此外 compute 投入存在边际递减。

---

### Q3：o1 / o3 / o3-mini / o4-mini 分别是什么关系？

**答案**：

| 模型 | 定位 | 特点 |
|------|------|------|
| **o1** | 第一代推理模型 | 内部 CoT，固定推理深度 |
| **o3** | 第二代主力 | 比 o1 更强，支持 `reasoning_effort` 参数 |
| **o3-mini** | 性价比选择 | 速度更快、价格更低，支持 low/medium/high |
| **o4-mini** | 轻量快速 | 最新，适合 Agent 高频调用 |

**记忆口诀**：o1 "开山"，o3 "旗舰"，o3-mini "性价比"，o4-mini "轻量"。

> **面试追问**：o1 为什么没有 reasoning_effort 参数？
> **追问答案**：o1 是固定推理深度，o3 系列才开放了这个旋钮。生产环境优先用 o3-mini。

---

### Q4：DeepSeek R1 和 Claude 3.7 Extended Thinking 的特点对比？

**答案**：

| 特性 | DeepSeek R1 | Claude 3.7 + Extended Thinking |
|------|-------------|--------------------------------|
| 开源 | ✅ | ❌ |
| 思考过程可见 | ✅ `reasoning_content` | ✅ `thinking` block |
| 思考 token 可控 | 间接 | ✅ `budget_tokens` |
| 价格 | 低（可自部署） | 中（API 计费） |
| 中文能力 | 强 | 中（英文为主） |

```python
# DeepSeek R1：可见思考过程
resp = requests.post("http://localhost:8000/v1/chat/completions", json={...})
print(resp.json()["choices"][0]["message"]["reasoning_content"])

# Claude 3.7 Extended Thinking
response = client.messages.create(
    model="claude-3-7-sonnet-20250219",
    thinking={"type": "enabled", "budget_tokens": 10000},
    max_tokens=20000,
    messages=[...],
)
```

> **面试追问**：R1 的 `reasoning_content` 在生产环境怎么用？
> **追问答案**：调试（看模型卡在哪一步）、教学（展示推理过程）、日志审计。**注意隐私**：思考过程可能含敏感信息。

---

## 2. Test-Time Compute Scaling Law（Q5–Q7）

---

### Q5：Test-time compute scaling 有哪些具体实现方式？

**答案**：

| 方式 | 说明 | 代表 |
|------|------|------|
| **延长思考时间** | 让模型多生成思考 token | o3 的 `reasoning_effort="high"` |
| **Self-Consistency** | 同 prompt 采样 N 次，多数投票 | CoT + 多数投票 |
| **Best-of-N** | 生成 N 个答案，reward model 选最优 | o1 内部训练方法 |
| **Tree Search / MCTS** | 推理时搜索多个推理路径 | 研究阶段 |
| **Process Reward** | 对每一步推理打分 | o1 训练用 PRM |
| **Verifier** | 推理后加验证模型检查答案 | 外部 verifier |

```python
def self_consistency(model, prompt, n=5):
    answers = [extract_answer(model.generate(prompt)) for _ in range(n)]
    from collections import Counter
    return Counter(answers).most_common(1)[0][0]
```

> **面试追问**：Self-consistency 的代价？
> **追问答案**：N 倍 API 调用成本 + N 倍延迟。适合高价值任务（医疗、法律），不适合普通客服。

---

### Q6：为什么 test-time compute scaling 是 2025 的范式转折？

**答案**：

**四点核心影响**：

1. **Agent Loop 形状变了**：ReAct 循环从 N 次（浅思考）→ M 次（深思考，M << N）
2. **Planning 成本内化**：Reasoning Model 内部自带 planning，外部 planner 需求降低
3. **Prompt Engineering 收益变小**：模型自带推理，prompt 可以大幅简化
4. **排行榜刷新**：SWE-bench ~30%（GPT-4o）→ ~70%（o1）；AIME ~50% → ~90%+

> **面试追问**：这对 prompt engineer 的职业有什么影响？
> **追问答案**：不是失业，而是工作重心从 "教模型怎么推理" 转向 "设计系统架构（选哪个模型 + 怎么编排）"。System prompt 设计仍然重要。

---

### Q7：Reasoning Model 的内部 CoT 和显式 CoT（"Let's think step by step"）有什么区别？

**答案**：

| 维度 | 显式 CoT | 内部 CoT（Reasoning Model） |
|------|---------|----------------------------|
| 触发方式 | Prompt 中写 "Let's think step by step" | 模型训练自动触发 |
| 思考过程可见性 | 可见（输出到 assistant） | 不可见（黑盒） |
| 思考深度 | 受输出长度限制 | 受 `reasoning_effort` 控制 |
| 可靠性 | 弱（容易跳过） | 强（训练时强化） |
| 灵活性 | 灵活 | 不灵活（只能调 effort） |

> **面试追问**：显式 CoT 还有用吗？
> **追问答案**：有，但用途变了。Reasoning Model 的 prompt 可以更简洁，不需要 "Let's think step by step"。显式 CoT 更适合标准 LLM 或轻量增强场景。

---

## 3. 模型选型与对比（Q8–Q11）

---

### Q8：o1 / o3 / R1 / Claude 3.7 Thinking / Gemini 2.5 Thinking 怎么选？

**答案**：

| 场景 | 推荐 | 理由 |
|------|------|------|
| 最强推理能力 | OpenAI o3 | 数学、代码、科学综合最强 |
| 性价比 + 开源 | DeepSeek R1 | 可自部署，价格低 |
| Agent 友好 | Claude 3.7 + Extended Thinking | prompt cache + extended thinking |
| 多模态推理 | Gemini 2.5 Pro Thinking | 支持图像 + 文本混合推理 |
| 快速原型 | o3-mini low | 便宜，延迟低 |
| 中文场景 | DeepSeek R1 / Qwen QwQ | 中文原生训练 |
| 实时场景（< 1s） | ❌ 不用 Reasoning Model | 延迟太高 |

> **面试追问**：为什么 Claude 3.7 对 Agent 友好？
> **追问答案**：Anthropic 原生支持 prompt caching（节省 90% 重复 token 成本）+ extended thinking（`budget_tokens` 精确控制）+ tool use。三者组合是生产 Agent 最佳实践。

---

### Q9：Reasoning Model 的价格结构怎么算？

**答案**：

| 模型 | Input Price | Output Price | 备注 |
|------|-------------|--------------|------|
| GPT-4o | $2.50/1M | $10.00/1M | 基准 |
| o1 | $15.00/1M | $60.00/1M | ≈ 6x GPT-4o |
| o3-mini | $1.10/1M | $4.40/1M | ≈ 2x GPT-4o |
| Claude 3.7 Sonnet | $3.00/1M | $15.00/1M | extended thinking 按 output 计费 |
| DeepSeek R1（API） | $0.55/1M | $2.19/1M | 低 |
| DeepSeek R1（自部署） | ~$0.10/1M | ~$0.40/1M | 大批量更优 |

**成本公式**：
$$
\text{总成本} = \text{input\_tokens} \times \text{input\_price} + \text{output\_tokens} \times \text{output\_price}
$$

Reasoning Model 的 output tokens 包含**思考 token + 答案 token**，可能比标准 LLM 多 5-20 倍。

> **面试追问**：如果 thought token 不计费怎么办？
> **追问答案**：o1 早期有 "thinking token 不计费" 政策，但 o3 系列已全部计入 output token。做成本估算时**必须按 output token 全算**。

---

### Q10：Gemini 2.0 Flash Thinking 和 2.5 Pro Thinking 的区别？

**答案**：

| 维度 | 2.0 Flash Thinking | 2.5 Pro Thinking |
|------|-------------------|------------------|
| 推理速度 | 快 | 慢 |
| 推理质量 | 中等 | 高 |
| 多模态 | 支持 | 支持（更强） |
| 价格 | 低 | 高 |
| 上下文长度 | 1M tokens | 1M tokens |

> **面试追问**：Gemini 的 thinking 和 OpenAI 的有什么不同？
> **追问答案**：Gemini 的 thinking token 以 `thought` 字段返回（可配置是否暴露），OpenAI o3 系列完全对用户隐藏。Gemini 在可观察性上略好。

---

### Q11：Qwen QwQ / Qwen3 系列在中国市场的定位？

**答案**：

| 模型 | 特点 |
|------|------|
| **Qwen QwQ-32B-Preview** | 开源推理模型，中文能力强 |
| **Qwen3** | 最新系列，含推理增强版本 |

**定位**：中文友好、可自部署、价格优势（比 OpenAI 低一个数量级）、企业合规（数据不出境）。

---

## 4. 工程实践：Reasoning Effort / Thinker+Doer / Cascade（Q12–Q15）

---

### Q12：`reasoning_effort` 参数怎么调？low / medium / high 分别适用什么场景？

**答案**：

| 参数 | 思考深度 | 适用场景 | 延迟 | 成本 |
|------|---------|---------|------|------|
| `low` | 浅思考 | 简单 QA、闲聊、分类 | 2-5s | 低 |
| `medium` | 中等 | 一般编程题、中等数学题 | 5-15s | 中 |
| `high` | 深度 | 竞赛数学、复杂规划、code review | 15-60s+ | 高 |

```python
def adaptive_solve(question, complexity_hint):
    effort_map = {"easy": "low", "medium": "medium", "hard": "high"}
    effort = effort_map.get(complexity_hint, "medium")
    return client.chat.completions.create(
        model="o3-mini", reasoning_effort=effort,
        messages=[{"role": "user", "content": question}],
    )
```

**调参策略**：
1. 默认 `medium`，简单问题 `low`，难题 `high`
2. 设置 timeout 兜底，超时降级到标准 LLM
3. A/B 测试：同一问题跑不同 effort，对比准确率和延迟

> **面试追问**：`reasoning_effort` 有没有量化指标？
> **追问答案**：OpenAI 没有公开具体 token 预算。实践中 low ≈ 1k-3k 思考 token，high ≈ 10k-50k+。需要自己实验。

---

### Q13：Thinker + Doer 双模型架构怎么设计？

**答案**：

**核心思想**：用 Reasoning Model 做深度规划（Thinker），用标准 LLM 做快速执行（Doer）。

```
用户请求
   │
   ▼
Thinker（o3 / R1 / Claude 3.7）→ plan + steps
   │
   ▼
Doer（GPT-4o-mini / Claude 3.5）→ 执行每个 step
   │
   ▼
最终答案 / 汇总
```

```python
def thinker_doer_agent(task):
    # Phase 1: Thinker 做深度规划
    plan = thinker.invoke(f"为以下任务制定计划：{task}")
    steps = parse_plan(plan)

    # Phase 2: Doer 执行每个 step
    results = [doer.invoke(f"执行：{step}") for step in steps]
    return aggregate(results)
```

**适用场景**：复杂任务需要规划 + 执行分离；成本敏感（Doer 比 Thinker 便宜 10x）。

> **面试追问**：Doer 能不能也用 Reasoning Model？
> **追问答案**：可以，但成本会爆炸。只有当某 step 特别复杂时才升级 Doer 到 Reasoning Model。

---

### Q14：Cascade（先快后慢）模式怎么实现？

**答案**：

**核心思想**：先用标准 LLM 快速试解，如果置信度低或复杂度高，再升级到 Reasoning Model。

```python
def cascade_solve(question, confidence_threshold=0.8):
    # Step 1: 快速试解
    fast = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": question}],
        response_format={"type": "json_object"},
    )
    result = json.loads(fast.choices[0].message.content)

    # Step 2: 判断是否需要升级
    if result.get("confidence", 0) >= confidence_threshold:
        return result["answer"]

    # Step 3: 升级到 Reasoning Model
    deep = client.chat.completions.create(
        model="o3-mini", reasoning_effort="high",
        messages=[{"role": "user", "content": f"深度思考：{question}"}],
    )
    return deep.choices[0].message.content
```

**成本节约效果**：90% 简单问题用快模型（成本 1/10），10% 难题升级到推理模型。

**置信度判断方式**：
1. 让模型直接输出置信度（`response_format=json`）
2. 关键词检测（"不确定"、"可能"）
3. 一致性检查（同问题多答一次，看是否一致）

---

### Q15：Reasoning Effort 调参有什么实战经验？

**答案**：

**调参 checklist**：

| 检查项 | 建议 |
|--------|------|
| 默认 effort | `medium`（平衡速度和质量） |
| 简单问题 | `low`（QA、分类、摘要） |
| 难题 | `high`（竞赛数学、代码 debug） |
| 超时保护 | 设置 30s-60s timeout，超时降级 |
| 成本监控 | 记录每次 effort 的 token 消耗和延迟 |
| A/B 测试 | 同一问题跑 low/medium/high，找 sweet spot |

```python
def measure_effort(question, effort):
    start = time.time()
    response = client.chat.completions.create(
        model="o3-mini", reasoning_effort=effort,
        messages=[{"role": "user", "content": question}],
    )
    return {
        "effort": effort,
        "latency_s": round(time.time() - start, 2),
        "total_tokens": response.usage.total_tokens,
    }
```

> **面试追问**：有没有办法自动选择 effort？
> **追问答案**：用标准 LLM 先做复杂度评估，根据评估结果动态选择 effort：
> ```python
> complexity = gpt4o_mini.classify(question)
> effort = {"simple": "low", "medium": "medium", "hard": "high"}[complexity]
> ```

---

## 5. RAG / Tool Use / Agent 集成（Q16–Q18）

---

### Q16：Reasoning Model + RAG 怎么组合？

**答案**：

**最佳实践**：检索阶段用快模型，生成阶段用 Reasoning Model 做深度合成。

```python
def rag_with_reasoning(question, vector_db):
    # 阶段 1: 快速检索
    chunks = vector_db.search(question, top_k=5)

    # 阶段 2: Reasoning Model 深度合成
    response = client.chat.completions.create(
        model="o3-mini", reasoning_effort="high",
        messages=[
            {"role": "system", "content": "根据参考资料回答，标注引用来源。"},
            {"role": "user", "content": f"参考资料：\n{format_chunks(chunks)}\n\n问题：{question}"},
        ],
    )
    return response.choices[0].message.content
```

**Reasoning Model 在 RAG 中的优势**：
1. **多跳推理**：能从多个 chunk 中推断出隐含关系
2. **引用验证**：内部推理时检查引用是否一致
3. **矛盾检测**：发现不同 chunk 之间的矛盾

> **面试追问**：放多少 RAG context 合适？
> **追问答案**：检索 5-10 个高质量 chunk，用 reranker 过滤，控制 context 在 10k-30k tokens 之间。

---

### Q17：Reasoning Model + Tool Use 怎么配合？有什么坑？

**答案**：

**核心原则**：**少而精**。Reasoning Model 适合 "一次深度思考 + 关键 tool call"，不适合频繁调用。

| 坑 | 说明 |
|----|------|
| **频繁 tool call** | 每次 tool call 打断思考流，延迟爆炸 |
| **thinking token 超限** | extended thinking + tool use 可能超过 max_tokens |
| **tool call 格式错误** | 偶尔不符合预期格式 |
| **streaming 中断** | tool call 中途无法流式返回 |

```python
def reasoning_agent_with_tools(question, tools):
    response = client.chat.completions.create(
        model="o3-mini", reasoning_effort="high",
        tools=tools,
        messages=[{"role": "user", "content": question}],
    )
    message = response.choices[0].message

    if message.tool_calls:
        results = [execute_tool(tc) for tc in message.tool_calls]
        final = client.chat.completions.create(
            model="o3-mini",
            messages=[*messages, {"role": "tool", "content": json.dumps(results)}],
        )
        return final.choices[0].message.content
    return message.content
```

> **面试追问**：为什么不适合频繁 tool call？
> **追问答案**：Reasoning Model 的思考是深度的、一次的。频繁打断导致每次思考都不够深入，反而降低准确率。标准 LLM 的 ReAct 才是 "思考 → 行动 → 观察 → 再思考" 的快速循环。

---

### Q18：Reasoning Model 在 Agent Loop 中替代了什么？

**答案**：

| 传统组件 | Reasoning Model 替代程度 |
|----------|------------------------|
| 外部 Planner | 部分替代（模型内部做规划） |
| CoT Prompt | 基本替代（模型自带推理） |
| Few-shot examples | 需求降低 |
| ReAct 循环次数 | 显著减少（M << N） |
| Tool selection | 更精准（思考后再选） |

**传统 ReAct**：`Thought → Action → Observation → ...`（循环 N 次，浅思考）

**Reasoning Model 版**：`[深度内部 CoT] → Action → Observation → ...`（循环 M 次，M << N）

> **面试追问**：ReAct 完全被淘汰了吗？
> **追问答案**：没有。实时性要求高的场景（< 1s）、简单任务、作为 Reasoning Model 的 fallback，ReAct 仍然有用。

---

## 6. 局限与权衡（Q19）

---

### Q19：Reasoning Model 的 5 大局限是什么？各自怎么应对？

**答案**：

| 局限 | 表现 | 应对策略 |
|------|------|---------|
| **慢** | 10s–60s+ | Cascade、timeout fallback |
| **贵** | API 价格 5-10x | 自部署（R1）、Cascade、Model Routing |
| **不可控** | 看不到内部 CoT | R1 开源可看 `reasoning_content`、日志审计 |
| **不能保证正确** | 还是会犯错 | Self-consistency、外部 verifier |
| **Streaming 弱** | 思考过程通常不能流式 | 返回 "正在思考..." loading indicator |

```python
class ReasoningAgentWithFallback:
    def solve(self, question, timeout=30):
        try:
            quick = fast_llm.invoke(question)
            if self.is_confident(quick):
                return quick
            return reasoning_llm.invoke(question, reasoning_effort="high", timeout=timeout)
        except TimeoutError:
            return fast_llm.invoke(question)
```

> **面试追问**：不可控怎么在生产环境监控？
> **追问答案**：
> 1. **延迟监控**：P95/P99 延迟告警
> 2. **成本监控**：token 消耗 vs 标准 LLM 对比
> 3. **质量监控**：抽样人工评估或自动评估
> 4. **超时率**：超时请求占比，过高需调整 effort 或 fallback

---

## 7. 面试综合题（Q20）

---

### Q20：给你一个生产级客服系统，用户提问平均响应时间要求 < 3s，你怎么引入 Reasoning Model？

**答案**：

**方案设计**：

```
用户提问 → Intent Classifier（< 500ms）
   │
   ├─ 简单 FAQ → FAQ 检索 + 快速回答（< 1s）
   │
   └─ 需要推理 → Cascade Agent（先快后慢，timeout 3s 兜底）
```

**关键设计决策**：

1. **Intent Classification 前置**：90% 简单问题直接路由，0 Reasoning Model 成本
2. **Cascade 模式**：复杂问题先 GPT-4o-mini 试答，置信度低再升级到 o3-mini
3. **Timeout 兜底**：o3 设 3s timeout，超时返回 GPT-4o-mini 快速答案
4. **Streaming + "正在思考"**：升级到 Reasoning Model 时先返回占位，缓解等待焦虑

```python
class CustomerServiceAgent:
    async def handle(self, question):
        intent = await self.classifier.classify(question)
        if intent == "simple_faq":
            return await self.faq_lookup(question)
        elif intent == "needs_reasoning":
            try:
                return await asyncio.wait_for(self.deep_reason(question), timeout=3.0)
            except asyncio.TimeoutError:
                return await self.fast_responder.answer(question)
        return "已转接人工客服。"
```

> **面试追问**：这个方案的弱点？
> **追问答案**：
> 1. Intent classifier 可能分错类
> 2. o3-mini medium effort 延迟可能还是超过 3s（需要实测）
> 3. Streaming 在 tool use 场景下不可用
> 4. 即使只有 10% 问题走 Reasoning Model，QPS 高时总成本也不低

---

## 面试技巧总结

### Reasoning Model 是不是替代了 ReAct？

**直接回答**：不是替代，而是**改变 ReAct 的形状**。

| 维度 | 传统 ReAct | Reasoning Model + ReAct |
|------|-----------|------------------------|
| 思考方式 | 每一步外部显式思考 | 内部隐式深度思考 |
| 循环次数 | N 次（多步） | M 次（M << N，更少步） |
| 每步质量 | 浅 | 深 |

**一句话总结**：
> Reasoning Model 让 ReAct 的 "Thought" 阶段从 "浅而多" 变成 "深而少"，是 ReAct 的进化版，不是终结者。

### 面试得分要点

| 要点 | 说明 |
|------|------|
| 提到 test-time compute | 核心创新点 |
| 能画出 Thinker + Doer 架构图 | 生产架构理解 |
| 提到 Cascade 模式 | 成本/延迟平衡意识 |
| 提到 RAG + Reasoning | 幻觉解决思路 |
| 诚实说局限 | 慢 / 贵 / 不可控 / streaming 弱 |
| 能写出实际代码 | 面试官通常会要求手写 |

---

## 常见面试题速查

| 问题 | 关键词 |
|------|--------|
| 什么是 Reasoning Model？ | 内部 CoT / test-time compute |
| Test-time compute scaling law | pre-training vs test-time / AIME / GPQA |
| o1 / o3 / R1 / Claude 3.7 区别 | 开源 / 价格 / agent 友好度 |
| reasoning_effort 怎么调 | low/medium/high / 延迟 vs 质量 |
| Thinker + Doer 架构 | 双模型 / 规划 vs 执行 |
| Cascade 先快后慢 | 置信度 / 成本节约 |
| Reasoning Model + RAG | 多跳推理 / 引用验证 |
| Reasoning Model + Tool Use | 少而精 / 不要频繁调用 |
| 局限是什么 | 慢 / 贵 / 不可控 / streaming 弱 |
| 和 ReAct 的关系 | 改变形状 / 不是替代 / 深而少 |

---

## 参考材料

- [OpenAI o1 System Card](https://openai.com/index/openai-o1-system-card/)
- [OpenAI o3 Blog](https://openai.com/index/openai-o3/)
- [DeepSeek R1 Paper](https://arxiv.org/abs/2501.12948)
- [Anthropic Claude 3.7 Sonnet](https://www.anthropic.com/news/claude-3-7-sonnet)
- [Karpathy on test-time compute](https://x.com/karpathy/status/1836503993638502597)
- [Simon Willison on o1](https://simonwillison.net/2024/Sep/12/openai-o1/)

---

*本文件覆盖 U24 核心知识点，20 道 Q&A，适合面试前快速复习。*
