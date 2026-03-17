# Lesson 76 - Structured Thinking Patterns
## 结构化思维模式：CoT / ToT / GoT

> 让 Agent 不只是"想"，而是"有结构地想"

---

## 为什么需要结构化思维？

普通 LLM 调用是黑盒：输入 → 输出，中间推理过程不透明，无法干预。

结构化思维模式让推理过程**可见、可控、可分叉**：

```
CoT:  A → B → C → 答案          （线性推理链）
ToT:  A → B1, B2, B3 → 最优     （树形探索）
GoT:  A ↔ B ↔ C → 网状推理      （图结构，节点间相互影响）
```

---

## 1. Chain-of-Thought（思维链）

最基础的结构化思维，强制 LLM "一步步想清楚"。

### 1.1 Zero-shot CoT

```python
# learn-claude-code 风格
def ask_with_cot(client, question: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"{question}\n\n请一步步思考，然后给出最终答案。"
        }]
    )
    return response.content[0].text
```

### 1.2 Few-shot CoT（带示例）

```python
COT_EXAMPLES = """
问题：用户账户余额 100，购买物品价格 75，手续费 2%，够吗？
思考：
1. 手续费 = 75 × 2% = 1.5
2. 总费用 = 75 + 1.5 = 76.5
3. 余额 100 > 76.5 ✓
答案：够，余额还剩 23.5

---
"""

def ask_with_few_shot_cot(client, question: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        system=f"你是精确的分析助手。用以下格式思考：\n{COT_EXAMPLES}",
        messages=[{"role": "user", "content": question}]
    )
    return response.content[0].text
```

### 1.3 Scratchpad CoT（结构化暂存）

将思考和输出分离，更易解析：

```python
from anthropic import Anthropic
import re

client = Anthropic()

def structured_cot(question: str) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"""{question}

请用以下格式回答：
<thinking>
[你的逐步分析过程]
</thinking>
<answer>
[最终答案]
</answer>"""
        }]
    )
    
    text = response.content[0].text
    thinking = re.search(r'<thinking>(.*?)</thinking>', text, re.DOTALL)
    answer = re.search(r'<answer>(.*?)</answer>', text, re.DOTALL)
    
    return {
        "thinking": thinking.group(1).strip() if thinking else "",
        "answer": answer.group(1).strip() if answer else text,
        "raw": text
    }

# 使用 Claude 原生 Extended Thinking（更强大）
def native_extended_thinking(question: str) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=16000,
        thinking={
            "type": "enabled",
            "budget_tokens": 10000  # 给思考过程分配的 token 预算
        },
        messages=[{"role": "user", "content": question}]
    )
    
    thinking_text = ""
    answer_text = ""
    
    for block in response.content:
        if block.type == "thinking":
            thinking_text = block.thinking
        elif block.type == "text":
            answer_text = block.text
    
    return {"thinking": thinking_text, "answer": answer_text}
```

---

## 2. Tree-of-Thought（思维树）

当问题有多种解法时，并行探索多条路径，选最优的。

### 2.1 BFS 风格（广度优先）

```python
import asyncio
from dataclasses import dataclass
from typing import List

@dataclass
class ThoughtNode:
    content: str
    score: float = 0.0
    children: List['ThoughtNode'] = None
    
    def __post_init__(self):
        if self.children is None:
            self.children = []

class TreeOfThought:
    def __init__(self, client, model="claude-sonnet-4-5"):
        self.client = client
        self.model = model
    
    async def generate_thoughts(self, problem: str, context: str, n: int = 3) -> List[str]:
        """生成 n 个候选思路"""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            messages=[{
                "role": "user",
                "content": f"""问题：{problem}
                
当前分析：{context}

请生成 {n} 个不同的下一步思路，每个思路独立成段，用 ---THOUGHT--- 分隔："""
            }]
        )
        
        raw = response.content[0].text
        thoughts = [t.strip() for t in raw.split("---THOUGHT---") if t.strip()]
        return thoughts[:n]
    
    async def evaluate_thought(self, problem: str, thought: str) -> float:
        """评估一个思路的价值（0-10）"""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=100,
            messages=[{
                "role": "user",
                "content": f"""问题：{problem}
思路：{thought}

这个思路有多大可能导向正确答案？打分 0-10，只输出数字："""
            }]
        )
        
        try:
            score = float(response.content[0].text.strip())
            return min(10.0, max(0.0, score))
        except:
            return 5.0
    
    async def solve(self, problem: str, depth: int = 3, width: int = 3) -> str:
        """BFS 树搜索"""
        # 初始思路
        root_thoughts = await self.generate_thoughts(problem, "（开始分析）", width)
        
        best_path = []
        current_level = [(t, t) for t in root_thoughts]  # (当前思路, 完整路径)
        
        for d in range(depth):
            # 评估当前层所有思路
            scored = []
            for thought, path in current_level:
                score = await self.evaluate_thought(problem, path)
                scored.append((score, thought, path))
            
            # 取 top-k
            scored.sort(reverse=True)
            top_k = scored[:width]
            
            if d == depth - 1:
                # 最后一层：选最优路径
                best_path = top_k[0][2]
                break
            
            # 展开下一层
            next_level = []
            for _, thought, path in top_k:
                new_thoughts = await self.generate_thoughts(problem, path, width)
                for new_t in new_thoughts:
                    next_level.append((new_t, f"{path}\n→ {new_t}"))
            
            current_level = next_level
        
        # 基于最优路径生成最终答案
        response = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": f"""问题：{problem}

探索路径：
{best_path}

基于以上推理路径，给出最终答案："""
            }]
        )
        
        return response.content[0].text

# 使用示例
async def main():
    from anthropic import Anthropic
    client = Anthropic()
    tot = TreeOfThought(client)
    
    result = await tot.solve(
        problem="如何设计一个每秒处理 10万 请求的 Agent 调度系统？",
        depth=2,
        width=3
    )
    print(result)
```

### 2.2 DFS 风格（OpenClaw/pi-mono 实战）

```typescript
// pi-mono 风格的 TypeScript 实现
interface ThoughtBranch {
  reasoning: string;
  confidence: number;
  subBranches?: ThoughtBranch[];
}

async function treeSearch(
  client: Anthropic,
  problem: string,
  maxDepth = 3
): Promise<string> {
  async function explore(context: string, depth: number): Promise<ThoughtBranch> {
    if (depth === 0) {
      return { reasoning: context, confidence: 0.5 };
    }

    // 生成候选分支
    const response = await client.messages.create({
      model: "claude-sonnet-4-5",
      max_tokens: 1024,
      messages: [{
        role: "user",
        content: `问题: ${problem}\n当前思路: ${context}\n\n生成2个不同方向的深入分析（JSON格式）：
[{"reasoning": "...", "is_promising": true/false}, ...]`
      }]
    });

    const branches = JSON.parse(extractJSON(response.content[0].text));
    const promisingBranch = branches.find((b: any) => b.is_promising) || branches[0];
    
    // 递归探索最有希望的分支
    const deeper = await explore(promisingBranch.reasoning, depth - 1);
    
    return {
      reasoning: promisingBranch.reasoning,
      confidence: deeper.confidence,
      subBranches: [deeper]
    };
  }

  const tree = await explore("开始分析", maxDepth);
  return tree.reasoning;
}
```

---

## 3. Graph-of-Thought（思维图）

最复杂的模式，节点之间可以互相引用和更新。适合需要**多角度交叉验证**的复杂推理。

```python
from typing import Dict, Set
import json

class GraphOfThought:
    """
    思维节点可以互相引用，形成网状推理结构
    适用于：安全审计、复杂 Bug 分析、多维度决策
    """
    
    def __init__(self, client):
        self.client = client
        self.nodes: Dict[str, dict] = {}
        self.edges: Set[tuple] = set()
    
    def add_node(self, node_id: str, content: str, node_type: str = "thought"):
        self.nodes[node_id] = {
            "id": node_id,
            "content": content,
            "type": node_type,  # thought / evidence / conclusion
            "refined": False
        }
    
    def add_edge(self, from_id: str, to_id: str, relation: str = "supports"):
        self.edges.add((from_id, to_id, relation))
    
    def get_neighbors(self, node_id: str) -> List[dict]:
        neighbors = []
        for (f, t, rel) in self.edges:
            if f == node_id or t == node_id:
                other = t if f == node_id else f
                if other in self.nodes:
                    neighbors.append({**self.nodes[other], "relation": rel})
        return neighbors
    
    def refine_node(self, node_id: str, problem: str) -> str:
        """根据相邻节点的信息精炼当前节点"""
        node = self.nodes[node_id]
        neighbors = self.get_neighbors(node_id)
        
        context = "\n".join([
            f"- [{n['type']}] {n['content']} (关系: {n['relation']})"
            for n in neighbors
        ])
        
        response = self.client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=512,
            messages=[{
                "role": "user",
                "content": f"""问题：{problem}

当前思维节点：{node['content']}

相关节点信息：
{context}

综合以上信息，精炼这个思维节点（可以修正、增强或验证）："""
            }]
        )
        
        refined = response.content[0].text
        self.nodes[node_id]["content"] = refined
        self.nodes[node_id]["refined"] = True
        return refined
    
    def solve(self, problem: str, iterations: int = 2) -> str:
        """多轮迭代精炼所有节点"""
        
        # Step 1: 初始化基础节点
        initial_response = self.client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": f"""{problem}

请从以下3个维度初步分析，每个维度独立成段：
1. 技术可行性
2. 潜在风险  
3. 实施成本

用 JSON 格式输出：
{{"feasibility": "...", "risks": "...", "cost": "..."}}"""
            }]
        )
        
        data = json.loads(extract_json(initial_response.content[0].text))
        
        self.add_node("n1", data["feasibility"], "thought")
        self.add_node("n2", data["risks"], "thought") 
        self.add_node("n3", data["cost"], "thought")
        
        # 建立关联边（风险影响成本，可行性影响风险）
        self.add_edge("n1", "n2", "impacts")
        self.add_edge("n2", "n3", "increases")
        self.add_edge("n1", "n3", "determines")
        
        # Step 2: 多轮精炼
        for _ in range(iterations):
            for node_id in list(self.nodes.keys()):
                self.refine_node(node_id, problem)
        
        # Step 3: 综合结论
        final_context = "\n".join([
            f"[{n['type']}] {n['content']}"
            for n in self.nodes.values()
        ])
        
        conclusion = self.client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": f"""问题：{problem}

精炼后的多维分析：
{final_context}

综合以上图状推理，给出最终建议："""
            }]
        )
        
        return conclusion.content[0].text
```

---

## 4. 在 OpenClaw Agent 中的实际应用

```typescript
// OpenClaw 工具调用前的 CoT 决策层
async function agentWithCoT(userMessage: string): Promise<void> {
  // 第一步：结构化思考应该调用哪些工具
  const planningResponse = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 1024,
    thinking: { type: "enabled", budget_tokens: 5000 },
    system: `你是 Agent 规划器。分析用户请求，决定需要哪些工具。
输出格式：
<plan>
1. 需要工具 X 因为 Y
2. 需要工具 Z 因为 W
</plan>`,
    messages: [{ role: "user", content: userMessage }]
  });
  
  // 提取计划
  const plan = extractPlan(planningResponse.content);
  
  // 第二步：按计划执行工具
  // ...执行阶段
  
  // 第三步：ToT 风格验证结果
  const verification = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 512,
    messages: [{
      role: "user",
      content: `计划：${plan}\n执行结果：${results}\n
是否达成用户目标？如未达成，哪里需要调整？`
    }]
  });
}
```

---

## 5. 选哪种模式？

| 场景 | 推荐模式 | 理由 |
|------|---------|------|
| 数学/逻辑推理 | CoT (Extended Thinking) | 线性，精确 |
| 方案设计/创意 | ToT | 需要探索多个方向 |
| 复杂 Bug 分析 | GoT | 多角度交叉验证 |
| 实时 Agent 决策 | CoT (快速) | 延迟敏感 |
| 离线批量分析 | ToT / GoT | 准确度优先 |

---

## 关键原则

1. **Claude Extended Thinking 是最简单的 CoT** — 不需要手写 prompt，直接开启
2. **ToT 有成本** — 探索 3 条路径 × 3 层 = 至少 9 次 LLM 调用
3. **GoT 适合验证型任务** — 多角度互相校验，但实现复杂
4. **可以组合使用** — CoT 内嵌于 ToT 的每个节点，效果最好
5. **记录推理路径** — `thinking` 块的内容要存 log，方便调试

---

## 参考

- [Anthropic Extended Thinking Docs](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- [Tree of Thoughts Paper](https://arxiv.org/abs/2305.10601)
- [Graph of Thoughts Paper](https://arxiv.org/abs/2308.09687)
- learn-claude-code: `examples/extended_thinking/`
