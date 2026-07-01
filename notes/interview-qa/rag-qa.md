# Interview QA · RAG 完整篇

> **来源**：U4 RAG 基础 + U15 RAG 进阶 + 生产实践 + 高频面试题
> **覆盖**：完整流程 / Chunking / Hybrid Search / Reranking / HyDE / GraphRAG / CRAG / RAGAS 评估 / 方案选型 / 失败模式 / 系统设计

---

## 一、RAG 完整流程

### Q1: RAG 完整流程是什么？三个核心阶段各做什么？

**Answer:**

```
文档 → Chunking → Embedding → VectorDB   ← 离线：摄取（Ingestion）
                                            ↓
用户问题 → Embedding → 检索 Top-K → [Rerank] → Context → LLM 生成   ← 在线
```

| 阶段 | 做什么 | 关键工具 |
|------|--------|----------|
| Ingestion | 加载 → 清洗 → 切分 → 向量化 → 入库 | PyMuPDF, RecursiveCharacterTextSplitter, FAISS |
| Retrieval | 问题向量化 → 相似度搜索 → 召回 Top-K | VectorDB, Hybrid Search, Reranker |
| Generation | 拼接 Context → 构造 Prompt → LLM 生成 | Chat LLM, Prompt Template |

**核心原则：** Ingestion 和 Retrieval 阶段**必须用同一个 Embedding 模型**，否则向量空间不一致，检索失效。

```python
# 完整流程
from langchain_community.document_loaders import PyMuPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS

# 1. Ingestion
docs = PyMuPDFLoader("doc.pdf").load()
chunks = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50).split_documents(docs)
db = FAISS.from_documents(chunks, OpenAIEmbeddings(model="text-embedding-3-small"))

# 2. Retrieval
query = "公司的年假政策是什么？"
retrieved = db.similarity_search(query, k=5)

# 3. Generation
context = "\n\n".join([d.page_content for d in retrieved])
prompt = f"""根据以下材料回答问题。如果材料中没有答案，明确说"找不到"，不要编造。

材料：
{context}

问题：{query}

回答："""
result = ChatOpenAI(model="gpt-4o-mini").invoke(prompt)
```

**面试追问：**
> Q: 摄取和检索用同一个 embedding 模型是为什么？
> A: 不同模型产生的向量空间不一致，cos similarity 失去意义。这是新手最常见错误。

---

### Q2: RAG vs Fine-tuning 的根本区别？怎么选？

**Answer:**

| 维度 | RAG | Fine-tuning |
|------|-----|-------------|
| 知识存储 | 外部 VectorDB | 模型权重 |
| 知识更新 | 实时，换库即可 | 需重新训练 |
| 成本 | 低 | 高（GPU + 时间） |
| 幻觉 | 少（外部约束） | 多（靠记忆） |
| 溯源 | 强（可引用来源） | 无 |
| 定制能力 | 弱（仅 prompt 控制） | 强（改变行为/风格） |

**决策框架：**
- 知识频繁更新 → RAG
- 需要溯源/可解释性 → RAG
- 改变模型行为/风格 → Fine-tuning
- 两者结合：Fine-tuning 让模型懂领域术语 + RAG 提供最新知识

**面试追问：**
> Q: 128K 长上下文还需要 RAG 吗？
> A: 需要。长上下文有"大海捞针"问题——信息被稀释，模型注意力下降。研究表明 Top-10 检索结果塞进短 context 比整本书塞进 128K 效果更好。

---

## 二、Chunking 策略

### Q3: 有哪些 Chunking 策略？各适合什么场景？

**Answer:**

| 策略 | 实现 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| **固定窗口** | 按字符/token 数切 | 简单可控 | 切断语义 | 通用 baseline |
| **重叠窗口** | 窗口 + overlap | 保留边界上下文 | 数据冗余 | **推荐起步** |
| **递归字符** | 按段落→句子优先级切 | 保留结构 | 长 chunk 不均 | 结构化文档 |
| **语义分割** | 按 embedding 距离切 | 真正语义边界 | 慢、贵 | 高质量 RAG |
| **Markdown 结构** | 按 heading/代码块切 | 完整语义块 | 依赖文档结构 | 技术文档 |

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=256, chunk_overlap=64,
    separators=["\n\n", "\n", "。", "！", "？", " ", ""],
)
chunks = splitter.split_text(long_text)
```

---

### Q4: Chunk Size 怎么选？有数据依据吗？

**Answer:**

| Chunk Size | Faithfulness | Answer Relevancy | 速度 |
|-----------|--------------|------------------|------|
| 128 | 0.78 | 0.72 | 最快 |
| **256** | **0.85** | **0.81** | **平衡（推荐起步）** |
| 512 | 0.86 | 0.83 | 中等 |
| 1024 | 0.83 | 0.79 | 慢 |

- 通用：`chunk_size=256, overlap=64` 起步
- 中文：适当加大到 320-384
- 代码库：用 AST parser 按函数/类切分
- 太短 → 信息碎片化；太长 → context 浪费

**面试追问：**
> Q: overlap 设太大有什么问题？
> A: Index 体积膨胀，检索返回大量重复内容，浪费 context window。经验值 10%-25%。

---

## 三、检索优化

### Q5: Hybrid Search 是什么？为什么向量 + BM25 结合更好？

**Answer:**

Hybrid Search = 向量检索（语义）+ BM25（关键词）的结合。

```
用户问题
    ├──→ 向量化 → VectorDB Top-50 → 结果集 A
    ├──→ 原文本 → BM25 Top-50 → 结果集 B
    └──→ RRF 融合 → Top-K → Reranker
```

**互补性：**

| 检索方式 | 强项 | 弱项 |
|----------|------|------|
| 向量检索 | 语义理解、同义词 | 专有名词、精确匹配 |
| BM25 | 精确关键词匹配 | 不理解同义词/语义 |

```python
from langchain.retrievers import EnsembleRetriever, BM25Retriever

vector_retriever = db.as_retriever(search_kwargs={"k": 10})
bm25_retriever = BM25Retriever.from_documents(chunks, k=10)

ensemble = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.5, 0.5],
)
results = ensemble.invoke(query)
```

**适用场景：** 专业术语多、向量化质量差、需要精确数字/ID 匹配。

---

### Q6: Reranker 为什么必要？Bi-Encoder vs Cross-Encoder？

**Answer:**

**为什么需要 Reranker：** 向量检索（Bi-Encoder）速度快但精度有限——它把 query 和 doc 分别编码，算 cos similarity。召回 Top-100 时很多"看起来像"但不相关的结果。

**两阶段架构：**
```
Query → Bi-Encoder（Top-100，快） → Cross-Encoder（Top-5，精确）
```

| 特性 | Bi-Encoder | Cross-Encoder |
|------|-----------|---------------|
| 编码方式 | 分别编码，算 cos similarity | 拼接后一起编码，输出分数 |
| 速度 | 快（预计算，ANN 检索） | 慢（每对都要推理） |
| 精度 | 较低 | 高 |
| 适用 | 大规模召回（百万级） | 精排（百级候选） |

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder('BAAI/bge-reranker-base')
pairs = [[query, doc.page_content] for doc in candidate_docs]
scores = reranker.predict(pairs)
ranked = sorted(zip(candidate_docs, scores), key=lambda x: x[1], reverse=True)[:5]
```

**面试追问：**
> Q: Cross-Encoder 不能直接用于大规模检索吗？
> A: 百万级 × Cross-Encoder 推理 = 不可行。必须先粗筛到百级。ColBERT 的 late interaction 是中间路线。

---

## 四、高级技术

### Q7: HyDE 原理？什么时候效果最好？

**Answer:**

**HyDE = Hypothetical Document Embeddings**：让 LLM 生成"假设答案"，用假设答案去检索真实文档。

```python
def hyde_retrieve(query, vectorstore, llm, k=5):
    hypothetical = llm.invoke(f"请生成一个详细回答以下问题的假设答案：{query}")
    return vectorstore.similarity_search(hypothetical.content, k=k)
```

**为什么有效：** 假设答案更长、包含更多关键词，产生更好的 embedding → 更好的召回。

**适用场景：** 问题表述模糊、缺少关键术语、知识库文档内容比问题描述更丰富。

---

### Q8: GraphRAG 是什么？和传统 RAG 有什么不同？

**Answer:**

**GraphRAG（Microsoft）** 用知识图谱增强检索，核心是多层级知识结构：

```
Level 0: 文档块（原始内容）
Level 1: 实体 + 关系（知识图谱）
Level 2: 社区摘要（每个主题簇的摘要）
Level 3: 全局摘要（整个知识库的高层摘要）
```

**传统 RAG vs GraphRAG：**

| 能力 | 传统 RAG | GraphRAG |
|------|----------|----------|
| 局部问题 | ✅ | ✅ |
| 全局性问题 | ❌（只能召回局部块） | ✅（层级摘要） |
| 多跳推理 | ❌ | ✅（关系链） |
| 构建复杂度 | 低 | 高（需建图谱） |

**适用场景：** 大型文档集合（企业 wiki、研究报告），需要多跳推理和全局理解。

---

### Q9: CRAG（Corrective RAG）是什么？

**Answer:**

**CRAG** = 检索后评估质量，不好就修正。

```
用户问题 → 检索 → [评估质量]
              ├─ 高质量 → 直接生成
              ├─ 低质量 → 重新检索（扩大范围）
              └─ 无结果 → 外部知识补充
```

**和 Self-RAG 的区别：**
- Self-RAG：生成过程中决定是否检索（per-token）
- CRAG：检索后评估质量，不信任时就纠正

---

### Q10: Self-RAG 是什么？

**Answer:**

**Self-RAG** 让模型在生成过程中自己判断是否需要检索、检索结果是否相关、以及是否要引用检索结果。

```
生成过程：... → [是否检索？] → 检索 → [结果相关？] → 是 → [引用？] → 继续生成
                           ↓ 否                    ↓ 否
                         不检索                   忽略结果
```

**核心贡献：** 不是每次生成都检索，而是按需检索，减少不必要的检索开销。

---

## 五、RAG 评估

### Q11: RAG 评估的三个维度？RAGAS 4 指标是什么？

**Answer:**

| 维度 | 指标 | 说明 |
|------|------|------|
| 检索质量 | Hit Rate, MRR, NDCG | 召回的文档是否相关 |
| 生成质量 | **RAGAS 4 指标** | 生成回答是否正确利用了检索结果 |
| 端到端 | 人工评估 / LLM-as-Judge | 整体回答质量 |

**RAGAS 4 大核心指标：**

| 指标 | 衡量什么 | 提升方法 |
|------|----------|----------|
| **Faithfulness** | 回答是否忠于检索到的上下文（有无幻觉） | 强化 prompt、reranker |
| **Answer Relevancy** | 回答与问题的相关性 | 更好的检索、更多 context |
| **Context Precision** | Top-K 检索结果的排序质量 | Hybrid Search、Reranker |
| **Context Recall** | 检索是否覆盖所有必要信息 | 更好的 chunking、HyDE、Multi-query |

```python
from ragas import evaluate
from ragas.metrics import Faithfulness, AnswerRelevancy, ContextPrecision, ContextRecall

result = evaluate(dataset, metrics=[
    Faithfulness(), AnswerRelevancy(), ContextPrecision(), ContextRecall(),
])
```

**校招目标：** Faithfulness > 0.85 + Answer Relevancy > 0.80。

---

### Q12: 如何计算 MRR 和 Hit Rate？

**Answer:**

```python
# Hit Rate：Top-K 中至少有一个相关文档的比例
def hit_rate(relevant_docs, retrieved_docs, k=5):
    return any(doc in relevant_docs for doc in retrieved_docs[:k])

# MRR：第一个相关文档排名的倒数均值
def mrr(relevant_docs, retrieved_docs, k=5):
    for i, doc in enumerate(retrieved_docs[:k]):
        if doc in relevant_docs:
            return 1 / (i + 1)
    return 0

# NDCG：考虑排序位置的 discounted gain
def ndcg(relevant_docs, retrieved_docs, k=5):
    dcg = sum(1 / math.log2(i + 2) for i, doc in enumerate(retrieved_docs[:k]) if doc in relevant_docs)
    idcg = sum(1 / math.log2(i + 2) for i in range(min(len(relevant_docs), k)))
    return dcg / idcg if idcg > 0 else 0
```

---

## 六、方案选型

### Q13: RAG vs Fine-tuning vs Long Context，怎么选？

**Answer:**

| 维度 | RAG | Fine-tuning | Long Context |
|------|-----|-------------|--------------|
| 知识容量 | 无限 | 受训练数据限制 | 受 context window 限制 |
| 知识更新 | 实时 | 需重新训练 | 不需要 |
| 成本 | 低 | 高 | 中等（按 token） |
| 延迟 | 有检索延迟（~100ms） | 纯推理 | 长 context 推理慢 |
| 幻觉 | 少 | 多 | 中等 |
| 溯源 | 强 | 无 | 无 |

**决策框架：**
```
知识会频繁更新？ ──→ RAG
需要改变模型行为？ ──→ Fine-tuning
需要全局理解长文档？ ──→ Long Context + RAG（互补）
```

> Q: 128K 长上下文还需要 RAG 吗？
> A: 需要。研究表明：Top-10 检索结果 + 短 context 比整本书塞进 128K 效果更好。

---

## 七、失败模式与排查

### Q14: RAG 常见失败模式？如何解决？

**Answer:**

| 失败模式 | 根因 | 解决方案 |
|----------|------|----------|
| **幻觉** | LLM 忽略约束 | 强化 prompt、Faithfulness 监控、后处理验证 |
| **检索不到** | Chunk 不当/Embedding 差 | 调 chunk size、换 embedding、Hybrid Search |
| **答非所问** | 检索排序差 | 加 Reranker、调 top-k |
| **答案不完整** | Top-K 太少 | 增加 k、Multi-query、HyDE |
| **Chunk 断裂** | 窗口切断语义 | 增大 overlap、语义切分 |
| **延迟过高** | 检索/LLM 慢 | Embedding 缓存、模型降级 |

```python
def diagnose(question, answer, contexts, llm):
    # 检查幻觉
    check = llm.invoke(f"回答中是否有信息不在上下文中？有则输出 HALLUCINATION: ...，否则输出 FAITHFUL\n回答：{answer}\n上下文：{contexts}")
    # 检查检索充分性
    coverage = llm.invoke(f"检索结果是否足以回答问题？不足则输出 INSUFFICIENT: ...\n问题：{question}\n检索：{contexts}")
    return check.content, coverage.content
```

---

### Q15: 生产 RAG 的关键工程问题？

**Answer:**

1. **索引更新**：增量更新（新文档只更新受影响部分）、版本管理、双写无停机切换
2. **多租户隔离**：每个租户独立 namespace，检索时过滤
3. **缓存**：Embedding 结果缓存 + Reranker 结果缓存
4. **监控**：Hit rate、Faithfulness（RAGAS 定期抽样）、P99 延迟
5. **A/B 测试**：对比不同 chunk_size / reranker / 检索策略

---

## 八、系统设计面试

### Q16: 设计题：设计一个企业知识库 RAG 系统

**Answer:**

**四步法答题框架：**

**第一步：需求澄清（必做！）**
- 数据规模？多久更新？
- 日活多少？QPS？
- 延迟要求？
- 多租户？

**第二步：高层架构**

```
摄取：文档 → Chunking → Embedding → VectorDB
检索：Query → Hybrid Search → Reranker → Top-K
生成：Context + Query → LLM → 回答（带来源）
```

**第三步：逐组件选型**

| 组件 | 推荐方案 | 理由 |
|------|----------|------|
| Embedding | text-embedding-3-small / BGE-M3 | 性价比高 |
| VectorDB | 小量 FAISS，大量 Pinecone/Milvus | 规模决定 |
| Chunking | chunk=256, overlap=64 | 通用 baseline |
| 检索 | Hybrid（向量 0.5 + BM25 0.5） | 召回率最大化 |
| Reranker | BGE-Reranker-Base | 开源最强 |
| LLM | GPT-4o-mini / Claude Haiku | 性价比 |

**第四步：评估指标**
- Faithfulness > 0.85（RAGAS）
- Answer Relevancy > 0.80
- P99 延迟 < 2s

---

### Q17: Faithfulness 只有 0.6，怎么系统性排查？

**Answer:**

```python
def diagnose_faithfulness(rag, test_cases):
    for case in test_cases:
        retrieved = rag.retrieve(case.question)
        # 1. 检索召回够吗？
        if not any(relevant(case.gt, doc) for doc in retrieved):
            print("RETRIEVAL_FAIL: 没召回到正确文档")
            continue
        # 2. Rerank 后正确文档在前吗？
        reranked = rag.rerank(retrieved, case.question)
        if not relevant(case.gt, reranked[0]):
            print("RERANK_FAIL")
        # 3. 生成有幻觉吗？
        answer = rag.generate(case.question, reranked)
        if hallucination(answer, retrieved):
            print("GENERATION_FAIL")
```

**排查顺序：** 检索召回 → Rerank 排序 → Prompt 约束 → 模型能力 → Chunk 质量

---

## 九、RAG vs Agent

### Q18: RAG 和 Agent（工具调用）的关系？

**Answer:**

RAG 和 Agent 互补而非互斥：

| 场景 | RAG | Agent |
|------|-----|-------|
| 知识库问答 | ✅ | ❌ |
| 操作外部系统 | ❌ | ✅ |
| 多步骤任务 | ❌ | ✅ |
| 需要溯源 | ✅ | ❌ |

**典型组合：**
```
用户："查 Q3 销售数据，写报告"
  ├─ Agent: 调用数据库查数据（Tool）
  ├─ RAG: 检索历史报告模板
  └─ 综合生成报告
```

---

## 十、速查与面试准备

### Q19: RAG 必背知识点清单

**高频必背（90%）：**
1. RAG 三段流程
2. Chunking 策略对比表
3. Hybrid Search 原理（BM25 + 向量互补）
4. Bi-Encoder vs Cross-Encoder
5. RAGAS 4 指标：Faithfulness / Answer Relevancy / Context Precision / Context Recall

**理解型（50%）：**
6. HyDE 原理
7. GraphRAG vs 传统 RAG
8. CRAG 自校正机制
9. RAG vs Fine-tuning 选型
10. 失败模式排查流程

**加分型（拉开差距）：**
11. ColBERT late interaction
12. RAG + Agent 协作模式
13. Context Compression 方法
14. 多租户隔离
15. 生产监控指标

---

### Q20: RAG 面试速查表

**技术选型：**

| 需求 | 推荐方案 |
|------|----------|
| 通用知识库 | Hybrid + BGE-Reranker + GPT-4o-mini |
| 术语密集领域 | 领域 Embedding + Hybrid + Fine-tune LLM |
| 大文档集合 | GraphRAG 或 Semantic Chunking |
| 高精度要求 | HyDE + Reranker + Compression |
| 多模态 | 图文双路检索 + LLaVA Captioning |

**推荐资源：**
- LangChain RAG 教程：https://python.langchain.com/docs/tutorials/rag/
- BGE-Reranker：https://huggingface.co/BAAI/bge-reranker-base
- RAGAS：https://github.com/explodinggradients/ragas
- GraphRAG 论文：https://arxiv.org/abs/2404.16130
- CRAG 论文：https://arxiv.org/abs/2401.15884

---

*最后更新：2026-07-01*
*来源：U4-RAG基础.md + U15-RAG进阶.md*
