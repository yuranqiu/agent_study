# U25 · Computer Use / GUI Agent（计算机使用 / 图形界面 Agent）

> **核心问题**：Agent 怎么"像人一样"操作电脑——点击、拖拽、输入、读屏？
> **预计学时**：5h
> **对应 Phase**：Phase 3（项目实战）
> **状态**：2025 顶级热点（Anthropic / OpenAI / 字节 / 阿里全部入局）

---

## 1. 什么是 Computer Use / GUI Agent

**Computer Use Agent**（CUA / GUI Agent）是能像人类一样**通过操作图形界面完成任务**的 Agent。区别于"传统"通过 API 调用的 Agent：

| 类型 | 动作 | 例子 |
|------|------|------|
| **API Agent** | 调 HTTP API、SQL、Shell | LangChain、AutoGen、ReAct |
| **Browser Agent** | 操作浏览器 DOM / Playwright | browser-use、Browserbase |
| **Computer Use Agent** | 看屏幕截图 + 操作鼠标键盘 | Anthropic Computer Use、OpenAI Operator |

GUI Agent 是**最通用**的 Agent 形态——理论上能做任何"人能做"的事，因为 GUI 是人机交互的最终接口。

代表项目：
- **Anthropic Computer Use**（2024-10，Claude 3.5 Sonnet 首发）
- **OpenAI Operator / CUA**（2025-01，o3 模型驱动）
- **ByteDance UI-TARS**（开源，2025）
- **Alibaba WebAgent / MobileAgent**
- **browser-use**（开源，Playwright + LLM）
- **Skywork / ShowUI / Aguvis**（学术界）

---

## 2. 核心范式：Screenshot + Action Loop

GUI Agent 的本质是一个 **观察-决策-行动** 循环：

```
观察 (Screenshot) → 理解 (Vision Model) → 决策 (Action) → 执行 (OS) → 观察 (新 Screenshot)
```

每一步：
1. **截图**：截取当前屏幕（或特定区域）
2. **视觉理解**：用多模态 LLM 理解屏幕内容（识别按钮、文本、图标）
3. **决策**：输出一个 action（点击坐标、输入文字、滚动、等待）
4. **执行**：通过 OS API（macOS / Windows / Linux / 浏览器）执行
5. **回到 1**

```python
# 伪代码
def computer_use_loop(task, max_steps=50):
    for step in range(max_steps):
        screenshot = capture_screen()
        action = vision_model.invoke(
            task=task,
            history=action_history,
            screenshot=screenshot,
        )
        # action 可能是 {"type": "click", "x": 100, "y": 200}
        # 或 {"type": "type", "text": "hello"}
        # 或 {"type": "done", "answer": "..."}
        if action["type"] == "done":
            return action["answer"]
        execute(action)
        action_history.append(action)
```

---

## 3. 主流框架 / 模型

### 3.1 Anthropic Computer Use

Anthropic 在 2024-10 把 Computer Use 内置进 Claude 3.5 Sonnet 的 API：

```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    tools=[
        {
            "type": "computer_20241022",  # 内置 tool
            "name": "computer",
            "display_width_px": 1920,
            "display_height_px": 1080,
        }
    ],
    messages=[
        {"role": "user", "content": "请打开 Safari 访问 google.com 并搜索 'AI agent'"}
    ],
)

# Claude 会返回 tool_use：
# {"name": "computer", "input": {"action": "screenshot"}}
# 然后你执行 → 把截图发回 → Claude 返回下一步
```

**Anthropic 提供的 13 个内置 action**：
- `screenshot` / `left_click` / `right_click` / `double_click`
- `type` (输入文字) / `key` (按键盘按键)
- `cursor_position` / `scroll` / `wait`
- `zoom` / `hold_key` / `left_mouse_down` / `left_mouse_up`

### 3.2 OpenAI Operator / CUA

OpenAI 在 2025-01 发布 **Operator**，背后是 **CUA (Computer-Using Agent)** 模型，融合了 o3 的推理能力 + GPT-4o 的视觉能力：

- 浏览器内自动化（Operator 浏览器）
- 桌面级控制（Operator 桌面版）
- 在 OSWorld / WebArena 排行榜上 SOTA

API 形态（推测，未完全公开）：
- 输入：截图 + 任务
- 输出：action JSON
- 内置 safety check（涉及支付、隐私时暂停）

### 3.3 ByteDance UI-TARS

字节开源的多模态 Agent 模型，端到端训练：

```python
# UI-TARS-1.5: https://github.com/bytedance/UI-TARS
from ui_tars import UITarsModel

model = UITarsModel("ui-tars-1.5-7b")
action = model.predict(
    screenshot=screen,
    instruction="点击 'Submit' 按钮",
    history=history,
)
```

**特点**：在 PC / Web / Mobile 多平台上统一，单模型多模态。

### 3.4 browser-use

开源、Python、基于 Playwright：

```python
from browser_use import Agent
from langchain_openai import ChatOpenAI

agent = Agent(
    task="在 hacker news 上找到今天得分最高的帖子",
    llm=ChatOpenAI(model="gpt-4o"),
)
result = agent.run()
print(result)
```

**特点**：比直接用 Playwright 简单，比 Computer Use 速度快（不依赖截图，只操作 DOM）。

### 3.5 Aguvis / ShowUI / SeeClick

学术界工作，把屏幕理解 + 决策统一成一个 end-to-end 模型：
- 直接从 screenshot 输出 click coordinates
- 不需要外部 orchestrator
- 在 ScreenSpot / WebArena 等基准上 SOTA

---

## 4. GUI Agent 的关键技术挑战

### 4.1 屏幕理解（Vision）

- **分辨率敏感**：小元素（小按钮、小图标）容易被忽略
- **OCR 准确率**：屏幕文字识别
- **图标 / 视觉元素理解**：不是所有 UI 都有文字标签
- **状态变化感知**：弹窗、loading、动画

**应对**：高分辨率截图（Anthropic 提供 zoom tool）+ 多次采样 + DOM 辅助（browser-use 同时用 Playwright 拿 DOM）

### 4.2 动作定位（Action Grounding）

- **精确点击坐标**：模型要能输出 `(x, y)` 像素坐标
- **OS 差异**：macOS / Windows / Linux 的事件 API 不同
- **拖拽**：hold_key + left_mouse_down + left_mouse_up 三步

**应对**：端到端训练（UI-TARS / Aguvis）+ post-processing（坐标归一化、约束在屏幕内）

### 4.3 长任务记忆（Memory）

- 50 步以上的任务，context 增长爆炸
- 需要 compaction / note-taking
- 失败要能回滚

**应对**：复用 U22 Context Engineering（compaction）+ U8 Memory（NOTES.md）

### 4.4 错误恢复

- 误点 / 误输入 / 死循环
- 检测到 stuck 怎么办
- 撤销（undo / Ctrl+Z）

**应对**：action history + stuck detection（重复同样 action N 次） + safety guard

### 4.5 安全（Safety）

GUI Agent 权限巨大，能做任何"人"能做的事。**安全设计是核心**：
- 涉及支付的 action → 暂停 + 人类确认
- 涉及个人隐私 → 拦截
- 不可逆操作（删除 / 发送） → 二次确认
- sandbox 执行（Docker / 虚拟机）

**参考 U18 Agent 安全与红队 + U23 Harness 组件 9（Permissions）**

---

## 5. 评测基准

GUI Agent 的评测是开放问题。当前主流基准：

| 基准 | 平台 | 任务类型 | 难度 |
|------|------|----------|------|
| **OSWorld** | Ubuntu / Windows / macOS | 真实 OS 任务（设置、文件管理、应用操作） | Hard |
| **WebArena** | Web | 真实网站任务（购物、订餐、论坛） | Hard |
| **VisualWebArena** | Web | 多模态任务（图片理解 + 交互） | Hard |
| **ScreenSpot** | 多平台 | 屏幕元素定位（点击 / 选择） | Medium |
| **Mind2Web** | Web | 真实网站多步任务 | Hard |
| **MobileAgentBench** | iOS / Android | 移动端任务 | Medium |
| **AndroidWorld** | Android | 真实 Android 任务 | Hard |

> **SOTA 状态（2025 中）**：Anthropic Computer Use 在 OSWorld 上 ~38%，OpenAI CUA ~36%，UI-TARS ~28%。都还有巨大提升空间。

---

## 6. 实战选型决策

| 场景 | 推荐 |
|------|------|
| **网页自动化**（抓取、表单、测试） | browser-use + Playwright（快、便宜、可控） |
| **真实跨应用任务**（在桌面 app 之间） | Anthropic Computer Use / OpenAI Operator |
| **移动端自动化** | UI-TARS / Aguvis / MobileAgent |
| **学术研究 / 自定义训练** | ShowUI / SeeClick / Aguvis |
| **生产 SaaS** | Operator API + 自家 safety layer |

**避免**：
- 用 Computer Use 做"网页抓取"（慢 + 贵，browser-use 更适合）
- 用 Computer Use 做"需要 100% 准确"的生产任务（还做不到）

---

## 7. 实战最小代码

### 7.1 用 Anthropic Computer Use 跑一个浏览器任务

```python
import anthropic
import pyautogui
import base64
from io import BytesIO

client = anthropic.Anthropic()
history = []
task = "在浏览器中打开 google.com 并搜索 'AI agent'"

while True:
    # 1. 截屏
    screenshot = pyautogui.screenshot()
    buffered = BytesIO()
    screenshot.save(buffered, format="PNG")
    img_b64 = base64.b64encode(buffered.getvalue()).decode()

    # 2. 构造消息
    if not history:
        messages = [{"role": "user", "content": [
            {"type": "text", "text": task},
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": img_b64}}
        ]}]
    else:
        messages = history

    # 3. Claude 决策
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        tools=[{
            "type": "computer_20241022",
            "name": "computer",
            "display_width_px": 1920,
            "display_height_px": 1080,
        }],
        messages=messages,
    )

    # 4. 执行 action
    action = response.content[0].input if response.stop_reason == "tool_use" else None
    if not action:
        break

    if action["action"] == "left_click":
        pyautogui.click(action["coordinate"][0], action["coordinate"][1])
    elif action["action"] == "type":
        pyautogui.typewrite(action["text"])
    elif action["action"] == "done":
        print("完成:", action.get("answer"))
        break
    # ... 处理其他 action
```

### 7.2 用 browser-use 跑同一个任务

```python
from browser_use import Agent
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o")
agent = Agent(
    task="在 google.com 搜索 'AI agent'，告诉我前 3 个结果",
    llm=llm,
)
result = agent.run()  # 自动用 Playwright 操作浏览器
print(result)
```

**对比**：browser-use 不依赖截图，直接操作 DOM，更快更便宜。但只能在浏览器内。

---

## 8. 面试视角

- **Q：Computer Use / GUI Agent 和传统 Agent 的区别？**
  → 传统 Agent 通过 API / DB / 文件操作；GUI Agent 通过截图 + OS 事件操作图形界面。GUI Agent 通用性更强（能做任何人能做的事），但速度慢、贵、不稳定。

- **Q：Anthropic Computer Use 怎么工作的？**
  → 内置 `computer` tool（13 个 action），模型接收 screenshot，输出 action JSON（click 坐标、type 文本等），执行端用 PyAutoGUI / xdotool 等。完整 loop：screenshot → Claude 决策 → 执行 → screenshot。

- **Q：browser-use 和 Anthropic Computer Use 怎么选？**
  → browser-use 只在浏览器内，基于 Playwright + DOM，速度快、便宜、稳定。Computer Use 跨整个 OS，慢、贵、通用。**生产环境** 优先 browser-use；**跨应用 / 桌面 app** 才用 Computer Use。

- **Q：GUI Agent 最大的挑战是什么？**
  → 屏幕理解（小元素、图标）+ 精确坐标 grounding + 长任务 memory + 错误恢复 + 安全（涉及支付/隐私要拦截）。在 OSWorld 上 SOTA 也才 ~38%，还有巨大提升空间。

- **Q：Anthropic Computer Use 和 OpenAI CUA 哪个强？**
  → 2025 中 OSWorld 排名：Anthropic ~38% / OpenAI ~36% / UI-TARS ~28%。差距不大，都还没到生产可用级别。

- **Q：GUI Agent 怎么保证安全？**
  → 沙箱执行（Docker / 虚拟机）+ 危险操作二次确认（支付、删除、发送）+ 隐私拦截 + 完整 action audit log + max_steps 限制。详见 U18 / U23。

---

## 9. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Anthropic: Computer Use Overview](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use) | Anthropic 官方 Computer Use 文档 |
| [Anthropic Computer Use 博客（2024-10）](https://www.anthropic.com/news/3-5-sonnet-and-computer-use) | 发布时的设计思路 |
| [OpenAI Operator 介绍](https://openai.com/index/introducing-operator/) | OpenAI CUA 官方介绍 |
| [OSWorld Benchmark](https://osworld.dev/) | 真实 OS 任务评测基准 |
| [WebArena](https://webarena.dev/) | 真实网页任务评测 |
| [browser-use GitHub](https://github.com/browser-use/browser-use) | 27k+ stars，最实用的开源方案 |
| [ByteDance UI-TARS](https://github.com/bytedance/UI-TARS) | 端到端多模态 GUI Agent 模型 |
| [Aguvis Paper](https://arxiv.org/abs/2412.04454) | 视觉 GUI Agent 的统一框架 |

---

## 10. 练习

1. **跑通最小例子**：用 browser-use 完成一个 "打开 google.com 搜索某词" 的任务
2. **Anthropic Computer Use**：在 macOS 上跑官方 demo，让 Claude 帮你设置系统偏好
3. **OSWorld 跑分**：clone OSWorld 仓库，跑一个简单任务看 SOTA 水平
4. **加 safety layer**：在你自己的 GUI Agent 实现里加上"涉及 'pay' / 'delete' / 'send' 关键字的 action 必须 human confirm"
5. **browser-use vs Computer Use 对比**：用同样任务分别跑两个方案，对比延迟、成本、成功率
6. **UI-TARS 微调**：用 Aguvis / SeeClick 的数据微调一个小模型，做自定义 GUI Agent

---

## 11. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U17 多模态 Agent | GUI Agent 是多模态 Agent 的典型应用（屏幕截图 = 视觉） |
| U20 编程 Agent 内部原理 | Coding Agent + Computer Use 是 2025 最强组合（Cursor / Claude Code 路线） |
| U22 Context Engineering | GUI Agent 长任务需要 compaction（U22）+ Notes.md（U8） |
| U23 Harness Engineering | GUI Agent 的 sandbox / permission / observability 是 Harness 11 组件的具体应用 |
| U18 Agent 安全与红队 | GUI Agent 权限最大，安全设计最关键 |
| U24 Reasoning Model | OpenAI CUA 内部用 o3 推理，决定下一步 action；与 U24 强协同 |

---

*最后更新：2026-06-30 新增 U25（Computer Use / GUI Agent）*
*来源：Anthropic / OpenAI / 字节 / 阿里 / 学术界 2025 工作，OSWorld / WebArena 基准*
