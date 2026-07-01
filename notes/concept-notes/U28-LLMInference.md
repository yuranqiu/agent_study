# U28 · LLM Inference & Cost/Latency 工程（推理与成本工程）

> **核心问题**：怎么让 Agent 调用 LLM 时**更快、更便宜、更稳定**？生产级 Agent 工程师必备的"经济学"基础。
> **预计学时**：5h
> **对应 Phase**：Phase 3 / 面试
> **状态**：核心工程基础（2024-2026 所有生产 Agent 必修）

---

## 1. 为什么 Agent 工程师要懂 LLM Inference

Agent 不只是"调 API"，生产环境要回答：
- 1000 QPS 怎么扛？延迟怎么压到 200ms？
- 月烧 10 万 token，成本怎么降到 1 万？
- 端到端可用性 99.9% 怎么保证？上游限流怎么办？
- 用户问"为什么这次响应慢了"——怎么 debug？

**核心洞察**：Agent 是 LLM 的客户端，LLM inference 的瓶颈直接决定 Agent 的体验。**不懂 inference 的 Agent 工程师写不出生产级系统**。

---

## 2. 5 大成本来源

一个 Agent 调一次 LLM API 的成本由 5 部分组成：

```
总成本 = input_token_cost + output_token_cost + cache_miss_penalty + retry_overhead + reasoning_overhead
```

| 来源 | 说明 | 优化方向 |
|------|------|----------|
| **Input tokens** | system + history + RAG context | Prompt 压缩、context 缓存、history 截断 |
| **Output tokens** | 模型生成的内容 | 控制 max_tokens、用更小模型（如果不是关键 step） |
| **Cache miss** | KV cache 没命中 | 优化 prompt 顺序、prefix sharing |
| **Retry** | 网络/限流重试 | 限流感知、指数退避、circuit breaker |
| **Reasoning overhead** | Reasoning Model 思考成本 | reasoning_effort 调档、按问题分级 |

---

## 3. 6 大优化技术

### 3.1 Prompt Caching（Anthropic 特色）

**Prompt Caching** 是 Anthropic / OpenAI 提供的关键优化：自动缓存 system prompt 和长 prefix，省 90% 成本。

```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "你是一个代码审查助手...",
            "cache_control": {"type": "ephemeral"},  # 标记可缓存
        },
        {
            "type": "text",
            "text": "以下是项目的 code style guide（很长）...",
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[{"role": "user", "content": "review 这段代码"}],
)

# 第一次：full cost
# 第二次（同样 system）：cache hit，只付增量 cost
# Anthropic: cache write +5%, cache hit -90%
```

**OpenAI 也有**：自动 caching（不对外暴露 cache_control，但内部做）。

**Google Gemini**：explicit cache API（更可控）。

**实战**：
- 把稳定的 system prompt / 长 context 标记为 cacheable
- RAG 检索结果一般也适合（同一段文本可能被多轮对话复用）
- 不要把"每次都变"的部分（user 输入）放在 cache_control 后

### 3.2 Context Compression

**Compaction**（详见 U22）：把长 history 压缩成摘要。

**Token Budget Management**：自己控制 prompt 大小：

```python
def build_prompt(system, history, user_msg, max_tokens=100000):
    # 1. 固定部分（system + tools + RAG）= 已知占用
    base_tokens = count_tokens(system) + count_tokens(tools) + count_tokens(rag)

    # 2. 剩余预算
    remaining = max_tokens - base_tokens

    # 3. 分配：history 60%, user 20%, buffer 20%
    history_budget = int(remaining * 0.6)
    user_budget = int(remaining * 0.2)

    # 4. 截断 history（保留最近 N 轮）
    history_truncated = truncate_history(history, history_budget)

    return assemble(system, history_truncated, user_msg)
```

### 3.3 Model Routing（按需选模型）

不是所有 step 都需要 GPT-4o / o1。**Model Routing** 按任务难度选模型：

```python
def route_model(task_complexity, cost_sensitive):
    if task_complexity == "simple":
        return "gpt-4o-mini"  # 便宜 30x
    elif task_complexity == "medium":
        return "claude-3-5-sonnet"  # 平衡
    elif task_complexity == "hard":
        return "o1" or "deepseek-r1"  # 推理
```

**节约实战**：把任务分成"简单抽取"（小模型）和"深度推理"（大模型），分别用最合适的模型。

详见 U24 Reasoning Model 的 Cascade 模式。

### 3.4 Batching & Concurrency

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def batch_process(queries, batch_size=20):
    results = []
    for i in range(0, len(queries), batch_size):
        batch = queries[i:i+batch_size]
        tasks = [client.chat.completions.create(...) for q in batch]
        results.extend(await asyncio.gather(*tasks))
    return results
```

**关键**：
- 异步 I/O（用 `AsyncOpenAI`）
- 控制并发（避免触发 rate limit）
- 错误隔离（一个失败不影响其他）

### 3.5 Streaming & TTFT

**TTFT**（Time To First Token）= 用户看到第一个 token 的延迟。Agent UX 关键指标。

```python
# Streaming 模式
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    stream=True,
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

**优势**：TTFT 可以压到 200-500ms，让用户感觉"快"。即使总时间 3s 也不觉得慢。

### 3.6 Local Serving（自部署）

当 API 成本不可接受时，自部署开源模型：

| 框架 | 特点 | 推荐场景 |
|------|------|----------|
| **vLLM** | 高吞吐，PagedAttention | 高 QPS 生产 |
| **SGLang** | RadixAttention，结构化输出 | 多轮对话 / structured output |
| **TGI** | Hugging Face 官方 | 通用 |
| **TensorRT-LLM** | NVIDIA 优化 | 极致性能 |
| **llama.cpp / ollama** | CPU / 边缘 | 离线 / 本地 |

```python
# vLLM 部署
vllm serve meta-llama/Llama-3.1-70B-Instruct \
    --tensor-parallel-size 4 \
    --gpu-memory-utilization 0.9 \
    --max-model-len 32768

# 调用
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-70B-Instruct",
    messages=[...],
)
```

---

## 4. 关键性能指标

| 指标 | 说明 | 目标值（生产） |
|------|------|----------------|
| **TTFT**（Time To First Token） | 用户看到第一个 token 的延迟 | < 500ms |
| **TPS**（Tokens Per Second） | 生成速度 | > 50 TPS（流式） |
| **End-to-end latency** | 从请求到完整响应 | < 3s（普通）/ < 30s（reasoning） |
| **Throughput** | QPS（Queries Per Second） | 视业务 |
| **Cost per 1k queries** | 千次请求成本 | 视业务 |
| **Cache hit rate** | 缓存命中率 | > 60%（理想 > 80%） |
| **Error rate** | 错误率 | < 0.5% |
| **Retry rate** | 重试率 | < 1% |

---

## 5. Reliability 工程

### 5.1 限流 & 重试

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10),
)
def call_llm_with_retry(messages):
    return client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
    )
```

### 5.2 Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failures = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.last_failure = 0
        self.state = "closed"  # closed / open / half-open

    def call(self, fn, *args):
        if self.state == "open":
            if time.time() - self.last_failure > self.reset_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()
        try:
            result = fn(*args)
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.threshold:
                self.state = "open"
            raise
```

### 5.3 Fallback Chain

```python
def call_with_fallback(messages):
    models = ["gpt-4o", "claude-3-5-sonnet", "deepseek-chat"]
    for model in models:
        try:
            return client_call(model, messages)
        except RateLimitError:
            continue
    raise AllModelsDownError()
```

### 5.4 Observability

- **日志**：每个 request 记录 model, tokens, latency, status
- **metrics**：QPS / latency P50/P95/P99 / error rate
- **trace**：跨多个 step / 多个 model 的调用链
- **工具**：LangSmith / Langfuse / Arize Phoenix / Helicone

---

## 6. Token Economics 实战

### 6.1 一个 Agent 调一次的成本拆解

假设一个 RAG Agent 处理 1 个用户问题：
- System prompt: 1k tokens
- RAG context (top-10 chunks): 5k tokens
- History: 2k tokens
- User question: 0.2k tokens
- Output: 0.5k tokens
- **Total: 8.7k tokens**

GPT-4o 价格：$2.50/1M input, $10.00/1M output
- Input: 8.2k * 2.5/M = $0.0205
- Output: 0.5k * 10/M = $0.005
- **Total: $0.0255 / query**

月 100 万 query = **$25,500 / month**（GPT-4o）

用 GPT-4o-mini（$0.15/$0.60 per 1M）：**$1.66 / 1M query = $1,660 / month**（节约 93%）

**用 prompt cache（90% hit）+ GPT-4o-mini for 简单 step + GPT-4o for 复杂 step**：月成本可压到 $500 以下。

### 6.2 节约 checklist

- [ ] **System prompt 用 cache_control**（Anthropic）或确保 prefix 稳定
- [ ] **RAG top-k 不要过大**（5-10 足够，过大浪费）
- [ ] **history 做截断或压缩**（保留关键决策，丢弃冗余）
- [ ] **能小模型就小模型**（不重要的 step 用 mini 模型）
- [ ] **Reasoning Model 按需用**（不要所有 step 都 o1）
- [ ] **并发请求但控制并发数**（避免 rate limit 重试浪费）
- [ ] **失败 fallback**（不要让一个错误打挂整个 pipeline）

---

## 7. vLLM / SGLang 关键概念

### 7.1 KV Cache

LLM 自回归生成时，每生成一个 token 要算 attention。**KV Cache** = 把历史的 K/V 缓存下来，避免重算。

- 第一次请求：算所有 K/V，存起来
- 后续 token：只算新 token 的 K/V，复用历史的
- **多请求 prefix 共享**：相同 system prompt 共享 KV cache → 大幅降成本

vLLM 的 **PagedAttention** 把 KV cache 分页管理，节省显存。

### 7.2 Continuous Batching

**传统 batching**：等一批请求全部完成才返回（短请求等长请求，浪费）
**Continuous batching**：随时插入新请求 / 移除完成的请求（高吞吐）

vLLM / SGLang 都默认 continuous batching。

### 7.3 Speculative Decoding

**Speculative Decoding**：用小模型（draft）先猜几个 token，大模型（target）批量验证。显著加速。

### 7.4 Quantization

**Quantization** = 把 FP16 量化到 INT8 / INT4 / FP8，节省显存 + 加速：
- **INT8**：质量损失小，2x 加速
- **INT4**：质量损失中等，3-4x 加速
- **FP8**（H100）：新硬件原生支持，2x 加速

工具：bitsandbytes / GPTQ / AWQ / SmoothQuant

---

## 8. 面试视角

- **Q：Agent 工程师为什么需要懂 LLM inference？**
  → Agent 是 LLM 的客户端，inference 性能/成本/可靠性直接决定 Agent 体验和商业可行性。生产 Agent 工程师必须懂 cost、latency、caching、reliability。

- **Q：Agent 的成本怎么优化？**
  → 5 大方向：① Prompt caching ② Model routing（按任务选模型） ③ Context compression ④ Batching & async ⑤ Reasoning Model 按需。生产系统 prompt cache + 模型路由 + context 截断通常能省 80%+ 成本。

- **Q：TTFT 和 TPS 是什么？怎么优化？**
  → TTFT = Time To First Token，影响"开始感觉快"；TPS = Tokens Per Second，影响"内容流速"。优化：streaming + 小模型做前置 + 缓存。

- **Q：vLLM 和 SGLang 怎么选？**
  → vLLM 通用，吞吐高，社区大。SGLang 在多轮对话 / structured output / prefix sharing 场景更强（有 RadixAttention）。两者底层都基于 continuous batching + PagedAttention。

- **Q：怎么设计 Agent 的限流 + 重试 + 熔断？**
  → 限流：固定窗口 / 滑动窗口 / token bucket。重试：指数退避（tenacity 库）。熔断：closed/open/half-open 三态。Fallback chain：多 model 多 provider 切换。

- **Q：什么是 KV cache？为什么要 cache？**
  → LLM 自回归生成时缓存历史的 Key/Value，避免重算 attention。多个请求共享相同 prefix（如 system prompt）可以共享 KV cache，省 90%+ 计算。

- **Q：production LLM serving 的关键指标？**
  → TTFT < 500ms，TPS > 50，end-to-end latency < 3s，QPS 视业务，error rate < 0.5%，cache hit rate > 60%。

---

## 9. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Anthropic Prompt Caching 文档](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) | Prompt caching 是生产 Agent 必修 |
| [OpenAI: Latency optimization](https://platform.openai.com/docs/guides/latency-optimization) | 官方延迟优化指南 |
| [vLLM Paper](https://arxiv.org/abs/2309.06180) | PagedAttention 原理 |
| [SGLang Paper](https://arxiv.org/abs/2312.07104) | RadixAttention / structured output |
| [Karpathy: LLM inference](https://www.youtube.com/watch?v=zjkBMFhNj_g) | 通俗讲解 LLM inference |
| [Athenese AI: LLM cost optimization](https://www.helicone.ai/blog/llm-cost-optimization) | 实战 cost 优化 |
| [Langfuse: LLM observability](https://langfuse.com/) | 主流 observability 工具 |
| [OpenLLMetry](https://opentelemetry.io/) | OpenTelemetry for LLM |

---

## 10. 练习

1. **测成本**：写一个脚本，用真实 API key 跑 1000 个 query，记录 token / cost / latency，输出 P50/P95/P99
2. **加 caching**：用 Anthropic prompt cache，重复 1000 个相同 system 的请求，对比 cache 命中前后的 cost
3. **Model routing**：实现一个"先 mini 后 o1"的 cascade router，对比纯 o1 的成本和质量
4. **Batching**：用 async + asyncio.gather 并发 100 个请求，对比串行的延迟
5. **Circuit breaker**：写一个简单的 circuit breaker，验证"上游挂掉时不再发请求"
6. **vLLM 本地部署**：用 vLLM 部署一个 7B 模型，测本地 QPS
7. **SLO 监控**：用 Langfuse / Helicone 监控自己的 agent，记录 latency / cost / error rate

---

## 11. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U1 LLM 基础 | LLM 怎么工作（autoregressive）→ 决定 inference 怎么优化 |
| U3 Prompt Engineering | Prompt 长度直接影响 cost / latency |
| U22 Context Engineering | Compaction / prompt caching 是 Context Engineering 的成本优化版 |
| U23 Harness Engineering | Reliability / observability 是 Harness 组件 6 + 11 |
| U24 Reasoning Model | Reasoning Model 改变了 token economics（更贵但更准） |
| U25 Computer Use | Computer Use 涉及多模态 / vision token，成本结构不同 |
| U26 Agent Skills | Skills lazy load 是 context cost 优化 |
| U29 Agent Deploy（U29） | 沙箱 / gateway 是 production reliability 的基础 |

---

*最后更新：2026-06-30 新增 U28（LLM Inference & Cost/Latency 工程）*
*来源：Anthropic / OpenAI 官方文档、vLLM / SGLang 论文、Karpathy 教学视频、Langfuse / Helicone 文档*
