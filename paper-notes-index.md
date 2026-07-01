# Paper Notes Index

> 按主题/论文组织的学习笔记

---

## 笔记目录

### 大模型基础

| 论文 | 核心贡献 | 笔记 |
|------|----------|------|
| (暂无笔记) | | |

### Agent 架构

| 论文 | 核心贡献 | 笔记 |
|------|----------|------|
| (暂无笔记) | | |

### Planning & Reasoning

| 论文 | 核心贡献 | 笔记 |
|------|----------|------|
| (暂无笔记) | | |

### Memory & Knowledge

| 论文 | 核心贡献 | 笔记 |
|------|----------|------|
| (暂无笔记) | | |

### Multi-Agent

| 论文 | 核心贡献 | 笔记 |
|------|----------|------|
| (暂无笔记) | | |

### Evaluation & Safety

| 论文 | 核心贡献 | 笔记 |
|------|----------|------|
| (暂无笔记) | | |

---

## 笔记规范

每个笔记文件命名：`YYYY-MM-DD_paper-name_summary.md`

内容结构：
1. **基本信息**：论文标题、作者、会议/期刊、链接
2. **一句话总结**：这篇论文解决了什么问题，核心方法是什么
3. **核心方法**：用流程图或示意图说明（ASCII 或文字描述）
4. **关键创新点**：2-3 条
5. **代码/实现细节**：如果有开源代码的位置
6. **面试相关**：这篇论文可能被问到的面试题
7. **疑问/思考**：读完后还有哪些不清楚的地方

---

## 推荐阅读路线

### 入门（必读）

1. **Attention is All You Need** (Vaswani et al., 2017) - Transformer 基础
2. **GPT-3** (Brown et al., 2020) - 展示 LLM 的涌现能力
3. **InstructGPT** (Ouyang et al., 2022) - RLHF 的奠基之作

### Agent 进阶

4. **ReAct** (Yao et al., 2022) - Synergy between reasoning and acting
5. **Toolformer** (Schick et al., 2023) - LLM 学会使用工具
6. **AutoGPT** (Significant Gravitas, 2023) - 自主 Agent 实践

### 系统设计

7. **DSPy** (Khattab et al., 2023) - 声明式 LLM 编程
8. **LangChain** (Anthropic, 2023) - LLM 应用框架
9. **AutoGen** (Microsoft, 2023) - 多 Agent 协作框架


## 必读论文 Top 30（按主题分类）

> 来源：业界共识 + 100-Agentic-AI-Interview-Questions + 本仓库 41 单元引用

### 大模型基础（必读 5 篇）

| # | 论文 | 年份 | 核心贡献 | 对应单元 |
| --- | --- | --- | --- | --- |
| 1 | Attention Is All You Need (Vaswani et al.) | 2017 | Transformer 架构，self-attention | U1 |
| 2 | GPT-3 (Brown et al.) | 2020 | 涌现能力，few-shot learning | U1 |
| 3 | InstructGPT (Ouyang et al.) | 2022 | RLHF 奠基，alignment | U30 |
| 4 | Llama 2 (Touvron et al.) | 2023 | 开源 LLM 标杆，指令微调 | U1 |
| 5 | Chinchilla (Hoffmann et al.) | 2022 | 数据缩放律，compute-optimal | U28 |

### RAG（必读 4 篇）

| # | 论文 | 年份 | 核心贡献 | 对应单元 |
| --- | --- | --- | --- | --- |
| 6 | RAG (Lewis et al.) | 2020 | RAG 范式，retrieval-augmented | U4 |
| 7 | Self-RAG (Asai et al.) | 2023 | 自反思 RAG，critique tokens | U4 |
| 8 | GraphRAG (Edge et al.) | 2024 | 知识图谱增强 RAG | U15 |
| 9 | HyDE (Gao et al.) | 2022 | 假设文档嵌入，提升召回 | U15 |

### Agent 架构（必读 6 篇）

| # | 论文 | 年份 | 核心贡献 | 对应单元 |
| --- | --- | --- | --- | --- |
| 10 | ReAct (Yao et al.) | 2022 | Reasoning + Acting 交替 | U5 |
| 11 | Toolformer (Schick et al.) | 2023 | LLM 学会使用工具 | U6 |
| 12 | BabyAGI (Nakajima) | 2023 | 任务驱动的自主 Agent | U7 |
| 13 | Plan-and-Solve (Wang et al.) | 2023 | 先规划后执行 | U7 |
| 14 | LLM Compiler (Cheng et al.) | 2024 | 并行规划，4x 加速 | U7 |
| 15 | ADaPT (Dagan et al.) | 2023 | 自适应任务分解 | U7 |

### Memory & Context（必读 4 篇）

| # | 论文 | 年份 | 核心贡献 | 对应单元 |
| --- | --- | --- | --- | --- |
| 16 | MemGPT / Letta (Packer et al.) | 2023 | 分层记忆，虚拟上下文 | U8 |
| 17 | RMT / Scratchpad (Saunders et al.) | 2022 | 外显推理步骤 | U8 |
| 18 | Effective Context Engineering (Anthropic) | 2025 | Context 工程官方定义 | U22 |
| 19 | AI Harness Engineering (arXiv 2605.13357) | 2025 | Harness 11 组件责任 | U23 |

### Planning & Reasoning（必读 3 篇）

| # | 论文 | 年份 | 核心贡献 | 对应单元 |
| --- | --- | --- | --- | --- |
| 20 | ToT (Yao et al.) | 2023 | Tree of Thoughts，多路径搜索 | U7 |
| 21 | GoT (Besta et al.) | 2024 | Graph of Thoughts，任意图结构 | U7 |
| 22 | o1 / OpenAI System Card | 2024 | 推理模型，test-time compute | U24 |

### Multi-Agent（必读 3 篇）

| # | 论文 | 年份 | 核心贡献 | 对应单元 |
| --- | --- | --- | --- | --- |
| 23 | AutoGen (Wu et al.) | 2024 | 多 Agent 对话框架 | U9 |
| 24 | ChatDev (Qian et al.) | 2023 | 软件公司模拟，角色扮演 | U9 |
| 25 | MetaGPT (Hong et al.) | 2023 | 结构化多 Agent 协作 | U9 |

### Evaluation & Safety（必读 5 篇）

| # | 论文 | 年份 | 核心贡献 | 对应单元 |
| --- | --- | --- | --- | --- |
| 26 | SWE-bench (Jimenez et al.) | 2024 | 真实软件工程基准 | U10 |
| 27 | AgentGym (Zhang et al.) | 2024 | 终身学习 Agent 环境 | U10 |
| 28 | RAGAS (Es et al.) | 2023 | RAG 评估框架，4 指标 | U10 |
| 29 | BIPIA (Chen et al.) | 2024 |  Indirect Prompt Injection 攻击 | U18 |
| 30 | AgentPoison (Zhan et al.) | 2024 | RAG 投毒攻击 | U18 |

### 面试高频追问映射

每篇论文可能的面试题：
- Attention Is All You Need → "Transformer 为什么能并行？Self-attention 复杂度是多少？"
- InstructGPT → "RLHF 三步是什么？PPO 为什么比 REINFORCE 好？"
- ReAct → "ReAct 和 CoT 的区别？什么时候用 ReAct？"
- BabyAGI → "这个系统的瓶颈在哪？怎么优化？"
- MemGPT → "分层记忆是怎么实现的？"
- o1 → "Test-time compute scaling law 是什么？和 pre-training 有什么不同？"
- BIPIA → "Indirect Prompt Injection 和直接注入的区别？怎么防御？"

---

*最后更新：2026-07-01*