# 核心概念笔记

> 用于记录 ReAct、Tool Use、Planning、Memory 等核心概念的个人理解（偏向"能用大白话讲清楚"）

## 笔记规范

每个概念一个文件，命名：`concept-name.md`

内容结构：
1. **官方定义**：来自论文或权威文档的一句话定义
2. **我的理解**：用类比或生活实例解释
3. **为什么需要这个**：它解决了什么问题
4. **实现要点**：如果要写代码实现，关键点是什么
5. **面试怎么问**：面试官可能怎么问，我应该怎么答
6. **相关概念**：和哪些其他概念有关联

## 当前笔记列表

| 文件 | 主题 | 备注 |
|------|------|------|
| U1-LLM基础.md | GPT原理/采样/RLHF | ✅ |
| U2-Embedding与VectorDB.md | 向量检索/ANN索引 | ✅ |
| U3-Prompt工程.md | Zero-shot/CoT/结构化输出 | ✅ |
| U4-RAG基础.md | RAG三段流程/Chunking | ✅ |
| U5-ReAct范式.md | 四步循环/与CoT区别 | ✅ |
| U6-ToolUse与FunctionCalling.md | JSON Schema/Tool Choice | ✅ |
| U7-Planning与Reasoning.md | GSM-SF/ToT/子目标分解 | ✅ |
| U8-Memory与Reflection.md | 短/长期记忆/Reflexion | ✅ |
| U9-MultiAgent系统.md | 协作模式/通信协议 | ✅ |
| U10-Agent评估与安全.md | 评测指标/红队/防御 | ✅ |
| U11-Memory框架对比.md | mem0/Letta/khoj三框架对比 | 🆕 |
| U12-Agent评估标准.md | SWE-bench/AgentGym/RAGAS | 🆕 |
| U13-A2A协议.md | Agent间通信协议 | 🆕 |
| U14-DSPy.md | 声明式Prompt优化/自动编译 | 🆕 |
| U15-RAG进阶.md | HyDE/GraphRAG/Reranking | 🆕 |
| U16-Embedding进阶.md | MTEB/ColBERT/微调方法 | 🆕 |
| U17-多模态Agent.md | GPT-4V/LLaVA/视觉Agent | 🆕 |
| U18-Agent安全与红队.md | 注入攻击/沙箱/红队方法论 | 🆕 |
| U19-向量数据库深度对比.md | FAISS/Chroma/Qdrant/Milvus | 🆕 |
| U20-编程Agent内部原理.md | Claude Code/SWE-agent架构 | 🆕 |
| UD-DL深度学习扫盲.md | Attention/Transformer直观理解 | ✅ |

---

*最后更新：2026-05-27*