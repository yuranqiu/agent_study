# Interview QA · Embedding 篇

> 来源：U2 Embedding 与 VectorDB / U16 Embedding 进阶 / 面试高频题
> 覆盖：Embedding 本质 / 相似度 / 索引算法 / API 用法 / 模型选型 / 对比学习 / Fine-tuning / MTEB / ColBERT / Cross-lingual

---

## 一、Embedding 基础概念篇

### Q: Embedding 的本质是什么？为什么语义相近的文本向量距离近？

**Answer:**

Embedding 是把文本（词、句、段）映射到一组数字向量空间的过程。

```
语义坐标
  ┌─────────────────────────────────┐
  │  动物 ──── 猫     狗            │
  │  交通工具 ──── 汽车  火车       │
  │  城市 ──── 北京   上海          │
  └─────────────────────────────────┘
```

**核心原理（面试必须说出来）：**
训练时让语义相近的文本（正样本对）在向量空间中距离近，语义不相关的文本（负样本对）距离远。用对比学习（Contrastive Learning）训练，最终形成一个"语义坐标"——坐标上位置近 = 语义相近。

**发展脉络：**

| 模型 | 年份 | 特点 |
|------|------|------|
| Word2Vec | 2013 | 一词一向量，一词多义无法区分 |
| BERT | 2018 | 上下文相关，同一词不同语境不同向量 |
| SBERT | 2019 | 专门优化句子相似度，速度比 BERT 快 |
| OpenAI Embedding | 2023 | 调用即用，最常用于 RAG |
| BGE / M3 | 2024 | 中文最强，支持多语言 |

**面试追问：**
- 追问：Embedding 模型怎么训练出"语义理解"？
  - 答：通过海量文本的对比学习——正样本对拉近、负样本对推远。模型看到语义相似的文本对（如"猫"和"狗"）时，loss 让它们的向量靠近；看到不相关的文本对（如"猫"和"汽车"）时，loss 让它们的向量远离。

---

### Q: 怎么验证一个 Embedding 模型的效果？

**Answer:**

**方法一：手工测试（快速验证）**
```python
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity

model = SentenceTransformer('BAAI/bge-m3')

# 测试语义相似
pairs = [
    ("如何预防感冒", "感冒的预防方法"),
    ("如何预防感冒", "今天天气不错"),
    ("人工智能", "机器学习"),
    ("人工智能", "烹饪技巧"),
]

for a, b in pairs:
    emb_a = model.encode([a])
    emb_b = model.encode([b])
    sim = cosine_similarity(emb_a, emb_b)[0][0]
    print(f'"{a}" vs "{b}" → {sim:.4f}')
```

**方法二：MTEB 基准测试（全面评估）**
```python
from mteb import MTEB
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('BAAI/bge-m3')
evaluation = MTEB(task_tasks=["retrieval/NQ"])
results = evaluation.run(model, output_folder="results/")
```

**方法三：业务场景验证（最可靠）**
在真实 RAG 流程中对比 embedding 更换后的检索命中率变化。

---

## 二、相似度计算篇

### Q: 余弦相似度和点积有什么区别？各用在什么场景？

**Answer:**

**公式对比：**

| 指标 | 公式 | 范围 | 说明 |
|------|------|------|------|
| 余弦相似度 | $\frac{A \cdot B}{\|A\| \|B\|}$ | [-1, 1] | 考虑方向，归一化 |
| 点积 | $A \cdot B$ | 无固定范围 | 不考虑长度 |

**什么时候等价？**
向量做了 L2 归一化后：$\text{dot}(A, B) = \text{cosine}(A, B)$

**代码实现：**
```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a = np.array(a)
    b = np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# 测试
print(cosine_similarity([1, 2, 3], [1, 2, 3]))     # 1.0
print(cosine_similarity([1, 2, 3], [-1, -2, -3]))  # -1.0
print(cosine_similarity([1, 0], [0, 1]))           # 0.0（正交）
```

**面试必须知道的点：**
OpenAI 返回的 embedding 是 L2 归一化的，所以直接点积就等于余弦相似度，不需要再除以范数。

**面试追问：**
- 追问：为什么 Embedding 系统通常用余弦相似度，而不用欧几里得距离？
  - 答：余弦只关心方向（语义），不关心长度。欧几里得距离受向量长度影响——一个文档长文本可能向量更长，但语义方向相近。

---

### Q: 欧几里得距离和余弦相似度各在什么场景下用？

**Answer:**

| 场景 | 选择 | 原因 |
|------|------|------|
| 语义相似度判断 | 余弦相似度 | 方向代表语义 |
| 向量空间聚类 | 欧几里得距离 | 关心实际距离 |
| 已 L2 归一化的向量 | 点积 | 等价于余弦，计算快 |
| 向量幅度有意义的场景 | 点积 | 幅度本身携带信息 |

**FAISS 中的选择：**
```python
import faiss
import numpy as np

dimension = 1536
vectors = np.random.rand(100, dimension).astype('float32')

# L2 距离索引（相当于欧几里得距离）
index_l2 = faiss.IndexFlatL2(dimension)
index_l2.add(vectors)

# 内积索引（相当于余弦相似度，需向量 L2 归一化）
vectors_normalized = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)
index_ip = faiss.IndexFlatIP(dimension)
index_ip.add(vectors_normalized)
```

---

## 三、索引算法篇

### Q: HNSW 和 IVF 的核心区别是什么？各适合什么场景？

**Answer:**

**核心对比：**

| 维度 | HNSW | IVF |
|------|------|-----|
| **原理** | 多层图结构（类似高速公路） | 先聚类，搜索时扫描类内 |
| **时间复杂度** | O(log N) | O(N/k)（k=聚类数） |
| **构建速度** | 慢 | 快 |
| **查询速度** | 极快 | 快（但依赖聚类质量） |
| **内存消耗** | 高 | 中等 |
| **召回率控制** | 不直观 | 可控（调聚类数） |
| **GPU 支持** | 有限 | 好 |

**面试能说的话：**
"HNSW 是图索引，层数越高跳数越少——搜索时从上往下走，能在 O(log N) 时间内找到近似最近邻。IVF 是倒排索引，先对向量做聚类，搜索时只扫描目标聚类内的向量。"

**实际选型：**

| 数据规模 | 推荐 | 原因 |
|----------|------|------|
| < 10 万 | HNSW | 快，易部署 |
| 10 万 - 100 万 | HNSW | 速度优势明显 |
| 100 万+ | IVF + PQ | 内存效率高 |
| 需要精确召回率控制 | IVF | 调聚类数即可 |
| 生产多租户场景 | Qdrant/Milvus | 原生过滤支持 |

**面试追问：**
- 追问：HNSW 的 M 和 efConstruction 参数怎么调？
  - 答：M 控制图中每个节点的连接数（越大越准但越慢）；efConstruction 控制构建时的搜索宽度（越大构建越准但越慢）。默认值通常够用，不用手动调。

---

### Q: 近似最近邻（ANNS）和精确最近邻有什么区别？为什么 RAG 中常用 ANNS？

**Answer:**

**精确最近邻（暴力遍历）：**
```python
# O(N) 复杂度，百万级数据时太慢
distances = []
for i, vec in enumerate(vectors):
    dist = np.linalg.norm(query - vec)
    distances.append((dist, i))
distances.sort()
top_k = distances[:k]
```

**ANNS（近似）：**
通过牺牲少量精度，换取指数级提速。

| 指标 | 精确 | ANNS |
|------|------|------|
| 召回率 | 100% | 95-99% |
| 百万级查询延迟 | 秒级 | 毫秒级 |
| 内存 | 中等 | 可能更高 |

**为什么 RAG 中用 ANNS：**
- 检索召回率 95% 已经够用（Reranker 阶段再精排）
- 延迟是硬指标（P99 < 500ms）
- 生产环境通常分层：向量检索粗召回 → Reranker 精排

---

### Q: FAISS 产品级代码怎么写？

**Answer:**

```python
import faiss
import numpy as np
from typing import List, Tuple

class FAISSVectorDB:
    def __init__(self, dimension: int, index_type: str = "hnsw"):
        self.dimension = dimension
        
        if index_type == "hnsw":
            # HNSW 索引
            self.index = faiss.IndexHNSWFlat(dimension, 32)
            self.index.hnsw.efConstruction = 200
            self.index.hnsw.efSearch = 100
        elif index_type == "ivf":
            # IVF 索引（需要先训练）
            quantizer = faiss.IndexFlatL2(dimension)
            self.index = faiss.IndexIVFFlat(quantizer, dimension, 100)
        else:
            self.index = faiss.IndexFlatL2(dimension)
    
    def add(self, vectors: np.ndarray):
        """批量添加向量"""
        if isinstance(vectors, list):
            vectors = np.array(vectors, dtype='float32')
        self.index.add(vectors.astype('float32'))
    
    def search(self, query: np.ndarray, k: int = 5) -> Tuple[np.ndarray, np.ndarray]:
        """检索 Top-K"""
        query = np.array([query], dtype='float32')
        distances, ids = self.index.search(query, k)
        return distances[0], ids[0]
    
    def save(self, path: str):
        faiss.write_index(self.index, path)
    
    def load(self, path: str):
        self.index = faiss.read_index(path)

# 使用示例
db = FAISSVectorDB(dimension=1536, index_type="hnsw")
db.add(vectors)  # vectors: [n_docs, 1536]
distances, ids = db.search(query_vector, k=5)
```

---

## 四、OpenAI Embedding API 篇

### Q: OpenAI Embedding API 怎么用？有哪些注意事项？

**Answer:**

**基础调用：**
```python
import openai

response = openai.embeddings.create(
    model="text-embedding-3-small",  # 或 "text-embedding-3-large"
    input=["北京是中国的首都", "上海是中国最大的城市"],
)
vectors = [item.embedding for item in response.data]
print(f"维度: {len(vectors[0])}")  # 1536 或 3072
```

**模型对比：**

| 模型 | 维度 | 价格（相对） | 适用场景 |
|------|------|-------------|----------|
| text-embedding-3-small | 1536 | 1x | 日常 RAG，性价比最高 |
| text-embedding-3-large | 3072 | 4x | 高精度需求，长文本 |

**重要注意事项（面试必说）：**
1. **L2 归一化**：OpenAI 返回的向量已经 L2 归一化，直接用点积 = 余弦相似度
2. **Batch 处理**：单次请求最多 2048 个输入，用 batch 处理更省 token
3. **Token 限制**：每个输入文本最长 8191 tokens
4. **延迟**：首次请求可能 100-300ms，后续并发请求可更低

```python
# Batch 处理（推荐）
def batch_embed(texts: List[str], batch_size: int = 100):
    all_vectors = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        response = openai.embeddings.create(
            model="text-embedding-3-small",
            input=batch,
        )
        all_vectors.extend([item.embedding for item in response.data])
    return all_vectors
```

**面试追问：**
- 追问：text-embedding-3-large 比 small 贵 4 倍，什么时候值得用？
  - 答：专业领域（医疗、法律）对语义区分度要求极高，或者 chunk 很长（需要高维压缩语义）。

---

### Q: 用 OpenAI Embedding + LangChain 怎么实现完整 RAG？

**Answer:**

```python
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

# 1. 加载文档
loader = TextLoader("your_document.txt")
docs = loader.load()

# 2. 切分
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # 每个 chunk 500 tokens
    chunk_overlap=50,    # 重叠 50 tokens
)
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

**Chunking 策略对比：**

| 策略 | 优点 | 缺点 |
|------|------|------|
| 固定窗口 | 简单粗暴 | 可能切断句子 |
| 重叠切分 | 保留上下文 | 数据冗余 |
| 语义分割 | 保证语义完整 | 实现复杂 |

---

## 五、模型选型篇

### Q: 中文场景选什么 Embedding 模型？BGE-M3 / M3E / text2vec 怎么选？

**Answer:**

**主流中文模型对比：**

| 模型 | 维度 | 参数 | 特点 | 适用场景 |
|------|------|------|------|----------|
| BGE-M3 | 1024 | 568M | 最强，多语言 | 专业场景，高质量需求 |
| M3E | 768 | 23M | 轻量，中文优化 | 资源有限，本地部署 |
| text2vec-large | 1024 | - | 早期开源 | 仅维护旧项目 |

**选型决策树：**

```
有 API 能力？
├── 是 → OpenAI text-embedding-3-small（最省事）
└── 否 → 需要本地部署？
    ├── 是 → GPU 够？
    │   ├── 是 → BGE-M3（质量最好）
    │   └── 否 → M3E（轻量，可 CPU 跑）
    └── 否 → 云端部署？
        ├── 是 → BGE-M3 部署在云 GPU
        └── 高隐私要求 → M3E（轻量，本地跑）
```

**面试能说的话：**
"优先看 MTEB 排行榜上的中文表现。BGE-M3 在中文上目前最强，但需要 GPU。M3E 轻量可以 CPU 跑，适合资源受限场景。OpenAI embedding 最省事，有问题再换开源。"

---

### Q: MTEB 排行榜怎么看？哪些指标重要？

**Answer:**

**MTEB（Massive Text Embedding Benchmark）**
- 包含 58 个数据集、8 个任务类型
- 覆盖：分类、聚类、成对分类、重排、检索、语义相似度、摘要、QA

**核心任务与指标：**

| 任务 | 衡量什么 | 数据示例 |
|------|----------|----------|
| Retrieval | 信息检索质量 | NQ, HotpotQA, FEVER |
| STS | 语义相似度判断 | STS-B, SICK-R |
| Pair Classification | 成对文本相似度 | SprintDuplicateQuestions |
| Reranking | 排序质量 | AskUbuntu, Mind |
| Classification | 文本分类 | Amazon-Polarity, SST-2 |

**主流模型检索任务表现：**

| 模型 | NQ | HotpotQA | FEVER | 平均 |
|------|-----|----------|-------|------|
| BGE-M3 | 63.4 | 69.0 | 82.1 | 71.5 |
| text-embedding-3-large | 61.2 | 67.3 | 80.5 | 69.7 |
| Voyage-3 | 64.1 | 70.2 | 83.4 | 72.6 |

**看榜单的方法：**
1. 先看整体平均分（综合评价）
2. 再看自己场景对应的任务（如 RAG 系统重点看 Retrieval）
3. 中文场景看中文子集（C-MTEB）

**面试追问：**
- 追问：MTEB 评分高就一定适合我的业务吗？
  - 答：不一定。MTEB 是公开数据集，你的业务场景（如医疗 RAG）需要自己验证。建议先看 Retrieval 任务，再在自己的真实数据上跑 Hit Rate。

---

### Q: 怎么用 MTEB 评估自己的 Embedding 模型？

**Answer:**

```python
from mteb import MTEB
from sentence_transformers import SentenceTransformer

# 1. 加载模型
model = SentenceTransformer('BAAI/bge-m3')

# 2. 选择评估任务
evaluation = MTEB(task_tasks=[
    "retrieval/NQ",
    "retrieval/HotpotQA",
    "STS/sts_benchmark",
])

# 3. 运行评估
results = evaluation.run(
    model,
    output_folder="results/",
    eval_splits=["test"],
)

# 4. 查看结果
for task_result in results:
    print(f"Task: {task_result.task_name}")
    for score in task_result.scores:
        print(f"  {score.metric}: {score.value:.4f}")
```

**评估流程：**
1. 先跑 MTEB 公开任务，看模型整体表现
2. 再在业务数据集上跑（如果有标注数据）
3. 看 Retrieval 任务的 Recall@K 和 MRR
4. 对比更换 embedding 前后的命中率变化

---

## 六、对比学习与 Fine-tuning 篇

### Q: 对比学习（Contrastive Learning）原理是什么？为什么能训练出好的 Embedding？

**Answer:**

**核心思想：让相似的文本靠近，不相似的文本远离。**

**训练数据：**
- **正样本对**：语义相近的文本（如"如何预防感冒" vs "感冒的预防方法"）
- **负样本对**：语义不相关的文本（如"如何预防感冒" vs "今天天气不错"）

**损失函数：**
```python
import torch
import torch.nn.functional as F

def contrastive_loss(emb_pos, emb_neg, temperature=0.07):
    # emb_pos: [batch, dim] 正样本对
    # emb_neg: [batch, dim] 负样本对
    
    # 正样本相似度
    sim_pos = F.cosine_similarity(emb_pos[0], emb_pos[1])
    
    # 负样本相似度
    sim_neg = F.cosine_similarity(emb_pos[0], emb_neg[1])
    
    # 组合 logits
    logits = torch.stack([sim_pos, sim_neg]) / temperature
    labels = torch.zeros(1, dtype=torch.long)  # 正样本在索引 0
    
    return F.cross_entropy(logits.unsqueeze(0), labels)
```

**Temperature 的作用：**
- Temperature 越小，logits 越"尖锐"，模型对正负样本的区分越严格
- Temperature 越大，越"平滑"，更易训练但区分度低
- 常用值：0.05 - 0.1

**InfoNCE 损失（更常用）：**
```python
def infonce_loss(emb_query, emb_pos, emb_neg, temperature=0.07):
    # emb_query: [batch, dim]
    # emb_pos: [batch, dim] 正样本
    # emb_neg: [batch, n_neg, dim] 负样本（In-batch negatives）
    
    # 正样本分数
    pos_score = torch.sum(emb_query * emb_pos, dim=1) / temperature
    
    # 负样本分数
    neg_scores = torch.sum(emb_query.unsqueeze(1) * emb_neg, dim=2) / temperature
    
    # 组合
    logits = torch.cat([pos_score.unsqueeze(1), neg_scores], dim=1)
    labels = torch.zeros(emb_query.size(0), dtype=torch.long, device=emb_query.device)
    
    return F.cross_entropy(logits, labels)
```

**面试追问：**
- 追问：InfoNCE 和 Triplet Loss 有什么区别？
  - 答：InfoNCE 支持多负样本（In-batch negatives），梯度更稳定；Triplet Loss 每次只有一个负样本，训练时负样本质量对结果影响大。

---

### Q: Embedding Fine-tuning 什么时候值得做？

**Answer:**

**必做场景：**
1. **领域术语特殊**：医疗、法律、金融——通用模型不认识专业术语
2. **现有模型表现差**：MTEB 或业务命中率不达标
3. **特定格式需求**：需要判断"是否来自同一政策文件"这种特定相似度

**不用做场景：**
1. 通用领域——直接用 OpenAI / BGE-M3
2. 数据量 < 1000 句对——效果提升有限
3. 对延迟要求极高的场景——微调后模型不能换

**微调数据要求：**

| 要求 | 说明 |
|------|------|
| 最少数据量 | 1000+ 句对 |
| 标注方式 | 正负样本对（label=1 同义, label=0 不同义） |
| 负样本质量 | 越难区分的负样本效果越好（Hard Negatives） |

**Sentence-Transformers 实战：**
```python
from sentence_transformers import SentenceTransformer, InputExample
from torch.utils.data import DataLoader

# 1. 准备数据
train_examples = [
    InputExample(texts=["如何预防感冒", "感冒的预防方法"], label=0.9),
    InputExample(texts=["如何预防感冒", "今天天气不错"], label=0.1),
    InputExample(texts=["劳动合同到期", "用人单位应提前三十日通知"], label=0.8),
]

# 2. 加载预训练模型
model = SentenceTransformer('BAAI/bge-m3')

# 3. 训练
train_dataloader = DataLoader(train_examples, shuffle=True, batch_size=8)
model.fit(
    train_objectives=[(train_dataloader, None)],
    epochs=3,
    warmup_steps=10,
    learning_rate=2e-5,
)
```

**面试能说的话：**
"Embedding fine-tuning 的核心是领域适配——让模型学会你业务中的语义关系。数据量要求 1000+，关键是负样本质量。生产环境中，先用现有模型，效果不够再 fine-tune。"

---

### Q: Hard Negative Mining 为什么重要？怎么构造高质量负样本？

**Answer:**

**为什么重要：**
模型在 Easy Negatives（明显不相关的样本）上训练很容易——只要稍微拉开距离就行。Hard Negatives（相似但不相关的样本）才是区分度来源——需要模型真正理解语义才能区分。

**构造策略：**

| 策略 | 方法 | 复杂度 | 效果 |
|------|------|--------|------|
| In-batch Negatives | 同 batch 内其他正样本对互为负样本 | 低 | 基础 |
| BM25 Negatives | 用 BM25 检索相关但不对应的文档 | 中 | 提升关键词匹配 |
| Cross-encoder 过滤 | 用强模型筛选真正的负样本 | 高 | 最高 |

**负样本配比建议：**

| 负样本类型 | 比例 | 作用 |
|-----------|------|------|
| 随机负样本 | 50% | 基础对比学习 |
| BM25 负样本 | 30% | 提升关键词匹配能力 |
| Hard Negatives | 20% | 提升细粒度区分能力 |

**实战代码：**
```python
# BM25 Hard Negatives 构造
from rank_bm25 import BM25Okapi

tokenized_corpus = [doc.split() for doc in documents]
bm25 = BM25Okapi(tokenized_corpus)

hard_negatives = []
for query, pos_doc in zip(queries, pos_docs):
    # BM25 检索 top-5
    scores = bm25.get_scores(query.split())
    top_indices = np.argsort(scores)[-5:]
    
    # 过滤掉正样本
    candidates = [documents[i] for i in top_indices if documents[i] != pos_doc]
    hard_negatives.extend(candidates[:2])  # 取前 2 个
```

---

## 七、ColBERT 多向量检索篇

### Q: 什么是 ColBERT？和标准 Embedding 有什么本质区别？

**Answer:**

**核心思想：为每个 token 生成独立向量，检索时用 Max-Pooling 计算细粒度交互。**

| 维度 | 标准 Embedding | ColBERT |
|------|--------------|---------|
| 向量数量 | 每个文档 1 个向量 | 每个 token 1 个向量 |
| 查询方式 | 单一向量 vs 单一向量 | 查询每个 token vs 文档每个 token |
| 延迟 | 低（一次点积） | 高（二次交互） |
| 细粒度信息 | 压缩丢失 | 保留 |

**Late Interaction 机制：**
```python
# 1. 文档编码：每个 token 一个向量
doc_vectors = bert_encode(doc)  # [seq_len, hidden_dim]

# 2. 查询编码：每个 token 一个向量
query_vectors = bert_encode(query)  # [query_len, hidden_dim]

# 3. 交互：查询每个 token 与文档所有 token 做点积，取最大值
scores = torch.matmul(query_vectors, doc_vectors.T)  # [query_len, doc_len]
max_scores = scores.max(dim=1)  # Max-Pooling over doc tokens
final_score = max_scores.sum()
```

**为什么比标准 Embedding 好：**
- 标准 Embedding 把整个文档压缩成一个向量，信息有损失
- ColBERT 保留了每个 token 的语义信息，短语匹配更精确

**面试追问：**
- 追问：ColBERT 延迟高怎么解决？
  - 答：ColBERTv2 用 Product Quantization 压缩向量；或者只对第二阶段精排用 ColBERT，第一阶段粗召回用标准 Embedding。

---

### Q: Dense 和 Sparse Embedding 各擅长什么？Hybrid Search 怎么实现？

**Answer:**

**对比：**

| 维度 | Dense | Sparse |
|------|-------|--------|
| 向量形式 | 连续值（如 1536 维） | 大部分为 0 |
| 代表 | OpenAI, BGE-M3 | BM25, SPLADE |
| 特点 | 语义理解强 | 关键词精确匹配强 |
| 适用场景 | 表述多样的语义查询 | 专业术语、专有名词 |
| 延迟 | 低 | 低 |

**Hybrid Search 融合：**
```python
def hybrid_search(query, dense_db, sparse_index, documents, k=10, alpha=0.5):
    """Reciprocal Rank Fusion 融合"""
    
    # Dense 检索
    dense_results = dense_db.similarity_search(query, k=k*2)
    dense_scores = {doc.id: score for doc, score in dense_results}
    
    # Sparse 检索 (BM25)
    tokenized_query = query.split()
    sparse_scores_raw = bm25.get_scores(tokenized_query)
    sparse_scores = {i: score for i, score in enumerate(sparse_scores_raw)}
    
    # Reciprocal Rank Fusion
    fused_scores = {}
    all_doc_ids = set(dense_scores.keys()) | set(sparse_scores.keys())
    
    for doc_id in all_doc_ids:
        d_rank = 1 / (k + list(dense_scores.keys()).index(doc_id)) if doc_id in dense_scores else 0
        s_rank = 1 / (k + list(sparse_scores.keys()).index(doc_id)) if doc_id in sparse_scores else 0
        fused_scores[doc_id] = alpha * d_rank + (1 - alpha) * s_rank
    
    return sorted(fused_scores.items(), key=lambda x: x[1], reverse=True)[:k]
```

---

## 八、Cross-lingual Embedding 篇

### Q: 什么是 Cross-lingual Embedding？跨语言 RAG 怎么实现？

**Answer:**

**核心挑战：**
不同语言的文本（如英文"What is AI?"和中文"什么是人工智能？"）要在同一个向量空间中距离近。

**代表模型：**

| 模型 | 支持语言 | 特点 |
|------|---------|------|
| LaBSE | 100+ | Google，专精跨语言句子相似度 |
| MUSE | 50+ | Facebook，多语言词典学习 |
| BGE-M3 | 100+ | 中英文最强，支持多语言 |

**跨语言 RAG 实现：**
```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 中文知识库
chunks = [
    "人工智能是计算机科学的一个分支",
    "机器学习是人工智能的子领域"
]

# 向量化（同一个模型，自动支持多语言）
vectors = [embeddings.embed_query(chunk) for chunk in chunks]

# 英文查询
query = "What is artificial intelligence?"
query_vector = embeddings.embed_query(query)

# 检索（自动对齐到同一个语义空间）
from sklearn.metrics.pairwise import cosine_similarity
similarity = cosine_similarity([query_vector], vectors)[0]
print(f"最相似的文档: {similarity.argmax()}")
```

**实际应用场景：**
- 国际企业的内部知识库（多语言）
- 跨境电商的跨语言客服
- 学术论文的多语言检索

---

## 九、生产实践篇

### Q: Embedding 在生产环境中的工程问题有哪些？

**Answer:**

**核心工程问题：**

| 问题 | 解决方案 |
|------|----------|
| **Embedding 计算延迟** | Batch 处理 + 缓存（相同文本不重复计算） |
| **向量库更新** | 增量更新 + 版本管理（双写过渡） |
| **多租户隔离** | 每个租户独立 namespace 或过滤字段 |
| **向量维度选择** | 精度 vs 存储成本权衡（1536 vs 3072） |
| **监控** | 检索命中率 + 生成质量定期评估 |

**缓存策略：**
```python
from functools import lru_cache

class CachedEmbeddings:
    def __init__(self, embeddings_client):
        self.client = embeddings_client
        self._cache = {}
    
    def embed(self, text: str) -> list[float]:
        if text not in self._cache:
            self._cache[text] = self.client.embed(text)
        return self._cache[text]
    
    def batch_embed(self, texts: list[str]) -> list[list[float]]:
        uncached = [t for t in texts if t not in self._cache]
        if uncached:
            new_vectors = self.client.embed_batch(uncached)
            for text, vector in zip(uncached, new_vectors):
                self._cache[text] = vector
        return [self._cache[t] for t in texts]
```

---

### Q: 生产级 RAG 的完整 Embedding 架构？

**Answer:**

```
知识库（文档）
    │
    ├──→ 切分（chunk_size=500, overlap=50）
    │
    ├──→ 向量化（Embedding 模型）
    │
    └──→ 存入 VectorDB（FAISS / Qdrant / Milvus）
            │
            │  查询流程
            ▼
        查询文本 ──→ 向量化 ──→ 向量检索（Top-100）
                                    │
                                    ├──→ 过滤（多租户 / 分类）
                                    │
                                    └──→ 精排（Reranker / ColBERT）
                                            │
                                            └──→ Top-10 → LLM 生成
```

**分层索引（Hierarchical Indices）：**
```
Top-100 (HNSW，粗召回) → Top-20 (HNSW + 过滤) → Top-5 (精确重排)
```

**面试追问：**
- 追问：什么时候需要向量版本管理？
  - 答：知识库更新后，旧文档的向量和知识库内容不一致。解决方案：软删除（加 `deleted_at` 字段）+ 向量重建 + 双写过渡。

---

## 十、面试总结篇

### Q: 面试必背：怎么选 Embedding 模型？

**Answer:**

**决策树：**

```
场景判断
├── 有 API 能力？
│   ├── 是 → OpenAI text-embedding-3-small（最省事）
│   └── 否 → 本地部署？
│       ├── 是 → GPU 够？
│       │   ├── 是 → BGE-M3（质量最好）
│       │   └── 否 → M3E（轻量，CPU 可跑）
│       └── 云部署？
│           ├── 是 → BGE-M3 部署云 GPU
│           └── 高隐私 → M3E（本地跑）
└── 专业领域？
    ├── 医疗 / 法律 → Fine-tune BGE-M3
    └── 通用 → 直接用 API 模型
```

**面试时的完整回答模板：**

"Embedding 模型选型主要看三点：**精度、成本、隐私**。

精度方面：先看 MTEB 排行榜中文表现，BGE-M3 目前中文最强，OpenAI embedding-3-large 国际表现好。

成本方面：1536 维的 small 模型存储和延迟都更低，3072 维的 large 贵 4 倍但精度提升有限——先用 small，效果不够再换 large。

隐私方面：有高隐私需求用本地开源模型（BGE-M3 或 M3E），否则直接用 API。

生产环境我建议分层：粗召回用标准 Embedding，精排用 ColBERT 或 Reranker。"

---

### Q: Embedding 相关的面试高频问题 TOP 5？

**Answer:**

| 排名 | 问题 | 考察点 |
|------|------|--------|
| 1 | Embedding 本质是什么？ | 语义坐标的理解 |
| 2 | 余弦 vs 点积，L2 归一化后的关系 | 相似度计算基础 |
| 3 | HNSW vs IVF 适用场景 | 索引算法理解 |
| 4 | 对比学习原理 | 模型训练基础 |
| 5 | 怎么评估 Embedding 模型 | 工程实践能力 |

---

### Q: 完整回答模板：为什么语义相近的文本向量距离近？

**Answer（面试标准回答）：**

"Embedding 本质是训练一个语义坐标系统。训练时，通过对比学习（Contrastive Learning），让正样本对（语义相近的文本）的向量距离近，负样本对（语义不相关的文本）的向量距离远。

训练完成后，这个语义空间中位置接近 = 语义相近。检索就是在同一个语义空间中，找到离查询向量最近的 k 个向量。

比如在 RAG 系统中，用户查询和文档被同一个模型编码后，语义相近的文档向量和查询向量自然距离近——这就是检索能生效的本质原因。"

---

*最后更新：2026-07-01*
