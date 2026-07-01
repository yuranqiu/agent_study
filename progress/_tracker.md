# 进度追踪

> 路线已重构为集群（Cluster）+ 单元（Unit）视图，弱化时间线。本文件用于追踪单元完成状态和累计投入。

## 当前焦点

**Phase 1：基础夯实** — C1（LLM/Prompt）+ C2（Embedding）+ C3（RAG）+ C4（Agent 范式）基础

入口单元：U1 → U3 → U2 → U4 → U6 → U5

---

## 单元完成状态

> 每完成一个单元，将 `todo` 改为 `done`，并记录笔记位置。

### C1 · LLM 与 Prompt 基础

- [ ] U1 · LLM 基础（`notes/concept-notes/U1-LLM基础.md`）
- [ ] U3 · Prompt 工程（`notes/concept-notes/U3-Prompt工程.md`）
- [ ] U14 · DSPy（`notes/concept-notes/U14-DSPy.md`）
- [ ] U24 · Reasoning Model（`notes/concept-notes/U24-ReasoningModel.md`）
- [ ] U30 · Post-training 与 Alignment（`notes/concept-notes/U30-PostTraining.md`）
- [ ] U32 · Fine-tuning 实践（`notes/concept-notes/U32-FineTuning.md`）
- [ ] U-DL · 深度学习扫盲（`notes/concept-notes/UD-DL深度学习扫盲.md`）

### C2 · Embedding 与检索

- [ ] U2 · Embedding 与 VectorDB（`notes/concept-notes/U2-Embedding与VectorDB.md`）
- [ ] U16 · Embedding 进阶（`notes/concept-notes/U16-Embedding进阶.md`）
- [ ] U19 · 向量数据库深度对比（`notes/concept-notes/U19-向量数据库深度对比.md`）

### C3 · RAG

- [ ] U4 · RAG 基础（`notes/concept-notes/U4-RAG基础.md`）
- [ ] U15 · RAG 进阶（`notes/concept-notes/U15-RAG进阶.md`）

### C4 · Agent 范式

- [ ] U5 · ReAct 范式（`notes/concept-notes/U5-ReAct范式.md`）
- [ ] U6 · Tool Use 与 Function Calling（`notes/concept-notes/U6-ToolUse与FunctionCalling.md`）
- [ ] U7 · Planning 与 Reasoning（`notes/concept-notes/U7-Planning与Reasoning.md`）

### C5 · Memory 与状态

- [ ] U8 · Memory 与 Reflection（`notes/concept-notes/U8-Memory与Reflection.md`）
- [ ] U11 · Memory 框架对比（`notes/concept-notes/U11-Memory框架对比.md`）

### C6 · Multi-Agent 与协议

- [ ] U9 · Multi-Agent 系统（`notes/concept-notes/U9-MultiAgent系统.md`）
- [ ] U13 · A2A 协议（`notes/concept-notes/U13-A2A协议.md`）
- [ ] U36 · MCP 深入（`notes/concept-notes/U36-MCPDeep.md`）

### C7 · 评估与安全

- [ ] U10 · Agent 评估与安全（`notes/concept-notes/U10-Agent评估与安全.md`）
- [ ] U12 · Agent 评估标准（`notes/concept-notes/U12-Agent评估标准.md`）
- [ ] U18 · Agent 安全与红队（`notes/concept-notes/U18-Agent安全与红队.md`）
- [ ] U34 · Guardrails 与 Output 防御（`notes/concept-notes/U34-Guardrails.md`）
- [ ] U35 · AgentOps / Eval 工具链（`notes/concept-notes/U35-AgentOps.md`）

### C8 · 多模态与编程 Agent

- [ ] U17 · 多模态 Agent（`notes/concept-notes/U17-多模态Agent.md`）
- [ ] U20 · 编程 Agent 内部原理（`notes/concept-notes/U20-编程Agent内部原理.md`）
- [ ] U25 · Computer Use / GUI Agent（`notes/concept-notes/U25-ComputerUse.md`）

### C9 · Loop Engineering 与长跑 Agent

- [ ] U21 · Loop Engineering（`notes/concept-notes/U21-LoopEngineering.md`）
- [ ] U27 · Goal Engineering（`notes/concept-notes/U27-GoalEngineering.md`）

### C10 · Agent 运行时工程（Context + Harness）

- [ ] U22 · Context Engineering（`notes/concept-notes/U22-ContextEngineering.md`）
- [ ] U23 · Harness Engineering（`notes/concept-notes/U23-HarnessEngineering.md`）
- [ ] U31 · Structured Output & Validation（`notes/concept-notes/U31-StructuredOutput.md`）
- [ ] U33 · Human-in-the-loop 设计（`notes/concept-notes/U33-HumanInTheLoop.md`）
- [ ] U37 · Subagents 与任务隔离（`notes/concept-notes/U37-Subagents.md`）
- [ ] U38 · Context 管理深入（`notes/concept-notes/U38-ContextManagement.md`）
### C11 · Agent 能力工程化（Skills + AGENTS.md）

- [ ] U26 · Agent Skills & AGENTS.md（`notes/concept-notes/U26-AgentSkills.md`）
- [ ] U41 · Agent 全栈项目学习路线（`notes/concept-notes/U41-AgentProjectPath.md`）

### C12 · 生产工程（Inference + Deploy）

- [ ] U28 · LLM Inference & Cost/Latency 工程（`notes/concept-notes/U28-LLMInference.md`）
- [ ] U29 · Agent 部署 & Sandbox DevOps（`notes/concept-notes/U29-AgentDeploy.md`）
- [ ] U39 · Permission & Auto Mode（`notes/concept-notes/U39-Permission.md`）
---

## 累计投入时间

| 单元 / 项目 | 开始 | 完成 | 投入时长 | 关键产出 |
|------------|------|------|----------|----------|
| - | - | - | - | - |

---

## 资源索引（按单元）

| 单元 | 概念笔记 | 工具速查 | 面试题 |
|------|----------|----------|--------|
| U1 | `notes/concept-notes/U1-LLM基础.md` | — | `notes/interview-qa/agent-core.md` |
| U2 | `notes/concept-notes/U2-Embedding与VectorDB.md` | — | `notes/interview-qa/agent-core.md` |
| U3 | `notes/concept-notes/U3-Prompt工程.md` | — | `notes/interview-qa/agent-core.md` |
| U4 | `notes/concept-notes/U4-RAG基础.md` | — | `notes/interview-qa/agent-core.md` |
| U5 | `notes/concept-notes/U5-ReAct范式.md` | `tools/langchain-guide.md` | `notes/interview-qa/agent-core.md` |
| U6 | `notes/concept-notes/U6-ToolUse与FunctionCalling.md` | `tools/langchain-guide.md`, `tools/smolagents-guide.md` | `notes/interview-qa/agent-core.md` |
| U7 | `notes/concept-notes/U7-Planning与Reasoning.md` | `tools/langchain-guide.md` | `notes/interview-qa/agent-core.md` |
| U8 | `notes/concept-notes/U8-Memory与Reflection.md` | `tools/langchain-guide.md` | `notes/interview-qa/agent-core.md` |
| U9 | `notes/concept-notes/U9-MultiAgent系统.md` | `tools/langchain-guide.md`, `tools/crewai-guide.md`, `tools/autogen-guide.md` | `notes/interview-qa/frameworks.md`, `notes/interview-qa/coding-agent.md` |
| U10 | `notes/concept-notes/U10-Agent评估与安全.md` | — | `notes/interview-qa/agent-core.md`, `notes/interview-qa/coding-agent.md` |
| U11 | `notes/concept-notes/U11-Memory框架对比.md` | `tools/langchain-guide.md`, `tools/llamaindex-guide.md` | `notes/interview-qa/memory-frameworks.md` |
| U12 | `notes/concept-notes/U12-Agent评估标准.md` | — | `notes/interview-qa/agent-eval.md` |
| U13 | `notes/concept-notes/U13-A2A协议.md` | `tools/a2a-guide.md` | — |
| U14 | `notes/concept-notes/U14-DSPy.md` | `tools/dspy-guide.md` | `notes/interview-qa/dspy-qa.md` |
| U15 | `notes/concept-notes/U15-RAG进阶.md` | `tools/llamaindex-guide.md` | `notes/interview-qa/advanced-rag.md` |
| U16 | `notes/concept-notes/U16-Embedding进阶.md` | `tools/llamaindex-guide.md` | — |
| U17 | `notes/concept-notes/U17-多模态Agent.md` | `tools/llamaindex-guide.md` | `notes/interview-qa/multi-modal-agent.md` |
| U18 | `notes/concept-notes/U18-Agent安全与红队.md` | — | `notes/interview-qa/security-red-team.md`, `notes/interview-qa/Agent-安全面试题.md` |
| U19 | `notes/concept-notes/U19-向量数据库深度对比.md` | — | `notes/interview-qa/frameworks.md` |
| U20 | `notes/concept-notes/U20-编程Agent内部原理.md` | — | `notes/interview-qa/coding-agent-internals.md` |
| U21 | `notes/concept-notes/U21-LoopEngineering.md` | — | — |
| U22 | `notes/concept-notes/U22-ContextEngineering.md` | — | — |
| U23 | `notes/concept-notes/U23-HarnessEngineering.md` | — | — |
| U24 | `notes/concept-notes/U24-ReasoningModel.md` | — | — |
| U25 | `notes/concept-notes/U25-ComputerUse.md` | — | — |
| U26 | `notes/concept-notes/U26-AgentSkills.md` | — | — |
| U27 | `notes/concept-notes/U27-GoalEngineering.md` | — | — |
| U28 | `notes/concept-notes/U28-LLMInference.md` | — | — |
| U29 | `notes/concept-notes/U29-AgentDeploy.md` | — | — |
| U30 | `notes/concept-notes/U30-PostTraining.md` | — | — |
| U31 | `notes/concept-notes/U31-StructuredOutput.md` | — | — |
| U32 | `notes/concept-notes/U32-FineTuning.md` | — | — |
| U33 | `notes/concept-notes/U33-HumanInTheLoop.md` | — | — |
| U34 | `notes/concept-notes/U34-Guardrails.md` | — | — |
| U35 | `notes/concept-notes/U35-AgentOps.md` | — | — |
| U36 | `notes/concept-notes/U36-MCPDeep.md` | `tools/mcp-guide.md` | — |
| U37 | `notes/concept-notes/U37-Subagents.md` | — | — |
| U38 | `notes/concept-notes/U38-ContextManagement.md` | — | — |
| U39 | `notes/concept-notes/U39-Permission.md` | — | — |
| U40 | `notes/concept-notes/U40-Verification.md` | — | — |
| U41 | `notes/concept-notes/U41-AgentProjectPath.md` | — | — |
| U-DL | `notes/concept-notes/UD-DL深度学习扫盲.md` | — | — |

---

## 路线调整记录

| 日期 | 调整内容 | 原因 |
|------|----------|------|
| 2026-05-27 | 新增 `tools/` 目录（5 个框架速查指南） | 用户要求学会所有工具的用法 |
| 2026-05-27 | 新增 `notes/interview-qa/`（3 个面试题文件） | 用户要求根据面经调整学习内容 |
| 2026-05-27 | U5-U10 每个单元补充工具文件 + 面试资料链接 | 对应到具体学习单元 |
| 2026-06-13 | 整体路线压缩：6阶段→4阶段，12个月→3.5个月 | 原计划未按时启动，聚焦秋招准备 |
| 2026-06-13 | Phase 1 从 4 周压缩为 3 周 | 时间紧迫，优先保证核心内容 |
| 2026-06-13 | 砍掉 Phase 3（LangChain 深度）和 Phase 5（进阶主题） | 面试不考源码级理解，聚焦实战 |
| 2026-06-30 | 路线重构为 Cluster + Unit，去除时间线 | 用户要求去除时间线，按主题组织 |
| 2026-06-30 | 补齐 U11-U20 + U-DL 单元（已存在但未在 README 索引） | 用户要求补充缺失的 agent 学习内容 |
| 2026-06-30 | 引入 8 个知识集群（C1-C8），建立单元→笔记→工具→面试题索引 | 把仓库已有资源全部串起来 |
| 2026-06-30 | 新增 C9 + U21（Loop Engineering），引入 6 个新兴项目 | 用户提出 Loop Engineering 概念 |
| 2026-06-30 | 新增 C10 + U22 + U23（Context + Harness Engineering） | 用户要求补全 Harness/Context Engineering 概念 |
| 2026-06-30 | 新增 C1+U24（Reasoning Model）、C8+U25（Computer Use）、C9+U27（Goal Engineering）、C11+U26（Skills）、C12+U28+U29（生产工程）共 6 个单元 + 2 个新集群 | 用户要求补充剩余核心方向 |
| 2026-06-30 | 新增 C1+U30+U32（Post-training + Fine-tuning 实战）、C7+U34+U35（Guardrails + AgentOps）、C10+U31+U33（Structured Output + HITL）共 6 个单元 | 用户要求补充后训练相关，并补足生产工程：Post-training 是 2024-2026 范式革命（RLHF/DPO/GRPO/RLVR），AgentOps 是 LLM 时代的 DevOps，HITL/GUARDRAILS/Structured Output 是生产必修 |
| 2026-06-30 | U33 HITL 深度增强（v2.0，12 节→19 节，536 行→958 行） | 用户要求检查 U33 深度并补充优秀教程和必读论文。完整列表略 |
| 2026-06-30 | 新增 5 个单元：U36 MCP 深入（C6）/ U37 Subagents 与任务隔离（C10）/ U38 Context 管理深入（C10）/ U39 Permission & Auto Mode（C12）/ U40 Verification & Adversarial Review（C10）| 用户要求"检查还有什么遗漏"——通过研究 Claude Code best practices + Anthropic Context Engineering + MCP 官方文档 + LangChain MCP adapter，发现 5 个生产必修单元缺失。版本 → 2.6 |
| 2026-06-30 | 深度增强 7 个薄弱单元 + 新增 U41 跨单元项目路线（v2.7）| 用户要求"再检查深度"——通过研究 datawhale/hello-agents (63k⭐) / all-in-rag (9k⭐) / learn-claude-code (69k⭐) 等优秀教程，发现 7 个单元深度不够（< 250 行）+ 缺跨单元整合。增强：U3 (+Anthropic 12 法则 + 10 模板) / U4 (+all-in-rag chunking + embedding 选型) / U7 (+Plan-and-Solve + LLM Compiler + ADaPT) / U8 (+mem0 4 层 + Reflexion 论文) / U10 (+SWE-bench + Langfuse Eval) / U14 (+DSPy GEval + MIPROv2) / U21 (+Lifetime Goal 模式 + 7 Pattern 深度)。新增 U41 (Agent 全栈项目学习路线)：6 个项目（P1-P6）整合所有概念，含 datawhale 教程引用。版本 → 2.7 |
