# U6 · Tool Use 与 Function Calling

> **核心问题**：模型怎么调用外部工具？
> **预计学时**：5h
> **对应 Phase**：Phase 2

---

## 学习目标（Gate 检验）

1. 能写出符合 OpenAI Function Calling 格式的 Tool Definition
2. 能解释 OpenAI Function Calling 和 Claude Tool Use 的异同
3. 能实现一个多 Tool 调用的完整链路（带依赖处理）
4. 能说出 Tool Choice 策略的适用场景

---

## 1. Tool Definition 格式

### OpenAI Function Calling 格式

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
          "description": "城市名称，如'北京'、'上海'"
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

### Claude Tool Use 格式

```json
{
  "name": "get_weather",
  "description": "Queries the weather for a given city",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "City name"
      }
    },
    "required": ["city"]
  }
}
```

### 核心异同

| 方面 | OpenAI | Claude |
|------|--------|--------|
| Schema 格式 | JSON Schema | JSON Schema（相同） |
| 参数包装 | `arguments` 是 JSON 字符串 | `input` 直接是对象 |
| Tool 选择策略 | `tool_choice` 控制强制/自动 | `tool_choice` 同 |
| 多 Tool 调用 | 同时返回多个 | 同时返回多个 |

---

## 2. Tool 执行循环

### 标准流程

```
用户问题 → LLM（带 Tool Definitions）→ 
    ├── 无 tool_call → 直接回答
    └── 有 tool_call → 执行 Tool（获取 Observation）→ 
        → 拼回消息 → LLM 继续推理 → 
            ├── 继续调 Tool
            └── 最终回答
```

### 带依赖的多 Tool 调用

一个 Tool 的输出可以作为另一个 Tool 的输入：

```
Action: search
Action Input: {"query": "北京今日天气"}
Observation: [网页摘要1, 网页摘要2, 网页摘要3]

Action: get_weather
Action Input: {"city": "北京"}
Observation: 晴，22°C
```

顺序由 LLM 的推理决定（Thought 中会说明依赖关系）。

---

## 3. Tool Choice 策略

### OpenAI 的 tool_choice

```python
# 方式 1：auto（默认）—— 让模型自己决定
tool_choice = "auto"

# 方式 2：强制使用某个 Tool
tool_choice = {"type": "function", "function": {"name": "get_weather"}}

# 方式 3：强制使用任一 Tool（但不是指定某个）
tool_choice = "required"
```

### 适用场景

| 策略 | 适用场景 |
|------|---------|
| `auto` | 通用场景，模型自由判断 |
| 强制某个 Tool | 已知需要特定 Tool（如确认用户意图时） |
| `required` | 每个回合必须调 Tool（用于强制使用工具的 Agent） |

### Claude 的 tool_choice

Claude 也有完全相同的选项：`auto`、`any`、`tool`，行为语义相同。

---

## 4. 多 Tool 调用策略

### 并行调用

当多个 Tool 之间没有依赖时，可以让 LLM 同时调用：

```python
# 第一次调用：模型选择多个无依赖的 Tool
tool_calls = [
    {"id": "call_1", "function": {"name": "get_weather", "arguments": '{"city": "北京"}'}},
    {"id": "call_2", "function": {"name": "get_weather", "arguments": '{"city": "上海"}'}},
]

# 并行执行（Python asyncio）
results = await asyncio.gather(
    get_weather("北京"),
    get_weather("上海"),
)
```

### 顺序依赖

当 Tool B 需要 Tool A 的输出时，必须等 A 执行完再调用 B。

### 失败处理

| 策略 | 处理方式 |
|------|---------|
| 重试 | Tool 执行失败，自动重试 1~2 次 |
| 返回错误 | Tool 返回错误消息，模型决定是否继续 |
| 降级 | Tool 不可用时，使用备用方案或告知用户 |

---

## 5. 实战：实现多 Tool 链路

```python
import json
import openai

client = openai.OpenAI()

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询城市天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"}
                },
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search",
            "description": "搜索引擎",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"}
                },
                "required": ["query"]
            }
        }
    }
]

def execute_tool(name: str, args: dict) -> str:
    if name == "get_weather":
        db = {"北京": "晴 22°C", "上海": "多云 25°C"}
        return db.get(args["city"], "未找到")
    elif name == "search":
        return f"关于「{args['query']}」的搜索结果..."
    return "未知工具"

def run(question: str, max_turns=10):
    messages = [
        {"role": "system", "content": (
            "你是一个 Tool Agent。格式：\n"
            "Thought: ...\n"
            "Action: tool_name\n"
            "Action Input: {...}\n"
            "Observation: ...\n"
            "Final Answer: ..."
        )},
        {"role": "user", "content": question}
    ]

    for _ in range(max_turns):
        resp = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools,
            tool_choice="auto",
        )
        msg = resp.choices[0].message
        messages.append(msg)

        if not msg.tool_calls:
            print(f"Final: {msg.content}")
            break

        for call in msg.tool_calls:
            args = json.loads(call.function.arguments)
            result = execute_tool(call.function.name, args)
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": result
            })

run("北京天气怎么样？上海呢？")
```

---

## 完成 Gate 检验

1. **Q1**：写出天气查询和数学计算两个 Tool 的完整 JSON Schema 定义。
2. **Q2**：OpenAI Function Calling 和 Claude Tool Use 的三个主要异同。
3. **Q3**：什么情况下需要用强制 Tool Choice（`tool_choice = "required"`）？
4. **Q4**：当 Tool B 需要 Tool A 的输出时，怎么实现顺序依赖？

---

## 练习清单

- [ ] 定义 3 个 Tool（天气、搜索、计算器）的完整 JSON Schema
- [ ] 用 OpenAI API 实现一个多 Tool 调用的完整链路
- [ ] 实现并行调用两个没有依赖的 Tool（asyncio.gather）
- [ ] 观察模型在什么场景下选择不调用 Tool

---

## 关联资源

- [OpenAI Function Calling 文档](https://platform.openai.com/docs/guides/function-calling)
- [Claude Tool Use 文档](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [LangChain Tools](https://python.langchain.com/docs/concepts/tools/)

---

## 6. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U1 LLM 基础|Tool Calling 依赖 LLM 的函数调用能力，U1 的模型能力是基础|
|U3 Prompt 工程|Tool Definition 的 description 是 prompt 设计的一部分|
|U5 ReAct 范式|ReAct 的 Action 就是 Tool Call，U6 是 U5 的底层机制|
|U9 Multi-Agent|Multi-Agent 系统依赖 U6 的 Tool 定义和调用|
|U36 MCP 深入|MCP 是 Tool 调用的标准协议，U6 讲原理 U36 讲协议|

## 7. 高级 Tool Use 模式（生产必修）

### 7.1 并行 Tool 调用

当多个 Tool 之间**没有依赖**时，可以让 LLM 一次返回多个 tool_call，并行执行：

```python
# OpenAI 原生支持并行 tool call
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=tools,
)
msg = response.choices[0].message

# msg.tool_calls 可能包含多个调用
if msg.tool_calls:
    # 用 asyncio 并行执行
    results = await asyncio.gather(*[
        execute_tool(tc.function.name, json.loads(tc.function.arguments))
        for tc in msg.tool_calls
    ])
```

**适用场景**：同时查多个城市的天气 / 同时搜索多个关键词 / 批量文件操作

### 7.2 Tool 结果缓存

相同的 tool call 参数 + 相同的输入 → 应该返回缓存的结果：

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_weather(city: str, date: str) -> str:
    return get_weather(city, date)
```

**注意**：缓存 TTL 很重要——天气数据会过时，用 `functools.cached_property` + TTL 更安全

### 7.3 Tool 描述优化

Tool 的 `description` 字段直接影响模型的选择准确率：

|写法|问题|改进|
|---|---|---|
|'查询天气'|太泛，模型不知道何时调用|'查询指定城市未来 24 小时的天气，返回温度和天气状况。输入：城市名称（中文）。'|
|'计算'|不明确|'执行数学表达式计算，支持四则运算。输入：表达式字符串，如 "(2+3)*4"'|

**原则**：description 要包含 **when_to_use** + **input_format** + **output_format**
*最后更新：2026-05-26*