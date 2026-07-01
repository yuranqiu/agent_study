# U16 · Embedding 进阶

> **核心问题**：怎么让 Embedding 适配你的专属场景？
> **预计学时**：6h
> **对应 Phase**：Phase 1

---

## 学习目标（Gate 检验）

1. 能解释 Dense Embedding 和 Sparse Embedding 的区别及各自适用场景
2. 能说出 ColBERT 多向量检索的原理和优势
3. 能用 MTEB 基准测试评估一个 Embedding 模型
4. 能设计 Embedding Fine-tuning 的数据配比和训练策略
5. 能对比主流向量数据库的过滤检索能力

---

## 1. Dense vs Sparse Embedding

### 两种 Embedding 范式

| 维度 | Dense Embedding | Sparse Embedding |
|------|----------------|------------------|
| **向量形式** | 连续值向量（如 1536 维） | 稀疏向量（大部分为 0） |
| **代表模型** | OpenAI text-embedding-3, BGE-M3 | BM25, SPLADE |
| **特点** | 语义理解强 | 关键词精确匹配强 |
| **适用场景** | 语义相似、表述多样的查询 | 专业术语、专有名词多的场景 |

### Sparse Embedding：BM25

BM25 是信息检索经典算法，本质是**改进的 TF-IDF**：

- TF（词频）饱和：词出现太多次不再增加权重
- IDF 调整：惩罚在太多文档中出现的词

```python
from rank_bm25 import BM25Okapi

# 文本切分
tokenized_corpus = [doc.split() for doc in documents]

# 构建 BM25 索引
bm25 = BM25Okapi(tokenized_corpus)

# 检索
query = "transformer architecture"
scores = bm25.get_scores(query.split())
top_docs = bm25.get_top_n(query.split(), documents, n=5)
```

### Sparse Embedding：SPLADE

SPLADE 用 BERT 生成稀疏向量——对每个词学习一个权重，形成"软关键词"向量。兼顾语义和关键词匹配。

```python
# SPLADE 示例（实际用库如 bert-splade）
from transformers import AutoModelForSparseEncoding

model = AutoModelForSparseEncoding.from_pretrained("naver/splade-cased")
```

### Hybrid Search（混合检索）

**同时用 Dense + Sparse**，取两者结果的并集或用 Reciprocal Rank Fusion 融合。

```python
def hybrid_search(query, dense_db, sparse_index, documents, k=10, alpha=0.5):
    # Dense 检索
    dense_results = dense_db.similarity_search(query, k=k*2)
    dense_scores = {doc.id: score for doc, score in dense_results}
    
    # Sparse 检索 (BM25)
    tokenized_query = query.split()
    sparse_scores_raw = bm25.get_scores(tokenized_query)
    sparse_scores = {
        i: score for i, score in enumerate(sparse_scores_raw)
    }
    
    # Reciprocal Rank Fusion
    fused_scores = {}
    all_doc_ids = set(dense_scores.keys()) | set(sparse_scores.keys())
    
    for doc_id in all_doc_ids:
        d_rank = 1 / (k + list(dense_scores.keys()).index(doc_id)) if doc_id in dense_scores else 0
        s_rank = 1 / (k + list(sparse_scores.keys()).index(doc_id)) if doc_id in sparse_scores else 0
        fused_scores[doc_id] = alpha * d_rank + (1 - alpha) * s_rank
    
    # 按融合分数排序
    ranked = sorted(fused_scores.items(), key=lambda x: x[1], reverse=True)
    return ranked[:k]
```

---

## 2. ColBERT：多向量检索

### 核心思想

ColBERT（Contextualized Late Interaction over BERT）为每个 token 生成独立向量，检索时用**Max-Pooling**计算相似度。

### 与传统 Embedding 的区别

| | 标准 Embedding（ANCE/DPR） | ColBERT |
|--|----------------------------|---------|
| 向量数量 | 每个文档 1 个向量 | 每个 token 1 个向量 |
| 查询方式 | 单一向量 vs 单一向量 | 查询每个 token 与文档每个 token |
| 延迟 | 低（只需一次点积） | 高（需要二次交互） |

### late interaction 机制

```python
# ColBERT 检索流程
# 1. 文档编码：每个 token 一个向量
doc_vectors =bert_encode(doc)  # [seq_len, hidden_dim]

# 2. 查询编码：每个 token 一个向量
query_vectors = bert_encode(query)  # [query_len, hidden_dim]

# 3. 交互：查询每个 token 与文档所有 token 做点积，取最大值
scores = torch.matmul(query_vectors, doc_vectors.T)  # [query_len, doc_len]
max_scores = scores.max(dim=1)  # Max-Pooling over doc tokens
final_score = max_scores.sum()
```

### ColBERTv2

- 向量压缩（用 Product Quantization）
- 改进的 reranking 策略
- 支持长文档（传统 ColBERT 受限于 512 tokens）

### 适用场景

- **长文档检索**：标准 embedding 压缩成长向量会丢失信息，ColBERT 保留细粒度
- **短语匹配重要**：专有名词、术语匹配精确度要求高的场景

---

## 3. Cross-lingual Embedding（跨语言 Embedding）

### 任务类型

| 任务 | 说明 | 示例 |
|------|------|------|
| **CLSN**（跨语言句子相似度） | 不同语言的句子相似度 | "Hello" vs "你好" |
| **CLIR**（跨语言信息检索） | 用一种语言检索另一种语言的知识库 | 英文 query 检索中文文档 |
| **MT**（机器翻译） | 端到端翻译（Embedding 是中间表示） | - |

### 代表模型

| 模型 | 支持语言 | 特点 |
|------|---------|------|
| **LaBSE** | 100+ | Google 开源，专精 CLSN |
| **MUSE** | 50+ | Facebook，多语言词典学习 |
| **BGE-M3** | 100+ | 中英文最强，支持多语言 |

### 实际应用：跨语言 RAG

```python
# 用户用英文问，检索中文知识库
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 文档（中文）
chunks = ["人工智能是计算机科学的一个分支", "机器学习是人工智能的子领域"]

# 向量化（用同一个模型，自动支持多语言）
vectors = [embeddings.embed_query(chunk) for chunk in chunks]

# 用户用英文问（embedding 会自动对齐到同一个语义空间）
query = "What is artificial intelligence?"
query_vector = embeddings.embed_query(query)

# 检索
similarity = cosine_similarity(query_vector, vectors[0])
```

---

## 4. Embedding Fine-tuning

### 什么时候需要 Fine-tuning

- 领域术语特殊（医疗、法律术语）
- 现有模型在特定场景效果差
- 需要特定格式/风格的相似度判断

### 对比学习框架

核心思想：让正样本（相似对）距离近，负样本（不相似对）距离远。

```python
import torch
from torch import nn
from sentence_transformers import SentenceTransformer, InputExample, Loader

# 1. 准备数据
train_examples = [
    InputExample(texts=["如何预防感冒", "感冒的预防方法"], label=0.9),
    InputExample(texts=["如何预防感冒", "今天天气不错"], label=0.1),
]

# 2. 加载预训练模型
model = SentenceTransformer('BAAI/bge-m3')

# 3. 对比学习训练
model.fit(
    train_objectives=[(train_examples, None)],
    epochs=3,
    warmup_steps=10,
    learning_rate=2e-5,
)
```

### Hard Negative Mining

负样本质量决定模型效果。策略：

1. **In-batch negatives**：同一个 batch 内其他正样本对互为负样本
2. **BM25 负采样**：用 BM25 检索相似但不相关的文档作为 hard negatives
3. **Cross-encoder 过滤**：用更强的 Cross-encoder 筛选真正的负样本

### 数据配比建议

| 负样本类型 | 比例 | 作用 |
|-----------|------|------|
| 随机负样本 | 50% | 基础对比学习 |
| BM25 负样本 | 30% | 提升关键词匹配能力 |
| Hard negatives | 20% | 提升细粒度区分能力 |

---

## 5. Embedding 评估：MTEB

### MTEB 是什么

Massive Text Embedding Benchmark（MTEB）是 Embedding 模型的综合评估基准，包含 58 个数据集、8 个任务类型。

### 核心任务类型

| 任务 | 数据集示例 | 衡量什么 |
|------|-----------|---------|
| **Classification** | Amazon-Polarity, SST-2 | 文本分类能力 |
| **Clustering** | ArXiv, Bird-spectrum | 按主题聚类能力 |
| **Pair Classification** | SprintDuplicateQuestions | 成对文本相似度 |
| **Reranking** | AskUbuntu, Mind | 排序能力 |
| **Retrieval** |NQ, HotpotQA, FEVER | 信息检索能力 |
| **STS**（语义相似度） | STS-B, SICK-R | 语义相似度判断 |
| **Summarization** | GSB, Reddit-TIFU | 生成摘要相关性 |

### 用 MTEB 评测模型

```python
from mteb import MTEB
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('BAAI/bge-m3')

# 选择评估任务
evaluation = MTEB(task_tasks=[" retrieval/NQ"])

# 评估
results = evaluation.run(model, output_folder="results/bge-m3")
```

### 主流模型榜单（检索任务）

| 模型 | NQ | HotpotQA | FEVER | 平均 |
|------|-----|----------|-------|------|
| BGE-M3 | 63.4 | 69.0 | 82.1 | 71.5 |
| text-embedding-3-large | 61.2 | 67.3 | 80.5 | 69.7 |
| Voyage-3 | 64.1 | 70.2 | 83.4 | 72.6 |

---

## 6. Advanced VectorDB 特性

### 过滤检索（Filtered Search）

生产环境中需要"只检索某租户的数据"或"只检索某分类下的文档"。

```python
# Qdrant 过滤示例
from qdrant_client import QdrantClient

client = QdrantClient(url="localhost", port=6333)

results = client.search(
    collection_name="my_collection",
    query_vector=query_vector,
    query_filter={
        "must": [
            {"key": "category", "match": {"value": "tech"}},
            {"key": "tenant_id", "match": {"value": "user_123"}}
        ]
    },
    limit=10
)
```

### 分层索引（Hierarchical Indices）

数据量大时建立分层索引：第一层用小模型快速筛选，第二层用精准模型排序。

```
Top-100 (HNSW，粗召回) → Top-20 (HNSW + 过滤) → Top-5 (精确重排)
```

### 向量版本管理

**问题**：知识库更新后，旧文档的向量和知识库内容不一致。

**方案**：
1. **软删除**：给文档加 `deleted_at` 字段，检索时过滤
2. **向量更新**：文档变了重建向量，更新时同时更新向量库
3. **双向量**：新旧版本向量同时存在，切换时无缝过渡

### 主流 VectorDB 对比

| 数据库 | 索引类型 | 过滤支持 | 分布式 | 适用场景 |
|--------|---------|---------|-------|---------|
| **Qdrant** | HNSW | 原生支持 | 是 | 生产环境，多租户 |
| **Weaviate** | HNSW + IVF | GraphQL | 是 | 混合搜索（向量+结构化） |
| **Milvus** | HNSW/IVF/PQ | 表达式过滤 | 是 | 超大规模，GPU 加速 |
| **Pinecone** | 自研 | 原生支持 | 是（云） | 免运维，云服务 |
| **Chroma** | HNSW | 有限 | 否 | 本地实验，轻量 |

### 选型建议

- **小规模实验** → Chroma（零配置，快速上手）
- **生产多租户** → Qdrant（过滤能力强，Rust 实现性能好）
- **超大规模** → Milvus（分布式，GPU 加速）
- **免运维** → Pinecone（云服务，按量付费）

---

## 完成 Gate 检验

1. **Q1**：Dense Embedding 和 Sparse Embedding 各擅长什么？Hybrid Search 的融合策略是什么？
2. **Q2**：ColBERT 的 late interaction 机制为什么能保留更多细粒度信息？它和标准 single-vector Embedding 的根本区别是什么？
3. **Q3**：Embedding Fine-tuning 时，Hard Negative Mining 为什么重要？如何构造高质量的负样本？
4. **Q4**：MTEB 评估的 8 个任务类型中，Reranking 和 Retrieval 有什么区别？
5. **Q5**：生产环境选择向量数据库时，过滤检索能力为什么重要？各数据库的过滤实现有什么区别？

---

## 练习清单

- [ ] 实现 BM25 检索，对比与向量检索的效果差异
- [ ] 实现 Hybrid Search（Reciprocal Rank Fusion）
- [ ] 用 MTEB 对比 BGE-M3 和 OpenAI text-embedding-3-small 的检索效果
- [ ] 设计 Embedding Fine-tuning 数据集（至少 100 条，包含正负样本）
- [ ] 在 Qdrant 中实现带过滤的检索（多租户场景）

---

## 关联资源

- [MTEB 官方 GitHub](https://github.com/embeddings-benchmark/mteb)
- [ColBERT 论文](https://arxiv.org/abs/2004.12865)
- [SPLADE 论文](https://arxiv.org/abs/2109.10086)
- [BGE-M3 模型](https://huggingface.co/BAAI/bge-m3)
- [Qdrant 文档](https://qdrant.tech/documentation/)
- [Sentence Transformers](https://www.sbert.net/)

---

## 7. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U2 Embedding 与 VectorDB|U2 是基础，U16 深入 MTEB、微调、领域适配|
|U4 RAG 基础|Embedding 质量直接决定 RAG 的检索质量|
|U15 RAG 进阶|HyDE、GraphRAG 等高级 RAG 依赖更深的 Embedding 理解|
|U19 向量数据库深度对比|Embedding 选型影响 VectorDB 的存储和检索策略|
|U32 Fine-tuning|领域 Embedding 的微调依赖 U32 的 fine-tuning 知识|
*最后更新：2026-05-27*