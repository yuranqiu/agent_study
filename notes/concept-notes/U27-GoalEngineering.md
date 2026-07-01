# U27 · Goal Engineering（目标工程）

> **核心问题**：怎么把"用户的模糊意图"转化成 Agent 可执行的"明确目标"？为什么目标定义的质量决定了 Agent 90% 的表现？
> **预计学时**：3h
> **对应 Phase**：Phase 2 / Phase 3
> **状态**：2026 新兴概念（cobusgreyling 兄弟项目，与 Loop Engineering 互补）

---

## 1. 什么是 Goal Engineering

**Goal Engineering** 由 Cobus Greyling 在 2026 推出，与 Loop Engineering 是姊妹概念：

> "Goal engineering complements loop engineering. Loops discover, goals finish."
> —— [Cobus Greyling: Goal Engineering](https://github.com/cobusgreyling/goal-engineering)

**核心问题**：

> 大部分 Agent 失败不是因为模型不够强，而是因为**目标定义不清楚**。

Goal Engineering 解决：
- 用户的"模糊意图" → Agent 可执行的"明确目标"
- 目标要可衡量、可验证、有边界
- 目标层级的拆分（lifetime goal → milestone → task）
- 目标的 completion criteria

代表项目：
- **[cobusgreyling/goal-engineering](https://github.com/cobusgreyling/goal-engineering)**（Cobus 兄弟项目，伴随 loop-engineering）
- **ultragoal**（oh-my-hermes 内置 skill）
- **SPEC.md / GOAL.md 模式**（LoopX / Loom / paper_agent 通用）

---

## 2. Goal Engineering 与 Loop Engineering 的关系

```
┌──────────────────────────────────────────┐
│ Goal Engineering (U27) ★                 │
│   "What is the goal?"                    │
│   - 目标定义 / 拆解 / 验收                │
│   - 模糊意图 → 明确目标                   │
│   - 一锤子定调                            │
├──────────────────────────────────────────┤
│ Loop Engineering (U21)                   │
│   "How does the goal get achieved over time?" │
│   - 持续运转 / 状态保持 / handoff         │
│   - 多次迭代 / 跨 session                 │
├──────────────────────────────────────────┤
│ Harness Engineering (U23)                │
│   "How does each turn run?"              │
└──────────────────────────────────────────┘
```

> **一句话**：**Goal Engineering 负责"目标"（开始时），Loop Engineering 负责"持续"（过程中）**。

---

## 3. Goal 的 5 大属性

一个**好的 Agent 目标**必须满足：

| 属性 | 说明 | 反例 |
|------|------|------|
| **Specific（具体）** | 不是"做个好的应用"，而是"做一个能展示 10 张图片的 React 页面" | "帮我做个东西" |
| **Measurable（可衡量）** | 有明确 success criteria | "让代码跑起来" |
| **Achievable（可达成）** | Agent 当前能力能做 | "训练一个超越 GPT-4 的模型" |
| **Relevant（相关）** | 对用户真的有用 | "顺便做一下 XX" |
| **Time-bound（有时限）** | 有截止时间或迭代次数 | "随便做" |

——这正是经典的 **SMART 原则**，但 Goal Engineering 给出了**Agent 场景下的具体化**。

---

## 4. 4 层目标结构

Cobus Greyling 把目标拆成 4 层：

```
Lifetime Goal          → "我的项目要 9 月前上线"
   ↓
Strategic Goal         → "v1.0 实现核心功能"
   ↓
Milestone              → "完成用户认证 + 数据持久化 + UI"
   ↓
Tactical Task          → "用 FastAPI 实现 /login 接口 + 写测试 + 跑通"
```

每一层都有自己的：
- **Owner**（谁负责）
- **Verification**（怎么验证完成）
- **Dependencies**（依赖什么）
- **Failure mode**（失败怎么办）

**Agent 工程师的关键工作**：在 Loop Engineering 的每个 turn 里，**判断当前在哪一层**。

---

## 5. Goal 定义实战

### 5.1 反面：模糊目标

```
用户：帮我做个网站
Agent：（开始瞎写）
```

### 5.2 正面：Goal-Engineered

```markdown
# GOAL.md

## Lifetime Goal
2026-09-01 前上线一个可对外展示的个人作品集网站。

## v1.0 Scope（Strategic Goal）
- 一个静态 SPA，能展示 10 个项目卡片
- 响应式，支持移动端
- 有联系表单（用 Formspree）
- 部署到 Vercel

## Acceptance Criteria
1. 主页加载 < 1.5s（LCP）
2. 10 个项目卡片正确显示，含图片 + 描述 + 链接
3. 移动端（375px 宽）布局正常
4. 联系表单提交成功收到 Formspree webhook
5. Lighthouse 性能分 > 90
6. URL: `https://portfolio.example.com` 可访问

## Out of Scope
- 后端
- 用户登录
- CMS

## Verification Plan
- 单元测试：组件渲染
- E2E 测试：Playwright 跑全流程
- 手动：手机真机看响应式

## Tech Stack（约束）
- React + Vite（不要 Next.js）
- Tailwind CSS
- 部署：Vercel
```

**关键**：**Acceptance Criteria 必须是机器可验证的**。

---

## 6. Goal Engineering 工具 / 规范

### 6.1 GOAL.md / SPEC.md 模式

主流 Loop / Harness 项目都约定**一个 GOAL.md 或 SPEC.md 文件**作为目标规范：

| 项目 | 文件 | 工具 |
|------|------|------|
| LoopX | `.codex/goals/<goal-id>/GOAL.md` | `loopx bootstrap` |
| Loom | 用户自然语言输入 → Loom 生成 plan | `/loom build ...` |
| paper_agent | User input + Spec | 自动 |
| oh-my-hermes | `ultragoal` skill | 显式调用 |
| Codex CLI | `AGENTS.md` + 用户任务 | 显式 |

### 6.2 ultragoal：长跑目标

oh-my-hermes 提供的 ultragoal skill 专门做"长跑目标"管理：
- 把 lifetime goal 拆成 milestone
- 每个 milestone 拆成 task
- 每个 task 都有 acceptance criteria
- failure 怎么办 / escalation

### 6.3 Stack Cookbook

Cobus Greyling 发布的 [stack cookbook](https://github.com/cobusgreyling/goal-engineering/blob/main/docs/stack-cookbook.md) 列出常见的 goal stack 组合：
- L1 / L2 / L3 复杂度怎么定
- goal vs task vs step 的区分
- 怎么 stack 多个 goal

---

## 7. Goal Engineering 在 Agent 循环里的位置

```
┌──────────────────────────────────────────────┐
│ TURN 1: Goal Engineering                      │
│   - 解析用户意图                              │
│   - 写出 GOAL.md / SPEC.md                    │
│   - 设置 acceptance criteria                  │
│   - 拆解成 milestone + task                   │
├──────────────────────────────────────────────┤
│ TURN 2..N: Loop Engineering                   │
│   - 每个 turn 检查"我现在在哪一层目标"        │
│   - 完成 task → 标记 → 进入下一个            │
│   - 失败 / stuck → 重新 goal re-engineer      │
├──────────────────────────────────────────────┤
│ FINAL: Verification                          │
│   - 跑 acceptance criteria                   │
│   - 验证完成                                  │
│   - 关闭 goal                                 │
└──────────────────────────────────────────────┘
```

---

## 8. 失败模式：常见的 Goal Engineering 错误

| 错误 | 后果 | 修复 |
|------|------|------|
| **Goal 太模糊** | Agent 乱做 | 用 SMART 原则 + 写 acceptance criteria |
| **Acceptance criteria 不可验证** | Agent 自我宣称完成 | 写机器可验证的（test / E2E / command） |
| **Goal scope 没边界** | 无限扩张 | 显式 Out of Scope |
| **Goal 层级混乱**（milestone 当 task 用） | 长跑目标失控 | 4 层结构 + 每一层 owner 明确 |
| **没 failure plan** | 失败时不知道怎么办 | 写"如果 X 失败怎么办" |
| **Goal 一直 re-engineer** | 反复重新开始 | 设 re-engineer 次数上限 |

---

## 9. Goal Engineering 与其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U3 Prompt Engineering | Goal 写清楚 prompt 也少写 |
| U7 Planning & Reasoning | Goal 是 planning 的输入 |
| U8 Memory | 把 GOAL.md 存到 memory，跨 session 持续 |
| U9 Multi-Agent | Multi-Agent 时每个 Agent 都有自己的 goal |
| U12 Agent 评估标准 | Acceptance criteria = eval 集 |
| U20 编程 Agent | 编程 Agent 最常因 Goal 模糊而失败 |
| U21 Loop Engineering | 互补：Goal 解决"做什么"，Loop 解决"怎么持续做" |
| U22 Context Engineering | GOAL.md 是 context 的一部分 |
| U23 Harness Engineering | Goal 状态属于 Harness 组件 5（Task state） |
| U26 Agent Skills | "Goal Engineering" 本身可以做成 Skill |

---

## 10. 面试视角

- **Q：什么是 Goal Engineering？**
  → 把"用户模糊意图"转化成"Agent 可执行明确目标"的工程化方法。强调 SMART + acceptance criteria + scope 边界。Loop Engineering 的姊妹概念。

- **Q：Goal Engineering 和 Loop Engineering 的区别？**
  → Goal 负责"开始时——做什么"，Loop 负责"过程中——怎么持续做"。Goal 是"图纸"，Loop 是"施工"。

- **Q：怎么写一个好的 Agent goal？**
  → 4 层结构（lifetime → strategic → milestone → task）+ SMART + 机器可验证的 acceptance criteria + 显式 out of scope + failure plan。

- **Q：为什么 Agent 经常做错事？**
  → 90% 的失败来自 goal 定义不清：模糊 / 不可验证 / 边界不明 / 没 failure plan。修复方法：Goal Engineering。

- **Q：Acceptance criteria 怎么写？**
  → 必须是机器可验证的：测试 / E2E / 命令输出 / 性能指标。**不要**写"代码看起来对"。

- **Q：Goal Engineering 和 Prompt Engineering 的关系？**
  → Goal 是"做什么"，Prompt 是"怎么说"。Goal 错了，prompt 再好也救不回来。

---

## 11. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Cobus Greyling: Goal Engineering](https://github.com/cobusgreyling/goal-engineering) | 概念 + 工具（goal doctor） |
| [Cobus Greyling: Goal Engineering Essay](https://cobusgreyling.substack.com/p/goal-engineering) | 概念入门 |
| [oh-my-hermes: ultragoal skill](https://github.com/rlaope/oh-my-hermes) | 长跑目标的实操模式 |
| [LoopX: lifetime-goal invariant](https://github.com/huangruiteng/loopx/blob/main/docs/architecture.md) | 目标在控制面里的实现 |
| [Matt Pocock: Skills - requirement clarification](https://github.com/mattpocock/skills) | Goal Engineering 的实操 skill |
| [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | Goal 与 agent 设计 |

---

## 12. 练习

1. **写 GOAL.md**：为自己当前的学习项目写一份 GOAL.md，包含 acceptance criteria 和 out of scope
2. **4 层拆解**：把一个 lifetime goal 拆到 task 层
3. **Acceptance criteria check**：拿一份已有的 prompt / 任务，检查它的 acceptance criteria 是否可机器验证
4. **Re-engineer 实验**：故意写一个模糊 goal，让 Agent 跑，看它怎么走偏；然后 re-engineer，再跑
5. **ultragoal 实践**：如果用 oh-my-hermes，跑一遍 ultragoal skill，体验长跑目标管理
6. **Goal vs Loop 对比**：在同一个项目上分别用"Goal-only"（一次性）和"Goal + Loop"（持续）模式跑，对比最终质量

---

*最后更新：2026-06-30 新增 U27（Goal Engineering）*
*来源：Cobus Greyling Goal Engineering 概念、oh-my-hermes ultragoal、LoopX lifetime-goal*
