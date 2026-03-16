# 67 - Semantic Tool Selection：语义工具选择

> 当 Agent 有 50+ 工具时，不要把所有工具都塞进 prompt——用向量相似度预筛，只给 LLM 最相关的那几个。

---

## 问题：工具爆炸

一个生产级 Agent 很容易积累几十上百个工具：

```
文件操作: read, write, edit, delete, move, copy, search_files...
网络请求: http_get, http_post, fetch_html, web_search...
代码执行: run_python, run_bash, run_node, run_sql...
数据库:   db_query, db_insert, db_update, db_schema...
通知:     send_email, send_slack, send_telegram, send_sms...
```

全部塞进 system prompt 会带来三个问题：

1. **Token 爆炸**：每次调用多花几千 token 描述没用的工具
2. **LLM 困惑**：工具太多，LLM 更容易选错或混淆
3. **延迟增加**：更大的 prompt → 更慢的首 token 时间

**解法**：在请求 LLM 之前，先用语义相似度筛出最相关的 K 个工具。

---

## 核心思路

```
用户输入 → 向量化
                ↓
        与工具库向量做相似度搜索
                ↓
        取 Top-K 相关工具 (如 K=8)
                ↓
        只把这 K 个工具传给 LLM
                ↓
        LLM 在小工具集中选择并调用
```

每个工具都有一个「工具描述向量」，预先计算好存入向量索引。每次用户发来消息，实时把消息向量化后做近似近邻查找。

---

## Python 实现（learn-claude-code 风格）

### 1. 工具注册与向量化

```python
# tool_registry.py
import numpy as np
from dataclasses import dataclass, field
from typing import Callable, Any
import json

@dataclass
class Tool:
    name: str
    description: str
    parameters: dict
    fn: Callable
    # 运行时填充
    embedding: np.ndarray | None = field(default=None, repr=False)

class ToolRegistry:
    def __init__(self, embed_fn: Callable[[str], np.ndarray]):
        self.tools: dict[str, Tool] = {}
        self.embed_fn = embed_fn
        # 向量矩阵（N × dim），用于批量相似度计算
        self._matrix: np.ndarray | None = None
        self._tool_names: list[str] = []

    def register(self, tool: Tool):
        """注册工具，同时计算其描述向量"""
        # 拼接 name + description 作为语义锚点
        text = f"{tool.name}: {tool.description}"
        tool.embedding = self.embed_fn(text)
        self.tools[tool.name] = tool
        self._matrix = None  # 缓存失效

    def _build_matrix(self):
        """懒构建向量矩阵"""
        self._tool_names = list(self.tools.keys())
        embeddings = [self.tools[n].embedding for n in self._tool_names]
        self._matrix = np.stack(embeddings)  # (N, dim)
        # L2 归一化，便于用点积代替 cosine
        norms = np.linalg.norm(self._matrix, axis=1, keepdims=True)
        self._matrix = self._matrix / (norms + 1e-8)

    def select_tools(self, query: str, top_k: int = 8) -> list[Tool]:
        """根据用户输入语义选出最相关的 K 个工具"""
        if self._matrix is None:
            self._build_matrix()

        query_vec = self.embed_fn(query)
        # 归一化查询向量
        query_vec = query_vec / (np.linalg.norm(query_vec) + 1e-8)

        # 点积 = cosine 相似度（因为都已归一化）
        scores = self._matrix @ query_vec          # (N,)
        top_indices = np.argsort(scores)[::-1][:top_k]

        return [self.tools[self._tool_names[i]] for i in top_indices]

    def to_claude_tools(self, selected: list[Tool]) -> list[dict]:
        """转为 Claude API 的 tools 格式"""
        return [
            {
                "name": t.name,
                "description": t.description,
                "input_schema": t.parameters,
            }
            for t in selected
        ]
```

### 2. 轻量 Embed 函数（无需外部 API）

生产环境推荐 `sentence-transformers`，快速原型可用简单的 TF-IDF：

```python
# embedder.py
from sentence_transformers import SentenceTransformer
import numpy as np

# 轻量模型，~80MB，CPU 上 <10ms/次
_model = SentenceTransformer("all-MiniLM-L6-v2")

def embed(text: str) -> np.ndarray:
    return _model.encode(text, normalize_embeddings=True)
```

如果不想引入 ML 依赖，可用 OpenAI / Anthropic 的 embedding API：

```python
import anthropic

client = anthropic.Anthropic()

def embed_via_api(text: str) -> np.ndarray:
    # 注意：Anthropic 目前没有独立 embedding 端点
    # 用 OpenAI text-embedding-3-small 更经济
    from openai import OpenAI
    oai = OpenAI()
    resp = oai.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return np.array(resp.data[0].embedding)
```

### 3. 在 Agent Loop 中使用

```python
# agent.py
import anthropic
from tool_registry import ToolRegistry, Tool
from embedder import embed

registry = ToolRegistry(embed_fn=embed)

# 注册所有工具（启动时一次性完成）
registry.register(Tool(
    name="read_file",
    description="Read the contents of a file from disk",
    parameters={"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"]},
    fn=lambda path: open(path).read()
))
registry.register(Tool(
    name="web_search",
    description="Search the web using Brave Search API",
    parameters={"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]},
    fn=lambda query: search_web(query)
))
# ... 注册更多工具 ...

client = anthropic.Anthropic()

def run_agent(user_message: str):
    # 1. 语义筛选：从全部工具中选出 Top-8
    selected = registry.select_tools(user_message, top_k=8)
    tools = registry.to_claude_tools(selected)

    print(f"🔍 从 {len(registry.tools)} 个工具中选出 {len(selected)} 个:")
    for t in selected:
        print(f"   - {t.name}")

    # 2. 用筛出的工具调用 LLM（prompt 更小，更精准）
    messages = [{"role": "user", "content": user_message}]
    
    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            return response.content[0].text

        # 处理工具调用
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                tool = registry.tools.get(block.name)
                result = tool.fn(**block.input) if tool else "Tool not found"
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result),
                })

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
```

---

## TypeScript 实现（pi-mono 风格）

```typescript
// tool-selector.ts
import Anthropic from "@anthropic-ai/sdk";

interface ToolDef {
  name: string;
  description: string;
  inputSchema: object;
  fn: (input: Record<string, unknown>) => Promise<unknown>;
  embedding?: number[];
}

class SemanticToolSelector {
  private tools: Map<string, ToolDef> = new Map();
  private matrix: number[][] = [];
  private toolNames: string[] = [];
  private dirty = true;

  constructor(private embedFn: (text: string) => Promise<number[]>) {}

  async register(tool: ToolDef): Promise<void> {
    const text = `${tool.name}: ${tool.description}`;
    tool.embedding = await this.embedFn(text);
    this.tools.set(tool.name, tool);
    this.dirty = true;
  }

  async selectTools(query: string, topK = 8): Promise<ToolDef[]> {
    if (this.dirty) this.buildMatrix();

    const queryVec = normalize(await this.embedFn(query));
    const scores = this.matrix.map((row) => dot(row, queryVec));

    return scores
      .map((score, i) => ({ score, name: this.toolNames[i] }))
      .sort((a, b) => b.score - a.score)
      .slice(0, topK)
      .map(({ name }) => this.tools.get(name)!);
  }

  private buildMatrix(): void {
    this.toolNames = [...this.tools.keys()];
    this.matrix = this.toolNames.map((name) =>
      normalize(this.tools.get(name)!.embedding!)
    );
    this.dirty = false;
  }

  toAnthropicTools(tools: ToolDef[]): Anthropic.Tool[] {
    return tools.map((t) => ({
      name: t.name,
      description: t.description,
      input_schema: t.inputSchema as Anthropic.Tool.InputSchema,
    }));
  }
}

function dot(a: number[], b: number[]): number {
  return a.reduce((sum, v, i) => sum + v * b[i], 0);
}

function normalize(v: number[]): number[] {
  const norm = Math.sqrt(v.reduce((s, x) => s + x * x, 0)) + 1e-8;
  return v.map((x) => x / norm);
}
```

---

## OpenClaw 中的实际应用

OpenClaw 目前有 **35+ 工具**（read, write, exec, browser, canvas, nodes, cron, message, memory...），并不是每次都全部传给 Claude。

在 `skillsystem` 里有类似的两层筛选逻辑：

1. **Policy 过滤**：根据权限和配置，移除不可用工具
2. **Skill 注入**：根据 skill 匹配，动态补充 skill-specific 工具

**完整的语义筛选**是下一步优化方向：当工具数超过阈值时，先跑一次轻量 embed 查询，把与当前消息不相关的工具剔除，再传给主模型。

---

## 性能对比

| 场景 | 工具数量 | 工具 Token 数 | 平均延迟 |
|------|----------|--------------|---------|
| 无筛选（全量传递） | 60 | ~8,000 | 2.1s |
| 语义筛选 Top-8 | 8 | ~1,100 | 0.95s |
| 节省 | 87% 减少 | 86% 减少 | 55% 更快 |

*数据来自 [LangChain Tool Selection Benchmark 2024]*

---

## 进阶：分层工具树

当工具超过 200 个时，两级筛选更合适：

```
第一层：工具类别（粗筛）
  ├── 文件操作 (12个工具)
  ├── 网络请求 (8个工具)
  ├── 数据库   (15个工具)
  └── 通知发送 (6个工具)

第二层：类别内精筛 → Top-K
```

```python
class HierarchicalToolSelector:
    def __init__(self):
        self.categories: dict[str, ToolRegistry] = {}
        self.category_registry = ToolRegistry(embed)  # 类别级别的 registry

    def select_tools(self, query: str, top_categories=3, top_tools_per_cat=3):
        # 先选出最相关的类别
        relevant_cats = self.category_registry.select_tools(query, top_k=top_categories)
        # 再从每个类别中选出最相关的工具
        result = []
        for cat in relevant_cats:
            cat_registry = self.categories[cat.name]
            result.extend(cat_registry.select_tools(query, top_k=top_tools_per_cat))
        return result
```

---

## Always Include（必选工具）

有些工具无论如何都应该出现，比如：

```python
ALWAYS_INCLUDE = {"finish_task", "ask_clarification", "error_report"}

def select_tools(query: str, top_k: int = 8) -> list[Tool]:
    semantic_selected = registry.select_tools(query, top_k=top_k)
    # 确保必选工具始终包含
    always = [registry.tools[n] for n in ALWAYS_INCLUDE if n in registry.tools]
    # 去重合并
    seen = {t.name for t in always}
    result = always + [t for t in semantic_selected if t.name not in seen]
    return result[:top_k + len(ALWAYS_INCLUDE)]
```

---

## 关键要点

- **预计算 embedding**：工具描述不变，启动时算好，请求时只算用户输入
- **归一化后用点积**：等价于 cosine，比欧氏距离快
- **Top-K 不要太大**：8-12 个工具是甜蜜点，再多边际收益递减
- **Always Include**：关键的控制流工具（如 finish/error）不要被筛掉
- **监控选中率**：记录哪些工具经常/从不被选中，优化描述质量

---

## 下节预告

**#68 - Agent 输出格式协商（Output Format Negotiation）**：Agent 如何根据调用方（API/CLI/聊天/语音）动态调整输出格式，结构化 vs. 自然语言的运行时切换。
