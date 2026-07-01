# U5 · ReAct 范式

> **核心问题**：Agent 怎么把推理和行动交替起来？
> **预计学时**：5h
> **对应 Phase**：Phase 2

---

## 学习目标（Gate 检验）

1. 能画 ReAct 四步循环图（Thought → Action → Observation → Next Thought）
2. 能解释 ReAct 和纯 CoT / 纯 Function Call 的本质区别
3. 能用 LangChain ReAct 实现一个带 3 个 Tool 的 Agent
4. 能说清楚 ReAct 在什么场景下优于其他方案

---

## 1. ReAct 是什么

### 全称

**Re**asoning + **Act**ion = ReAct

### 核心思想

Agent 在执行任务时，交替进行**推理**和**行动**，而不是一次性规划完再执行，也不是盲目行动。

- **Thought（思考）**：分析当前状态，决定下一步做什么
- **Action（行动）**：执行一个动作（如调用 Tool、查数据库）
- **Observation（观察）**：获取行动的结果
- **Next Thought（下一步思考）**：基于观察更新认知，循环直到完成

### 与其他范式的对比

| 范式 | 过程 | 适用场景 |
|------|------|---------|
| 纯 CoT（Chain-of-Thought） | 推理 → 推理 → 推理 → 最终答案 | 数学推导、逻辑分析 |
| 纯 Function Call | 用户指令 → 直接调 Tool → 返回结果 | 简单指令（查天气、设闹钟） |
| ReAct | 推理 → 行动 → 观察 → 推理 → 行动 → ... → 最终答案 | 需要外部知识的复杂任务 |

### 为什么 ReAct 有优势

纯 CoT 的问题是"推理链可以很长，但模型不知道外部事实"——模型在推理过程中可能产生幻觉（自己推着推着跟现实对不上）。

ReAct 通过在推理过程中插入 Action 和 Observation，让模型**不断用真实反馈修正推理方向**，避免走偏。

---

## 2. 四步循环详解

```
┌─────────────────────────────────────────────────────┐
│  用户问题                                           │
└─────────────────────┬───────────────────────────────┘
                      ↓
              ┌───────────────┐
              │   Thought     │  分析：我需要知道北京天气
              │  （推理）      │  决策：调用天气查询工具
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │   Action      │  调用 weather_tool(city="北京")
              │   （行动）     │
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │  Observation  │  收到：{temp: 22°C, weather: 晴}
              │   （观察）     │
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │  Next Thought │  分析：天气已查到，可以回答用户了
              │ （下一步推理） │
              └───────┬───────┘
                      ↓
              ┌───────────────┐
              │   Final Answer │  输出：北京今天天气晴，气温 22°C
              └───────────────┘
```

---

## 3. 为什么需要 Thought

### 直觉

Thought 是让模型"说出来它在想什么"。这有三个好处：

1. **可调试**：从外部可以看到模型的推理过程
2. **自我纠正**：模型在说出推理后发现矛盾，可以自己纠正
3. **减少幻觉**：说出来比闷着想更暴露问题

### 经典案例：数学推理

```
问题：小明有 10 个苹果，掉了 3 个，又买了 5 个，现在有几个？

纯 CoT 模型：11 个。（直接出答案，可能算对也可能算错）

ReAct 模型：
Thought：首先计算当前数量。10 - 3 = 7，然后 7 + 5 = 12。
Action：执行加法 7 + 5
Observation：结果是 12
Next Thought：12 是最终答案
Final Answer：12 个
```

---

## 4. 实现：用 LangChain ReAct Agent

### 完整代码

```python
# pip install langchain-openai langchain-community

from langchain_openai import ChatOpenAI
from langchain.agents import AgentType, Tool, create_react_agent
from langchain_core.tools import tool
from langchain import hub

# 1. 定义 Tools
@tool
def calculator(expr: str) -> str:
    """执行数学计算，只接受简单的四则表达式。"""
    try:
        result = eval(expr, {"__builtins__": {}})
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"

@tool
def get_weather(city: str) -> str:
    """查询城市天气。输入城市名称，返回天气和温度。"""
    # 模拟天气数据（实际接入 API）
    weather_db = {
        "北京": "晴，22°C",
        "上海": "多云，25°C",
        "成都": "小雨，18°C",
    }
    return weather_db.get(city, f"未找到 {city} 的天气数据")

@tool
def search(query: str) -> str:
    """搜索引擎。输入查询词，返回摘要信息。"""
    # 模拟搜索结果（实际接入 Google/Bing API）
    return f"搜索结果：关于「{query}」的信息来自权威来源。"

# 2. 注册 Tools
tools = [calculator, get_weather, search]

# 3. 创建 Agent
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
agent = create_react_agent(llm, tools, hub.pull("hwchase17/react"))

# 4. 运行
result = agent.invoke({
    "input": "北京今天多少度？如果加上上海的温度，再乘以 2，结果是多少？"
})
print(result["output"])
```

### 运行输出（示例）

```
Thought：我需要先查询北京和上海的天气，然后用计算器算结果。
Action：get_weather
Action Input：{"city": "北京"}
Observation：北京天气晴，22°C

Thought：好的，北京天气已查到。现在查上海。
Action：get_weather
Action Input：{"city": "上海"}
Observation：上海天气多云，25°C

Thought：现在用计算器：(22 + 25) * 2
Action：calculator
Action Input：{"expr": "(22 + 25) * 2"}
Observation：94

Thought：已经得到最终结果。
Final Answer：北京今天 22°C，上海 25°C，按照你的计算，(22 + 25) * 2 = 94。
```

---

## 5. 从零手写 ReAct Agent（不含框架）

```python
import json
import openai

client = openai.OpenAI()

# 定义 Tools
tools = [
    {
        "type": "function",
        "function": {
            "name": "calculator",
            "description": "执行数学计算",
            "parameters": {
                "type": "object",
                "properties": {"expr": {"type": "string"}},
                "required": ["expr"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询城市天气",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }
    }
]

def calculator(expr: str) -> str:
    return str(eval(expr))

def get_weather(city: str) -> str:
    db = {"北京": "晴 22°C", "上海": "多云 25°C", "成都": "小雨 18°C"}
    return db.get(city, "未找到数据")

# ReAct 循环
MAX_ITER = 10

def run_react(question: str):
    messages = [
        {"role": "system", "content": "你是一个 ReAct Agent。\n\n你必须交替进行 Thought（推理）和 Action（行动）。\n\n格式示例：\nThought: 我需要查询天气\nAction: get_weather\nAction Input: {\"city\": \"北京\"}\nObservation: 北京天气晴，22°C\n\n如果问题已解决，输出：Final Answer: ..."},
        {"role": "user", "content": question}
    ]

    for i in range(MAX_ITER):
        # 调用 LLM
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools,
            tool_choice="auto",
        )

        msg = response.choices[0].message
        messages.append(msg)

        # 没有 tool_call → 直接回答
        if not msg.tool_calls:
            print(f"\nFinal Answer: {msg.content}")
            break

        # 执行每个 tool_call
        for call in msg.tool_calls:
            name = call.function.name
            args = json.loads(call.function.arguments)
            print(f"\nAction: {name}")
            print(f"Action Input: {args}")

            if name == "calculator":
                result = calculator(**args)
            elif name == "get_weather":
                result = get_weather(**args)
            else:
                result = "未知的 tool"

            obs = {"role": "tool", "tool_call_id": call.id, "content": result}
            messages.append(obs)
            print(f"Observation: {result}")

if __name__ == "__main__":
    run_react("北京多少度？温度加 10 等于多少？")
```

---

## 完成 Gate 检验

1. **Q1**：画 ReAct 四步循环图，并标注每步在做什么。
2. **Q2**：ReAct 和纯 CoT 的本质区别是什么？为什么复杂任务需要 ReAct？
3. **Q3**：为什么 Agent 需要"说出来 Thought"？有什么好处？
4. **Q4**：从零手写一个带两个 Tool 的 ReAct Agent 循环（不需要框架）。

---

## 练习清单

- [ ] 画 ReAct 四步循环的 Mermaid 图
- [ ] 用 LangChain ReAct Agent 实现一个带 3 个 Tool 的 Agent 并跑通
- [ ] 从零手写一个不含框架的 ReAct Agent（参考上面的实现）
- [ ] 对比 ReAct 和纯 Function Call 在同一个复杂问题上的输出差异

---

## 关联资源

- [ReAct 论文](https://arxiv.org/abs/2210.03629)（Abstract + Figure 1，30 分钟）
- [LangChain ReAct Agent](https://python.langchain.com/docs/tutorials/agent_executor/)
- [LangChain ReAct Agent 模板](https://smith.langchain.com/hub/hwchase17/react)

---

## 6. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U1 LLM 基础|ReAct 依赖 LLM 的推理能力，U1 的 next-token prediction 是底层|
|U3 Prompt 工程|ReAct 的 System Prompt 设计依赖 U3 的技巧|
|U6 Tool Use|ReAct 的 Action/Observation 循环就是 U6 的 Tool Calling|
|U7 Planning|ReAct 是轻量 planning，U7 的 ToT/Plan-and-Solve 是更复杂的规划|
|U21 Loop Engineering|ReAct 循环是 Loop Engineering 的基础案例|

*最后更新：2026-05-26*