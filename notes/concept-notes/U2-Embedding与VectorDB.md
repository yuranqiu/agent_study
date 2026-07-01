# U2 · Embedding 与 VectorDB

> **核心问题**：怎么让模型"找到"相关知识？
> **预计学时**：5h
> **对应 Phase**：Phase 1

---

## 学习目标（Gate 检验）

1. 能解释 Embedding 是什么（不只是"转成向量"）
2. 能用手写或代码实现余弦相似度计算
3. 能说出 HNSW 和 IVF 的适用场景差异
4. 能在本地建一个向量库并实现相似检索

---

## 1. Embedding：从词到向量

### 直观理解

Embedding 是把文本（词、句、段）映射到一组数字（向量）的过程。语义相似的文本在向量空间中距离近。

类比：把"猫"和"狗"都映射到靠近的位置（都是动物），"汽车"放在另一个区域（交通工具）。

### 发展历程

| 模型 | 年份 | 输入 | 特点 |
|------|------|------|------|
| Word2Vec | 2013 | 单词 | 词→固定向量，一词多义无法区分 |
| BERT | 2018 | 句子/词段 | 上下文相关，同一个词不同语境向量不同 |
| Sentence-BERT (SBERT) | 2019 | 句子 | 专门优化句子级相似度，比 BERT 快 |
| OpenAI Embedding API | 2023 | 句子/段落 | 调用即用，最常用于 RAG |
| BGE / M3 | 2024 | 多语言 | 开源最强之一，支持中文 |

### 本质（面试能说出来）

> Embedding 把文本转成向量，使得语义相近的内容在向量空间中距离近。相似度检索就是在这个空间里找最近的邻居。

### OpenAI Embedding 用法

```python
import openai

response = openai.embeddings.create(
    model="text-embedding-3-small",  # 或 "text-embedding-3-large"
    input="北京是中国的首都",
)
vector = response.data[0].embedding
print(f"维度: {len(vector)}")  # text-embedding-3-small = 1536 维
```

---

## 2. 余弦相似度与向量距离

### 余弦相似度（最常用）

$$\text{cosine\_similarity}(A, B) = \frac{A \cdot B}{\|A\| \|B\|}$$

值的范围是 [-1, 1]：
- **1**：方向完全相同（最相似）
- **0**：正交，无相关性
- **-1**：方向完全相反

### 点积（Dot Product）

直接用原始向量点积作为相似度衡量。不需要归一化，但结果范围不固定。

在向量做了 L2 归一化的情况下，点积 = 余弦相似度。

### 代码实现

```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a = np.array(a)
    b = np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# 测试
a = [1.0, 2.0, 3.0]
b = [1.0, 2.0, 3.0]
c = [-1.0, -2.0, -3.0]

print(cosine_similarity(a, b))  # 1.0（完全相同）
print(cosine_similarity(a, c))  # -1.0（完全相反）
```

### 实战注意

OpenAI 返回的 embedding 是 L2 归一化的，所以直接用点积就等于余弦相似度，不需要再除以范数。

---

## 3. ANNS：近似最近邻搜索

### 为什么需要近似？

精确的最近邻搜索（暴力遍历所有向量）时间复杂度是 O(N)，当向量数量到百万级时太慢。ANNS 通过牺牲少量精度，换取指数级提速。

### HNSW（Hierarchical Navigable Small World）

- **原理**：多层图结构，从上往下逐层搜索（类似高速公路网络）
- **优点**：查询快（O(log N)），不需要GPU
- **缺点**：构建慢，占内存
- **适用场景**：数据量不大（百万级），需要快速部署

### IVF（Inverted File Index）

- **原理**：先对向量做聚类，搜索时只扫描目标聚类内的向量
- **优点**：可解释性强，可控召回率
- **缺点**：需要额外调参（聚类数）
- **适用场景**：需要精确控制召回率

### 其他算法

| 算法 | 特点 |
|------|------|
| FAISS (Facebook) | 支持 HNSW/IVF/PQ，常用于生产 |
| SPTAG (Microsoft) | 支持 HNSW/ANNOY，中文优化 |
| Qdrant | 向量数据库，支持过滤检索 |
| Chroma | 轻量，适合本地和实验 |
| Milvus | 分布式，支持 GPU 加速 |

### 面试能说的话

> "HNSW 是一个多层图，层数越高跳数越少，搜索时从上往下走，能在 O(log N) 时间内找到近似的最近邻。IVF 是倒排索引，先聚类再扫描类内向量，适合需要精确控制召回率的场景。"

---

## 4. Embedding 模型选型

### OpenAI API（最省事）

| 模型 | 维度 | 价格 | 适用 |
|------|------|------|------|
| text-embedding-3-small | 1536 | 便宜 | 日常 RAG |
| text-embedding-3-large | 3072 | 贵一倍 | 高精度需求 |

### 开源中文模型

| 模型 | 维度 | 特点 |
|------|------|------|
| BGE-M3 | 1024 | 性能最强，支持多语言 |
| M3E | 768 | 中文优化，轻量 |
| text2vec-large | 1024 | 早期中文模型 |

### 选模型的原则

1. **精度优先**：先用 OpenAI embedding（或 BGE-M3），有质量问题再换
2. **维度决定存储**：维度越高存储越大（text-embedding-3-large 的 3072 维比 1536 大一倍）
3. **本地 vs API**：有高隐私需求用本地开源模型（如 BGE-M3），否则直接用 API

---

## 5. 实战：用 FAISS 建向量库

```python
# 安装：pip install faiss-cpu（或 faiss-gpu）

import faiss
import numpy as np

# 1. 准备数据
documents = [
    "北京是中国的首都",
    "上海是中国最大的城市",
    "成都被誉为天府之国",
    "杭州有西湖风景区",
    "广州是广东省的省会",
]

# 2. 模拟向量化（实际用 OpenAI Embedding API）
dimension = 4  # 简化维度
vectors = np.random.rand(len(documents), dimension).astype('float32')

# 3. 构建 FAISS 索引
index = faiss.IndexFlatL2(dimension)  # L2 距离
index.add(vectors)

# 4. 查询
query = np.random.rand(1, dimension).astype('float32')
k = 3  # 返回最近 3 个
distances, ids = index.search(query, k)

print("最近邻文档:")
for i, idx in enumerate(ids[0]):
    print(f"  {i+1}. {documents[idx]} (距离: {distances[0][i]:.4f})")
```

真实流程中，把 `vectors` 换成 OpenAI Embedding API 的返回结果即可。

---

## 6. 完整 RAG Embedding 流程

```
文档 → 切块（chunking）→ 向量化 → 存入 VectorDB
                                        ↓
查询 → 向量化 → 检索（相似度 Top-K）→ 拼接 context → LLM 生成
```

### Chunking 策略对比

| 策略 | 优点 | 缺点 |
|------|------|------|
| 固定窗口（512 tokens）| 简单粗暴 | 可能切断句子 |
| 重叠切分（128 tokens overlap）| 保留上下文 | 数据冗余 |
| 语义分割（按句子/段落）| 保证语义完整 | 实现复杂 |

### 实战：LangChain 实现

```python
# 安装：pip install langchain-openai langchain-community faiss-cpu

from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

# 1. 加载文档
loader = TextLoader("your_document.txt")
docs = loader.load()

# 2. 切分
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# 3. 向量化 + 建库
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
db = FAISS.from_documents(chunks, embeddings)

# 4. 检索
query = "北京是哪个国家的首都？"
results = db.similarity_search(query, k=3)
for r in results:
    print(r.page_content)
```

---

## 完成 Gate 检验

1. **Q1**：Embedding 本质是什么？为什么语义相近的文本向量距离近？
   - 提示：训练时让语义相近的文本向量在空间中靠近，形成一种"语义坐标"。

2. **Q2**：余弦相似度和点积什么时候等价？
   - 提示：向量做了 L2 归一化之后。

3. **Q3**：HNSW 和 IVF 的核心区别是什么？各适合什么场景？
   - 提示：HNSW 是图索引，适合快读；IVF 是倒排索引，适合精确控制召回率。

4. **Q4**：用自己的话描述 Embedding 模型选型的主要考虑因素。
   - 提示：精度 vs 成本 vs 隐私，开源 vs API，维度与存储。

---

## 练习清单

- [ ] 用 `tiktoken` 或 OpenAI SDK 算一个中文段落的 token 数
- [ ] 用 FAISS 建一个本地向量库，存入 10 条文档，实现 Top-3 相似检索
- [ ] 对比固定窗口（chunk_size=256）和重叠切分（overlap=50）的检索质量差异

---

## 关联资源

- [OpenAI Embeddings 文档](https://platform.openai.com/docs/guides/embeddings)
- [FAISS GitHub](https://github.com/facebookresearch/faiss)
- [HNSW 论文](https://arxiv.org/abs/1608.07834)（Introduction 看 10 分钟即可）
- [LangChain RAG 教程](https://python.langchain.com/docs/tutorials/rag/)
- [BGE-M3 模型](https://huggingface.co/BAAI/bge-m3)

---


## 7. 与本仓库其他单元的关系

|关联单元|关系|
|---|---|
|U1 LLM 基础|Embedding 模型本身是 LLM 的一种，U1 的 Transformer 知识是基础|
|U4 RAG 基础|Embedding 是 RAG 的核心组件，U2 的技术在 U4 中直接使用|
|U15 RAG 进阶|高级 RAG 技术（HyDE、GraphRAG）依赖更深的 Embedding 理解|
|U16 Embedding 进阶|U2 是基础，U16 深入 MTEB、微调、领域适配|
|U19 向量数据库深度对比|U2 讲原理，U19 对比具体产品的选型|

## 8. Embedding 进阶概念（面试加分）
### 8.1 对比学习（Contrastive Learning）

Embedding 模型的训练核心是对比学习：
- **正样本对**：语义相近的文本（如 '猫' 和 '狗'）→ 向量距离近
- **负样本对**：语义不相关的文本（如 '猫' 和 '汽车'）→ 向量距离远
- **损失函数**：InfoNCE / NT-Xent，最大化正样本相似度、最小化负样本相似度

```python
# 简化的对比损失
import torch
import torch.nn.functional as F

def contrastive_loss(emb_pos, emb_neg, temperature=0.07):
    # emb_pos: [batch, dim] 正样本对
    # emb_neg: [batch, dim] 负样本对
    sim_pos = F.cosine_similarity(emb_pos[0], emb_pos[1])
    sim_neg = F.cosine_similarity(emb_pos[0], emb_neg[1])
    logits = torch.stack([sim_pos, sim_neg]) / temperature
    labels = torch.zeros(1, dtype=torch.long)
    return F.cross_entropy(logits.unsqueeze(0), labels)
```

### 8.2 Embedding Fine-tuning 实战

|场景|做法|
|---|---|
|通用领域|直接用 OpenAI / BGE-M3|
|专业领域（医疗/法律）|用领域数据微调 embedding 模型|
|多语言混合|选支持中文的多语言模型（BGE-M3 / M3E）|

**微调数据量**：至少 1000+ 句对，每对标注相似度（1=同义, 0=不同义）

### 8.3 面试能说的话

> 'Embedding 模型的训练用对比学习——正样本对拉近、负样本对推远。选模型时，优先看 MTEB 排行榜上的中文表现，BGE-M3 在中文上目前最强。生产环境先试用 OpenAI embedding，有问题再换开源。'
*最后更新：2026-05-26*