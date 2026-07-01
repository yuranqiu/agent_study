# U39 · Permission & Auto Mode（权限与自动模式）

> **核心问题**：怎么让 agent 安全地**自主执行**有副作用的操作（写文件 / 跑命令 / 调 API）？怎么在"自主"和"安全"之间找平衡？
> **预计学时**：3h
> **对应 Phase**：Phase 3（项目实战）/ Phase 5
> **状态**：2025-2026 工业级标配（Claude Code / Codex / Cursor 都有 permission 系统）

---

## 1. 什么是 Agent Permission

**Agent Permission** = 决定 agent 在每个 step **能做什么 / 不能做什么**的系统。

**问题来源**：
- Coding agent 能写文件、跑命令、push 代码——**做错一步就灾难**
- LLM 输出不可预测——**不能完全信任**
- 用户希望 agent 自主——**但不能给 root 权限裸跑**

**4 大 Permission 模式**（从最严到最松）：

| 模式 | 适用 | 风险 |
|------|------|------|
| **Always confirm** | 危险操作（部署 / 支付） | 低 |
| **Confirm per session** | 一般项目 | 中 |
| **Allowlist + confirm unknown** | 自主工作流 | 中低 |
| **Auto mode** | 高信任场景 | 中 |

---

## 2. 主流框架的 Permission 实现

### 2.1 Claude Code Permissions

**3 个层次**：

```bash
# 1. Permission allowlist（永久允许某些 tool/命令）
/permissions

# 2. Permission modes（运行时模式）
claude --permission-mode default
claude --permission-mode accept-edits
claude --permission-mode plan
claude --permission-mode auto

# 3. Sandboxing（OS-level 隔离）
/sandbox
```

#### Default mode

```bash
# 每次潜在修改都问
claude --permission-mode default
```

**适用**：新手 / 第一次跑陌生 repo

#### Accept-edits mode

```bash
# 自动接受文件编辑，但仍问 Bash / 网络操作
claude --permission-mode accept-edits
```

**适用**：熟悉 codebase，主要改文件不跑命令

#### Plan mode

```bash
# 只读不写，Agent 只 plan 不 execute
claude --permission-mode plan
```

**适用**：exploration / 大设计 / 第一次改陌生代码

#### Auto mode（杀手特性）

```bash
# 用 classifier model 自动判断哪些操作需要 confirm
claude --permission-mode auto -p "fix all lint errors"
```

**原理**：
- 一个**小 classifier model**（不是 LLM）实时判断每个 action
- 分类"safe" → 自动通过
- 分类"risky" → 问人类
- 训练目标：识别"scope escalation"（破坏性命令、未知基础设施、hostile content）

**优势**：
- 比"全 confirm"少 90% 打断
- 比"全 auto"安全（有 classifier 兜底）
- 适合长跑任务（U21 Loop Engineering）

### 2.2 Claude Code Hooks（U33 也有提，详细看这里）

```json
// ~/.claude/settings.json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash|Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "if echo $CLAUDE_TOOL_INPUT | grep -qE 'rm -rf|drop table'; then exit 2; fi"
      }]
    }],
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "npx eslint --fix"
      }]
    }],
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "npm test"
      }]
    }]
  }
}
```

**Hook 类型**（Claude Code 全套）：
- `PreToolUse`：tool 调用前（拒绝 / 放行 / 修改）
- `PostToolUse`：tool 调用后（cleanup / validate）
- `Stop`：session 结束前（最后检查）
- `SubagentStart`：subagent 启动时
- `SubagentStop`：subagent 结束
- `Notification`：通知
- `UserPromptSubmit`：用户输入时

### 2.3 OpenAI Codex CLI

```bash
# Sandbox mode（默认）
codex --sandbox=workspace-write

# 不问任何事
codex --approval-mode=never

# 每次都问
codex --approval-mode=always

# 让 model 自己决定
codex --approval-mode=on-request
```

### 2.4 Cursor

```json
// .cursor/settings.json
{
  "permissions": {
    "allow": ["Bash(git *)", "Read", "Edit"],
    "deny": ["Bash(rm -rf *)", "Bash(sudo *)"]
  }
}
```

---

## 3. 4 大 Permission 策略

### 3.1 永久 allowlist

**适用**：你完全信任的命令（`git status`、`ls`、`pytest`）。

```json
// Claude Code
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff*)",
      "Bash(npm test)",
      "Bash(npm run lint)"
    ]
  }
}
```

### 3.2 永久 denylist

**适用**：永远不该 agent 跑的命令。

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf /)",
      "Bash(rm -rf ~)",
      "Bash(curl * | bash)",  // pipe to bash
      "Bash(sudo *)",
      "Bash(chmod 777 *)",
      "Bash(eval *)"
    ]
  }
}
```

### 3.3 询问模式（按上下文判断）

**适用**：危险但偶尔需要的操作（`git push` / `npm publish` / `kubectl apply`）。

```json
{
  "permissions": {
    "ask": [
      "Bash(git push*)",
      "Bash(kubectl apply*)",
      "Bash(rm -rf * | head -100)"
    ]
  }
}
```

### 3.4 Pattern-based Hook（regex + script）

**适用**：复杂判断（"deploy only if on main branch"）。

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "if echo $CLAUDE_TOOL_INPUT | grep -qE 'deploy' && git branch --show-current | grep -v main; then echo 'Refusing: can only deploy from main' >&2; exit 2; fi"
      }]
    }]
  }
}
```

---

## 4. Auto Mode 深度

### 4.1 Classifier Model 设计

Auto mode 背后的 classifier model 不是 LLM，是**专门训练的小模型**。

**输入**：action description + context
**输出**：safe / risky / unknown

**训练数据**：
- Anthropic 内部标注了 100k+ actions
- "safe" = 常见的 read / 写文件 / git 操作
- "risky" = sudo / rm -rf / push --force / 不可逆操作
- "unknown" = 罕见 / 自定义命令

**关键设计**：classifier **不允许 escalate scope**——
- 不能"git push"到任意 remote
- 不能"rm"非 git-tracked 文件
- 不能"eval"任意代码

### 4.2 Auto Mode 限制

```bash
# Auto mode 在 -p (non-interactive) 下：
# - 8 次 classifier 拒绝后 → 自动 abort
# - 没有人类 fallback → 安全第一
claude --permission-mode auto -p "long task"
```

**使用建议**：
- ✅ 适合：长跑 exploration / 已知 codebase / 信任的任务
- ❌ 不适合：陌生 codebase / 不可逆操作 / 高风险任务

### 4.3 Hybrid：Auto + Hook

```json
{
  "permissionMode": "auto",
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "deny-list-checker.sh"  // 你自己写的 denylist
      }]
    }]
  }
}
```

**优势**：auto mode 处理 95% 决策，hook 兜底 5% 真正危险的。

---

## 5. Sandbox 隔离（OS Level）

**Permission 是 prompt-level 限制，Sandbox 是 OS-level 限制**。

### 5.1 4 种 Sandbox 方案

| 方案 | 隔离度 | 性能开销 | 适用 |
|------|--------|----------|------|
| **Process** | 弱 | 0% | 不可信代码禁止 |
| **Docker** | 中 | 1-2s 启动 | 标准 coding agent |
| **gVisor** | 强 | 5-10% | 多租户 / 半可信 |
| **Firecracker** | 强 | <125ms | Serverless / 高 QPS |
| **完整 VM** | 极强 | 秒-分钟 | 金融/医疗 |

### 5.2 Claude Code Sandbox

```bash
# 启用 sandbox
/sandbox

# 配置 sandbox 规则（filesystem + network）
{
  "sandbox": {
    "filesystem": {
      "allow": ["~/projects/**"],
      "deny": ["~/.ssh", "~/.aws"]
    },
    "network": {
      "allowDomains": ["github.com", "pypi.org"]
    }
  }
}
```

**效果**：agent 在 sandbox 里跑，所有 file / network access 都受限制。即使 agent 失控，sandbox 保护主系统。

### 5.3 Codex Sandbox

```bash
# workspace-write: 只能写当前 workspace
codex --sandbox=workspace-write

# danger-full-access: 完全访问（不推荐）
codex --sandbox=danger-full-access
```

---

## 6. 5 大 Permission 反模式

| 反模式 | 后果 | 修复 |
|--------|------|------|
| **完全 auto，无 allowlist** | agent 跑任意命令 | 加 allowlist |
| **完全 deny** | agent 啥都干不了 | 用 ask 而非 deny |
| **denylist 不更新** | 新的危险操作没拦住 | 定期更新 denylist |
| **hook 出错静默** | hook 失败 agent 仍继续 | hook 失败 = 拒绝 |
| **sandbox 配置错** | sandbox 失效 | 测试 sandbox 是否真的隔离 |

---

## 7. 实战：完整 Claude Code Permission 配置

```json
// ~/.claude/settings.json
{
  "permissionMode": "default",
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(npm run build)",
      "Bash(ls*)",
      "Bash(cat*)",
      "Bash(find*)",
      "Bash(grep*)",
      "Read",
      "Edit"
    ],
    "deny": [
      "Bash(rm -rf /)",
      "Bash(rm -rf ~)",
      "Bash(sudo *)",
      "Bash(curl * | bash)",
      "Bash(eval *)",
      "Bash(chmod 777 *)",
      "Bash(:(){:|:&};:)"  // fork bomb
    ]
  },
  "sandbox": {
    "filesystem": {
      "allow": ["/Users/me/projects/**"],
      "deny": ["~/.ssh", "~/.aws", "~/.gnupg"]
    },
    "network": {
      "allowDomains": [
        "github.com",
        "pypi.org",
        "registry.npmjs.org",
        "*.amazonaws.com"
      ]
    }
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "validate-bash-command.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "npm test"
          }
        ]
      }
    ]
  }
}
```

**配套脚本** `validate-bash-command.sh`：
```bash
#!/bin/bash
# 验证 Bash 命令是否安全
INPUT="$CLAUDE_TOOL_INPUT"

# 禁止的 patterns
if echo "$INPUT" | grep -qE '(rm -rf|curl.*\|.*bash|eval|sudo|chmod 777)'; then
  echo "BLOCKED: dangerous pattern detected" >&2
  exit 2
fi

# 写文件到 /etc /usr /var 禁止
if echo "$INPUT" | grep -qE '(/etc/|/usr/|/var/)'; then
  echo "BLOCKED: writing to system directory" >&2
  exit 2
fi

exit 0
```

---

## 8. 面试视角

- **Q：Agent Permission 为什么重要？**
  → LLM 输出不可预测 + coding agent 能做不可逆操作（写文件、跑命令、push 代码）= 必须有 permission 系统。**生产 Agent 必修**。

- **Q：4 大 Permission 模式？**
  → ① Always confirm（最严，危险操作）② Confirm per session（一般项目）③ Allowlist + confirm unknown（自主工作流）④ Auto mode（高信任场景，classifier model 兜底）。

- **Q：Claude Code 的 4 个 permission mode？**
  → default（每次问）/ accept-edits（自动接受文件编辑，但问 Bash）/ plan（只读不写）/ auto（classifier model 自动判断）。**auto 适合长跑任务，8 次拒绝后自动 abort**。

- **Q：Allowlist vs Denylist vs Ask？**
  → Allowlist = 永远允许（git status / ls）。Denylist = 永远拒绝（rm -rf /）。Ask = 默认询问（git push / deploy）。**实战**：allowlist 越宽越自主，denylist 越严越安全。

- **Q：Hooks 怎么用？**
  → Claude Code 的 hooks 是 bash 脚本，挂在 PreToolUse / PostToolUse / Stop / SubagentStart 等事件上。**exit 0 = 放行，exit 2 = 拒绝**（stderr 作为 reason）。Hook 比 Permission 灵活——可以写复杂逻辑。

- **Q：Auto mode 背后的 classifier model 怎么工作？**
  → 小模型（不是 LLM），输入 action + context，输出 safe / risky / unknown。训练数据是 100k+ 内部标注的 actions。**关键限制**：不允许 scope escalation（不能 push 到任意 remote，不能 rm 非 tracked 文件）。

- **Q：Permission 和 Sandbox 区别？**
  → Permission = prompt-level 限制（agent 收到拒绝信号）。Sandbox = OS-level 限制（agent 物理无法访问）。**两者结合**：sandbox 兜底（即使 agent 失控），permission 优化（少打断）。

- **Q：怎么设计一个 permission system？**
  → 4 层：① Allowlist（永远安全）② Denylist（永远危险）③ Ask（中等风险）④ Hook（复杂判断 + audit log）。**生产加 sandbox**（OS-level 隔离）+ 监控（per-action audit log）。

- **Q：Permission 的 5 大反模式？**
  → ① 完全 auto 无 allowlist ② 完全 deny ③ Denylist 不更新 ④ Hook 静默失败 ⑤ Sandbox 配置错（没真隔离）。

---

## 9. 必读材料

| 材料 | 链接 | 为什么读 |
|------|------|----------|
| **Claude Code: Permissions** | [code.claude.com/docs/en/permissions](https://code.claude.com/docs/en/permissions) | 官方 permission 文档 |
| **Claude Code: Permission modes** | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) | 4 种 mode 详解 |
| **Claude Code: Sandboxing** | [code.claude.com/docs/en/sandboxing](https://code.claude.com/docs/en/sandboxing) | OS-level 隔离 |
| **Claude Code: Hooks** | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks) | Hook 系统（含 Pre/Post/Stop）|
| **OpenAI Codex CLI: Sandboxing** | [github.com/openai/codex](https://github.com/openai/codex) | Codex 的 sandbox 配置 |
| **Anthropic: Building effective agents** | [anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents) | "orchestrator-workers" 模式 |

---

## 10. 练习

1. **配置个人 Claude Code Permission**：写一份 `~/.claude/settings.json`，加 allowlist / denylist / hooks
2. **写一个 hook**：用 bash 脚本拒绝 `rm -rf` 和 `curl * | bash` 模式
3. **跑 auto mode**：用 `claude --permission-mode auto` 跑一个长任务，观察哪些 action 自动通过
4. **配置 sandbox**：用 `/sandbox` 限制 agent 只能访问 `~/projects/`
5. **测试 hook 失败**：故意写一个会失败的 hook，观察 agent 行为
6. **Audit log**：给所有 action 加 log，事后 review 哪些被 classifier 标为 risky

---

## 11. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U18 Agent 安全与红队 | Permission 是防御深度的一部分 |
| U23 Harness Engineering | Permission 是 Harness 组件 9（Permissions）的具体实现 |
| U29 Agent Deploy | Sandbox 隔离是 U29 部署的关键 |
| U33 Human-in-the-loop | Permission 与 HITL 互补（auto mode + HITL 兜底）|
| U37 Subagents | Subagent 可有独立 permissions（如 security-reviewer 只能 Read）|

---

*最后更新：2026-06-30 新增 U39（Permission & Auto Mode）*
*来源：[Claude Code: Permissions](https://code.claude.com/docs/en/permissions) · [Claude Code: Permission modes](https://code.claude.com/docs/en/permission-modes) · [Claude Code: Sandboxing](https://code.claude.com/docs/en/sandboxing) · [Claude Code: Hooks](https://code.claude.com/docs/en/hooks) · [OpenAI Codex CLI](https://github.com/openai/codex)*
