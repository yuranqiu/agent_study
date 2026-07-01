# Interview QA · 高级 RAG 篇

> 来源：RAG 进阶概念 / 生产实践 / 面试高频题
> 覆盖：Query Expansion / HyDE / Reranking / Compression / 评估 / 生产架构

---

## 一、检索增强策略篇

### Q: 什么是 Query Expansion？有哪些方法？

**Answer:**

用户问题往往短且模糊，直接检索效果差。Query Expansion 通过改写/扩充问题来提升检索质量。

**常用方法：**

| 方法 | 描述 |
|------|------|
| **Step-back** | 抽象到更高层次的问题（如"怎么做"→"基本原理是什么"） |
| **Sub-query decomposition** | 拆解成多个子问题并行检索，再合并结果 |
| **Query rewriting** | 用 LLM 重写为更清晰的搜索 query |

```python
# Step-back Example
original_query = "为什么 Transformer 比 CNN 效果好？"
step_back_query = "Transformer 和 CNN 的核心区别是什么？"

# 双重检索
results = await asyncio.gather(
    vectorstore.search(original_query, k=5),
    vectorstore.search(step_back_query, k=5),
)
```

---

### Q: 什么是 HyDE？原理是什么？

**Answer:**

- **HyDE = Hypothetical Document Embeddings（假设性文档嵌入）**
- 让 LLM 生成一个"假设答案"，用这个假设答案去检索相关文档
- 假设答案包含幻觉但语义方向正确，能匹配到真实相关文档

```python
def hyde_retrieve(query, vectorstore, llm, k=5):
    # 1. 生成假设答案
    hypothetical = llm.invoke(f"请生成一个回答以下问题的假设答案：{query}")
    
    # 2. 用假设答案检索
    results = vectorstore.similarity_search(hypothetical.content, k=k)
    return results
```

**适用场景：** 用户问题表述模糊、缺少关键术语时效果明显。

---

### Q: 什么是 REFEED？

**Answer:**

用 LLM 首轮生成的结果反过来增强检索。具体做法：挑出生成回答中引用的文档，重新检索相关内容补充 context。

---

## 二、Reranking 篇

### Q: 两阶段检索架构是什么？

**Answer:**

```
Query → 向量检索（Top-100，快速） → Reranker（精确重排） → Top-10 → LLM
```

- 第一阶段用向量检索快速召回候选集
- 第二阶段用交叉编码器（Cross-Encoder）做精确相关性排序

---

### Q: Reranker 用什么类型的模型？为什么不能用向量模型替代？

**Answer:**

| 模型 | 类型 | 特点 |
|------|------|------|
| BGE-Reranker | 交叉编码器 | 开源最强，中文支持好 |
| Cohere Rerank | API | 效果好，但收费 |
| BERT-base + pairwise | 自训练 | 可针对领域微调 |

**为什么不能替代：**
- 向量模型（Bi-Encoder）：把 query 和 doc 分别编码，算 cos similarity，速度快但精度有限
- 交叉编码器（Cross-Encoder）：把 query+doc 一起输入，输出相关性分数，精度高但速度慢
- 两者互补：向量检索快但粗筛，Reranker 慢但精排

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder('BAAI/bge-reranker-base')

pairs = [[query, doc] for doc in candidate_docs]
scores = reranker.predict(pairs)

ranked = sorted(zip(candidate_docs, scores), key=lambda x: x[1], reverse=True)
```

---

## 三、Context Compression 篇

### Q: 为什么需要 Context Compression？

**Answer:**

召回的文档往往很长，包含很多无关信息。Compression 把每个文档压缩成只保留与 query 相关的部分。

**核心目标：** 减少无关信息，提高 LLM 的注意力集中度。

---

### Q: Context Compression 有哪些方法？

**Answer:**

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

## 四、评估篇

### Q: RAG 评估的三大维度是什么？

**Answer:**

| 维度 | 指标 | 说明 |
|------|------|------|
| **检索质量** | Hit Rate, MRR, NDCG | 检索到的文档是否相关 |
| **生成质量** | RAGAS, RAGScore | 生成回答是否正确利用了检索结果 |
| **端到端** | 人工评估 / LLM-as-Judge | 整体回答质量 |

---

### Q: RAGAS 的三个核心指标是什么？

**Answer:**

| 指标 | 衡量什么 |
|------|----------|
| **Context Precision** | 检索到的文档按相关性排序的精确度 |
| **Answer Faithfulness** | 生成回答是否忠实于检索到的上下文（不幻觉） |
| **Answer Relevance** | 回答与问题的相关性 |

---

### Q: 如何计算 MRR 和 Hit Rate？

**Answer:**

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

## 五、RAG vs Fine-tuning 篇

### Q: RAG 和 Fine-tuning 的核心区别？

**Answer:**

| 维度 | RAG | Fine-tuning |
|------|-----|------------|
| **知识更新** | 实时（换知识库就行） | 需要重新训练 |
| **成本** | 低（无需训练） | 高（需要 GPU + 训练时间） |
| **幻觉** | 较少（有外部知识约束） | 较多（靠自己记忆） |
| **定制能力** | 弱（只能提示词控制） | 强（能改变模型行为） |
| **适用场景** | 知识库问答、事实性问答 | 风格迁移、领域术语、复杂推理 |

---

### Q: 什么时候用 RAG，什么时候用 Fine-tuning？

**Answer:**

**用 RAG：**
1. 知识会频繁更新（产品文档、新闻）
2. 需要溯源和可解释性（答案要带来源）
3. 多来源异构知识（PDF + Web + 数据库）
4. 成本敏感（不想付训练成本）

**用 Fine-tuning：**
1. 风格/格式固定（特定行业的报告格式）
2. 领域术语密集（医疗、法律——需要模型"学会"术语）
3. 交互模式复杂（多轮对话、特定角色扮演）
4. 延迟要求高（不想每次都检索）

---

### Q: RAG + Fine-tuning 如何结合？

**Answer:**

```
知识库（RAG）+ 训练好的领域能力（Fine-tuning）
```

例如：医疗 RAG 系统，用医学数据微调的模型既懂医学术语，又能在知识库中检索最新指南。

---

## 六、生产架构篇

### Q: 生产级 RAG 的完整架构？

**Answer:**

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

---

### Q: 生产 RAG 的关键工程问题？

**Answer:**

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

---

### Q: 多模态 RAG 怎么处理？

**Answer:**

**图片 + 文字 RAG：**

| 方案 | 流程 |
|------|------|
| Captioning | 图片 → GPT-4V/LLaVA 描述 → 文字 → 标准 RAG |
| 双路检索 | 用户问题 → 文字检索 + 图片向量检索 → 合并结果 → LLM |

**表格 RAG：**
1. 表格识别：用 TableTransformers 定位表格
2. HTML 表格保持结构：转为 HTML 格式的 table，保留 cell 关系
3. Serializer：用专用工具转成 JSON 或 markdown

**代码 RAG：**
- 用 AST（抽象语法树）解析，按函数/类级别切分
- 保留代码的 import 依赖关系作为 metadata
- 代码专属 Embedding 模型（如 CodeBGE）

---

*最后更新：2026-05-27*
