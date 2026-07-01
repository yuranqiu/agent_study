# Tool Guide · AutoGen

> 覆盖：安装 / 核心概念 / 最小可运行代码 / 常见坑点 / 面试高频问法

---

## 1. 安装

```bash
pip install autogen-agentchat
```

---

## 2. 核心概念

| 组件 | 说明 |
|------|------|
| `AssistantAgent` | LLM 驱动，执行代码/生成响应 |
| `UserProxyAgent` | 模拟用户，触发代码执行/人类输入 |
| `GroupChat` | 多 Agent 群聊协调 |
| `ConversableAgent` | 基类，支持自定义对话行为 |

---

## 3. 最小可运行代码

### 3.1 基本对话（Assistant + UserProxy）

```python
from autogen import AssistantAgent, UserProxyAgent

assistant = AssistantAgent(
    "assistant",
    llm_config={"model": "gpt-4o-mini"}
)
user = UserProxyAgent("user")

user.initiate_chat(assistant, message="写一个快速排序")
```

### 3.2 带代码执行

```python
from autogen import AssistantAgent, UserProxyAgent

assistant = AssistantAgent(
    "assistant",
    llm_config={"model": "gpt-4o-mini"}
)
user = UserProxyAgent(
    "user",
    code_execution_config={"work_dir": ".", "use_docker": False}
)

user.initiate_chat(assistant, message="计算 1+1 并打印结果")
```

### 3.3 多 Agent GroupChat

```python
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager

coder = AssistantAgent("coder", llm_config={"model": "gpt-4o-mini"})
reviewer = AssistantAgent("reviewer", llm_config={"model": "gpt-4o-mini"})

groupchat = GroupChat(
    agents=[coder, reviewer],
    messages=[],
    max_round=10
)
manager = GroupChatManager(groupchat=groupchat, llm_config={"model": "gpt-4o-mini"})

user.initiate_chat(manager, message="写一个排序算法并 review")
```

### 3.4 自定义 Tool

```python
from autogen import AssistantAgent

@register_function
def get_weather(city: str) -> str:
    return f"{city} 天气晴"

assistant = AssistantAgent(
    "assistant",
    llm_config={"model": "gpt-4o-mini"},
    function_map={"get_weather": get_weather}
)
```

---

## 4. 常见坑点

| 坑 | 说明 | 解法 |
|----|------|------|
| 代码不执行 | 没配 `code_execution_config` | 加 `code_execution_config={"use_docker": False}` |
| `max_turn` 限制 | 达到上限对话中断 | 调 `max_turn` 参数 |
| GroupChat 死循环 | 消息没有出口 | 设 `max_round` 或自定义 `speaker_selection_method` |
| 模型不支持 function calling | AutoGen 需要 function calling | 用支持 FC 的模型（GPT-4 / Claude） |

---

## 5. 面试高频问法

### Q: AutoGen 核心组件？

→ AssistantAgent（LLM 驱动）+ UserProxyAgent（用户模拟/代码执行）+ GroupChat（多 Agent 协调）

### Q: AssistantAgent 和 UserProxyAgent 的区别？

→ AssistantAgent：LLM 驱动生成响应，可以执行代码
→ UserProxyAgent：模拟用户，可以触发代码执行、接受人类输入

### Q: AutoGen 如何实现多 Agent 协作？

→ `GroupChat` + `GroupChatManager`
→ 多个 Agent 在群聊中互相消息，由 manager 协调

### Q: AutoGen 代码执行机制？

→ UserProxyAgent 内置 code execution
→ 设置 `code_execution_config={"work_dir": ".", "use_docker": False}`

---

*最后更新：2026-05-27*