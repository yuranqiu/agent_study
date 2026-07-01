# U21 · Loop Engineering（循环工程）

> **核心问题**：怎么让 Agent 在跨小时、跨天、跨会话、跨人类的循环里持续工作，而不是每次重启都从零开始？
> **预计学时**：5h
> **对应 Phase**：Phase 5 / Phase 3（项目实战阶段必学）
> **状态**：emerging（2026 年中新兴概念，尚未形成统一标准）

---

## 1. 什么是 Loop Engineering

Loop Engineering 是在 **Harness Engineering**（U23）和 **Context Engineering**（U22）之上的最外层抽象。三者形成清晰的分层：

- **Prompt Engineering**（U3）：怎么写好一段 prompt——一次性决策
- **Context Engineering**（U22）：塞什么进 context window——每个 turn 都要做 curation
- **Harness Engineering**（U23）：模型周围的 runtime substrate（11 个组件、H0-H3 阶梯）——一次 session 内
- **Loop Engineering**（U21，本单元）：让 harness **跨会话跨人类持续运转** 的工程——跨 session

> **核心命题**：你不要 prompt Agent，你要 design the loop that prompts your agent。
> —— Peter Steinberger / Boris Cherny（Claude Code 负责人）

单次 ReAct 循环解决"一次对话里的推理和行动"，但真实项目里：
- 目标需要跑几天（multi-day engineering / research / benchmark）
- Agent 会被压缩、被中断、被切换（session restart / model swap / quota）
- 中间需要人介入决策（human gate）
- 多个 agent 可能撞同一个 todo（multi-agent collision）
- 失败要可恢复、可审计、可回放

Loop Engineering 解决的是这些**跨 session 的状态漂移问题**。

---

## 2. 学习目标（Gate 检验）

1. 能说出 Loop Engineering 与 Harness / Context / Goal Engineering 的关系
2. 能列出 Loop 的 6 大基本块（primitives）
3. 能区分 L1 / L2 / L3 三个 loop 成熟度等级
4. 能用一个具体场景（比如"每天自动 triage issue"）画出 loop 流程图
5. 能用 `loop-init` / `loop-audit` / `loop-cost` 跑一次自己的 loop 并打分

---

## 3. 五个基本块 + Memory（cobusgreyling 框架）

| Primitive | 在 loop 里的职责 |
|-----------|------------------|
| **Automations / Scheduling** | 按节奏触发（cron / heartbeat / `/loop`） |
| **Worktrees** | 安全的并行执行（每个 agent 一个隔离分支） |
| **Skills** | 持久化的项目知识（怎么 build / test / deploy） |
| **Plugins & Connectors** | 通过 MCP 触达真实工具（git / DB / CI） |
| **Sub-agents** | Maker / Checker 分离（implements / verifies） |
| **+ Memory / State** | 跨会话的持久脊柱（`STATE.md` / `LOOP.md`） |

> **Anatomy of a Loop**：
> ```
> Schedule → Triage Skill → Read/Write State → Worktree →
>   Implementer Sub-agent → Verifier Sub-agent → MCP/Git →
>     {Human Gate?} → Commit/PR 或 Escalate → 回到 Schedule
> ```

---

## 4. Loop 的成熟度等级（Phased Rollout）

| 等级 | 描述 | 风险 |
|------|------|------|
| **L1** | report-only / propose-only，只观察和汇报 | 最低 |
| **L2** | assisted fix，agent 给 patch，由人合并 | 中 |
| **L3** | unattended 自动合并、自动部署 | 高 |

**必须按 L1 → L2 → L3 渐进**，不要一上来就 L3。L1 至少要跑 1-2 周积累 trust。

---

## 5. 7 个生产级 Loop 模式（cobusgreyling 整理）

| Pattern | 节奏 | Token 成本 | 典型场景 |
|---------|------|------------|----------|
| Daily Triage | 1d–2h | Low | 每天扫一遍 issue 列表，分类标注 |
| PR Babysitter | 5–15m | High | 监控 PR，处理 review comment，rebase |
| CI Sweeper | 5–15m | Very High | 修 CI 失败的小问题 |
| Dependency Sweeper | 6h–1d | Medium | 自动提 PR 更新依赖 |
| Changelog Drafter | 1d | Low | 自动生成 changelog 草稿 |
| Post-Merge Cleanup | 1d–6h | Low | 合并后清理 branch / label |
| Issue Triage | 2h–1d | Low | 自动给 issue 打 label / assign |

每个 pattern 都有对应的 starter kit（clone-and-run）和 5 分钟 quickstart。

---

## 6. Loop 的失败模式（必须知道）

| 失败模式 | Loop 的应对 |
|----------|-------------|
| Partial completion | bounded tasks + 显式 result 文件 + 续跑路由 |
| Goal drift | confirmed scope + architecture contract + compact context pack |
| Self-check bias | implementer / verifier 分离 + 证据写回 |
| Token waste | 持久化 context summary + 增量检索 |
| Handoff gap | 显式 handoff packet（goal / status / evidence / next） |
| Loop 失控 | quota 机制（`should-run` 决策） + 心跳退出条件 |

---

## 7. 主流 Loop Engineering 工具 / 项目

### 7.1 LoopX（控制面层）

**Repo**: [huangruiteng/loopx](https://github.com/huangruiteng/loopx)
**定位**: 跨 Agent 运行时的本地控制面（Codex / Claude Code / Cursor / shell 都能接）
**核心抽象**: 目标（goal）+ 闸门（gate）+ 待办（todo claim）+ 证据（evidence）+ 配额（quota）+ 交接（handoff）

```bash
curl -fsSL https://raw.githubusercontent.com/huangruiteng/loopx/main/scripts/install-from-github.sh | bash
export PATH="$HOME/.local/bin:$PATH"
loopx doctor
loopx status
loopx quota should-run
loopx todo claim --goal-id my-project
```

**心智模型**：LoopX 不是 executor，是**控制面**。你已有的 Codex / Claude Code / Cursor 跑实际的 loop turn，LoopX 负责保持目标、边界、证据、交接在 turn 之间不丢。

### 7.2 Loom（交付层）

**Repo**: [valkor-ai/loom](https://github.com/valkor-ai/loom)
**定位**: 编程 Agent 的开源自适应交付层
**核心抽象**: Dynamic workflow → planning → bounded task → evidence → review → repair → handoff

```bash
curl -fsSL https://github.com/valkor-ai/loom/releases/latest/download/install.sh | bash -s -- --agent claude-code
# 在 Claude Code 里：
/loom build a visitor registration system
/loom continue
/loom review
```

**心智模型**：Loom 是"在 Agent 之上"的一层，把每次"我想做一个 feature"都包成一个**有状态可恢复的交付循环**。

### 7.3 paper_agent（垂直案例：论文精读）

**Repo**: [whut09/paper_agent](https://github.com/whut09/paper_agent)
**定位**: 论文精读的 Agent Harness + Loop Engineering
**核心抽象**: Paper → Parse → Extract → Synthesize → Verify → Report → User Feedback → Correction Memory → Prompt Patch

> 该项目明确把架构分三层：
> - **Agent Harness**：DAG executor + node 协议（`requires` / `produces` / `NodeResult`）
> - **Harness Engineering**：Grounding Map + Asset Manifest + Verifier Agent
> - **Loop Engineering**：correction memory + 自动 prompt patch

### 7.4 oh-my-hermes（多 agent 编排）

**Repo**: [rlaope/oh-my-hermes](https://github.com/rlaope/oh-my-hermes)
**定位**: Hermes Agent 的 workflow 插件集
**核心抽象**: `loop` / `ultragoal` / `ralplan` / `deep-interview` 等 skill，把 loop 拆成可视阶段

### 7.5 cobusgreyling/loop-engineering（pattern 库 + 工具）

**Repo**: [cobusgreyling/loop-engineering](https://github.com/cobusgreyling/loop-engineering)
**Stars**: ~4.3k（截至 2026-06）
**定位**: Loop Engineering 的"标准库"——patterns + starters + CLI 工具

```bash
npx @cobusgreyling/loop-init . --pattern daily-triage --tool grok
npx @cobusgreyling/loop-audit . --badge
npx @cobusgreyling/loop-cost --pattern ci-sweeper --cadence 15m
```

**心智模型**：cobusgreyling 是 Loop Engineering 这个概念的**布道者和 pattern 集大成者**，必读。

### 7.6 Nexent（无代码平台）

**Repo**: [ModelEngine-Group/nexent](https://github.com/ModelEngine-Group/nexent)
**Stars**: ~5.4k
**定位**: 零代码生成生产级 AI Agent 的平台，**"Harness Engineering"** 概念的代表
**用法**: 可视化配置，导出可部署的 Agent

---

## 8. Loop Engineering 必读材料

| 材料 | 为什么读 |
|------|----------|
| [Cobus Greyling: Loop Engineering（Substack）](https://cobusgreyling.substack.com/p/loop-engineering) | 概念入门，原作者一手叙述 |
| [Addy Osmani: Loop Engineering](https://addyosmani.com/blog/loop-engineering/) | 大佬背书 + 工程视角的解读 |
| [LoopX 文档：Loop Engineering principles and pitfalls](https://github.com/huangruiteng/loopx/blob/main/docs/product/loop-engineering-principles-and-pitfalls.md) | 控制面视角的 7 条原则 + 8 个坑 |
| [Loom Technical Report](https://zonodqioyxil6r3k.public.blob.vercel-storage.com/Loomline-v0.pdf) | 交付层的白皮书 |
| [cobusgreyling/loop-engineering docs](https://github.com/cobusgreyling/loop-engineering) | 7 个生产 pattern + 工具链 |
| [paper_agent README](https://github.com/whut09/paper_agent) | 垂直案例，看 Loop Engineering 怎么落地 |

---

## 9. 面试视角

Loop Engineering 是 2026 年中非常新的方向，面试可能问：

- **Q：Loop Engineering 和 Harness Engineering 的区别？**
  → Harness（U23）是一次 session 内的 runtime substrate（11 个组件、H0-H3 阶梯）；Loop（U21）是跨 session 的状态持久 + 决策交接（goal / gate / evidence / handoff / quota）。Loop 建立在 Harness 之上——Harness 决定"一次 turn 怎么跑"，Loop 决定"多个 turn / 多个 session 怎么续上"。

- **Q：怎么设计一个每天自动 triage issue 的 Agent loop？**
  → 答案参考 §5 的 Daily Triage pattern：schedule → read STATE → fetch issues → classify（sub-agent）→ write labels → write status → next tick。

- **Q：Loop 失控怎么办？**
  → quota `should-run` 决策：等用户、跑安全侧路、自修复、静默退出。**写 spend 只在 validated writeback 之后**，不在 quiet skip。

- **Q：Loop Engineering 有哪些已知坑？**
  → Token 爆炸、verifier bias、comprehension debt、auto-merge 风险、跨 loop 冲突。详见 §6 失败模式表。

---

## 10. 练习

1. **跑一个 starter**：用 `npx @cobusgreyling/loop-init` 初始化一个 `daily-triage` loop，paste 到自己的 repo 试试
2. **画 anatomy**：手画一个 loop 的 mermaid 图（参考 §3 的图）
3. **加 verifier 分离**：把现有的 ReAct Agent 拆成 implementer + verifier 两个 sub-agent，看准确率变化
4. **写 STATE**：为一个多日项目写 `STATE.md` + `LOOP.md`，模拟"agent 重启后能恢复"的场景
5. **自我审计**：用 `npx @cobusgreyling/loop-audit . --suggest` 看自己 loop 的 Loop Ready 分数

---

## 11. 与本仓库其他单元的关系

| U7 Planning & Reasoning | Loop 里要分阶段 / 决策树 |
| U8 Memory & Reflection | Loop Engineering 的 Memory 是跨 session 持久化 |
| U9 Multi-Agent | Sub-agent 协作是 loop 的核心 |
| U12 Agent 评估标准 | loop-level eval（不是单次 eval） |
| U20 编程 Agent 内部原理 | 编程 Agent 是 loop engineering 最活跃的落地场景 |
| U22 Context Engineering | Context（塞什么进 window）是 loop 持续运转的载体 |
| U23 Harness Engineering | Harness（11 组件 / H0-H3）是 loop 单 turn 的执行 runtime |

*最后更新：2026-06-30 新增 U21（Loop Engineering）*
*来源：GitHub Trending 2026-06、cobusgreyling 博客、LoopX / Loom / paper_agent / oh-my-hermes 官方文档*

---

## 12. 完整 Loop Engineering 实战（基于 LoopX / oh-my-hermes）

### 12.1 Lifetime Goal 模式

```python
# 完整实现：lifetime goal 跨 session 持续
class LifetimeGoalLoop:
    def __init__(self, llm, db):
        self.llm = llm
        self.db = db  # Postgres 存 state
        self.state = LifetimeState()

    def bootstrap(self, goal_id, objective, goal_doc):
        """初始化 lifetime goal（只跑一次）"""
        self.state.goal_id = goal_id
        self.state.objective = objective
        self.state.created_at = datetime.now()
        # 拆成 milestones
        self.state.milestones = self.llm.invoke(
            f"把目标 '{objective}' 拆成 3-5 个可验证的 milestone：\n"
            f"每个 milestone 1 句话 + acceptance criteria"
        )
        self.save()

    def tick(self):
        """每个 tick 跑一次：检查进度 + 决定下一步"""
        # 1. 加载 state
        self.load()
        if self.state.is_done():
            return "GOAL_COMPLETE"

        # 2. 选下一个 milestone
        next_milestone = self.state.next_milestone()
        if not next_milestone:
            return "ALL_MILESTONES_DONE"

        # 3. 跑 milestone
        result = self.run_milestone(next_milestone)

        # 4. 更新 state
        self.state.update_milestone(next_milestone, result)
        self.save()

        # 5. 决定下一步
        if result.failed:
            return self.handle_failure(next_milestone, result)
        elif result.success:
            return "MILESTONE_DONE"
        else:
            return "RETRY"

    def run_milestone(self, milestone):
        # 1. Triage（用 LLM 决定怎么做）
        plan = self.llm.invoke(
            f"为 milestone '{milestone.desc}' 制定 3-5 步执行计划"
        )
        # 2. Executor（用 subagent 跑）
        result = executor_subagent.run(plan)
        # 3. Verification
        if milestone.verify(result):
            return Result(success=True, output=result)
        else:
            return Result(success=False, output=result)
```

### 12.2 Goal Doc 实战模板

```markdown
# GOAL.md

## Objective
Implement a real-time chat system that supports 10k concurrent users, end-to-end encrypted messages, and offline message sync.

## Acceptance Criteria
1. WebSocket server can handle 10k concurrent connections
2. Messages are encrypted with X3DH + Double Ratchet
3. Offline messages sync within 5 seconds when reconnected
4. p99 latency < 100ms

## Out of Scope
- Voice/video calls
- Message reactions
- Read receipts

## Milestones
- [ ] M1: WebSocket server with 10k connection benchmark
- [ ] M2: E2E encryption protocol implementation
- [ ] M3: Offline message storage and sync
- [ ] M4: Load test suite + CI integration
- [ ] M5: Production deployment with monitoring
```

**实战**：GOAL.md 是 Loop 的**源代码**。LLM 看到它就知道整个目标。

---

## 13. 7 个生产 Pattern 深度（参考 cobusgreyling）

### 13.1 Daily Triage

**问题**：每天 100+ GitHub issues，怎么决定先做哪个？

```python
class DailyTriageLoop:
    def schedule(self):
        # 每天早上 9 点跑
        return cron("0 9 * * *", self.tick)

    def tick(self):
        # 1. 拉新 issues
        new_issues = github.fetch_issues(state="open", since=last_run)

        # 2. Triage（用 LLM 分类）
        for issue in new_issues:
            triage = llm.invoke(f"分析 issue，决定：\n{issue.body}\n"
                f"返回 JSON: { priority, labels, estimated_effort, assign_to }")
            # 3. 应用分类
            github.update_issue(issue.number, **triage)

        # 4. 通知 owner
        slack.send("#dev", f"今日新 issue: {len(new_issues)}, 已自动分类")
```

### 13.2 PR Babysitter

```python
class PRBabysitterLoop:
    def schedule(self):
        return cron("*/15 * * * *", self.tick)  # 每 15 分钟

    def tick(self):
        # 找 review 超过 24h 的 PR
        stale_prs = github.search_prs(review_requested_age_hours__gt=24)

        for pr in stale_prs:
            # 用 LLM 总结 PR 状态
            summary = llm.invoke(f"总结 PR #{pr.number}：当前状态 + 阻碍 review 的原因")
            # 通知 reviewer
            slack.dm(pr.reviewers, f"PR 等你 review：{summary}")
```

### 13.3 CI Sweeper

```python
class CISweeperLoop:
    def tick(self):
        # 找 CI 失败的 PR
        failed = github.search_prs(checks="failure")

        for pr in failed:
            # 用 subagent 尝试自动修复
            fixed = subagent.run(f"修复 PR #{pr.number} 的 CI 失败，失败原因：{pr.failure_log}")
            if fixed.success:
                github.push_fix(pr, fixed)
            else:
                slack.notify_owner(pr, "CI 失败，需要人工")
```

**实战**：CI Sweeper 节省 50% CI 失败的人工排查时间。

---

## 14. Loop Engineering 失败模式深度

### 14.1 5 大失败模式细节

| 失败模式 | 早期信号 | 修复 |
|---------|---------|------|
| **Partial completion** | state 有 step 但没标 done | 强制 idempotent + done flag 必填 |
| **Goal drift** | milestone 越加越多 | 每周重新 review GOAL.md，砍掉不相关的 |
| **Self-check bias** | verifier 老说"pass" | 多 verifier + adversarial review |
| **Token waste** | context 重复 load 同一文件 | 强制 file path + cached state |
| **Handoff gap** | 换 session 后不知道状态 | 强制写 state.md + summary |

### 14.2 反模式：Loop 失控

**症状**：
- 跑了一晚上烧了 $200 token 没产出
- 同一任务反复跑 100 次
- LLM 越跑越偏

**预防**：
```python
# 1. 设 quota
MAX_TOKENS_PER_GOAL = 50_000
if state.total_tokens > MAX_TOKENS_PER_GOAL:
    state.pause("budget exceeded")
    notify_human("Loop paused, please review")

# 2. 检测 stuck
if state.consecutive_no_progress > 5:
    state.pause("stuck detected")
    notify_human("Agent seems stuck")

# 3. 检测 token waste
last_10_actions = state.history[-10:]
if all(a.is_duplicate_of_earlier() for a in last_10_actions):
    state.pause("duplicate actions detected")
```

**核心**：**Loop 应该是可中断的**（L1 → L2 → L3 渐进暴露）。
