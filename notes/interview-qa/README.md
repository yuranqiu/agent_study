# Interview QA · 面试题总览

> 来源：Navedphysicist/100-Agentic-AI-Interview-Questions + SihaoLiu/vibe-agentic-interview-notes

---

## 文件清单

|| 文件 | 内容 | 篇幅 |
|------|------|------|
| `agent-core.md` | Agent 核心概念：基础 / ReAct / Tool Use / Memory / Planning / Multi-Agent / 设计模式 | ~150 题 |
| `frameworks.md` | 框架篇：LangChain / CrewAI / AutoGen / MCP / smolagents / Dify | ~80 题 |
| `coding-agent.md` | Coding Agent 进阶（Claude Code / Codex）：背景知识 / 细节梳理 / 工作流 / 系统设计 / 概念哲学 | ~38 题 |
| `memory-frameworks.md` | Memory 框架：mem0 / Letta / khoj 对比，架构设计题 | 🆕 ~30 题 |
| `agent-eval.md` | Agent 评估：SWE-bench / AgentGym / RAGAS / Trulens / 幻觉检测 | 🆕 ~25 题 |
| `dspy-qa.md` | DSPy：声明式 Prompt / Signature / Teleprompter / 编译器 | 🆕 ~20 题 |
| `prompt-qa.md` | Prompt 工程（U3）：Zero-shot/Few-shot/CoT / System Prompt 4 要素 / Anthropic 12 法则 / XML 标签 / Pydantic Mode / Prompt 注入防御 / Temperature 调参 | 🆕 ~40 题 |
| `rag-qa.md` | RAG 完整篇（U4+U15）：流程/Chunking/Hybrid/Rerank/GraphRAG/CRAG/评估/选型/系统设计 | 🆕 ~20 题 |
| `advanced-rag.md` | 高级 RAG：HyDE / GraphRAG / Reranking / Chunking / 评估 | 🆕 ~35 题 |
| `planning-qa.md` | Planning 与推理（U7）：GSM-SF / ToT / PoT / Plan-and-Solve / LLM Compiler / ADaPT / 子目标缓存 / 状态回溯 / 失败恢复 / 系统设计 | 🆕 ~30 题 |
| `security-red-team.md` | Agent 安全：注入攻击 / 沙箱 / 红队方法论 / 防御设计 | 🆕 ~25 题 |
| `multi-modal-agent.md` | 多模态 Agent：GPT-4V / LLaVA / 视觉架构 / 文档理解 | 🆕 ~25 题 |
| `coding-agent-internals.md` | Coding Agent 内部原理：harness / 权限模型 / 沙箱 / 上下文压缩 | 🆕 ~30 题 |
| `multi-agent-qa.md` | Multi-Agent 系统：协作模式 / 通信协议 / LangGraph State / 拓扑 / Writer-Reviewer Loop / 系统设计 | 🆕 ~27 题 |
| `deploy-ops-qa.md` | Agent 部署与运维：4层架构 / Sandbox / Session / Gateway / CI-CD / 可观测性 / Eval-Driven | 🆕 ~30 题 |
| `safety-permission-qa.md` | Agent 安全 & 权限：注入攻击 / Guardrails / Permission 模式 / HITL / 红队 / 架构设计 | 🆕 ~22 题 |
| `inference-qa.md` | LLM Inference & Cost/Latency：Prefill/Decode / KV Cache / Prompt Caching / TTFT/TPS / vLLM-SGLang-TGI / Batching / Model Routing / Streaming / Cost 优化 / 延迟压到 200ms | 🆕 ~70 题 |
| `context-harness-qa.md` | Context + Harness 工程（U22+U23）：Context Engineering 官方定义 / Context Rot / 5 大组件 / Compaction/Note-taking/Sub-agent / Just-in-time vs Pre-loaded / Harness 11 组件 / H0-H3 四级阶梯 / Claude Code 案例 | 🆕 ~40 题 |
| `reasoning-qa.md` | Reasoning Model（U24）：o1 / R1 / Claude 3.7 Thinking / Test-time Compute / Reasoning Effort / Thinker+Doer / Cascade / 局限 / 选型 / 面试题 | 🆕 ~35 题 |

---

## 使用方法

### 面试前三问

1. **这个知识点背后能讲出什么问题？**（不只是是什么，还要为什么）
2. **能写出最小可运行的代码吗？**（不是背，是理解后能实现）
3. **在真实项目里哪个场景会用到？**（关联自己的项目经验）

### 学习优先级

```
高频必背：ReAct 四步循环 / Function Calling 格式 / Tool Definition 字段
理解即可：ToT / Reflexion / Generative Agents 论文细节
扩展视野：Coding Agent 篇的开放思辨题（系统设计/哲学）
```

---

## 面试追问链路（示例）

```
问：什么是 ReAct？
答：推理 + 行动交替的范式
追问：和 Function Calling 的区别？
答：FC 是直接调 tool，ReAct 中间插入推理和观察
追问：ReAct 在什么场景下比 CoT 好？
答：需要外部知识、避免幻觉的场景
追问：手写一个 ReAct Agent 的核心循环
答：（写出 while 循环 + tool_call 处理逻辑）
```

---

## 按框架索引

| 框架 | 面试题文件 |
|------|-----------|
| LangChain | `frameworks.md` → LangChain 篇 |
| CrewAI | `frameworks.md` → CrewAI 篇 |
| AutoGen | `frameworks.md` → AutoGen 篇 |
| MCP | `frameworks.md` → MCP 篇 |
| ReAct 范式 | `agent-core.md` → ReAct 篇 |
| Tool Use | `agent-core.md` → Tool Use 篇 |
| Memory | `agent-core.md` → Memory 篇 |
| Planning | `agent-core.md` → Planning 篇 |
| Multi-Agent | `agent-core.md` → Multi-Agent 篇 |
| Prompt 工程 | `prompt-qa.md` 全文 |
| RAG 完整篇 | `rag-qa.md` 全文 |
| LLM Inference / Cost | `inference-qa.md` 全文 |
| Agent 安全 & 权限 | `safety-permission-qa.md` 全文 |
| Claude Code / Codex | `coding-agent.md` 全文 |

---

## 综合学习指南

| 资源 | 文件 |
|------|------|
| U42 面试真题与学习顺序 | `notes/concept-notes/U42-InterviewGuide.md` |

---

*最后更新：2026-07-01*
