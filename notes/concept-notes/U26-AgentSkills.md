# U26 · Agent Skills & AGENTS.md（Agent 能力工程化）

> **核心问题**：怎么把"项目知识 / 工具说明 / 最佳实践"以**可发现、可加载、可组合**的方式提供给 Agent，而不是堆在 system prompt 里？
> **预计学时**：4h
> **对应 Phase**：Phase 3（项目实战）
> **状态**：2025-2026 工业界新共识（Anthropic Skills、AGENTS.md、Cline、Cursor Rules 等）

---

## 1. 什么是 Agent Skills

**Agent Skills** 是 2024 末到 2025 年由 Anthropic / 社区推动的**能力工程化模式**：把"Agent 怎么用工具、怎么遵循项目规范、怎么执行特定任务"封装成**可发现、可加载、可组合**的模块，而不是堆进 system prompt。

**核心思想**：
- **Don't put everything in system prompt**——prompt 越长越贵、越容易冲突
- **Skill = a directory of instructions + resources**——按需加载
- **Discoverable**——Agent 知道有什么 skill 可用
- **Composable**——多个 skill 可以组合
- **Versioned**——和代码一样版本管理

代表实现：
- **Anthropic Skills**（Claude.ai / Claude Code / Agent SDK 通用）
- **AGENTS.md**（GitHub 主推的开放标准）
- **.cursorrules**（Cursor 专用）
- **CLAUDE.md**（Claude Code 专用）
- **Cline .clinerules**（Cline 专用）
- **Aider conventions**（Aider 专用）

---

## 2. 三大主流规范的对比

| 规范 | 来源 | 适用 Agent | 文件 | 特点 |
|------|------|-----------|------|------|
| **AGENTS.md** | GitHub / 社区共识 | 通用（任何 Agent） | `AGENTS.md` | 开放标准、单一文件 |
| **CLAUDE.md** | Anthropic | Claude Code | `CLAUDE.md` | 自动加载、可拆分到子目录 |
| **Skills** | Anthropic Agent SDK | Claude / Agent SDK | `SKILL.md` + 资源目录 | 模块化、组合、按需加载 |
| **.cursorrules** | Cursor | Cursor IDE | `.cursorrules` | 简单、单文件 |
| **.clinerules** | Cline | Cline | `.clinerules` | 简单 |

> **2026 趋势**：AGENTS.md 正在成为事实标准，被多家工具支持。GitHub、Cursor、Claude Code、Aider、Cline 都在向它靠拢。

---

## 3. AGENTS.md 标准

### 3.1 文件结构

```markdown
# AGENTS.md

## Build / Test
- 用 `uv run pytest` 跑测试
- 用 `ruff check .` 做 lint

## Code Style
- 所有 public 函数必须有 docstring
- 用 type hints，不要用 `Any`
- 错误处理：用 exception，不要用 return code

## Project Layout
- `src/` 是主代码
- `tests/` 是测试
- `docs/` 是文档

## PR Conventions
- 一个 PR 一个功能
- 标题用 [Conventional Commits](https://conventionalcommits.org/)
- 必须有 issue 链接

## Agent-Specific Notes
- 不要修改 `migrations/` 目录
- 数据库 schema 改动必须同时改 `models.py` 和 `schema.sql`
```

**关键**：**简洁**。AGENTS.md 不应该超过 100 行。详细内容放链接或子目录。

### 3.2 多级 AGENTS.md

GitHub 规范支持多级 AGENTS.md（嵌套加载，子目录的 AGENTS.md 优先级更高）：

```
repo/
├── AGENTS.md                 # 全局规则
├── src/
│   ├── api/
│   │   └── AGENTS.md         # API 模块特定规则
│   └── frontend/
│       └── AGENTS.md         # 前端模块特定规则
└── docs/
    └── AGENTS.md             # 文档模块特定规则
```

Agent 在 `src/api/` 目录工作时，会同时加载 root AGENTS.md + src/api/AGENTS.md。

### 3.3 GitHub 主推的开放标准

GitHub 在 2025 推动 AGENTS.md 成为**跨工具标准**，原因：
- 一次写，多 Agent 通用（不用为每个 Agent 写一份）
- 在 PR Review 时可见（code review 的人能 review Agent 规则）
- 跟代码一起版本管理

---

## 4. Anthropic Skills（最成熟的实现）

### 4.1 什么是 Skill

Anthropic Skills = **a directory of instructions + resources**：

```
my-skill/
├── SKILL.md              # 入口：描述 skill 是干嘛的，何时用
├── scripts/              # 可执行脚本
│   └── run.sh
├── references/           # 文档资源
│   ├── api-docs.md
│   └── conventions.md
└── assets/               # 模板、示例
    └── template.json
```

**SKILL.md 格式**：
```markdown
---
name: code-review
description: Review code for style, correctness, and security
when_to_use: 当用户要求 review 代码时
---

# Code Review Skill

## Steps
1. 用 `git diff` 看当前改动
2. 读改动的文件
3. 对照 `references/conventions.md` 检查
4. 输出 review 报告

## Output Format
- 用 Markdown
- 按严重程度排序：critical / warning / nit
- 每个问题给出 file:line
```

### 4.2 怎么用 Skills

**Claude.ai**：上传 skill 文件夹
**Claude Code**：放在 `~/.claude/skills/` 或项目 `.claude/skills/`
**Agent SDK**：用 `Skill(loader=...)` 加载

**Skills 的运行时行为**：
- **Discovery**：Agent 启动时知道有哪些 skill 可用
- **Lazy loading**：skill 资源按需加载（不一次性塞进 context）
- **Composition**：多个 skill 可以链式调用

### 4.3 Skill vs System Prompt vs Sub-agent

| 维度 | System Prompt | Skill | Sub-agent |
|------|---------------|-------|-----------|
| 用途 | 通用行为 | 特定任务方法 | 隔离的子任务 |
| 加载时机 | 每次都加载 | 按需加载 | 调用时启动 |
| 复杂度 | 简单 | 中等 | 高 |
| Context 占用 | 全部在 | 选择性 | 隔离 |

**实际选型**：
- **简单的项目规则** → AGENTS.md
- **特定任务方法论**（code review、debug、deploy） → Skill
- **复杂独立任务**（研究、benchmark、refactor） → Sub-agent

---

## 5. 实战：用 AGENTS.md 规范项目

### 5.1 最小可用的 AGENTS.md

```markdown
# AGENTS.md

## Build
- `uv sync` 安装依赖
- `uv run pytest` 跑测试
- `uv run ruff check .` 做 lint

## Layout
- `src/` 主代码
- `tests/` 测试
- `docs/` 文档

## Conventions
- Type hints everywhere
- Public functions need docstrings
- Errors: use exceptions, not return codes

## Don'ts
- Don't modify `migrations/`
- Don't push to main directly
```

放在 repo 根目录，git commit。**所有 Agent 自动加载**。

### 5.2 进阶：分层 AGENTS.md

```markdown
# src/api/AGENTS.md

## API Conventions
- 路由用 kebab-case：`/user-profile`
- 所有 endpoint 必须有 OpenAPI 注解
- 错误返回统一用 `{"error": "code", "message": "..."}`

## Testing
- 每个 endpoint 至少 3 个测试：success / 4xx / 5xx
- 用 `httpx.AsyncClient` + `pytest-asyncio`
```

放在 `src/api/`，只在 Agent 操作该目录时加载。

### 5.3 Skill 示例：Code Review Skill

```markdown
# skills/code-review/SKILL.md
---
name: code-review
description: Review code for style, correctness, and security
---

# Code Review

## When to use
- 用户说"review 一下" / "检查代码" / "PR review"
- 不适用：用户问"怎么写"

## Steps
1. `git diff main..HEAD` 看改动
2. 读改动的每个文件
3. 按顺序检查：
   - Correctness（逻辑错误、边界 case）
   - Security（注入、auth 漏洞、敏感数据）
   - Performance（N+1 query、O(n²) 循环）
   - Style（项目 conventions）
4. 输出 review 报告

## Output Format
```markdown
## Critical (must fix)
- `src/api/auth.py:42` — 密码用明文比较，应该用 `bcrypt`

## Warnings (should fix)
- `src/api/users.py:15` — N+1 query

## Nits
- `src/api/utils.py:8` — 缺 docstring
```
```

---

## 6. 工具映射（2025-2026）

| 工具 | 配置方式 | 加载时机 |
|------|----------|----------|
| **Claude Code** | `CLAUDE.md` + `.claude/skills/` | 自动 |
| **Cursor** | `.cursorrules` + AGENTS.md（2025+ 支持） | 自动 |
| **Cline** | `.clinerules` + AGENTS.md | 自动 |
| **Aider** | `CONVENTIONS.md` + `--read` flag | 显式 |
| **GitHub Copilot Coding Agent** | `.github/agents.md`（实验） | 任务时 |
| **Codex CLI** | `AGENTS.md` | 自动（2025+） |
| **OpenAI Codex Web** | Repository-level instructions | 任务时 |

---

## 7. Skill 设计原则

### 7.1 好的 Skill

- **单一职责**：一个 skill 干一件事
- **明确触发条件**：`when_to_use` 字段清晰
- **可执行步骤**：不是抽象建议，是可执行 steps
- **输出格式明确**：规定 markdown / JSON / 等等
- **包含示例**：典型用例 + 输出
- **资源按目录组织**：references / scripts / assets

### 7.2 不好的 Skill

- 太泛（"做一个好代码"）—— 没可执行步骤
- 把所有内容塞进 SKILL.md —— 违背 lazy load 原则
- 没有 when_to_use —— Agent 不知道何时触发
- 没有 output format —— Agent 输出不一致
- 没有示例 —— 第一次跑会乱猜

### 7.3 Skill vs System Prompt 的取舍

- **稳定 + 全局 + 短** → 放 system prompt
- **特定任务 + 长 + 可复用** → 放 Skill
- **项目特定规则** → 放 AGENTS.md
- **Skills 不要超 200 行**——超过就拆

---

## 8. 面试视角

- **Q：什么是 AGENTS.md？和 CLAUDE.md、.cursorrules 有什么区别？**
  → AGENTS.md 是 GitHub 推动的**跨工具开放标准**，单一 Markdown 文件，放在 repo 根目录，所有 Agent 通用。CLAUDE.md 是 Claude Code 专用，.cursorrules 是 Cursor 专用。趋势是 AGENTS.md 成为事实标准。

- **Q：Anthropic Skills 是什么？和 system prompt 有什么区别？**
  → Skill 是一个目录（SKILL.md + resources），按需 lazy load，可发现、可组合。System prompt 每次都全量加载。Skill 适合特定任务方法论，system prompt 适合全局短规则。

- **Q：什么时候用 Skill vs Sub-agent？**
  → Skill = 同一 context 内的"做事方法"，Sub-agent = 独立 context 的"独立任务"。Skill 轻，Sub-agent 重。**用 Skill** 当任务能在主 context 内完成时；**用 Sub-agent** 当任务需要隔离的 context 或独立资源时。

- **Q：多级 AGENTS.md 怎么工作？**
  → GitHub 规范：每个目录可以有 AGENTS.md。Agent 操作某目录时，加载从根到该目录路径上所有 AGENTS.md，越近的优先级越高。允许"全局规则 + 模块特定规则"分层。

- **Q：怎么写一个好的 SKILL.md？**
  → 单一职责 + 明确 when_to_use + 可执行 steps + 明确 output format + 包含示例 + resources 按目录组织。**不要**超 200 行——超了就拆。

- **Q：AGENTS.md 怎么版本管理？**
  → 跟代码一起 git 管理。PR review 时要 review AGENTS.md 的改动。重大规则改动要写 changelog。

---

## 9. 必读材料

| 材料 | 为什么读 |
|------|----------|
| [GitHub: AGENTS.md 规范](https://github.com/openai/agents.md) 或 [agents.md](https://agents.md/) | 开放标准 |
| [Anthropic: Skills 文档](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/overview) | 最成熟的 Skill 实现 |
| [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | Skill 在 Agent 中的位置 |
| [Claude Code: CLAUDE.md](https://code.claude.com/docs/en/memory) | Claude Code 的 memory / project 规则 |
| [Cursor: .cursorrules 文档](https://docs.cursor.com/context/rules) | Cursor 的规则系统 |
| [Matt Pocock: Skills](https://github.com/mattpocock/skills) | 一组高质量示例 skills（requirement clarification / TDD / handoff） |

---

## 10. 练习

1. **写第一个 AGENTS.md**：给自己的项目写一份 50 行内的 AGENTS.md
2. **写第一个 Skill**：写一个 "code-review" Skill（参考 §5.3），用 Anthropic Agent SDK 跑通
3. **多级 AGENTS.md**：在项目里设置 root + 子目录两层 AGENTS.md
4. **Skill vs Prompt 对比**：同样任务，分别用 system prompt 和 Skill 跑，对比 token 成本和输出质量
5. **跨工具兼容**：在同一个 repo 里写 AGENTS.md，看 Claude Code / Cursor / Codex 哪个自动识别

---

## 11. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U3 Prompt Engineering | Skills 是"项目级 prompt engineering"——可加载、可版本管理 |
| U9 Multi-Agent | Sub-agent 加载独立 Skill 是常见模式 |
| U20 编程 Agent 内部原理 | CLAUDE.md / AGENTS.md / Skills 是 Coding Agent 的"上下文设计"核心 |
| U22 Context Engineering | Skills 是 Context Engineering 的工程化产物（lazy load + on-demand） |
| U23 Harness Engineering | Skills 属于 Harness 组件 4（Project Memory）的实现细节 |
| U25 Computer Use | GUI Agent 也可以用 Skills 加载"特定应用的操作知识" |

---

*最后更新：2026-06-30 新增 U26（Agent Skills & AGENTS.md）*
*来源：Anthropic Skills 文档、GitHub AGENTS.md 规范、Matt Pocock Skills、Cursor/Cline/Aider 文档*
