# Interview QA · LLM Inference & Cost/Latency 工程

> 来源：U28-LLMInference.md + 生产工程实践
> 覆盖：Prefill / Decode / KV Cache / Prompt Caching / 性能指标 / 部署框架 / Batching / Model Routing / Streaming / Cost 优化

---

## 一、推理流程基础篇

### Q: LLM 推理的两个阶段（prefill 和 decode）是什么？各占多少时间？

**Answer:**

LLM 自回归生成分两个阶段：

| 阶段 | 做什么 | 计算特点 | 时间占比 |
|------|--------|----------|----------|
| **Prefill** | 处理完整输入，计算所有 token 的 K/V 并缓存 | 计算密集（一次完整 forward pass） | 10-30% |
| **Decode** | 每次生成 1 个 token，复用缓存 K/V | 内存带宽密集（读 K/V 矩阵） | 70-90% |

```python
# 伪代码：理解两个阶段
def llm_generate(prompt):
    # === PREFILL（一次性）===
    kv_cache = prefill_forward(prompt)  # 约 50-200ms
    
    # === DECODE（逐 token 循环）===
    generated = []
    next_token = "<sos>"
    while next_token != "<eos>":
        next_token, kv_cache = decode_one_step(kv_cache, next_token)  # 10-50ms/step
        generated.append(next_token)
    
    return generated
```

**核心洞察：**
- Prefill 延迟 ∝ 输入长度（O(n)）
- Decode 延迟 ∝ 生成长度（O(m)），受限于 GPU 显存带宽
- 优化核心：**减少 prefill 重复计算**（KV Cache） + **提高 decode 吞吐**（continuous batching）

**追问：** 为什么 decode 比 prefill 慢？→ Decode 是 memory-bound，每次只读不重算，GPU SM 利用率低

---

### Q: Transformer 自回归生成为什么慢？

**Answer:**

1. **串行依赖**：每个 token 必须等前一个生成完，无法并行
2. **内存带宽瓶颈**：decode 阶段每步要读整个 KV Cache（70B 模型 32k seq ≈ 128GB）
3. **低计算利用率**：GPU 擅长并行，decode 每次只处理 1 个 token

**量化**：70B 模型 batch=1 时 ≈ 10-30ms/token；batch=32 时可压到 5ms/token（但每个请求 P95 延迟上升）

---

## 二、KV Cache 核心篇

### Q: KV Cache 是什么？为什么对 LLM 推理至关重要？

**Answer:**

**KV Cache** = 缓存 Transformer 每一层中已计算 token 的 Key 和 Value 向量。

```
第 1 步（token "今"）: 计算 K₁, V₁ → 存入 Cache
第 2 步（token "天"）: 读取 K₁, V₁ + 计算 K₂, V₂ → 存入 Cache
第 3 步（token "气"）: 读取 K₁,V₁,K₂,V₂ + 计算 K₃, V₃ → 存入 Cache
```

**为什么重要：**
- **避免重算**：无 KV Cache，每生成 1 个 token 要重跑整个序列（O(n²)）
- **内存 vs 计算权衡**：KV Cache 占显存，但节省大量 FLOPs

**KV Cache 大小估算（70B 模型，FP16，32k seq_len）：**
```
2 × 80 层 × 64 heads × 128 head_dim × 32k tokens × 2 bytes ≈ 128 GB
```

**vLLM 的 PagedAttention**：把 KV Cache 分页管理（类似虚拟内存），解决显存碎片，支持多请求共享 prefix → 内存利用率提升 2-4x

**追问：** KV Cache 和 Prompt Caching 的区别？→ KV Cache 是模型层的计算缓存（K/V 向量）；Prompt Caching 是 API 层的输入缓存

---

## 三、Prompt Caching 实战篇

### Q: Anthropic Prompt Caching（cache_control）原理？怎么用？

**Answer:**

**原理**：Anthropic API 自动缓存前缀的 KV 计算结果。后续请求前缀相同则直接复用。

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=512,
    system=[
        {"type": "text", "text": "你是一个代码审查助手...", 
         "cache_control": {"type": "ephemeral"}},
        {"type": "text", "text": "Code Style Guide:\n" + long_style_guide, 
         "cache_control": {"type": "ephemeral"}},
    ],
    messages=[{"role": "user", "content": "review 这段代码"}],
)
# 第一次：full cost
# 第二次（相同 system）：cache hit，只付增量 cost
```

**关键约束：**
- `cache_control` 必须放在 system 消息块或消息列表的最近用户消息之前
- 缓存 TTL = 5 分钟
- 缓存大小最小 1024 tokens
- **缓存断裂**：前缀变化则缓存失效

**成本对比（1M tokens，Claude 3.5 Sonnet）：**
| 场景 | Input 成本 |
|------|-----------|
| 无缓存 | $3.00 / 1M input tokens |
| 缓存写入 | $3.75 / 1M（+25%） |
| 缓存命中 | $0.30 / 1M（**-90%**） |

**实战技巧：**
- System prompt + RAG 检索结果 = 最佳缓存候选（稳定、长、复用率高）
- 用户输入不要放在 cache_control 后（每次都变，无意义）

**追问：** OpenAI 和 Google 怎么实现？→ OpenAI 自动 caching（不暴露 API）；Google Gemini 有 explicit cache API（更可控）

---

## 四、性能指标篇

### Q: TTFT / TPS / E2E Latency 的定义、关系和目标值？

**Answer:**

| 指标 | 定义 | 影响因素 | 目标值 |
|------|------|----------|--------|
| **TTFT**（Time To First Token） | 发出请求到收到第一个 token | Prefill 延迟、网络 RTT | < 500ms（理想 < 200ms） |
| **TPS**（Tokens Per Second） | 生成阶段的 token 输出速度 | 模型大小、batch size、硬件 | > 50 TPS |
| **E2E Latency** | 从请求到完整响应 | TTFT + 生成 token 数 / TPS | < 3s（普通）/ < 30s（reasoning） |

```
用户感知延迟模型：
├── TTFT = 500ms → "立刻开始响应" ✅
├── TTFT = 3s   → "卡住了" ❌（即使之后快速输出也不好）
└── TPS = 60    → "像真人打字"，体验好
```

**优化优先级：**
1. **先保 TTFT**（> 1s 就感觉慢）
2. **再提 TPS**（影响内容流速感）
3. **最后压 E2E**

**追问：** 为什么 streaming 能改善体验？→ TTFT 心理感知：看到输出滚动 → "系统在工作"，即使总时间相同也觉得快

---

## 五、部署框架对比篇

### Q: vLLM / SGLang / TGI 三大框架对比？

**Answer:**

| 特性 | vLLM | SGLang | TGI |
|------|------|--------|-----|
| **核心优势** | PagedAttention，高吞吐 | RadixAttention，prefix 共享 | HuggingFace 官方，易部署 |
| **Prefix Sharing** | ✅ 有限 | ✅ **RadixAttention（树形缓存）** | ⚠️ 基础 |
| **Structured Output** | ⚠️ 需 EAGLE | ✅ **原生支持**（JSON/regex） | ⚠️ 基础 |
| **社区 & 生态** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **适用场景** | 通用高 QPS | 多轮对话 / structured output | 快速原型 |

```python
# vLLM 部署
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-70B-Instruct",
    messages=[{"role": "user", "content": "Explain RLHF"}],
    max_tokens=512,
)
```

**追问：** RadixAttention 和普通 prefix sharing 的区别？→ RadixAttention 用 Radix Tree 组织 KV Cache，支持动态插入/删除/共享，适合多轮对话

---

## 六、Batching / 异步 / 并发优化篇

### Q: Batching、Continuous Batching、异步并发的区别？Python 怎么实现？

**Answer:**

**静态 Batching vs Continuous Batching：**

| 类型 | 机制 | 缺点 |
|------|------|------|
| **静态 Batching** | 等整批请求完成才返回 | 短请求被长请求阻塞 |
| **Continuous Batching** | 随时插入新请求 / 移除完成的 | 实现复杂 |

**Python 异步并发实现：**
```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def batch_process(queries, batch_size=20):
    results = []
    for i in range(0, len(queries), batch_size):
        batch = queries[i:i + batch_size]
        tasks = [client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": q}],
            max_tokens=256,
        ) for q in batch]
        batch_results = await asyncio.gather(*tasks, return_exceptions=True)
        results.extend(batch_results)
    return results

queries = ["What is AI?", "Explain RLHF", ...] * 100
results = asyncio.run(batch_process(queries, batch_size=20))
```

**并发优化要点：**
- **控制并发数**：避免触发 rate limit（重试 + 退避反而浪费时间）
- **错误隔离**：`return_exceptions=True`，单个失败不影响其他
- **连接池**：复用 HTTP 连接（`AsyncOpenAI` 默认做）

**追问：** 并发 100 个请求，为什么 TPS 不线性增长？→ GPU 显存和 batch 限制，超过 sweet spot 后排队

---

## 七、Model Routing 按需选模型篇

### Q: Model Routing 是什么？怎么在生产中实现？

**Answer:**

**Model Routing** = 根据任务复杂度、成本预算，动态选择最合适的模型。

```python
def route_model(complexity: str) -> str:
    if complexity == "trivial":
        return "gpt-4o-mini"          # $0.15/$0.60，快
    elif complexity == "medium":
        return "claude-3-5-sonnet"    # $3/$15，平衡
    elif complexity == "hard":
        return "o1"                    # $15/$60，强推理
    else:
        return "deepseek-chat"        # $0.27/$1.10，成本优先
```

**Agent Pipeline 分级示例：**

| Agent Step | 复杂度 | 推荐模型 | 理由 |
|-----------|--------|----------|------|
| 意图分类 | trivial | gpt-4o-mini | 准确率够，成本极低 |
| 工具参数提取 | medium | claude-3-5-sonnet | 需要精准 JSON |
| 多步规划 | hard | o1 / deepseek-r1 | 推理链复杂 |
| 最终答案 | medium | claude-3-5-sonnet | 质量 + 速度平衡 |

**Cascade 模式（先小后大）：**
```python
async def cascade_answer(question: str):
    answer = await call_llm("gpt-4o-mini", question)
    if confidence(answer) > 0.8:
        return answer  # 够了，省大模型
    return await call_llm("o1", question)

# 成本对比（100 万 query）：
# 全用 o1: $75,000
# Cascade（90% mini + 10% o1）: $7,500（节约 97%）
```

**追问：** 怎么判断任务复杂度？→ 关键词/意图分类器 / 历史准确率统计 / few-shot 分类

---

## 八、Streaming 实现篇

### Q: SSE Streaming 怎么实现？TTFT 怎么优化？

**Answer:**

```python
from openai import OpenAI

client = OpenAI()

# 非流式（等全部生成完）
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Explain RLHF"}],
    max_tokens=512,
)
# 用户等待 3s 才看到任何输出

# === STREAMING（边生成边输出）===
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Explain RLHF"}],
    max_tokens=512,
    stream=True,
)

full_text = ""
for chunk in stream:
    delta = chunk.choices[0].delta.content or ""
    print(delta, end="", flush=True)
    full_text += delta
```

**FastAPI 后端 SSE：**
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app = FastAPI()

async def generate_stream(messages):
    stream = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        stream=True,
    )
    for chunk in stream:
        if chunk.choices[0].delta.content:
            yield f"data: {json.dumps({'content': chunk.choices[0].delta.content})}\n\n"
    yield "data: [DONE]\n\n"

@app.get("/chat")
async def chat(query: str):
    return StreamingResponse(
        generate_stream([{"role": "user", "content": query}]),
        media_type="text/event-stream",
    )
```

**TTFT 优化技巧：**
1. **小模型做意图分类**（200ms 内完成）→ 立即返回"正在处理"
2. **流式生成**：第一个 token 到达即展示（感知延迟 ≈ TTFT）
3. **Prompt 压缩**：减少 prefill 时间
4. **本地化部署**：避免网络 RTT（50-100ms）

**追问：** SSE 和 WebSocket 哪个好？→ SSE 简单（HTTP 原生支持），WebSocket 双向但更重。LLM 生成是单向的，SSE 足够

---

## 九、Cost 优化实战篇

### Q: Agent 调 LLM 的成本怎么拆解？怎么压到极致？

**Answer:**

**成本公式：**
```
总成本 = input_tokens × input_price + output_tokens × output_price 
       + cache_penalty + retry_overhead + reasoning_overhead
```

**实战案例（RAG Agent，100 万 query/月）：**

| 优化手段 | 月成本 | 收益 |
|---------|--------|------|
| 原始（GPT-4o，无缓存） | $29,000 | — |
| + Prompt Caching | $10,300 | 节约 64% |
| + Model Routing（70% mini） | $4,550 | 再节约 56% |
| + Context 截断 | $3,200 | **总节约 89%** |

**Token Budget 管理：**
```python
def build_prompt_with_budget(system, tools, history, user_msg, max_tokens=100_000):
    base = system + tools
    available = max_tokens - count_tokens(base) - count_tokens(user_msg) - 1000
    
    # 从最新开始往前取（保留最近的决策）
    truncated = truncate_history(history, available * 0.6)
    return assemble(base, truncated, user_msg)
```

**Model 分级策略：**

| 任务 | 推荐模型 | 成本对比 |
|------|---------|---------|
| 简单分类/抽取 | gpt-4o-mini | 比 GPT-4o 便宜 95% |
| 工具调用/JSON | claude-3-5-haiku | 便宜 73% |
| 一般对话 | claude-3-5-sonnet | 平衡 |
| 复杂推理 | o1 / deepseek-r1 | 贵但准 |

**追问：** 怎么预估新 pipeline 成本？→ 先跑 100 次样本，记录平均 token 用量 × 目标 QPS × 模型价格 = 月成本

---

## 十、Reliability 工程篇

### Q: 生产 Agent 的限流、重试、熔断怎么设计？

**Answer:**

```python
from tenacity import retry, stop_after_attempt, wait_exponential_jitter

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=10),
    retry=lambda e: isinstance(e, RateLimitError | ServiceUnavailableError),
)
def call_llm_with_retry(messages, model="gpt-4o"):
    return client.chat.completions.create(
        model=model,
        messages=messages,
        timeout=30,
    )

def fallback_chain(messages):
    """多模型 fallback"""
    models = ["gpt-4o", "claude-3-5-sonnet", "deepseek-chat"]
    for model in models:
        try:
            return call_llm_with_retry(messages, model)
        except (RateLimitError, ServiceUnavailableError):
            continue
    raise AllModelsDownError()
```

**完整 Reliability Stack：**
```
Client → Rate Limiter → Circuit Breaker → Retry + Backoff → Fallback Chain → Observability
```

**关键指标：**
- Error rate < 0.5%
- Retry rate < 1%
- P95 Latency < 5s
- Cache hit rate > 60%

---

## 十一、把延迟压到 200ms 实战

### Q: Interview tips：怎么把 Agent 推理延迟压到 200ms？

**Answer:**

**目标分解：**
```
200ms = 20ms(网络) + 30ms(prompt处理) + 100ms(prefill) + 50ms(生成首token)
```

**完整 checklist：**

| 优化手段 | 预期收益 | 实现难度 |
|---------|---------|---------|
| **本地部署**（vLLM，同机房） | -50ms RTT | 中 |
| **小模型做前置**（分类/路由） | -80ms prefill | 低 |
| **Prompt 压缩**（< 2k tokens） | -50ms prefill | 低 |
| **Prompt Caching**（5min TTL） | -30ms | 低 |
| **量化**（INT8/FP8） | -30% decode 时间 | 中 |
| **Speculative Decoding** | +30% TPS | 中 |
| **Streaming + 骨架屏** | 感知延迟 -200ms | 低 |
| **Connection Keep-Alive** | -10ms RTT | 低 |

```python
# 完整低延迟实现
async def low_latency_chat(query: str, cached_system: str):
    start = time.perf_counter()
    
    # 1. 小模型意图分类（50ms）
    intent = await classify_intent(query)
    model = "gpt-4o-mini" if intent == "simple" else "gpt-4o"
    
    # 2. 带 cache 的流式请求
    stream = await client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": cached_system},  # cache hit!
            {"role": "user", "content": query},
        ],
        stream=True,
    )
    
    # 3. 流式返回
    first_token_time = None
    full_text = ""
    async for chunk in stream:
        if first_token_time is None:
            first_token_time = (time.perf_counter() - start) * 1000
        delta = chunk.choices[0].delta.content or ""
        full_text += delta
    
    return {
        "ttft_ms": round(first_token_time, 1),    # 目标 80-150ms
        "total_ms": round((time.perf_counter() - start) * 1000, 1),
    }
```

**预期结果：**
- TTFT ≈ 80-150ms（本地部署 + 小模型 + cache）
- Total (100 tokens) ≈ 300-500ms

---

## 十二、多模态推理成本篇

### Q: Vision / 多模态模型的推理成本有什么不同？

**Answer:**

**视觉 token 开销（GPT-4o）：**
```
1024×1024 图（低细节） ≈ 85 tokens
1024×1024 图（高细节） ≈ 1100 tokens
2048×2048 图（高细节） ≈ 4200 tokens
```

**成本影响：**
- RAG 系统如果包含图片，context window 被快速耗尽
- 解决方案：图片预处理（缩略图 / OCR 提取文字替代原始图）

```python
def optimize_multimodal_input(image_path: str):
    # OCR 提取文字（100 tokens）vs 原始图（1000 tokens）
    text = ocr_extract(image_path)  # 成本降低 90%
    return text
```

**Agent Computer Use（U25）：** 每次截图 ≈ 500-1000 tokens → 多轮操作成本飙升

**追问：** 怎么在多模态 Agent 中控制成本？→ 用小模型做图像分类（判断是否需要完整视觉），仅在需要时用大模型

---

## 十三、可观测性工程篇

### Q: 生产 LLM Agent 的可观测性怎么做？

**Answer:**

**三大支柱：**

```python
import time
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def traced_llm_call(messages, model, agent_step):
    with tracer.start_as_current_span(f"llm.{agent_step}") as span:
        span.set_attribute("model", model)
        span.set_attribute("prompt_tokens", estimate_tokens(messages))
        
        start = time.perf_counter()
        response = await client.chat.completions.create(
            model=model,
            messages=messages,
        )
        latency_ms = (time.perf_counter() - start) * 1000
        
        span.set_attribute("latency_ms", latency_ms)
        span.set_attribute("cost_usd", calculate_cost(response.usage, model))
        
        return response
```

**关键 Metrics 与告警阈值：**

| 指标 | 告警阈值 | 工具 |
|------|---------|------|
| P95 Latency | > 5s | Langfuse / Helicone |
| Error Rate | > 1% | Prometheus + Grafana |
| Cache Hit Rate | < 50% | Langfuse |
| Token 用量/日 | > 预算 120% | Custom alert |

**追问：** LangSmith vs Langfuse？→ LangSmith 是 LangChain 官方，Langfuse 是开源的，支持任何框架

---

## 面试总结 Checklist

### 必背题（高频）
- [ ] **Prefill vs Decode**：两个阶段的区别、时间占比、优化方向
- [ ] **KV Cache**：是什么、为什么重要、PagedAttention 原理
- [ ] **Prompt Caching**：Anthropic 的 cache_control、成本收益、适用场景
- [ ] **TTFT / TPS / Latency**：三个指标的定义、目标值、优化手段
- [ ] **vLLM vs SGLang**：核心差异、适用场景、RadixAttention

### 理解题（中等）
- [ ] **Continuous Batching**：为什么比静态 batching 好
- [ ] **Model Routing**：怎么实现、Cascade 模式、成本收益
- [ ] **Streaming**：SSE 实现、TTFT 优化
- [ ] **Reliability**：限流、重试、熔断的完整设计
- [ ] **Token Economics**：成本拆解、分级策略、预算管理

### 实战题（高级）
- [ ] **怎么把延迟压到 200ms？** → 完整 checklist
- [ ] **生产 Agent 的可观测性** → metrics / tracing / alerting
- [ ] **多模态推理成本** → 图片 token 开销、优化策略

---

*最后更新：2026-07-01*
