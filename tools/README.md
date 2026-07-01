# Tools 工具速查指南

> 每个主流框架一份 SKILL.md 格式的速查指南，包含：安装 / 核心概念 / 最小可运行代码 / 常见坑点 / 面试高频问法

---

## 目录

|| 文件 | 框架 | 状态 |
|------|------|------|
| `langchain-guide.md` | LangChain / LangGraph | ✅ |
| `llamaindex-guide.md` | LlamaIndex | ✅ |
| `crewai-guide.md` | CrewAI | ✅ |
| `autogen-guide.md` | AutoGen | ✅ |
| `mcp-guide.md` | MCP (Model Context Protocol) | ✅ |
| `smolagents-guide.md` | smolagents | ✅ |
| `a2a-guide.md` | A2A (Agent-to-Agent) 协议 | 🆕 |
| `dspy-guide.md` | DSPy (声明式 Prompt 优化) | 🆕 |

---

## 使用方法

1. **学习新框架前**：先通读对应 guide，建立整体概念
2. **写代码时**：当速查手册，复制粘贴最小代码片段
3. **面试前**：背"面试高频问法"部分，对照检查理解

---

## 各框架定位速查

```
主要场景是 RAG + 私有数据？
├── 是 → LlamaIndex（专注检索优化）
└── 否 → 需要写代码实现 Agent？
    ├── 是 → 需要 Multi-Agent？
    │   ├── 是 → 角色明确分工清晰 → CrewAI
    │   ├── 是 → 需要复杂状态机/Cycles → LangGraph
    │   ├── 是 → 微软生态、代码执行 → AutoGen
    │   └── 否 → 需要 RAG + Tool + Memory 全家桶 → LangChain
    └── 否 → 需要可视化搭建 → Dify / Coze
```

---

## 工具文件规范

每个 guide 包含：
- **安装命令**（一行 pip install）
- **核心概念速查表**（关键类 + 作用）
- **最小可运行代码**（3-5 个精选例子，可直接复制跑）
- **常见坑点**（表格形式：坑 / 说明 / 解法）
- **面试高频问法**（5-8 道题 + 答案要点）

---

*最后更新：2026-05-27*