# Interview QA · Agent 核心

> 来源：Navedphysicist/100-Agentic-AI-Interview-Questions + vibe-agentic-interview-notes
> 覆盖：基础概念 / 设计模式 / ReAct / Tool Use / Memory / Planning / Multi-Agent

---

## 一、基础概念篇

### Q: 什么是 Generative AI？和传统 AI 的区别是什么？

**Answer:**
- Generative AI 创建新内容（文本/图像/音频），而非只分析或分类已有数据
- 传统 AI = 预测/分类；GenAI = 生成创造
- 代表架构：LLM、GAN、VAE
- 关键能力：Zero-shot / Few-shot learning，无需针对任务训练

**面试追问：** GPT 和 GAN 的区别？→ 一个是自回归语言模型，一个是生成器-判别器对抗训练

---

### Q: 什么是 Agentic AI？和普通 LLM 的区别是什么？

**Answer:**
- Agentic AI = 能感知环境、做决策、执行动作达成目标的自主系统
- 核心区别：**主动行为**，不需要人类持续输入
- 五大组件：Perception / Planning / Memory / Reasoning / Action
- 代表产品：GitHub Copilot Agents、Devin、Claude Code

---

### Q: LLM 和 SLM 的区别是什么？什么时候用 SLM？

**Answer:**
- SLM = 参数量更少（通常 < 10B），推理更快，成本更低，可本地/边缘部署
- 代表：Phi-3、Mistral-7B、Gemma
- 适用：资源受限场景、特定垂直任务、隐私敏感（不上云）
- Trade-off：能力上限不如大模型，但够用的场景成本更低

---

### Q: Tokenization 是什么？为什么重要？

**Answer:**
- 把原始文本切成模型能处理的 token（词/子词/字符）
- 方法：BPE、WordPiece、SentencePiece
- 重要性：API 定价按 token 计、Context Window 有限、跨语言效率不同
- 子词 tokenization 平衡词表大小和处理未知词能力

**追问：** 一个中文字是多少 token？→ 通常 1-2 个 token（取决于模型）

---

## 二、ReAct 范式篇

### Q: 什么是 ReAct？和 CoT、Function Call 的区别？

**Answer:**

| 范式 | 过程 | 适用场景 |
|------|------|---------|
| CoT | 推理→推理→推理→答案 | 数学推导、逻辑分析 |
| Function Call | 指令→直接调Tool→结果 | 简单指令（查天气） |
| **ReAct** | 推理→行动→观察→推理→...→答案 | 需要外部知识的复杂任务 |

**ReAct 核心**：交替进行推理和行动，用真实反馈修正推理方向，避免幻觉

---

### Q: ReAct 四步循环是什么？手写伪代码

**Answer:**

```
Thought: 分析当前状态，决定下一步
Action: 调用 Tool（search, calculate, etc.）
Observation: 获取 Tool 返回结果
→ 如果完成 → Final Answer
→ 如果未完成 → 回到 Thought 继续循环
```

**伪代码：**
```python
while not done and iteration < max_iter:
    thought = llm.reason(context)          # 推理
    action = llm.select_tool(thought)     # 选 Tool
    obs = execute_tool(action)             # 执行
    context += f"Action: {action}\nObs: {obs}"
return llm.generate_final(context)
```

---

### Q: ReAct 在什么场景下优于其他方案？

**Answer:**
- 复杂多步骤任务，需要外部知识（数据库/搜索/API）
- 模型需要在行动后验证假设是否正确
- 避免幻觉：长推理链中途插入真实反馈
- 场景：研究助手、代码调试、多跳问答

---

## 三、Tool Use 篇

### Q: OpenAI Function Calling 格式（手写 JSON Schema）

**Answer:**

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "查询城市天气，返回温度和天气状况",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {
          "type": "string",
          "description": "城市名称，如'北京'"
        },
        "unit": {
          "type": "string",
          "enum": ["celsius", "fahrenheit"],
          "description": "温度单位，默认 celsius"
        }
      },
      "required": ["city"]
    }
  }
}
```

---

### Q: OpenAI Function Calling 和 Claude Tool Use 的区别

**Answer:**
- OpenAI：`tools` 参数传入，`function_call` 指定模型行为
- Claude：`tools` 数组，每项有 `name` + `description` + `input_schema`
- 核心相同：都是让模型决定调用哪个工具，传入结构化参数
- 细节：Claude 支持 `tool_choice: {type: "tool", name: "..."}` 指定固定工具

---

### Q: Tool Choice 策略有哪些？适用场景？

**Answer:**

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| `auto` | 模型自己决定调用哪个 | 通用场景 |
| `none` | 不调用工具，直接回复 | 简单问答 |
| `required` | 必须调用一个工具 | 需要外部数据时 |
| `{name: "..."}` | 强制调用指定工具 | 固定工作流 |

---

### Q: 实现多 Tool 调用链路（带依赖处理）

**Answer:**

```python
tools = [get_weather, search_web, calculate]

def run_agent(query, max_iter=10):
    messages = [{"role": "user", "content": query}]
    for i in range(max_iter):
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools,
            tool_choice="auto"
        )
        msg = response.choices[0].message
        if msg.tool_calls:
            for tc in msg.tool_calls:
                tool_name = tc.function.name
                args = json.loads(tc.function.arguments)
                result = execute_tool(tool_name, args)  # 执行工具
                messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})
        else:
            return msg.content  # 无 tool call，直接回复
```

---

## 四、Memory 篇

### Q: Agent 的 Memory 分哪几类？

**Answer:**

| 类型 | 描述 | 示例 |
|------|------|------|
| **Short-term** | 维持当前对话上下文 | 最近 N 条消息 |
| **Long-term** | 跨会话持久记忆 | 用户偏好、项目知识 |
| **Episodic** | 记录特定事件/经历 | "上次开会讨论了 X" |
| **Procedural** | 存储操作流程/工作流 | "这类任务的标准步骤" |

---

### Q: 如何实现滑动窗口记忆？

**Answer:**

```python
class SlidingWindowMemory:
    def __init__(self, window_size=10):
        self.window_size = window_size
        self.messages = []

    def add(self, message):
        self.messages.append(message)
        if len(self.messages) > self.window_size:
            self.messages.pop(0)  # 移除最早的

    def get_context(self):
        return "\n".join(self.messages)
```

---

### Q: 为什么要用 VectorDB 做长期记忆？

**Answer:**
- 滑动窗口有上限，不能无限增长
- VectorDB 通过语义相似度检索，不受时间顺序限制
- 可以跨会话记忆"用户上次问过 X，用的方案是 Y"
- 结合 RAG：在推理时检索相关记忆片段，拼入 context

---

## 五、Planning 篇

### Q: 什么是 GSM-SF（Goal-Step-Motion-Subgoal-Formulation）？

**Answer:**
一种任务分解框架：
1. **Goal**：定义最终目标
2. **Step**：拆解为可执行步骤
3. **Subgoal**：每个大步骤的子目标
4. **Formulation**：把子目标映射为具体 Action

**本质**：把模糊的"我要做网站"变成可追溯的步骤树

---

### Q: 什么是 ToT（Tree of Thoughts）？和 CoT 的区别？

**Answer:**
- CoT：线性推理链，每步只选一个方向
- ToT：树状探索，每步可以生成多个候选路径，然后评估选择最优
- ToT 适合：需要回溯、多方案比较的复杂任务（如算法设计）
- 实现代价更高，需要更多 token

---

### Q: 什么是 Self-Reflection / Reflexion？

**Answer:**
Agent 在执行完一个步骤后，显式反思：
- "我做到了吗？"
- "距离目标还有多远？"
- "我犯的错误是什么？"

Reflexion 是带外部评估的自我反思：通过额外 LLM 评价执行结果，写入记忆供后续迭代使用。

---

## 六、Multi-Agent 篇

### Q: Multi-Agent 和 Single Agent 的核心区别是什么？

**Answer:**
- Single Agent：一个 LLM 做所有决策
- Multi-Agent：多个专业 Agent 协作，每个专注特定角色/任务
- 优势：并行处理、专业化、避免单点上下文爆炸
- 问题：协调成本、消息路由、状态同步

---

### Q: Agent 之间如何通信？

**Answer:**
- **星型**：主 Agent 协调，子 Agent 不直接互通（Claude Code sub-agent 模型）
- **网状**：Agent 间直接 SendMessage（Agent Team 模型）
- **广播**：一个 Agent 消息广播给所有 Agent

---

### Q: 什么是 Orchestrator Pattern？

**Answer:**
中央协调者分解任务、分派给子 Agent、收集结果、聚合输出。
- 适合：多步骤流程（研究→写作→审核）
- 协调者需要维护任务状态和依赖关系

---

## 七、设计模式篇

### Q: 什么是 Prompt Chaining？

**Answer:**
把复杂任务分解为多个串行 Prompt，每个输出作为下一个输入。
```
分析任务 → 写大纲 → 生成内容 → 审核修改
```
- 优点：模块化、易调试、错误隔离
- 缺点：串行延迟

---

### Q: 什么是 Router Pattern？

**Answer:**
根据输入内容动态路由到不同的专业处理分支。
```
用户问题 → 路由判断（intent分类）→ billing/technical/general
```
- 优点：专业化处理、扩展性好
- 实现关键：准确的 intent classification

---

### Q: 什么是 Tool Integration Pattern？

**Answer:**
Agent 动态选择可用工具，基于任务描述匹配最合适的工具。
- 工具注册表：所有可用工具的描述和签名
- 选择策略：Rule-based / LLM-driven / Semantic similarity

---

## 八、综合理解篇（开放题）

### Q: 手写一个最小可运行的 ReAct Agent

**Answer:**

```python
import json
from openai import OpenAI

client = OpenAI()

tools = [
    {
        "type": "function",
        "function": {
            "name": "search",
            "description": "Search the web",
            "parameters": {"type": "object", "properties": {"query": {"type": "string"}}}
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "Calculate math expression",
            "parameters": {"type": "object", "properties": {"expr": {"type": "string"}}}
        }
    }
]

def execute_tool(name, args):
    if name == "search": return f"Results for {args['query']}"
    if name == "calculate": return str(eval(args["expr"]))

def run_agent(query, max_iter=5):
    messages = [{"role": "user", "content": query}]
    for _ in range(max_iter):
        response = client.chat.completions.create(
            model="gpt-4o-mini", messages=messages, tools=tools
        )
        choice = response.choices[0]
        if choice.finish_reason == "stop":
            return choice.message.content
        if choice.message.tool_calls:
            tc = choice.message.tool_calls[0]
            result = execute_tool(tc.function.name, json.loads(tc.function.arguments))
            messages.append({"role": "assistant", "content": None, "tool_calls": [tc]})
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})
    return "max iterations reached"

print(run_agent("What's 15 * 23?"))
```

---

### Q: Agent 的评估指标有哪些？

**Answer:**

| 维度 | 指标 | 说明 |
|------|------|------|
| 准确性 | Task Success Rate | 任务是否完成 |
| 效率 | 迭代次数 / Token 消耗 | 成本控制 |
| 可靠性 | 错误率 / 恢复能力 | 容错 |
| 安全性 | 越界操作次数 | 权限控制 |
| 可解释性 | 决策链路清晰度 | Trace/log |

---

*最后更新：2026-05-27*