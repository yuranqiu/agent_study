# U41 · Agent 全栈项目学习路线（Project-Driven Learning Path）

> **核心问题**：学完所有概念单元后，怎么"按项目学"——把分散的概念整合成可展示的端到端项目？哪些项目既能深度整合多个概念，又能在简历 / 面试中展示？
> **预计学时**：60-100h
> **对应 Phase**：Phase 3（项目实战）
> **状态**：跨单元整合（参考 [datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents) 16 章 + [all-in-rag](https://github.com/datawhalechina/all-in-rag) 10 章结构）

---

## 1. 为什么需要"项目驱动"学习

**问题**：41 个单元覆盖了所有核心概念，但**分散的概念不等于能用的能力**。

**解决**：通过**完整项目**把多个单元串起来，形成**端到端作品**。

**项目驱动的 3 大价值**：

1. **能力整合**：把 U5 ReAct + U6 Tool + U8 Memory + U22 Context + U33 HITL 整合成 1 个项目
2. **简历可展示**：面试时 demo 比讲理论有效 10 倍
3. **概念 → 工程**：从"知道"到"做出"——发现真实问题（速度 / 成本 / 可靠性）

---

## 2. 6 个项目推荐（按难度递增）

### 项目 1：ReAct 智能体原型（入门）

**对应单元**：U1 / U3 / U5 / U6 / U-DL

**目标**：用 150 行 Python 从零实现一个 ReAct agent

**功能**：
- 至少 3 个 tool（天气 / 搜索 / 计算器）
- Tool Calling 通过 OpenAI Functions/Tools
- ReAct 循环可配置 max_iterations（10 步后停）
- 输出 trace log（每步 Thought/Action/Observation）
- 流式输出

**时间**：20-30h

**参考**：[datawhale/hello-agents 第四章 智能体经典范式构建](https://github.com/datawhalechina/hello-agents)

**评估标准**：能调通 3 个 tool 完整调用链 + 跑 10 个测试 query 100% 成功

---

### 项目 2：RAG 知识库助手（核心）

**对应单元**：U2 / U4 / U8 / U15 / U16 / U19 / U22 / U31

**目标**：构建一个生产级 RAG 系统，支持 PDF / Markdown / Notion 多源

**功能**：
- Document loading（多格式）+ 清洗
- Chunking 策略对比（固定 / 重叠 / 语义 / 层次化）
- 多模态 embedding（OpenAI + bge-m3）
- 向量库（Chroma / Milvus / Qdrant 任选）
- Hybrid 检索（BM25 + 向量）+ Reranker
- RAGAS 4 大指标评估
- 流式输出 + source citation
- Web UI（Gradio / Streamlit）

**时间**：30-40h

**参考**：[datawhale/all-in-rag 第八章 项目一](https://github.com/datawhalechina/all-in-rag) + [datawhale/hello-agents 第八章 记忆与检索](https://github.com/datawhalechina/hello-agents)

**评估标准**：
- RAGAS faithfulness > 0.85
- answer relevancy > 0.80
- 端到端 demo 可演示

---

### 项目 3：Multi-Agent 协作系统（进阶）

**对应单元**：U9 / U11 / U13 / U22 / U31 / U36

**目标**：用 LangGraph / CrewAI / AutoGen 实现 multi-agent 协作

**功能**：
- 3-5 个 agent（Planner / Researcher / Writer / Reviewer / Executor）
- A2A 或 MCP 协议通信
- Shared memory（短期 + 长期）
- 工具调用 + 结构化输出
- Human-in-the-loop（U33：让人类审批关键决策）
- Streaming output

**时间**：40-50h

**参考**：[datawhale/hello-agents 第六章 框架开发实践](https://github.com/datawhalechina/hello-agents) + [jjyaoao/HelloAgents](https://github.com/jjyaoao/helloagents)

**评估标准**：
- 3 个 agent 协作完成"研究 → 写作 → 审校"任务
- 演示 trace（看 agent 之间的 handoff）

---

### 项目 4：Computer Use / GUI Agent（前沿）

**对应单元**：U17 / U20 / U25 / U29 / U34

**目标**：实现一个能操作浏览器 / 桌面应用的 agent

**功能**：
- 截图 + 视觉理解（GPT-4V / Claude Vision）
- Action grounding（输出 click 坐标）
- 多步任务执行（"打开 Chrome，搜索 xxx"）
- Safety guardrails（U34：危险操作前 confirm）
- Sandbox 执行（Docker）
- Trace + replay

**时间**：40-50h

**参考**：[browser-use](https://github.com/browser-use/browser-use) + [UI-TARS](https://github.com/bytedance/UI-TARS) + [Anthropic Computer Use 文档](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use)

**评估标准**：能完成 5+ 个真实浏览器任务（如订机票、查邮件）

---

### 项目 5：Reasoning + Tool Use Agent（推理核心）

**对应单元**：U24 / U6 / U7 / U9 / U28

**目标**：实现一个能用 Reasoning Model 调工具的 agent

**功能**：
- DeepSeek R1 / o1 / Claude 3.7 extended thinking 集成
- Reasoning trace 可视化（让 reasoning_effort 可见）
- Tool use in reasoning（reasoning 中途调 tool）
- Cascade 路由（简单问题 mini 模型，难问题 o1）
- Self-consistency（采样 N 次，多数一致）
- 成本 + 延迟监控（U28）

**时间**：30-40h

**参考**：[DeepSeek R1 论文](https://arxiv.org/abs/2501.12948) + [Anthropic Extended Thinking](https://www.anthropic.com/news/claude-3-7-sonnet)

**评估标准**：
- AIME 题 30%+ 准确率
- 成本 < GPT-4o baseline

---

### 项目 6：Long-Running Agent + Loop Engineering（深度）

**对应单元**：U21 / U22 / U23 / U27 / U29 / U33 / U37 / U40

**目标**：实现一个能跨 session 持续工作的 long-running agent

**功能**：
- LoopState 持久化（Postgres + Redis）
- 5 大基本块（Schedule / Triage / State / Worktree / Implementer）
- L1 → L3 渐进暴露
- Async HITL（Redis queue + Slack 通知）
- Subagent 委派（context 隔离）
- Adversarial review（writer / reviewer 循环）
- Verification subagent
- DPO 训练集自动生成
- Auto mode + permission

**时间**：60-80h

**参考**：[LoopX](https://github.com/huangruiteng/loopx) + [cobusgreyling/loop-engineering](https://github.com/cobusgreyling/loop-engineering) + [Anthropic Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

**评估标准**：
- 跨 session 恢复成功率 > 90%
- 演示一个 24 小时的 agent loop

---

## 3. 6 个项目的单元覆盖矩阵

| 项目 | 覆盖的单元 | 难度 | 时间 | 简历价值 |
|------|----------|------|------|----------|
| **P1 ReAct 原型** | U1 / U3 / U5 / U6 / U-DL | ⭐ | 20-30h | ★★★ |
| **P2 RAG 知识库** | U2 / U4 / U8 / U15 / U16 / U19 / U22 / U31 | ⭐⭐ | 30-40h | ★★★★ |
| **P3 Multi-Agent 协作** | U9 / U11 / U13 / U22 / U31 / U36 | ⭐⭐⭐ | 40-50h | ★★★★★ |
| **P4 Computer Use / GUI Agent** | U17 / U20 / U25 / U29 / U34 | ⭐⭐⭐ | 40-50h | ★★★★★ |
| **P5 Reasoning + Tool Use** | U24 / U6 / U7 / U9 / U28 | ⭐⭐⭐ | 30-40h | ★★★★★ |
| **P6 Long-Running + Loop Engineering** | U21 / U22 / U23 / U27 / U29 / U33 / U37 / U40 | ⭐⭐⭐⭐ | 60-80h | ★★★★★ |

**推荐组合**（校招+Agent 工程岗）：
- **必做**：P1（打基础）+ P2（核心 RAG 经验）
- **选做 1**：P3 或 P5（看你想偏 multi-agent 还是 reasoning）
- **加分**：P4 或 P6（前沿话题，差异化）

---

## 4. 项目驱动的学习循环

```
项目选定 → 单元复习（先看相关单元的概念笔记）
  ↓
MVP 设计（5-10 行伪代码：先跑通最小流程）
  ↓
逐功能实现（每个功能对应 1-2 个单元的练习）
  ↓
Eval-driven 改进（用 U12 / U35 的工具测质量）
  ↓
复盘 + 文档（README + 架构图 + 关键 trade-off）
```

**关键原则**：
- **先 MVP 后完美**：P1 第一版 50 行能跑就行
- **每功能配 eval**：加 RAG 必测 faithfulness；加 agent 必测 task completion
- **每项目配 README**：架构图 + 关键 trade-off + 跑通截图
- **每项目配 demo**：Streamlit / Gradio，让面试官能玩

---

## 5. 6 个项目的"加分项"（差异化）

每个项目都有 common path 和 differentiator。**面试要讲 differentiator**：

| 项目 | Common Path | Differentiator（加分）|
|------|-------------|---------------------|
| **P1 ReAct** | 跑通 3 个 tool | 加 streaming + trace log + unit test |
| **P2 RAG** | 跑通 PDF → answer | 加 hybrid 检索 + reranker + RAGAS eval dashboard |
| **P3 Multi-Agent** | writer + reviewer 协作 | 加 A2A / MCP 协议 + subagent 委派 + HITL |
| **P4 Computer Use** | 跑通浏览器任务 | 加 sandbox + 错误恢复 + multi-step planning |
| **P5 Reasoning** | o1 调 tool | 加 cascade 路由 + self-consistency + cost monitoring |
| **P6 Long-Running** | 跨 session 持续 | 加 human-in-the-loop + adversarial review + DPO 训练 |

---

## 6. 实战：从项目到简历的转化

**每个项目完工后**：

1. **README.md**（关键）：
   - 1 句话项目定位
   - 架构图（Mermaid / draw.io）
   - 关键 trade-off 解释
   - "Why I built it this way" 段落
2. **博客文章**（加分）：
   - 写一篇 1500 字的 "How I built X" 文章
   - 包含 1 个具体问题的解决过程（最有价值的内容）
3. **Demo 视频**（强加分）：
   - 5-10 分钟 screen recording
   - 跑通主流程 + 展示 trace / eval 结果
4. **量化指标**（面试必问）：
   - "Faithfulness 0.85 vs baseline 0.72"（P2）
   - "Task completion 92%, 成本 $0.003/query"（P3）
   - "8 个 production loop pattern 跑通 24h"（P6）

---

## 7. 校招项目选择建议

**目标**：3 个月做 2-3 个能讲清楚的项目。

**推荐组合**（3 个月，2.5h/天）：

| 月 | 重点 | 项目 | 覆盖单元 |
|----|------|------|----------|
| **M1** | 基础 | P1 ReAct 原型 | U1 / U3 / U5 / U6 / U-DL |
| **M2** | 核心 | P2 RAG 知识库 | U2 / U4 / U15 / U22 / U31 |
| **M3** | 进阶 | P3 Multi-Agent 或 P5 Reasoning | U9 / U11 / U13 / U24 |

**每个项目 1 个月 = ~75h = 完整 MVP + 加分项**。

---

## 8. 项目参考资源汇总

### datawhale 系
- [hello-agents](https://github.com/datawhalechina/hello-agents)（63k⭐）— 16 章 + 13 Extra，覆盖 LLM/ReAct/MCP/RAG/Agentic RL 全栈
- [all-in-rag](https://github.com/datawhalechina/all-in-rag)（9k⭐）— RAG 技术全栈（10 章 + 2 项目）
- [agent-skills-with-anthropic](https://github.com/datawhalechina/agent-skills-with-anthropic)（1.3k⭐）— 吴恩达课程
- [post-training-of-llms](https://github.com/datawhalechina/post-training-of-llms) — SFT/GRPO 训练
- [handy-multi-agent](https://github.com/datawhalechina/handy-multi-agent) — CAMEL 框架
- [hugging-multi-agent](https://github.com/datawhalechina/hugging-multi-agent) — MetaGPT

### 工业级项目
- [jjyaoao/HelloAgents](https://github.com/jjyaoao/helloagents) — 自研多 agent 框架
- [datawhalechina/self-llm](https://github.com/datawhalechina/self-llm) — 开源 LLM 部署 + 微调
- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) — Claude Code 从 0 到 1

### 论文必读
- [ReAct 论文](https://arxiv.org/abs/2210.03629) — ReAct
- [DeepSeek R1](https://arxiv.org/abs/2501.12948) — Reasoning + GRPO
- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — Multi-agent 设计

---

## 9. 面试视角

- **Q：做过哪些 Agent 项目？**
  → 答：用 STAR 法（Situation / Task / Action / Result），重点讲 Result（量化指标 + trade-off）。**展示 demo 而非讲理论**。

- **Q：哪个项目最有挑战？**
  → 答：选 P3 / P5 / P6 之一（高难度），讲**真实遇到的问题**（不是讲你解决了什么，而是讲你挣扎了 + 怎么 debug 的）。

- **Q：怎么选项目？**
  → 答：**覆盖核心 + 差异化**。覆盖核心（ReAct / RAG / Multi-Agent）让面试官知道基础扎实；差异化（Reasoning / Computer Use / Loop Engineering）让你从其他候选人中脱颖而出。

- **Q：项目用什么模型？**
  → 答：**有理由地选**。比如："P2 用 gpt-4o-mini 因为成本 + 95% faithfulness 已经够；P5 用 o1-mini 因为 reasoning 能力比 gpt-4o 强 30%"——**不是"用最好的"，而是"用最合适的"**。

---

## 10. 必读材料

| 材料 | 链接 | 为什么读 |
|------|------|----------|
| **hello-agents** | [datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents) | 63k⭐ 中文最系统的 Agent 教程（16 章 + 13 Extra）|
| **all-in-rag** | [datawhalechina/all-in-rag](https://github.com/datawhalechina/all-in-rag) | RAG 技术全栈（10 章 + 2 项目）|
| **HelloAgents 自研框架** | [jjyaoao/helloagents](https://github.com/jjyaoao/helloagents) | OpenAI 原生 API 自研 multi-agent 框架 |
| **learn-claude-code** | [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | Claude Code 复刻 |
| **agentic-ai** | [datawhalechina/agentic-ai](https://github.com/datawhalechina/agentic-ai) | 吴恩达 Agentic AI 课程 |

---

## 11. 练习

1. **选 1 个项目**：根据时间和目标选 P1 / P2 / P3 之一
2. **先跑 MVP**：50 行 Python 跑通最小流程
3. **加 eval**：用 U12 / U35 工具测质量
4. **写 README**：架构图 + trade-off + 量化指标
5. **博客文章**：写 1500 字 "How I built X"
6. **Demo 视频**：5-10 分钟 screen recording

---

## 12. 与本仓库其他单元的关系

U41 是**跨单元整合**单元，覆盖所有 41 个单元的项目实践。详见每个项目的"对应单元"列。

---

*最后更新：2026-06-30 新增 U41（Agent 全栈项目学习路线）*
*来源：[datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents)（63k⭐ 中文最系统 Agent 教程）· [datawhalechina/all-in-rag](https://github.com/datawhalechina/all-in-rag)（9k⭐ RAG 全栈）· [jjyaoao/HelloAgents](https://github.com/jjyaoao/helloagents) · [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) · [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)*
