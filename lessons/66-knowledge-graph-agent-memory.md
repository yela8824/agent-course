# 66 - Knowledge Graph for Agent Memory 知识图谱增强记忆

> 向量检索只能找"相似"——知识图谱能找"关联"。  
> 把 Agent 的记忆从"键值对"升级为"关系网络"。

---

## 为什么需要知识图谱？

RAG（第 34 课）靠语义相似度检索，解决了"找到相关段落"的问题。  
但它回答不了这类问题：

- *"用户 A 和用户 B 都购买过哪些共同的皮肤？"*
- *"这个 bug 是哪个 PR 引入的，那个 PR 又改动了哪些文件？"*
- *"Order 最终会流经哪些系统？"*

这些问题的本质是**图遍历**，需要知识图谱（Knowledge Graph）。

```
传统记忆:  key → value  (flat, no relationships)
向量记忆:  embedding → chunk  (semantic, but no structure)
知识图谱:  entity → [relation → entity]  (structured relationships)
```

---

## 核心概念

```
Node (实体)   ──[Edge (关系)]──▶  Node (实体)
User:Alice    ──[purchased]──▶   Item:AK-47 | Asiimov
User:Alice    ──[played]──▶      Battle:room_42
Battle:room_42 ──[contains]──▶  Item:M4A4 | Howl
```

三元组 `(subject, predicate, object)` 是知识图谱的最小单元。

---

## 用 Python 实现一个轻量级 Agent KG

用 `networkx` 作为图引擎，让 Agent 通过工具读写知识图谱。

```python
# kg_agent.py
import json
import networkx as nx
from pathlib import Path

KG_PATH = Path("agent_knowledge.json")

class KnowledgeGraph:
    def __init__(self):
        self.g = nx.DiGraph()
        self._load()

    def _load(self):
        if KG_PATH.exists():
            data = json.loads(KG_PATH.read_text())
            for node, attrs in data["nodes"]:
                self.g.add_node(node, **attrs)
            for src, dst, attrs in data["edges"]:
                self.g.add_edge(src, dst, **attrs)

    def save(self):
        data = {
            "nodes": list(self.g.nodes(data=True)),
            "edges": [(s, d, a) for s, d, a in self.g.edges(data=True)],
        }
        KG_PATH.write_text(json.dumps(data, ensure_ascii=False, indent=2))

    def add_triple(self, subject: str, predicate: str, obj: str, **attrs):
        """添加三元组 (主体, 关系, 客体)"""
        self.g.add_node(subject)
        self.g.add_node(obj)
        self.g.add_edge(subject, obj, relation=predicate, **attrs)
        self.save()
        return f"Added: {subject} --[{predicate}]--> {obj}"

    def query(self, node: str, relation: str | None = None, depth: int = 1):
        """查询节点的邻居，可按关系类型过滤"""
        results = []
        for neighbor in nx.neighbors(self.g, node):
            edge = self.g[node][neighbor]
            rel = edge.get("relation", "unknown")
            if relation is None or rel == relation:
                results.append({
                    "from": node,
                    "relation": rel,
                    "to": neighbor,
                    "attrs": {k: v for k, v in edge.items() if k != "relation"},
                })
        return results

    def find_path(self, source: str, target: str):
        """找两个实体之间的最短关系路径"""
        try:
            path = nx.shortest_path(self.g, source, target)
            steps = []
            for i in range(len(path) - 1):
                rel = self.g[path[i]][path[i+1]].get("relation", "?")
                steps.append(f"{path[i]} --[{rel}]--> {path[i+1]}")
            return " → ".join(steps)
        except nx.NetworkXNoPath:
            return f"No path between {source} and {target}"

    def common_neighbors(self, node_a: str, node_b: str):
        """找两个节点共同指向的邻居（协同过滤基础）"""
        neighbors_a = set(nx.neighbors(self.g, node_a))
        neighbors_b = set(nx.neighbors(self.g, node_b))
        return list(neighbors_a & neighbors_b)


# Agent 工具定义
kg = KnowledgeGraph()

TOOLS = [
    {
        "name": "kg_add",
        "description": "向知识图谱添加一条关系三元组",
        "input_schema": {
            "type": "object",
            "properties": {
                "subject": {"type": "string", "description": "主体实体"},
                "predicate": {"type": "string", "description": "关系类型"},
                "object": {"type": "string", "description": "客体实体"},
            },
            "required": ["subject", "predicate", "object"],
        },
    },
    {
        "name": "kg_query",
        "description": "查询某个实体的所有关联关系",
        "input_schema": {
            "type": "object",
            "properties": {
                "node": {"type": "string"},
                "relation": {"type": "string", "description": "可选，按关系类型过滤"},
            },
            "required": ["node"],
        },
    },
    {
        "name": "kg_path",
        "description": "查找两个实体之间的关系路径",
        "input_schema": {
            "type": "object",
            "properties": {
                "source": {"type": "string"},
                "target": {"type": "string"},
            },
            "required": ["source", "target"],
        },
    },
    {
        "name": "kg_common",
        "description": "找两个实体共同关联的对象（用于推荐、关联分析）",
        "input_schema": {
            "type": "object",
            "properties": {
                "node_a": {"type": "string"},
                "node_b": {"type": "string"},
            },
            "required": ["node_a", "node_b"],
        },
    },
]


def handle_tool(name: str, inputs: dict) -> str:
    if name == "kg_add":
        return kg.add_triple(inputs["subject"], inputs["predicate"], inputs["object"])
    elif name == "kg_query":
        result = kg.query(inputs["node"], inputs.get("relation"))
        return json.dumps(result, ensure_ascii=False)
    elif name == "kg_path":
        return kg.find_path(inputs["source"], inputs["target"])
    elif name == "kg_common":
        return json.dumps(kg.common_neighbors(inputs["node_a"], inputs["node_b"]))
    return "Unknown tool"
```

---

## 在 learn-claude-code 风格的 Agent Loop 中使用

```python
import anthropic

client = anthropic.Anthropic()

def run_kg_agent(user_question: str):
    messages = [{"role": "user", "content": user_question}]
    
    system = """你是一个知识图谱助手。用户问你关系和关联问题时，
先用 kg_query/kg_path/kg_common 查询图谱，
学到新关系时用 kg_add 保存。"""

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1024,
            system=system,
            tools=TOOLS,
            messages=messages,
        )

        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            # 提取文字回复
            for block in response.content:
                if hasattr(block, "text"):
                    print(block.text)
            break

        # 处理工具调用
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = handle_tool(block.name, block.input)
                print(f"[KG Tool] {block.name}({block.input}) → {result}")
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })

        messages.append({"role": "user", "content": tool_results})


# 使用示例
if __name__ == "__main__":
    # 先填充一些数据
    kg.add_triple("User:Alice", "purchased", "Item:AK47-Asiimov")
    kg.add_triple("User:Bob", "purchased", "Item:AK47-Asiimov")
    kg.add_triple("User:Alice", "purchased", "Item:M4A4-Howl")
    kg.add_triple("Battle:room_42", "involves", "User:Alice")
    kg.add_triple("Battle:room_42", "involves", "User:Bob")
    kg.add_triple("Battle:room_42", "used_item", "Item:AWP-Dragon")

    run_kg_agent("Alice 和 Bob 有什么共同购买的物品？他们有在同一个对战房间吗？")
```

**输出示例：**
```
[KG Tool] kg_common({"node_a": "User:Alice", "node_b": "User:Bob"}) → ["Item:AK47-Asiimov"]
[KG Tool] kg_path({"source": "User:Alice", "target": "User:Bob"}) → 
  User:Alice --[involves]--> Battle:room_42 --[involves]--> User:Bob

Alice 和 Bob 都购买了 AK47-Asiimov。
他们通过 Battle:room_42 产生了关联，曾经参与过同一场对战。
```

---

## OpenClaw 中的知识图谱实践

OpenClaw 的 `memory_search` 工具底层是向量相似度。  
要升级到知识图谱，可以在 workspace 里维护一个 JSON 格式的图：

```typescript
// memory/knowledge-graph.json
{
  "nodes": {
    "Project:MysteryBox": { "type": "project", "status": "active" },
    "User:Boss": { "type": "person", "role": "owner" },
    "System:Grafana": { "type": "infra", "url": "http://grafana.prod.skinsmanage.com" }
  },
  "edges": [
    { "from": "User:Boss", "to": "Project:MysteryBox", "relation": "owns" },
    { "from": "Project:MysteryBox", "to": "System:Grafana", "relation": "monitored_by" }
  ]
}
```

Agent 在每次任务结束后，把新发现的关系写入图谱。  
下次回答"MysteryBox 的监控在哪里"时，直接图遍历，无需语义猜测。

---

## pi-mono 中的图谱集成思路

```typescript
// tools/knowledge-graph.ts
import Graph from "graphology";
import { writeFileSync, readFileSync } from "fs";

const KG_PATH = "memory/kg.json";

function loadGraph(): Graph {
  const g = new Graph({ type: "directed" });
  try {
    const data = JSON.parse(readFileSync(KG_PATH, "utf8"));
    g.import(data);
  } catch {}
  return g;
}

export const knowledgeGraphTools = {
  kg_add: {
    description: "添加知识三元组到图谱",
    schema: z.object({
      subject: z.string(),
      predicate: z.string(),
      object: z.string(),
    }),
    execute: async ({ subject, predicate, object }) => {
      const g = loadGraph();
      if (!g.hasNode(subject)) g.addNode(subject);
      if (!g.hasNode(object)) g.addNode(object);
      g.addDirectedEdge(subject, object, { relation: predicate });
      writeFileSync(KG_PATH, JSON.stringify(g.export()));
      return `✓ ${subject} --[${predicate}]--> ${object}`;
    },
  },
  
  kg_traverse: {
    description: "从某节点出发，深度遍历关联关系",
    schema: z.object({
      start: z.string(),
      max_depth: z.number().default(2),
    }),
    execute: async ({ start, max_depth }) => {
      const g = loadGraph();
      const visited = new Set<string>();
      const result: string[] = [];

      function dfs(node: string, depth: number) {
        if (depth > max_depth || visited.has(node)) return;
        visited.add(node);
        g.outEdges(node).forEach(edge => {
          const target = g.target(edge);
          const rel = g.getEdgeAttribute(edge, "relation");
          result.push(`${"  ".repeat(max_depth - depth)}${node} --[${rel}]--> ${target}`);
          dfs(target, depth + 1);
        });
      }

      dfs(start, max_depth);
      return result.join("\n") || `No relations found for ${start}`;
    },
  },
};
```

---

## 向量 + 图谱：混合记忆架构

最强的 Agent 记忆系统是**向量检索 + 图遍历**的组合：

```
用户问题
    │
    ├─▶ 向量检索 → 找到相关的"入口节点"
    │
    └─▶ 图遍历 → 从入口节点展开关联关系
              │
              └─▶ 综合上下文 → 回答
```

```python
def hybrid_search(question: str) -> str:
    # Step 1: 向量找入口
    similar_nodes = vector_store.search(question, top_k=3)
    
    # Step 2: 图遍历展开
    context_parts = []
    for node in similar_nodes:
        relations = kg.query(node["id"])
        context_parts.append(f"Node: {node['id']}")
        for rel in relations:
            context_parts.append(f"  → {rel['relation']}: {rel['to']}")
    
    return "\n".join(context_parts)
```

---

## 什么时候用知识图谱？

| 场景 | 推荐方案 |
|------|---------|
| "找相似内容" | 向量检索（RAG） |
| "找关联关系" | 知识图谱 |
| "跨实体推理" | 知识图谱 + 图遍历 |
| "推荐系统" | 协同过滤 → 知识图谱 |
| "溯源分析" | 事件溯源 + 知识图谱 |

---

## 小结

1. **知识图谱**用三元组 `(subject, predicate, object)` 表达关系
2. **networkx / graphology** 是轻量级实现选择
3. **4 个核心操作**：add_triple、query、find_path、common_neighbors
4. **混合架构**：向量找入口 + 图遍历展开，覆盖所有记忆场景
5. 适合 Agent 的**长期结构化记忆**，补向量检索的短板

> 💡 **一句话记住**：RAG 告诉你"在哪"，知识图谱告诉你"如何连接"。
