# U40 · Verification / Adversarial Review（验证与对抗式审查）

> **核心问题**：怎么让 agent **自己验证自己的输出**？怎么用 adversarial review 模式抓出 LLM 的 self-bias 错误？怎么让"我以为做完了"变成"真的做完了"？
> **预计学时**：3h
> **对应 Phase**：Phase 3（项目实战）/ Phase 4
> **状态**：2025-2026 工业级标配（Claude Code / OpenAI / Cursor 都强调）

---

## 1. 为什么 Verification 至关重要

**核心问题**：LLM agent 经常**自己觉得做完了但实际没做完**。

**症状**：
- "I've completed the task!" → 实际只跑了一半
- 测试通过 → 但没覆盖 edge case
- 代码看起来对 → 但有 race condition
- 写了 100 行 → 但漏了关键的 error handling

**根因**：
- LLM 没有"自我验证"的内在机制
- LLM 有 **self-bias**（对自己的输出过度自信）
- LLM 不知道"不知道什么"

**Claude Code 官方论断**：

> "Without a check it can run, 'looks done' is the only signal available, and **you become the verification loop**: every mistake waits for you to notice it."

---

## 2. 4 大 Verification 模式（按强度）

### 2.1 模式 1：让 Agent 自己跑 check

**最简单**："做完 task 后跑 X 验证"。

```bash
# 单 prompt 模式
"Implement the validateEmail function. 
 Run the tests after implementing."

# Claude 自己跑 pytest 看结果
```

**vs**：

```bash
# 没有 verification（差）
"Implement the validateEmail function"
# → Claude 说"done" → 实际有 bug
```

### 2.2 模式 2：跨 session 用 /goal condition

```bash
# 设置 /goal 作为持续 condition
/goal "All tests pass and no lint errors"

# 每次 turn 后，evaluator re-check
# Claude 一直工作直到 condition 满足
```

**适合**：长跑 autonomous 任务（U21 Loop Engineering）。

### 2.3 模式 3：Deterministic Gate（Stop hook）

```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "command",
        "command": "npm test"
      }]
    }]
  }
}
```

**效果**：
- Claude 想结束 turn
- Stop hook 跑 `npm test`
- 测试失败 → Claude 必须继续（不能结束 turn）
- 8 次失败后 Claude 才能 override 结束

**适合**：所有项目都应该用——硬性保证不"半成品"。

### 2.4 模式 4：Adversarial Review（second opinion）

```bash
# Writer/Reviewer pattern
"Implement the rate limiter"

"Use a subagent to review the rate limiter diff against PLAN.md.
 Check that every requirement is implemented,
 the listed edge cases have tests,
 and nothing outside the task's scope changed.
 Report gaps, not style preferences."
```

**杀手特性**：
- Reviewer 是 **fresh context**——不被 writer 的"自我感觉良好"影响
- Reviewer 只看 diff + spec，不看 reasoning process
- Reviewer 倾向 find gaps（这是你让它做的）

**适合**：大功能 / 长 autonomous 任务。

---

## 3. 5 大 Verification 工具

### 3.1 Test Suite（最常见）

```python
# pytest / vitest / go test
def test_validate_email():
    assert validate_email("user@example.com") is True
    assert validate_email("invalid") is False
    assert validate_email("user@.com") is False
    assert validate_email("") is False
```

**生产必加**：每个 task 完成都跑 test suite。

### 3.2 Lint / Type Check

```bash
# Python
ruff check .
mypy src/

# TypeScript
eslint --fix
tsc --noEmit
```

**PostToolUse hook** 自动跑：
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "npx eslint --fix"
      }]
    }]
  }
}
```

### 3.3 Visual Diff

```bash
# Claude Code 在 Chrome 中可视化对比
"Implement this design. take a screenshot of the result and compare it to the original. list differences and fix them"
```

**适合**：前端 / UI work。

### 3.4 Build Verification

```bash
# 编译 / 打包
npm run build
go build
cargo build
```

**意义**：编译通过 = 代码至少在语法和类型层面是正确的。

### 3.5 Custom Domain Checks

```python
# 金融 agent：检查 transaction log
verify_no_double_spend(transactions)
verify_audit_trail(transactions)

# 医疗 agent：检查 HIPAA compliance
check_phi_redaction(response)
```

---

## 4. Adversarial Review 深度

### 4.1 Writer/Reviewer 模式

```python
class WriterReviewerLoop:
    def __init__(self, max_iterations=3):
        self.writer = writer_subagent
        self.reviewer = reviewer_subagent
        self.max_iterations = max_iterations
    
    def run(self, task):
        for i in range(self.max_iterations):
            # Writer 工作
            code = self.writer.run(task)
            
            # Reviewer 审查（fresh context）
            review = self.reviewer.run(
                f"Review this code for correctness, security, performance:\n{code}\n"
                f"Original task:\n{task}"
            )
            
            if review.approved:
                return code
            
            # Writer 修正
            task = f"{task}\n\nReviewer found these issues:\n{review.feedback}\nPlease fix."
        
        return code  # max iterations reached
```

### 4.2 Verification Subagent

```markdown
# .claude/agents/code-reviewer.md
---
name: code-reviewer
description: Reviews code for correctness, edge cases, and consistency with the plan
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior code reviewer. Review the diff for:

1. **Correctness**: Does it actually implement the requirements?
2. **Edge cases**: Empty inputs, null values, boundary conditions, error states
3. **Race conditions**: Concurrency, shared state, atomicity
4. **Security**: Input validation, auth checks, injection
5. **Tests**: Are edge cases covered? Do tests actually test the behavior?
6. **Scope**: Did the implementation go beyond what was asked?

Report gaps, not style preferences. For each finding:
- File:line reference
- Description
- Suggested fix

If you find no issues, say so explicitly.
```

**关键**："Report gaps, not style preferences" — 防止 reviewer 跑偏到 nitpicking。

### 4.3 Reviewer 反模式：Over-Engineering

> "A reviewer prompted to find gaps will usually report some, even when the work is sound, because that is what it was asked to do. **Chasing every finding leads to over-engineering**."

**反模式**：reviewer 提了 20 个"问题"，writer 一一修复，引入不必要的抽象。

**修复**：
- Reviewer prompt 加 "Flag only gaps that affect correctness or stated requirements. Treat the rest as optional."
- 区分 critical / warning / nit
- 接受 nit 不修（不追求完美）

---

## 5. 实战：完整 Verification Pipeline

```python
class VerificationPipeline:
    def __init__(self, agent):
        self.agent = agent
        self.checks = [
            self.check_tests_pass,
            self.check_lint_passes,
            self.check_types_pass,
            self.check_security_audit,
            self.check_diff_against_plan,
        ]
    
    def run_with_verification(self, task, plan=None):
        # 1. Agent 实施
        result = self.agent.run(task)
        code = result.code
        diff = result.diff
        
        # 2. 跑所有 checks
        for check in self.checks:
            verdict = check(code, diff, plan)
            if not verdict.passed:
                # 让 agent 修复
                result = self.agent.run(
                    f"Fix these issues:\n{verdict.feedback}"
                )
                code = result.code
                diff = result.diff
        
        # 3. Adversarial review（可选）
        if needs_deep_review(task):
            review = reviewer_subagent.run(
                f"Review this diff against the plan:\n{plan}\nDiff:\n{diff}"
            )
            if not review.approved:
                result = self.agent.run(f"Address these gaps:\n{review.gaps}")
        
        return result
```

---

## 6. Self-Evidence vs Self-Assertion

**核心原则**（Claude Code 团队）：

> "Have Claude show **evidence** rather than asserting success: the test output, the command it ran and what it returned, or a screenshot of the result. Reviewing evidence is faster than re-running the verification yourself, and it works for sessions you weren't watching."

**对比**：

```python
# ❌ Self-assertion（差）
"I've implemented the feature."
"Tests pass."
"It's done."

# ✓ Self-evidence（好）
"Implementation complete. Evidence:
 - `npm test`: 23 passed, 0 failed
 - `npm run build`: built successfully in 4.2s
 - `git diff main..HEAD --stat`: 4 files changed, 87 insertions
 - Lint: 0 errors"
```

**实战 prompt**：
```bash
"Implement X. After implementing, run the verification commands and 
 include the actual test output / build log / lint output in your 
 final response. Don't just say 'done'."
```

---

## 7. Adversarial Review 实战

### 7.1 完整 Review Template

```markdown
You are reviewing a code change. Be specific and cite evidence.

## Review Plan
The change is supposed to: {plan}

## Diff
```diff
{diff}
```

## Check
1. **Completeness**: Does every item in the plan have a corresponding change?
2. **Edge cases**: Empty input, null, boundary, large input, concurrent
3. **Tests**: Do tests cover the new behavior? Do they actually test it?
4. **Scope**: Did the change include anything outside the plan?
5. **Correctness**: Any obvious bugs?
6. **Naming**: Are new symbols named clearly?

## Output Format
For each finding:
- **Severity**: Critical / Warning / Nit
- **File:line**
- **Issue**: what's wrong
- **Fix**: suggested code

End with one of:
- APPROVED: ship it
- NEEDS FIX: list critical/warning items
```

### 7.2 完整 Prompt

```bash
"Use a subagent to review this PR diff against PLAN.md.

The review should:
1. Check that every requirement is implemented
2. Verify the listed edge cases have tests
3. Confirm nothing outside the task's scope changed
4. Flag only gaps that affect correctness or stated requirements
5. Treat style preferences as optional (don't include them as findings)

Report findings as Critical / Warning / Nit with file:line references.
End with APPROVED or NEEDS FIX."
```

---

## 8. 5 大 Verification 反模式

| 反模式 | 后果 | 修复 |
|--------|------|------|
| **No verification** | agent "完成"但实际有问题 | 强制 + Stop hook 跑 test |
| **Self-assertion only** | "我完成了"但没跑 check | 要求 "show test output" |
| **Reviewer 找 nits** | over-engineering | 加 "flag only correctness" |
| **同一 context review** | Reviewer 被 writer 影响 | 用 subagent / fresh context |
| **Verify but don't fix** | 检查发现问题但不修 | Verification = 修直到通过 |

---

## 9. Verification 决策树

```
任务完成 → 是否能跑自动 check？
├── 能（test / build / lint）
│   ├── 在同一 prompt 跑（最快）
│   ├── /goal 持续跑（长任务）
│   └── Stop hook 强制跑（硬保证）
└── 不能（UI / 业务逻辑 / 主观）
    ├── Visual diff（前端）
    ├── Adversarial review（复杂逻辑）
    └── Human review（关键业务）

Reviewer → 是否需要 fresh context？
├── 需要（避免 self-bias）
│   ├── Subagent（U37）
│   ├── 不同 session
│   └── 强制 /clear 后再 review
└── 不需要（trivial 改动）
```

---

## 10. 实战：完整代码 review 流程

```python
async def code_review_pipeline(task, plan, max_iterations=3):
    # Phase 1: Implementation
    code = await implementer.run(task)
    
    # Phase 2: Deterministic gates
    for gate in [run_tests, run_lint, run_type_check, run_build]:
        verdict = await gate(code)
        if not verdict.passed:
            code = await implementer.run(f"Fix:\n{verdict.feedback}")
    
    # Phase 3: Adversarial review
    for i in range(max_iterations):
        review = await reviewer_subagent.run(
            f"Review this code:\n{code}\n\nPlan:\n{plan}\n\n"
            f"Flag only correctness gaps, not style."
        )
        if review.approved:
            return code
        
        # Filter for critical/warning only
        critical_issues = [f for f in review.findings if f.severity in ("critical", "warning")]
        if not critical_issues:
            return code  # only nits left
        
        code = await implementer.run(f"Fix these issues:\n{critical_issues}")
    
    return code
```

---

## 11. 面试视角

- **Q：为什么 LLM agent 需要专门的 verification 机制？**
  → LLM 有 **self-bias**（对自己输出过度自信）——"I've completed" 经常实际只完成 70%。没有 verification check，agent 会"自欺欺人"。**必须 externalize verification**（test / build / lint / review subagent）。

- **Q：4 大 verification 模式？**
  → ① 让 agent 跑 check（同一 prompt 跑 test）② /goal 持续 condition（长任务）③ Stop hook 强制（硬保证）④ Adversarial review（subagent 审查，fresh context）。**4 种模式可叠加**。

- **Q：Writer/Reviewer 模式为什么有效？**
  → Reviewer 是 **fresh context**——不被 writer 的"自我感觉良好"影响。**Self-bias 是 LLM 最大的失败模式之一**，subagent 隔离是有效对策。

- **Q：怎么防止 reviewer 跑偏到 over-engineering？**
  → Prompt 加 "Flag only gaps that affect correctness or stated requirements. Treat style preferences as optional"。**区分 critical / warning / nit**——只修 critical 和 warning，接受 nit 不修。

- **Q：Self-assertion vs Self-evidence？**
  → Self-assertion = "我完成了"（差）。Self-evidence = "我完成了，证据：tests 23 passed, build 4.2s, lint 0 errors"（好）。**实战 prompt**："include actual test output / build log / lint output in your final response"。

- **Q：Stop hook 怎么用？**
  → Claude Code Stop hook 在 Claude 想结束 turn 时跑你的命令。**exit 0 = 放行，exit 2 = 拒绝**。**8 次拒绝后** Claude 可以 override 结束。配置 `npm test` 当 Stop hook = 硬保证不"半成品"。

- **Q：Adversarial review 怎么设计？**
  → 3 步：① 找 fresh context 的 reviewer（subagent）② 明确 review 范围（"check that every requirement is implemented"）③ 输出格式（APPROVED / NEEDS FIX + findings with file:line）。

- **Q：Verification 5 大反模式？**
  → ① No verification ② Self-assertion only（"我完成了"无证据）③ Reviewer 找 nits（over-engineering）④ 同一 context review（self-bias）⑤ Verify but don't fix（发现问题不修）。

- **Q：什么时候用 stop hook vs /goal vs subagent review？**
  → **Stop hook** = 硬保证（所有项目都该用）/**/goal** = 长跑 autonomous 任务（设 condition，evaluator re-check）/**Subagent review** = 大功能 / 关键代码（writer/reviewer 隔离，fresh context）。

---

## 12. 必读材料

| 材料 | 链接 | 为什么读 |
|------|------|----------|
| **Claude Code: Best practices** | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices) | "Give Claude a way to verify" + "Adversarial review" |
| **Claude Code: Hooks** | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks) | Stop hook 实战 |
| **Claude Code: Subagents** | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents) | 写 reviewer subagent |
| **Anthropic: Building effective agents** | [anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents) | "Evaluator-optimizer" 模式 |
| **Karpathy on LLM self-bias** | [karpathy.ai](https://karpathy.ai/) | LLM 自我评估的局限性 |

---

## 13. 练习

1. **Stop hook 实战**：在 Claude Code 配 Stop hook 跑 `npm test`，故意让测试失败，看 agent 怎么修
2. **Writer/Reviewer 循环**：用 subagent 实现 writer + reviewer，跑 3 个 iteration 看收敛
3. **Self-evidence prompt**：让 agent 跑 task 后必须贴 test output / build log，不能只说"done"
4. **Visual diff**：用 Claude Code Chrome 模式做 UI 任务，截图对比
5. **Audit log**：把所有 verification 结果写 log，事后分析哪些 check 经常 fail
6. **Multi-stage review**：实现 security + performance + tests 三个 subagent，并行 review

---

## 14. 与本仓库其他单元的关系

| 关联单元 | 关系 |
|----------|------|
| U12 Agent 评估标准 | Verification 是 Agent eval 的一部分 |
| U23 Harness Engineering | Verification 是 Harness 组件 8（Verification）的具体实现 |
| U33 Human-in-the-loop | Adversarial review 失败时升级到 HitL |
| U35 AgentOps | Verification metrics 是 U35 的关键监控点 |
| U37 Subagents | Subagent 是 Adversarial review 的核心模式 |
| U38 Context Management | Verification 让 agent 在 fixed context 里工作 |
| U39 Permission | Stop hook 是一种 permission gate |

---

*最后更新：2026-06-30 新增 U40（Verification / Adversarial Review）*
*来源：[Claude Code: Best practices](https://code.claude.com/docs/en/best-practices) · [Claude Code: Hooks](https://code.claude.com/docs/en/hooks) · [Claude Code: Subagents](https://code.claude.com/docs/en/sub-agents) · [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)*
