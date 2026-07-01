# Tool Guide · LlamaIndex

> 覆盖：安装 / 核心概念 / 最小可运行代码 / 常见坑点 / 面试高频问法
> 对比：LlamaIndex vs LangChain vs LangGraph

---

## 1. 安装

```bash
pip install llama-index llama-index-agent-openai llama-index-llms-openai
```

---

## 2. 核心概念速查

### 2.1 三大框架对比

| 维度 | LangChain | LangGraph | **LlamaIndex** |
|------|-----------|-----------|----------------|
| 定位 | 全能型 LLM 应用框架 | 有状态 Multi-Agent 编排 | **专注 RAG + 数据检索** |
| 核心能力 | Tool / Chain / Agent / Memory | Cycles / 状态机 / 人机交互 | **索引 / 检索 / 问答** |
| 学习曲线 | 中 | 高 | 低 |
| RAG 支持 | 支持但分散 | 需自己搭 | **原生强，专注优化检索质量** |

**LlamaIndex 的核心定位**：让你的私有数据能被 LLM 使用（RAG 专家）

### 2.2 关键概念

| 概念 | 说明 |
|------|------|
| `Document` | 文本/HTML/PDF/MD 的封装，含 text + metadata |
| `Node` | Document 切分后的块，保留与父文档的关系 |
| `Index` | 索引结构（Vector / Keyword / Composite） |
| `Retriever` | 从索引中拉取相关节点 |
| `QueryEngine` | 检索 + 组装 prompt + 调用 LLM 生成答案 |
| `Response Synthesizer` | 把检索结果合并成最终回答 |

### 2.3 Index 类型

```python
from llama_index.core import VectorStoreIndex  # 向量检索（最常用）
from llama_index.core import KeywordTableIndex  # 关键词检索
from llama_index.core import SummaryIndex  # 摘要索引
from llama_index.core import ComposableGraph  # 组合多个索引
```

---

## 3. 最小可运行代码

### 3.1 最简单 RAG（5 行）

```python
from llama_index.core import VectorStoreIndex
from llama_index.readers.file import MarkdownReader

# 加载文档 → 建索引 → 查询
docs = MarkdownReader().load_data("./notes/")
index = VectorStoreIndex.from_documents(docs)
query_engine = index.as_query_engine()
response = query_engine.query("什么是 ReAct 范式？")
print(response)
```

### 3.2 自定义 Embedding + 向量库（Chroma）

```python
from llama_index.core import VectorStoreIndex
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.core import Settings

# 配置 embedding
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# 用 Chroma 向量库
vector_store = ChromaVectorStore(collection_name="my_notes", persist_dir="./chroma_db")
index = VectorStoreIndex.from_documents(docs, vector_store=vector_store)

query_engine = index.as_query_engine()
response = query_engine.query("用户问题")
```

### 3.3 带过滤条件的检索

```python
from llama_index.core import VectorStoreIndex

index = VectorStoreIndex.from_documents(docs)

# 元数据过滤：只检索 tag="AI" 的文档
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core.retrievers import VectorIndexRetriever

retriever = VectorIndexRetriever(
    index=index,
    filters=["tag", "==", "AI"],  # 元数据过滤
    similarity_top_k=5
)
query_engine = RetrieverQueryEngine(retriever=retriever)
```

### 3.4 Query Pipeline（链式查询）

```python
from llama_index.core.query_pipeline import QueryPipeline

pipeline = QueryPipeline()
pipeline.add_modules({
    "retriever": index.as_retriever(similarity_top_k=3),
    "synthesizer": index.as_query_engine()
})

# 自定义链路：检索 → 后处理 → 生成
response = pipeline.run(query="什么是 Tool Use？")
```

### 3.5 组合多个索引（Composable Graph）

```python
from llama_index.core import ComposableGraph
from llama_index.core import SummaryIndex

# 为不同数据源建不同索引
index_notes = VectorStoreIndex.from_documents(notes_docs)
index_reports = VectorStoreIndex.from_documents(report_docs)
index_papers = VectorStoreIndex.from_documents(paper_docs)

# 组合成图
graph = ComposableGraph.from_indices(
    [index_notes, index_reports, index_papers],
    root_id="notes"
)

# 查询时自动跨索引检索
query_engine = graph.as_query_engine()
```

### 3.6 自定义 Tool（Agent 用）

```python
from llama_index.core.tools import QueryEngineTool, ToolMetadata

# 把 QueryEngine 封装成 Agent Tool
tool = QueryEngineTool(
    query_engine=index.as_query_engine(),
    metadata=ToolMetadata(
        name="notes_search",
        description="搜索个人笔记知识库，回答关于学习记录的问题"
    )
)

# 配合 Agent 使用
from llama_index.agent.openai import OpenAIAgent

agent = OpenAIAgent.from_tools([tool])
response = agent.chat("我上周学了哪些概念？")
```

---

## 4. 高级用法

### 4.1 Node 后处理（重排序 / 压缩）

```python
from llama_index.core.postprocessor import SimilarityPostprocessor, KeywordPostprocessor

query_engine = index.as_query_engine(
    node_postprocessors=[
        SimilarityPostprocessor(threshold=0.7),  # 低于 0.7 的结果丢弃
        KeywordPostprocessor(filter_keywords=["敏感词"])
    ]
)
```

### 4.2 HyDE（假设性文档增强）

```python
# HyDE：先让 LLM 生成"假设答案"，再用假设答案去检索
from llama_index.core import VectorStoreIndex

index = VectorStoreIndex.from_documents(docs)
query_engine = index.as_query_engine(
    similarity_top_k=5,
    # 启用 HyDE（需要更多 token 但检索更准）
)
```

### 4.3 Auto-Retrieval（自动判断检索什么）

```python
from llama_index.core import VectorStoreIndex

# LLM 自动判断用哪个索引，不需要手动选择
index = VectorStoreIndex.from_documents(all_docs)
query_engine = index.as_query_engine(
    recursive=True,  # 自动跨子索引检索
    retriever_mode="default"
)
```

---

## 5. 常见坑点

| 坑 | 说明 | 解法 |
|----|------|------|
| 检索质量差 | chunk_size 太大/太小 | 调 `SentenceSplitter(chunk_size=512, chunk_overlap=50)` |
| 上下文太长 | 检索了太多节点超出 window | 加 `similarity_top_k=3` 限制 + postprocessor 过滤 |
| 更新索引慢 | 每次更新重建整个索引 | 用 `DynamicRetriever` 或增量索引 |
| 中文分词 | 默认对中文不友好 | 用 `JinaEmbedding` 或 `BAAI/bge-large-zh` |
| 元数据丢失 | 切分后 Node 丢失 metadata | 切分时保留 `include_metadata=True` |

---

## 6. 面试高频问法

### Q: LlamaIndex 和 LangChain 的核心区别？

→ **LangChain**：全能框架，Tool/Chain/Memory/Agent 全覆盖，RAG 是其中一小部分
→ **LlamaIndex**：专注 RAG + 数据检索，索引/检索质量深度优化
→ 选择：如果你的场景 70% 是 RAG + 私有数据 → 用 LlamaIndex；如果需要复杂 Agent 编排 → 用 LangChain

### Q: LlamaIndex 的核心流程？

→ Document → Node（切分）→ Index（向量化）→ Retriever（检索）→ QueryEngine（组装 LLM 生成）

### Q: Document / Node / Index 的关系？

→ Document = 原始文档（一个 PDF / Markdown / 文本）
→ Node = Document 切分后的块，保留父子关系和 metadata
→ Index = 索引结构，加速检索（通常是向量索引）

### Q: 什么是 ComposableGraph？

→ 把多个索引组合成一个图，查询时自动跨索引检索
→ 适用：不同数据源建不同索引，再组合查询

### Q: 如何提升 RAG 检索质量？

→ 1. 调 chunk_size（512 左右通常较好）+ chunk_overlap
→ 2. 用元数据过滤缩小范围
→ 3. 加 Node 后处理（重排序/关键词过滤）
→ 4. 用 HyDE 假设性文档增强
→ 5. 用更好的 embedding 模型（如 bge-large-zh）

### Q: LlamaIndex 支持哪些向量库？

→ 支持：Chroma / FAISS / Milvus / Pinecone / Weaviate / Qdrant / 一切兼容 LangChain VectorStore 的

---

## 7. 和 LangChain 的选择决策树

```
主要场景是 RAG + 私有数据？
├── 是 → LlamaIndex（专注检索优化）
└── 否 → 需要 Tool 调用 / Multi-Agent / 复杂 Chain？
    ├── 是 → LangChain
    └── 否 → 需要状态机 + Cycles？
        ├── 是 → LangGraph
        └── 否 → 只是简单 LLM 调用 → 直接用 SDK
```

---

*最后更新：2026-05-27*