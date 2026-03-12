# 34 - RAG：检索增强生成

> 让 Agent 拥有"长期外部记忆"，突破 Context Window 限制

---

## 为什么需要 RAG？

大模型有两个硬伤：

1. **知识截止日期** —— 训练数据有时效，不知道最新信息
2. **Context Window 有限** —— 塞不下整个知识库

RAG（Retrieval-Augmented Generation）的思路很简单：
> **用户问问题 → 先搜相关文档 → 把文档塞进 Context → 让 LLM 基于文档回答**

```
用户问题
   │
   ▼
[Embed Query] ──→ 向量数据库搜索 ──→ 取出 Top-K 文档块
                                          │
                                          ▼
                              [System Prompt + 文档 + 问题] ──→ LLM ──→ 答案
```

---

## 核心组件

### 1. 文档切块（Chunking）

把长文档切成适合 LLM 处理的小块。

```python
# learn-claude-code 风格的简单切块
def chunk_text(text: str, chunk_size: int = 512, overlap: int = 50) -> list[str]:
    """
    按字符数切块，带重叠避免丢失边界信息
    """
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start += chunk_size - overlap  # 滑动窗口，overlap 保留上下文
    return chunks

# 更智能：按语义边界切（段落、句子）
def chunk_by_paragraph(text: str) -> list[str]:
    paragraphs = text.split('\n\n')
    return [p.strip() for p in paragraphs if p.strip()]
```

**切块策略对比：**

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 固定大小 | 简单、均匀 | 可能截断语义 | 通用文本 |
| 按段落 | 保留语义完整性 | 块大小不均 | Markdown、文章 |
| 按句子 | 粒度细 | 单句可能缺乏上下文 | 问答型文档 |
| 递归切分 | 平衡语义与大小 | 实现稍复杂 | 推荐首选 |

---

### 2. 向量嵌入（Embedding）

把文本转成数字向量，语义相近的向量距离近。

```python
import anthropic

client = anthropic.Anthropic()

def embed_text(text: str) -> list[float]:
    """
    使用 Voyage AI（Anthropic 推荐）生成嵌入向量
    """
    import voyageai
    vo = voyageai.Client()
    result = vo.embed([text], model="voyage-3")
    return result.embeddings[0]

# 或者用 OpenAI embeddings（更通用）
from openai import OpenAI

def embed_openai(text: str) -> list[float]:
    client = OpenAI()
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding
```

---

### 3. 向量数据库（Vector Store）

存储和检索向量。简单实现用 numpy，生产用 Chroma/Pinecone/pgvector。

```python
import numpy as np
from dataclasses import dataclass

@dataclass
class Document:
    id: str
    text: str
    embedding: list[float]
    metadata: dict

class SimpleVectorStore:
    """纯 Python 实现，用于教学理解原理"""
    
    def __init__(self):
        self.documents: list[Document] = []
    
    def add(self, doc: Document):
        self.documents.append(doc)
    
    def search(self, query_embedding: list[float], top_k: int = 5) -> list[Document]:
        """余弦相似度搜索"""
        if not self.documents:
            return []
        
        query_vec = np.array(query_embedding)
        scores = []
        
        for doc in self.documents:
            doc_vec = np.array(doc.embedding)
            # 余弦相似度 = 点积 / (范数之积)
            similarity = np.dot(query_vec, doc_vec) / (
                np.linalg.norm(query_vec) * np.linalg.norm(doc_vec)
            )
            scores.append((similarity, doc))
        
        # 按相似度降序排列，取 Top-K
        scores.sort(key=lambda x: x[0], reverse=True)
        return [doc for _, doc in scores[:top_k]]
```

**生产级向量数据库推荐：**

```python
# Chroma（本地，开箱即用）
import chromadb

client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection("knowledge_base")

# 批量插入
collection.add(
    ids=["doc1", "doc2"],
    documents=["文档内容1", "文档内容2"],
    metadatas=[{"source": "file1.md"}, {"source": "file2.md"}]
)

# 检索
results = collection.query(
    query_texts=["用户的问题"],
    n_results=5
)
```

---

### 4. 检索 + 生成（The RAG Pipeline）

```python
import anthropic

client = anthropic.Anthropic()

def rag_query(
    question: str,
    vector_store: SimpleVectorStore,
    embed_fn,
    top_k: int = 5
) -> str:
    """完整的 RAG 流程"""
    
    # Step 1: 将问题转成向量
    query_embedding = embed_fn(question)
    
    # Step 2: 检索相关文档
    relevant_docs = vector_store.search(query_embedding, top_k=top_k)
    
    # Step 3: 构建上下文
    context = "\n\n---\n\n".join([
        f"[来源: {doc.metadata.get('source', 'unknown')}]\n{doc.text}"
        for doc in relevant_docs
    ])
    
    # Step 4: 让 LLM 基于上下文回答
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system="""你是一个知识库助手。请严格基于提供的上下文回答问题。
如果上下文中没有相关信息，请直接说"我在知识库中没有找到相关信息"，
不要编造答案。""",
        messages=[{
            "role": "user",
            "content": f"""上下文信息：

{context}

---

问题：{question}"""
        }]
    )
    
    return response.content[0].text
```

---

## RAG 作为 Agent Tool

在 Agent 中，RAG 最自然的用法是**封装成一个工具**，让 LLM 自主决定何时检索。

```python
# pi-mono 风格：RAG 工具定义
rag_tool = {
    "name": "search_knowledge_base",
    "description": """搜索内部知识库。当需要查找：
    - 公司文档、规范、流程
    - 产品说明书
    - 历史数据和记录
    时使用此工具。不要用于实时信息（用 web_search 代替）。""",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索查询，用自然语言描述你想找的内容"
            },
            "top_k": {
                "type": "integer",
                "description": "返回的文档数量，默认 5，复杂问题可以用 10",
                "default": 5
            }
        },
        "required": ["query"]
    }
}

# Tool 执行函数
def execute_rag_tool(query: str, top_k: int = 5) -> str:
    query_embedding = embed_text(query)
    docs = vector_store.search(query_embedding, top_k)
    
    if not docs:
        return "知识库中没有找到相关内容。"
    
    results = []
    for i, doc in enumerate(docs, 1):
        results.append(f"**[{i}] {doc.metadata.get('source', 'unknown')}**\n{doc.text}")
    
    return "\n\n".join(results)
```

---

## OpenClaw 中的实战：Skills 知识库

OpenClaw 的 Skills 系统本质上就是一种 RAG！

```
用户请求 → 匹配 skill description（语义搜索）→ 注入 SKILL.md 到 Context → 执行
```

```javascript
// OpenClaw 伪代码（概念示意）
async function selectSkill(userMessage: string, skills: Skill[]): Promise<Skill | null> {
    // 用 LLM 做语义匹配（轻量级 RAG）
    const prompt = `
用户请求：${userMessage}

可用技能：
${skills.map(s => `- ${s.name}: ${s.description}`).join('\n')}

哪个技能最匹配？返回技能名，没有匹配则返回 null。`;
    
    const result = await llm.complete(prompt);
    return skills.find(s => s.name === result.trim()) || null;
}
```

---

## 进阶技巧

### Hybrid Search（混合搜索）

纯向量搜索有时候会漏掉关键词精确匹配。生产系统通常用**向量搜索 + BM25 关键词搜索**混合：

```python
def hybrid_search(query: str, vector_store, bm25_index, alpha: float = 0.7) -> list:
    """
    alpha: 向量搜索权重（0.7 表示 70% 向量 + 30% 关键词）
    """
    vector_results = vector_store.search(embed_text(query), top_k=10)
    keyword_results = bm25_index.search(query, top_k=10)
    
    # RRF (Reciprocal Rank Fusion) 融合排名
    scores = {}
    for rank, doc in enumerate(vector_results):
        scores[doc.id] = scores.get(doc.id, 0) + alpha * (1 / (rank + 60))
    for rank, doc in enumerate(keyword_results):
        scores[doc.id] = scores.get(doc.id, 0) + (1 - alpha) * (1 / (rank + 60))
    
    sorted_ids = sorted(scores, key=scores.get, reverse=True)
    return sorted_ids[:5]
```

### Re-ranking（重排序）

检索到的文档不一定最相关，用小模型重排提升精度：

```python
# 用 Cohere Re-rank API
import cohere

co = cohere.Client()

def rerank_documents(query: str, docs: list[str], top_n: int = 3) -> list:
    results = co.rerank(
        query=query,
        documents=docs,
        top_n=top_n,
        model="rerank-english-v3.0"
    )
    return [docs[r.index] for r in results.results]
```

---

## 常见坑

| 问题 | 原因 | 解法 |
|------|------|------|
| 检索到不相关内容 | 切块粒度太细，丢失上下文 | 增大 chunk_size 或加 overlap |
| 明明有答案但没检索到 | 查询和文档措辞不同 | 用 Query Expansion 扩展查询词 |
| LLM 忽略检索内容 | System Prompt 没强调 | 明确要求"只基于上下文回答" |
| 检索速度慢 | 向量数据库没加索引 | 用 HNSW 索引（Chroma 默认支持）|
| 内容过时 | 知识库没及时更新 | 加版本字段 + 定时重新索引 |

---

## 小结

RAG = **检索** + **生成**，解决 LLM 的两大痛点：

1. **知识时效性** → 知识库随时更新，模型不需要重训
2. **上下文限制** → 按需检索，只塞进相关内容

在 Agent 中，RAG 最佳实践是封装成工具，让 Agent 自主决定何时检索、检索什么。

**下一步可以探索：**
- GraphRAG（知识图谱 + RAG）
- Agentic RAG（多步检索、自动改写查询）
- Multimodal RAG（图片、表格也能检索）

---

*课程代码参考：[learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | [pi-mono](https://github.com/badlogic/pi-mono)*
