# U15 · RAG 进阶

> **核心问题**：怎么让 RAG 从"能用"到"好用"？
> **预计学时**：6h
> **对应 Phase**：Phase 1

---

## 学习目标（Gate 检验）

1. 能解释 Query Expansion 和 HyDE 的原理及适用场景
2. 能实现 Reranking + Compression 完整流程
3. 能对比三种 RAG 评估指标的特点
4. 能分析 RAG vs Fine-tuning 的权衡并做出选择
5. 能设计一个生产级 RAG 系统架构

---

## 1. 高级检索策略

### Query Expansion（查询扩展）

用户问题往往短且模糊，直接检索效果差。Query Expansion 通过改写/扩充问题来提升检索质量。

**常用方法**：
- **Step-back**：抽象到更高层次的问题（如"怎么做"→"这个技术的基本原理是什么"）
- **Sub-query decomposition**：拆解成多个子问题并行检索，再合并结果
- **Query rewriting**：用 LLM 重写为更清晰的搜索 query

```python
# Step-back Example
original_query = "为什么 Transformer 比 CNN 效果好？"
step_back_query = "Transformer 和 CNN 的核心区别是什么？"

# 双重检索：既用原查询，也用后退查询
results = await asyncio.gather(
    vectorstore.search(original_query, k=5),
    vectorstore.search(step_back_query, k=5),
)
combined = deduplicate(flatten(results))
```

### HyDE（Hypothetical Document Embeddings）

让 LLM 生成一个"假设答案"，用这个假设答案去检索相关文档。假设答案包含幻觉但语义方向正确，能匹配到真实相关文档。

```python
def hyde_retrieve(query, vectorstore, llm, k=5):
    # 1. 生成假设答案
    hypothetical = llm.invoke(f"请生成一个回答以下问题的假设答案：{query}")
    
    # 2. 用假设答案检索
    results = vectorstore.similarity_search(hypothetical.content, k=k)
    return results
```

**适用场景**：用户问题表述模糊、缺少关键术语时效果明显。

### REFEED（Retrieval Feedback）

用 LLM 首轮生成的结果反过来增强检索。具体做法：挑出生成回答中引用的文档，重新检索相关内容补充 context。

---

## 2. Reranking 与 Compression

### 两阶段检索架构

```
Query → 向量检索（Top-100，快速） → Reranker（精确重排） → Top-10 → LLM
```

第一阶段用向量检索快速召回候选集，第二阶段用交叉编码器（Cross-Encoder）做精确相关性排序。

### Reranker 模型

| 模型 | 类型 | 特点 |
|------|------|------|
| BGE-Reranker | 交叉编码器 | 开源最强，中文支持好 |
| Cohere Rerank | API | 效果好，但收费 |
| BERT-base + pairwise training | 自训练 | 可针对领域微调 |

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder('BAAI/bge-reranker-base')

# 输入：[query, doc] 对
pairs = [[query, doc] for doc in candidate_docs]
scores = reranker.predict(pairs)

# 按分数排序
ranked = sorted(zip(candidate_docs, scores), key=lambda x: x[1], reverse=True)
```

### Context Compression（上下文压缩）

召回的文档往往很长，包含很多无关信息。Compression 把每个文档压缩成只保留与 query 相关的部分。

**方法一：LLM-based Compression**
```python
def compress_doc(query, doc, llm):
    prompt = f"""根据问题，从以下文档中提取最相关的部分。

问题：{query}

文档：{doc}

请输出压缩后的内容（保留与问题相关的核心信息，去除无关细节）：
"""
    return llm.invoke(prompt).content
```

**方法二：Recompile**
用更小的专门模型（如 Llama 3B）做压缩，平衡效果和速度。

**方法三：Filter-only**
不做压缩，只过滤掉与 query 不相关 sentences（用关键词 overlap 或轻量 embedding）。

---

## 3. 多模态 RAG

### 图片 + 文字 RAG

**方案 1：图片描述（Captioning）**
```
图片 → GPT-4V / LLaVA 描述 → 文字 → 标准 RAG
```

**方案 2：双路检索**
```
用户问题 → 文字检索 + 图片向量检索 → 合并结果 → LLM
```

### 表格 RAG

表格容易被切分破坏结构。处理方式：

1. **表格识别**：用 TableTransformers 定位表格
2. **HTML 表格保持结构**：转为 HTML 格式的 table，保留 cell 关系
3. **Serializer**：用专用工具（如 TableParser）转成 JSON 或 markdown

```python
# 示例：表格作为独立 chunk
table_content = """
| 模型 | 参数量 | 精度 |
|------|--------|------|
| GPT-4 | 1T | SOTA |
"""

# 把表格作为一个完整的 semantic chunk
chunk = Document(page_content=table_content, metadata={"type": "table"})
```

### 代码 RAG

代码有特殊结构（函数、类、import 关系）。普通文本切分会破坏代码语义。

- 用 AST（抽象语法树）解析，按函数/类级别切分
- 保留代码的 import 依赖关系作为 metadata
- 代码专属 Embedding 模型（如 CodeBGE）

---

## 4. RAG 评估

### 三大评估维度

| 维度 | 指标 | 说明 |
|------|------|------|
| **检索质量** | Hit Rate, MRR, NDCG | 检索到的文档是否相关 |
| **生成质量** | RAGAS, RAGScore | 生成回答是否正确利用了检索结果 |
| **端到端** | 人工评估 / LLM-as-Judge | 整体回答质量 |

### RAGAS（RAG Assessment Score）

核心指标：
- **Context Precision**：检索到的文档按相关性排序的精确度
- **Answer Faithfulness**：生成回答是否忠实于检索到的上下文（不幻觉）
- **Answer Relevance**：回答与问题的相关性

```python
# 用 LLM-as-Judge 评估 RAGAS
def evaluate_ragas(question, answer, contexts):
    prompt = f"""评估以下 RAG 系统的质量：

问题：{question}

检索到的上下文：{contexts}

生成的回答：{answer}

请评估：
1. 回答是否忠实于上下文（没有添加不存在的信息）？
2. 回答是否与问题相关？
3. 回答是否完整？

评分：1-5 分，1=完全不合格，5=优秀
"""
    # 用 LLM 评分
```

### 检索评估指标

```python
# MRR (Mean Reciprocal Rank)
def calculate_mrr(queries, relevant_docs, retrieved_docs, k=10):
    reciprocal_ranks = []
    for query, relevant, retrieved in zip(queries, relevant_docs, retrieved_docs):
        for i, doc in enumerate(retrieved[:k]):
            if doc in relevant:
                reciprocal_ranks.append(1 / (i + 1))
                break
    return sum(reciprocal_ranks) / len(reciprocal_ranks)

# Hit Rate
def calculate_hit_rate(queries, relevant_docs, retrieved_docs, k=10):
    hits = sum(1 for relevant, retrieved in zip(queries, relevant_docs) 
              if any(doc in relevant for doc in retrieved[:k]))
    return hits / len(queries)
```

---

## 5. RAG vs Fine-tuning

### 核心区别

| 维度 | RAG | Fine-tuning |
|------|-----|------------|
| **知识更新** | 实时（换知识库就行） | 需要重新训练 |
| **成本** | 低（无需训练） | 高（需要 GPU + 训练时间） |
| **幻觉** | 较少（有外部知识约束） | 较多（靠自己记忆） |
| **定制能力** | 弱（只能提示词控制） | 强（能改变模型行为） |
| **适用场景** | 知识库问答、事实性问答 | 风格迁移、领域术语、复杂推理 |

### 什么时候用 RAG

1. **知识会频繁更新**（产品文档、新闻）
2. **需要溯源和可解释性**（答案要带来源）
3. **多来源异构知识**（PDF + Web + 数据库）
4. **成本敏感**（不想付训练成本）

### 什么时候用 Fine-tuning

1. **风格/格式固定**（特定行业的报告格式）
2. **领域术语密集**（医疗、法律——需要模型\"学会\"术语）
3. **交互模式复杂**（多轮对话、特定角色扮演）
4. **延迟要求高**（不想每次都检索，RAG 有额外延迟）

### 两者结合（RAG + Fine-tuning）

```
知识库（RAG）+ 训练好的领域能力（Fine-tuning）
```

例如：医疗 RAG 系统，用医学数据微调的模型既懂医学术语，又能在知识库中检索最新指南。

---

## 6. 生产级 RAG 架构

### 完整架构图

```
                    ┌─────────────────────────────────┐
                    │         Query Processing          │
                    │  (Query Expansion / HyDE / ...)  │
                    └─────────────────┬─────────────────┘
                                      │
                    ┌─────────────────▼─────────────────┐
                    │         Vector Search (Top-100)     │
                    └─────────────────┬─────────────────┘
                                      │
                    ┌─────────────────▼─────────────────┐
                    │           Reranker (Top-10)         │
                    └─────────────────┬─────────────────┘
                                      │
                    ┌─────────────────▼─────────────────┐
                    │        Context Compression         │
                    └─────────────────┬─────────────────┘
                                      │
                    ┌─────────────────▼─────────────────┐
                    │              LLM Generation        │
                    └─────────────────────────────────┘
```

### 关键工程问题

**1. 索引更新**
- 增量更新：新文档来了只更新受影响的索引部分
- 版本管理：知识库变更需要 track，方便回滚
- 双写：新旧索引同时存在，切换时无停机

**2. 多租户隔离**
- 每个租户独立向量空间（或用 namespace 隔离）
- 检索时只查当前租户的数据

**3. 缓存**
- Embedding 结果缓存（相同文本的 embedding 不重复计算）
- Reranker 结果缓存（相同的 query-doc 对不重复计算）

**4. 监控**
- 检索命中率监控（多少 query 召回的文档完全不相关）
- 生成质量监控（RAGAS 定期抽样评估）
- 延迟监控（P99 检索延迟 < 500ms）

**5. A/B 测试**
- 对比不同 chunk_size 的效果
- 对比不同 reranker 的效果
- 对比不同 retrieval 策略的效果

---

## 完成 Gate 检验

1. **Q1**：Query Expansion 和 HyDE 的核心思想各是什么？什么时候用 HyDE 效果更好？
2. **Q2**：两阶段检索（向量检索 → Reranker）中，Reranker 用什么类型的模型？为什么不能用向量模型替代？
3. **Q3**：RAGAS 评估的三个核心指标是什么？各衡量什么？
4. **Q4**：什么场景适合 RAG，什么场景适合 Fine-tuning？两者可以怎么结合？

---

## 练习清单

- [ ] 实现 HyDE 检索流程，对比原 query 检索和 HyDE 检索的效果差异
- [ ] 用 BGE-Reranker 实现两阶段检索，计算 MRR 和 Hit Rate
- [ ] 实现一个简单的 Context Compression（基于 LLM）
- [ ] 设计一个多模态 RAG 流程（图片描述 + 文字检索）
- [ ] 调研生产 RAG 系统的监控指标，设计一个监控面板

---

## 关联资源

- [HyDE 论文](https://arxiv.org/abs/2212.10496)
- [RAGAS 评估框架](https://github.com/explodinggradients/ragas)
- [BGE-Reranker 模型](https://huggingface.co/BAAI/bge-reranker-base)
- [MTEB 检索评估榜单](https://huggingface.co/spaces/mteb/leaderboard)
- [LangChain RAG 教程（进阶）](https://python.langchain.com/docs/tutorials/rag/)

---

## 7. 与本仓库其他单元的关系

|关联单元|关系|
|-------|---|
|U2 Embedding 与 VectorDB|高级 RAG 技术依赖更深的 Embedding 理解|
|U4 RAG 基础|U4 是基础，U15 深入高级 RAG 技术|
|U16 Embedding 进阶|MTEB 等评测依赖 U16 的 Embedding 深度知识|
|U19 向量数据库深度对比|高级检索策略依赖 U19 的 VectorDB 选型知识|
|U22 Context Engineering|RAG 检索结果是 Context 的核心来源|

*最后更新：2026-05-27*