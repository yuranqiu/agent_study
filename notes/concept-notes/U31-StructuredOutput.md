# U31 · Structured Output & Validation（结构化输出与验证）

> **核心问题**：怎么让 LLM 输出**可靠、可解析、符合 schema** 的结构化数据？这是 Agent 工具调用、自动化工作流、数据抽取的基础。
> **预计学时**：4h
> **对应 Phase**：Phase 2 / 项目实战
> **状态**：生产必修（2024-2026 范式已稳定）

---

## 1. 为什么 Structured Output 重要

Agent 工程的 80% 时间花在**解析 LLM 输出**上。如果 LLM 输出不稳定：
- Tool calling 失败（schema 不匹配）
- 数据 pipeline 断裂（解析报错）
- 下游系统崩溃（类型错乱）

**核心问题**：LLM 是生成模型，**它的输出本质是 token 序列，不是结构化对象**。我们必须"驯服"它。

```
LLM 输出 (string) → 解析 → 结构化对象 (JSON / Pydantic) → 下游消费
      ↑                                              ↑
      不稳定 ──────────── 不可控 ────────────────────  不可用
```

---

## 2. 4 大技术层级

```
Level 4: 自定义语法（最严）
    └─ JSON Schema + Function Calling
Level 3: Schema 强制
    └─ OpenAI Structured Outputs / Anthropic Tool Use
Level 2: 后处理 + 重试
    └─ Instructor / Outlines / Guidance
Level 1: 基础 Prompt
    └─ "Please output JSON"
```

**Level 越高，可靠性越高，成本越高**。生产系统通常用 Level 3 + Level 2 组合。

---

## 3. Level 1：基础 Prompt 引导

```python
prompt = """
Extract the user info from the text below. Output JSON only.

Text: "John is 30 years old and works at Anthropic"

JSON:
"""
response = llm.invoke(prompt)
data = json.loads(response)  # 经常失败！
```

**问题**：
- 模型可能在 JSON 前后加 `Here is the JSON: ...`
- JSON 语法错（缺引号、缺逗号）
- schema 错（字段名拼错、类型错）

**仅适合**：原型 / 一次性脚本 / 容忍错误的人类在环场景。

---

## 4. Level 2：库辅助 + 后处理

### 4.1 Instructor（最流行）

[Instructor](https://github.com/jxnel/instructor) = Pydantic + LLM，自动 retry + 验证。

```python
import instructor
from pydantic import BaseModel
from openai import OpenAI

client = instructor.from_openai(OpenAI())

class User(BaseModel):
    name: str
    age: int
    company: str | None = None

user = client.chat.completions.create(
    model="gpt-4o",
    response_model=User,  # ← Pydantic model 直接作为 response schema
    messages=[{"role": "user", "content": "John is 30 years old and works at Anthropic"}],
    max_retries=3,  # 自动重试
)
print(user)  # User(name='John', age=30, company='Anthropic')
```

**核心特性**：
- **Pydantic 集成**：直接用 Pydantic model 作为 schema
- **自动 retry**：JSON 解析失败 / validation 失败自动重试
- **Validation hook**：自定义 validator（用 `field_validator` / `model_validator`）
- **Streaming**：支持流式 Pydantic 对象

### 4.2 Outlines

[Outlines](https://github.com/outlines-dev/outlines) = 用正则/JSON Schema/Grammar 约束 LLM 输出。

```python
import outlines
from outlines import models, generate

model = models.transformers("Qwen/Qwen2.5-7B-Instruct")

# JSON Schema 模式
schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer", "minimum": 0, "maximum": 150},
    },
    "required": ["name", "age"],
}

generator = generate.json(model, schema)
user = generator("John is 30 years old and works at Anthropic")
print(user)  # {"name": "John", "age": 30}
```

**核心特性**：
- **Logit masking**：在生成时**直接屏蔽不合规 token**（不是后处理）
- **支持多种约束**：JSON Schema / Regex / CFG / Context-free grammar
- **本地模型友好**：不需要 OpenAI API

### 4.3 Guidance（Microsoft）

[Guidance](https://github.com/guidance-ai/guidance) = 用模板 + 约束生成。

```python
from guidance import models, gen

model = models.OpenAI("gpt-4o")

with model.session() as session:
    session += f"Extract user info.\n"
    session += f"Name: {gen('name', regex=r'[A-Z][a-z]+')}\n"
    session += f"Age: {gen('age', regex=r'\d{1,3}')}\n"
    session += f"JSON: {{\"name\": \"{gen('name_json', stop='\"')}\", \"age\": {gen('age_json', regex=r'\d+')}}}"
```

**特点**：基于 token 级别的控制，灵活但 API 复杂。

### 4.4 Pydantic + LangChain

LangChain 也支持 Pydantic：

```python
from langchain_core.pydantic_v1 import BaseModel
from langchain_openai import ChatOpenAI

class User(BaseModel):
    name: str
    age: int

llm = ChatOpenAI(model="gpt-4o").with_structured_output(User)
user = llm.invoke("John is 30 years old")
```

---

## 5. Level 3：Provider 原生 Structured Outputs

### 5.1 OpenAI Structured Outputs

OpenAI 2024-08 推出，**100% schema 遵循**（不是概率，是 constrained decoding）。

```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    messages=[{
        "role": "user",
        "content": "Extract user info from: John is 30 years old"
    }],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "user_extraction",
            "schema": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "age": {"type": "integer"},
                },
                "required": ["name", "age"],
                "additionalProperties": False,
            },
            "strict": True,  # ← 关键：100% 保证 schema
        },
    },
)

user = json.loads(response.choices[0].message.content)
```

**关键区别**：
- 旧 `json_object` mode：只是 prompt 引导，**模型可以输出任何 JSON**
- 新 `json_schema` + `strict: True`：constrained decoding，**token-level 约束**

### 5.2 Anthropic Tool Use (structured)

Anthropic 用 tool use 实现 structured output：

```python
import anthropic
client = anthropic.Anthropic()

tools = [{
    "name": "extract_user",
    "description": "Extract user info from text",
    "input_schema": {
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "age": {"type": "integer"},
        },
        "required": ["name", "age"],
    },
}]

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_user"},
    messages=[{"role": "user", "content": "John is 30 years old"}],
)

# tool_use block 一定符合 input_schema
tool_input = response.content[0].input
```

### 5.3 Gemini Function Calling

类似 Anthropic，用 function calling 实现。

### 5.4 vLLM / Outlines（本地模型）

```python
import vllm
from outlines import generate, models

# 本地部署 + 约束生成
llm = models.vllm("meta-llama/Llama-3.1-8B-Instruct")
generator = generate.json(llm, schema)
result = generator("...")
```

---

## 6. Level 4：自定义 Grammar / EBNF

最高级——用**CFG（上下文无关文法）**完全约束生成。

### 6.1 应用场景

- **代码生成**：必须遵循某语言的语法
- **SQL 生成**：必须合法 SQL
- **JSON-LD / XML**：严格 schema

### 6.2 工具

- **Outlines**（最方便）
- **Guidance**
- **lm-format-enforcer**
- **jsonformer**

```python
# Outlines 用 context-free grammar
import outlines
from outlines import models, generate
from outlines.types import Grammar

# 自定义 EBNF
sql_grammar = Grammar("""
    start: select_statement
    select_statement: "SELECT " column_list " FROM " table_name
    column_list: column ("," column)*
    column: /[a-zA-Z_][a-zA-Z0-9_]*/
    table_name: /[a-zA-Z_][a-zA-Z0-9_]*/
""")

generator = generate.cfg(models.transformers("Qwen/Qwen2.5-7B"), sql_grammar)
query = generator("Find all users in the users table")
print(query)  # "SELECT * FROM users"（合法 SQL）
```

---

## 7. 实战选型

| 场景 | 推荐 |
|------|------|
| **OpenAI / Anthropic API** | 用 Provider 原生（Level 3） |
| **本地开源模型 + 生产稳定性** | Outlines（logit masking） |
| **已有 Pydantic 代码** | Instructor（增量集成） |
| **复杂 schema / 嵌套** | OpenAI Structured Outputs + Pydantic |
| **代码生成** | Outlines + Grammar |
| **快速原型** | Instructor（最少代码） |

---

## 8. 实战：完整的 Agent Tool 定义

```python
from pydantic import BaseModel, Field
from typing import Literal
import instructor
from openai import OpenAI

# 1. 定义 Tool schema
class GetWeatherInput(BaseModel):
    city: str = Field(description="城市名，如'北京'、'上海'")
    unit: Literal["celsius", "fahrenheit"] = Field(default="celsius")

class SendEmailInput(BaseModel):
    to: str = Field(description="收件人邮箱")
    subject: str = Field(description="邮件主题")
    body: str = Field(description="邮件正文")

# 2. 让 LLM 选择调用哪个 tool
client = instructor.from_openai(OpenAI())

tool_call = client.chat.completions.create(
    model="gpt-4o",
    response_model=Union[GetWeatherInput, SendEmailInput],  # ← 多个 tool 任选
    messages=[{"role": "user", "content": "查一下北京天气"}],
)

# 3. 解析 + dispatch
if isinstance(tool_call, GetWeatherInput):
    result = weather_api(city=tool_call.city, unit=tool_call.unit)
elif isinstance(tool_call, SendEmailInput):
    result = email_api(to=tool_call.to, subject=tool_call.subject, body=tool_call.body)
```

**关键**：用 Pydantic + Union，LLM 自动选择调用哪个 tool，**且 schema 100% 保证**。

---

## 9. 验证（Validation）三件套

### 9.1 Pydantic Validators

```python
from pydantic import BaseModel, field_validator, model_validator

class SearchQuery(BaseModel):
    query: str
    max_results: int = 10
    safe_search: bool = True

    @field_validator("query")
    @classmethod
    def query_not_empty(cls, v):
        if not v.strip():
            raise ValueError("query 不能为空")
        return v.strip()

    @field_validator("max_results")
    @classmethod
    def max_results_in_range(cls, v):
        if v < 1 or v > 100:
            raise ValueError("max_results 必须在 1-100")
        return v
```

### 9.2 重试策略

```python
# Instructor 自动 retry
user = client.chat.completions.create(
    response_model=User,
    messages=[...],
    max_retries=3,  # ← 失败重试 3 次
    validation_context={"forbid_words": ["bad", "evil"]},  # 传给 validator
)
```

### 9.3 回退策略

```python
# 多模型 fallback
def extract_with_fallback(text):
    for model in ["gpt-4o", "gpt-4o-mini", "claude-3-5-sonnet"]:
        try:
            return client.chat.completions.create(
                model=model, response_model=User, messages=[...]
            )
        except Exception:
            continue
    raise AllModelsFailedError()
```

---

## 10. Agent 工具调用的设计模式

### 10.1 简单 tool（1 个）

```python
tool = {"type": "function", "function": {"name": "get_weather", ...}}
```

### 10.2 多 tool 互斥（Union）

```python
response_model = Union[GetWeather, SendEmail, Search]
```

### 10.3 多 tool 并行（同 prompt 多个 tool）

```python
# OpenAI parallel tool calls
response = client.chat.completions.create(
    model="gpt-4o",
    tools=[weather_tool, search_tool, email_tool],
    parallel_tool_calls=True,  # ← 一次返回多个 tool call
    messages=[...],
)
# response.choices[0].message.tool_calls 是 list
```

### 10.4 嵌套 tool（tool 输出作为 tool 输入）

```python
class ComplexTask(BaseModel):
    plan: list[SubTask]  # 嵌套子任务
    
class SubTask(BaseModel):
    tool_name: str
    args: dict
```

---

## 11. 面试视角

- **Q：怎么保证 LLM 输出符合 schema？**
  → 三层防线：① Provider 原生（OpenAI Structured Outputs / Anthropic Tool Use，token-level 约束） ② 库（Instructor / Outlines，constrained decoding 或自动重试） ③ 验证（Pydantic validator）。**生产必须三层都用**。

- **Q：JSON mode 和 Structured Outputs 有什么区别？**
  → JSON mode 只是 prompt 引导，模型**可能**输出任何 JSON。Structured Outputs（`strict: true`）用 constrained decoding 在 token level 屏蔽不合规 token，**100% 保证**符合 schema。

- **Q：Instructor 是什么？**
  → Instructor = Pydantic + LLM 库。提供自动 retry、Pydantic 验证 hook、流式输出、Union 模式（多 tool 互斥）。最小代码量接入 structured output。

- **Q：Outlines 和 Instructor 怎么选？**
  → Instructor 适合 OpenAI / Anthropic API。Outlines 适合本地模型（用 logit masking 在生成时屏蔽不合规 token）。功能有重叠，按部署场景选。

- **Q：Agent tool call 怎么保证 schema 100% 可靠？**
  → ① Tool schema 用 Pydantic 严格定义（带 validator） ② Provider 用 structured outputs（constrained decoding） ③ Post-hoc 用 Pydantic 验证 ④ 失败 retry + fallback。**只信 prompt 工程是不够的**。

- **Q：怎么用 Pydantic 定义一个嵌套的 tool schema？**
  → 用嵌套 Pydantic model + `list[SubModel]` / `Union[Model1, Model2]`。OpenAI Structured Outputs 和 Instructor 都支持。

---

## 12. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Instructor 文档](https://github.com/jxnel/instructor) | 实战首选 |
| [Outlines 文档](https://outlines-dev.github.io/outlines/) | 本地模型首选 |
| [OpenAI Structured Outputs 指南](https://platform.openai.com/docs/guides/structured-outputs) | Provider 原生 |
| [Pydantic 文档](https://docs.pydantic.dev/) | schema 验证基础 |
| [Anthropic Tool Use 文档](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview) | Claude tool calling |

---

## 13. 练习

1. **Instructor 实战**：用 Pydantic 定义一个 User model，用 Instructor 抽取
2. **Outlines 实战**：用本地 Qwen 模型 + Outlines 做 JSON 抽取
3. **OpenAI Structured Outputs**：用 strict mode 定义 tool，验证 100% schema 遵循
4. **多 tool Union**：定义 3 个互斥 tool，让 LLM 自动选
5. **Validation hook**：写一个 Pydantic validator，校验 email 格式 + 长度限制
6. **失败 retry + fallback**：实现一个 3 次重试 + 多 model fallback 的抽取器

---

## 14. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U3 Prompt 工程 | JSON Mode 是 Level 1 prompt engineering，U31 升级到 Level 3/4 |
| U6 Tool Use & Function Calling | U6 讲 tool definition 格式，U31 讲怎么保证 100% schema 遵循 |
| U14 DSPy | DSPy 用 Signature 定义 schema，是另一种 structured output 范式 |
| U23 Harness Engineering | Structured Output 是 Harness 组件 8（Verification）的具体实现 |
| U34 Guardrails | Guardrails 在 U31 之上加 input/output 验证 + 业务规则 |

---

*最后更新：2026-06-30 新增 U31（Structured Output & Validation）*
*来源：Instructor / Outlines / OpenAI Structured Outputs / Pydantic 官方文档*
