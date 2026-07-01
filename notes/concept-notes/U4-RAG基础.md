# U4 · RAG 基础

> **核心问题**：怎么让模型用你自己的知识回答？
> **预计学时**：5h
> **对应 Phase**：Phase 1

---

## 学习目标（Gate 检验）

1. 能画出 RAG 完整流程图（文档→切分→向量化→检索→生成）
2. 能比较三种 Chunking 策略的优缺点
3. 能实现一次完整的本地 RAG 流程（LangChain 或手写）
4. 能解释 Reranking 在 RAG 中的作用

---

## 1. RAG 三段流程

```
文档 → 切分（Chunking） → 向量化（Embedding） → 存入 VectorDB
                                                      ↓
用户问题 → 向量化 → 相似度检索（Top-K） → 拼接为上下文 → LLM 生成
```

### 三个阶段

1. **Ingestion（摄取）**：文档加载 → 清洗 → 切分 → 向量化 → 入库
2. **Retrieval（检索）**：问题向量化 → VectorDB 搜索 → 召回 Top-K
3. **Generation（生成）**：把检索结果作为 context 拼接 → LLM 生成回答

### 直观理解

RAG 像是在让模型"开卷考试"——把知识库作为参考资料提供给模型，而不是靠模型死记硬背。

---

## 2. Document Loading 与清洗

### 常用工具

| 工具 | 支持格式 |
|------|---------|
| LangChain DocumentLoaders | PDF, HTML, JSON, Markdown, CSV, Slack, Notion... |
| PyMuPDF | PDF（提取文字和布局） |
| pdfplumber | PDF（表格提取强） |
| unstructured | 通用，支持 100+ 格式 |

### 常见问题

- PDF 扫描件（没有文字层）→ 需要 OCR
- 表格数据 → 需要专门提取（pdfplumber 或 TableTransformers）
- 特殊符号 / 乱码 → 需要清洗正则替换

---

## 3. Chunking 策略

### 固定窗口切分（最常用）

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,    # 按字符数切分
    chunk_overlap=50,   # 重叠 50 字符保上下文
    separators=["\n\n", "\n", "。", "！", "？", " "],  # 优先级顺序
)
chunks = splitter.split_text(long_text)
```

### 三种策略对比

| 策略 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| 固定窗口 | 按字符/ token 数切 | 简单可控 | 可能切断句子 |
| 重叠切分 | 窗口 + overlap | 保留上下文 | 数据冗余 |
| 语义分割 | 按句子/段落切 | 保证语义完整 | 需要 NLP 工具 |

### Chunk Size 怎么选

| 场景 | 建议 chunk_size |
|------|----------------|
| 短问答（FAQ） | 64~128 tokens |
| 中等段落（文档段落） | 256~512 tokens |
| 长文档（论文、报告） | 512~1024 tokens |
| 代码库 | 函数级切分（用 AST parser） |

### 实用原则

- **太短**：缺少上下文，信息碎片化
- **太长**：超过模型 context，检索精度下降
- **重叠**：50 字符 overlap 是经验值，兼顾上下文和避免切断句子

---

## 4. Embedding 与 VectorDB

### 流程

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

# 向量化
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 入库
db = FAISS.from_documents(documents, embeddings)

# 检索
results = db.similarity_search(query, k=3)
```

### 多模态 RAG（图片+文字）

- 方案 1：图片描述（GPT-4V）→ 转成文字 → 走标准 RAG
- 方案 2：多向量检索（一个文档块同时有文字向量和图片向量）

---

## 5. Retrieval 优化

### Top-K 调参

- `k=3`：日常简单问答够用
- `k=10~20`：复杂问题需要更多召回
- 但 `k` 太大 → context 超过 LLM context window → 需要 reranking 或 compression

### Reranking（重排序）

检索回来的 Top-K 不一定按相关性排序。Reranking 是用更大的模型（如 BGE-Reranker）做更精准的排序。

```python
# 流程：VectorDB 检索（快速，100 条） → Reranker（精确，重排） → Top-5 → LLM
```

### Hybrid Search（混合检索）

同时用关键词检索（BM25）和向量检索，取两者结果的并集或交集。

适用场景：专业术语多、向量化质量差的领域。

### Self-RAG（自检索增强）

模型在生成过程中自己判断是否需要检索补充知识，而不是每次都检索。

---

## 6. Generation 提示词设计

### 核心原则

把检索结果明确标注来源，让 LLM 知道"这是给的知识，不是自己编的"。

### 基础 Prompt

```
你是一个专业的知识库问答助手。

根据以下参考材料回答用户问题。如果材料中没有答案，明确说"我无法从材料中找到答案"，不要编造。

参考材料：
{context}

用户问题：{question}

回答要求：
- 只基于材料回答，不要添加材料外的信息
- 如果有引用，用 [来源N] 标注
- 回答语言与问题一致
```

---

## 7. 完整示例：LangChain RAG

```python
# pip install langchain-openai langchain-community faiss-cpu pymupdf

import os
from langchain_community.document_loaders import PyMuPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_openai import ChatOpenAI

# 1. 加载 PDF
loader = PyMuPDFLoader("your_document.pdf")
docs = loader.load()

# 2. 切分
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# 3. 向量化 + 建库
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
db = FAISS.from_documents(chunks, embeddings)

# 4. 检索
query = "文档中的核心结论是什么？"
docs = db.similarity_search(query, k=5)

# 5. 生成
context = "\n\n".join([d.page_content for d in docs])
llm = ChatOpenAI(model="gpt-4o-mini")
prompt = f"""根据以下材料回答问题。

材料：
{context}

问题：{query}

回答："""

result = llm.invoke(prompt)
print(result.content)
```

---

## 完成 Gate 检验

1. **Q1**：画 RAG 完整流程图，并标注每个阶段在做什么。
2. **Q2**：固定窗口 / 重叠切分 / 语义分割 各适合什么场景？
3. **Q3**：Reranking 在 RAG 中的作用是什么？为什么需要它？
4. **Q4**：Hybrid Search 是什么？什么时候用它？

---

## 练习清单

- [ ] 用 PyMuPDFLoader 加载一个 PDF，提取文字
- [ ] 用 RecursiveCharacterTextSplitter 切分，对比不同 chunk_size 的效果
- [ ] 建一个本地 FAISS 向量库，实现完整 RAG 流程
- [ ] 尝试加一个 Reranker 看检索质量变化（可选）

---

## 关联资源

- [LangChain RAG 教程](https://python.langchain.com/docs/tutorials/rag/)
- [Chunking Strategies for RAG](https://pub.aimind.io/2024/chunking-strategies-for-rag/)
- [BGE-Reranker](https://huggingface.co/BAAI/bge-reranker)
- [Hybrid Search 原理](https://www.pinecone.io/search/hybrid-search)

---

*最后更新：2026-05-26*
---

## 8. Chunking 策略深度对比（all-in-rag 实战）

> 参考 [datawhalechina/all-in-rag 第二章 文本分块](https://github.com/datawhalechina/all-in-rag)

### 8.1 5 大策略对比

| 策略 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| **固定窗口** (`chunk_size=512`) | 简单、可预测 | 切断语义 | 通用 baseline |
| **重叠窗口** (`chunk_size=512, overlap=128`) | 保留边界语义 | 仍按字符切 | 大多数场景（推荐）|
| **递归字符** (按段落/句子) | 保留语义结构 | 长 chunk 不均 | 结构化文档 |
| **语义切分** (按 embedding 距离) | 真正语义边界 | 慢、贵 | 高质量 RAG |
| **Markdown/HTML 结构** (按 heading) | 完整语义块 | 依赖文档结构 | 技术文档 |

### 8.2 Chunk Size 选择

**经验数据**（来自社区基准）：

| Chunk Size | Faithfulness | Answer Relevancy | 速度 |
|-----------|--------------|------------------|------|
| 128 | 0.78 | 0.72 | 最快 |
| **256** | **0.85** | **0.81** | 平衡 |
| 512 | 0.86 | 0.83 | 中等 |
| 1024 | 0.83 | 0.79 | 慢 |
| 2048 | 0.75 | 0.72 | 最慢 |

**实战建议**：**chunk_size=256, overlap=64** 起步；中文文档 +128；超长文档用 512。

### 8.3 实战代码：递归字符切分（最常用）

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=256,
    chunk_overlap=64,
    # 优先按这些分隔符切
    separators=["\n\n", "\n", "。", "！", "？", " ", ""],
    length_function=len,
    is_separator_regex=False,
)

chunks = text_splitter.split_text(long_document)
print(f"Split into {len(chunks)} chunks")
```

**为什么"递归"**：先按段落切，太长再按句子切，再长按词切。**保留语义结构**。

### 8.4 实战代码：Markdown 结构切分

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

# 1. 按 markdown header 切（保留 heading 上下文）
md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "Header 1"),
        ("##", "Header 2"),
        ("###", "Header 3"),
    ]
)
md_chunks = md_splitter.split_text(markdown_doc)

# 2. 再按 chunk size 切（避免单 chunk 太大）
final_splitter = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=64)
final_chunks = final_splitter.split_documents(md_chunks)
```

**好处**：每个 chunk 带着 heading 上下文，retrieval 质量高。

---

## 9. Embedding 模型选型（all-in-rag 第三章）

### 9.1 主流 Embedding 模型对比（2025）

| 模型 | 维度 | MTEB 平均 | 成本 | 中文 | 开源 |
|------|------|-----------|------|------|------|
| **text-embedding-3-small** (OpenAI) | 1536 | 0.62 | $0.02/1M | 中 | ❌ |
| **text-embedding-3-large** (OpenAI) | 3072 | **0.64** | $0.13/1M | 中 | ❌ |
| **BAAI/bge-m3** | 1024 | 0.625 | 免费 | **强** | ✅ |
| **BAAI/bge-large-zh-v1.5** | 1024 | 0.62 | 免费 | **强** | ✅ |
| **Cohere embed-english-v3.0** | 1024 | 0.65 | $0.10/1M | 弱 | ❌ |
| **voyage-3** | 1024 | **0.67** | $0.06/1M | 中 | ❌ |

**选型决策**：
- **英文为主 + 质量优先**：voyage-3 > text-embedding-3-large > bge-m3
- **中文为主**：bge-large-zh-v1.5 或 bge-m3（开源 + 强中文）
- **预算紧 + 通用**：text-embedding-3-small（性价比高）
- **本地 / 离线**：bge-m3（开源 SOTA）

### 9.2 Embedding 微调（进阶）

```python
# 用 sentence-transformers 微调 bge
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader

model = SentenceTransformer('BAAI/bge-base-zh-v1.5')

# 1. 准备训练数据（query, positive, negative）
train_data = [
    InputExample(texts=['RAG 是什么', 'RAG 通过检索增强生成...', '神经网络是模仿生物神经...']),
    # ... 几千条
]

# 2. 训练
train_dataloader = DataLoader(train_data, shuffle=True, batch_size=16)
train_loss = losses.CachedMultipleNegativesRankingLoss(model)
model.fit(train_objectives=[(train_dataloader, train_loss)], epochs=3, output_dir='./ft-embedding')
```

**经验**：微调后通常 +5-10% RAGAS。

---

## 10. 完整实战：手写 RAG（50 行）

```python
from openai import OpenAI
import numpy as np
from typing import List

class MiniRAG:
    def __init__(self, openai_client: OpenAI):
        self.client = openai_client
        self.chunks: List[str] = []
        self.embeddings: np.ndarray = None

    def add_documents(self, documents: List[str], chunk_size=256, overlap=64):
        # 1. 简单切分（生产用 RecursiveCharacterTextSplitter）
        all_chunks = []
        for doc in documents:
            for i in range(0, len(doc), chunk_size - overlap):
                chunk = doc[i:i + chunk_size]
                if len(chunk) > 50:  # 过滤太短
                    all_chunks.append(chunk)
        self.chunks = all_chunks

        # 2. Embedding
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=self.chunks,
        )
        self.embeddings = np.array([d.embedding for d in response.data])
        print(f"Added {len(self.chunks)} chunks")

    def search(self, query: str, top_k=3) -> List[str]:
        # 1. Embedding query
        q_emb = np.array(
            self.client.embeddings.create(model="text-embedding-3-small", input=[query]).data[0].embedding
        )
        # 2. Cosine similarity
        scores = self.embeddings @ q_emb / (
            np.linalg.norm(self.embeddings, axis=1) * np.linalg.norm(q_emb)
        )
        # 3. Top-K
        top_indices = np.argsort(scores)[-top_k:][::-1]
        return [self.chunks[i] for i in top_indices]

    def query(self, question: str) -> str:
        chunks = self.search(question)
        context = "\n\n".join(chunks)
        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"基于以下上下文回答问题。如果上下文中没有答案，回答 'I don't know'。\n\n上下文：{context}\n\n问题：{question}",
            }],
        )
        return response.choices[0].message.content

# 使用
rag = MiniRAG(OpenAI())
rag.add_documents([long_doc_1, long_doc_2])
print(rag.query("什么是 RAG？"))
```

**生产改进**：
- 加 Reranker（U15）
- 加 RAGAS eval
- 加 streaming output
- 用 LangChain / LlamaIndex 替换


## 9. 与本仓库其他单元的关系

| 关联单元 | 关系 |
| --- | --- |
| U2 Embedding 与 VectorDB | RAG 的检索层依赖 U2 的 Embedding + VectorDB 技术 |
| U3 Prompt 工程 | RAG 的 generation 阶段依赖 U3 的 prompt 设计 |
| U15 RAG 进阶 | U4 是基础，U15 深入高级 RAG 技术 |
| U22 Context Engineering | RAG 检索结果是 Context 的重要组成部分 |
| U31 Structured Output | RAG 输出的验证依赖 U31 的 structured output 技术 |