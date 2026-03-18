# Agent DAG 执行引擎（Directed Acyclic Graph Execution Engine）

> 用有向无环图建模 Agent 工作流，实现依赖感知的并行执行

---

## 为什么需要 DAG？

大多数 Agent 工作流都有这个问题：

```
任务A → 任务B → 任务C  # 串行，慢
```

但实际上很多任务之间没有依赖关系，完全可以并行：

```
任务A ──┐
任务B ──┼──→ 任务D（汇总）
任务C ──┘
```

**DAG（有向无环图）** 就是解决这个问题的标准方案：
- 每个节点是一个任务（工具调用、子 Agent、LLM 推理）
- 每条边是依赖关系
- 没有环路（避免死锁）
- 拓扑排序决定执行顺序
- 无依赖的节点可以并行执行

---

## 核心概念

```
Node（节点）: 一个可执行单元（工具/Agent/函数）
Edge（边）:   依赖关系 A → B 表示"B 依赖 A 的输出"
DAG:          由节点和边组成的有向无环图
Topology Sort: 拓扑排序，确保执行顺序合法
Wave（波次）: 同一批可以并行执行的节点
```

---

## Python 实现（教学版）

```python
# dag_engine.py
import asyncio
from dataclasses import dataclass, field
from typing import Any, Callable, Awaitable
from collections import defaultdict, deque

@dataclass
class DagNode:
    id: str
    fn: Callable[..., Awaitable[Any]]
    deps: list[str] = field(default_factory=list)  # 依赖的节点 id
    result: Any = None
    error: Exception | None = None

class DagEngine:
    def __init__(self):
        self.nodes: dict[str, DagNode] = {}

    def add_node(self, node: DagNode):
        self.nodes[node.id] = node

    def _topo_waves(self) -> list[list[str]]:
        """拓扑排序，返回执行波次（同波次可并行）"""
        in_degree = {nid: 0 for nid in self.nodes}
        for node in self.nodes.values():
            for dep in node.deps:
                in_degree[node.id] += 1  # 每个依赖+1入度

        # 重新计算：统计有多少节点依赖某节点
        in_degree = defaultdict(int)
        for node in self.nodes.values():
            for dep in node.deps:
                in_degree[node.id] += 1

        waves = []
        remaining = set(self.nodes.keys())

        while remaining:
            # 当前没有未满足依赖的节点
            wave = [
                nid for nid in remaining
                if all(
                    dep not in remaining  # 依赖已完成
                    for dep in self.nodes[nid].deps
                )
            ]
            if not wave:
                raise ValueError(f"DAG 有环路！剩余节点: {remaining}")
            waves.append(wave)
            remaining -= set(wave)

        return waves

    async def run(self) -> dict[str, Any]:
        """执行整个 DAG，返回所有节点结果"""
        waves = self._topo_waves()
        results: dict[str, Any] = {}

        for i, wave in enumerate(waves):
            print(f"⚡ 第 {i+1} 波（并行 {len(wave)} 个节点）: {wave}")

            # 并行执行本波次所有节点
            async def run_node(nid: str):
                node = self.nodes[nid]
                # 把依赖节点的结果作为参数传入
                dep_results = {dep: results[dep] for dep in node.deps}
                try:
                    node.result = await node.fn(**dep_results)
                    results[nid] = node.result
                except Exception as e:
                    node.error = e
                    raise

            await asyncio.gather(*[run_node(nid) for nid in wave])

        return results


# ── 使用示例 ──────────────────────────────────────────
async def fetch_user(user_id: str = "u123") -> dict:
    await asyncio.sleep(0.1)  # 模拟 IO
    return {"id": user_id, "name": "Alice"}

async def fetch_orders() -> list:
    await asyncio.sleep(0.1)
    return [{"id": "o1", "amount": 100}]

async def fetch_balance() -> float:
    await asyncio.sleep(0.1)
    return 999.99

async def generate_report(fetch_user, fetch_orders, fetch_balance) -> str:
    return (
        f"用户: {fetch_user['name']}\n"
        f"订单数: {len(fetch_orders)}\n"
        f"余额: {fetch_balance}"
    )

async def main():
    engine = DagEngine()
    engine.add_node(DagNode("fetch_user",    fetch_user))
    engine.add_node(DagNode("fetch_orders",  fetch_orders))
    engine.add_node(DagNode("fetch_balance", fetch_balance))
    engine.add_node(DagNode(
        "generate_report",
        generate_report,
        deps=["fetch_user", "fetch_orders", "fetch_balance"]
    ))

    results = await engine.run()
    print("\n📊 最终报告:\n", results["generate_report"])

asyncio.run(main())
```

**输出：**
```
⚡ 第 1 波（并行 3 个节点）: ['fetch_user', 'fetch_orders', 'fetch_balance']
⚡ 第 2 波（并行 1 个节点）: ['generate_report']

📊 最终报告:
 用户: Alice
订单数: 1
余额: 999.99
```

三个 fetch 并行执行，总耗时 0.1s，而不是串行的 0.3s。

---

## TypeScript 实现（pi-mono 风格）

```typescript
// dag-engine.ts
type NodeFn<T> = (deps: Record<string, any>) => Promise<T>;

interface DagNode<T = any> {
  id: string;
  fn: NodeFn<T>;
  deps?: string[];
}

class DagEngine {
  private nodes = new Map<string, DagNode>();

  add<T>(node: DagNode<T>): this {
    this.nodes.set(node.id, node);
    return this; // 链式调用
  }

  private topoWaves(): string[][] {
    const completed = new Set<string>();
    const waves: string[][] = [];
    const remaining = new Set(this.nodes.keys());

    while (remaining.size > 0) {
      const wave = [...remaining].filter(id => {
        const node = this.nodes.get(id)!;
        return (node.deps ?? []).every(dep => completed.has(dep));
      });

      if (wave.length === 0) {
        throw new Error(`DAG cycle detected! Remaining: ${[...remaining]}`);
      }

      waves.push(wave);
      wave.forEach(id => {
        remaining.delete(id);
        completed.add(id);
      });
    }

    return waves;
  }

  async run(): Promise<Record<string, any>> {
    const results: Record<string, any> = {};
    const waves = this.topoWaves();

    for (const [i, wave] of waves.entries()) {
      console.log(`⚡ Wave ${i + 1} [${wave.join(', ')}]`);

      await Promise.all(
        wave.map(async (id) => {
          const node = this.nodes.get(id)!;
          const depResults = Object.fromEntries(
            (node.deps ?? []).map(dep => [dep, results[dep]])
          );
          results[id] = await node.fn(depResults);
        })
      );
    }

    return results;
  }
}

// ── 使用示例（Agent 研究工作流）──────────────────────
const engine = new DagEngine()
  .add({ id: "search_web",   fn: async () => ["result1", "result2"] })
  .add({ id: "search_docs",  fn: async () => ["doc1", "doc2"] })
  .add({ id: "search_code",  fn: async () => ["snippet1"] })
  .add({
    id: "synthesize",
    deps: ["search_web", "search_docs", "search_code"],
    fn: async ({ search_web, search_docs, search_code }) => ({
      summary: `综合 ${search_web.length + search_docs.length + search_code.length} 条结果`
    })
  })
  .add({
    id: "write_report",
    deps: ["synthesize"],
    fn: async ({ synthesize }) => `# 报告\n${synthesize.summary}`
  });

const results = await engine.run();
console.log(results.write_report);
```

---

## OpenClaw 中的 DAG 实践

OpenClaw 的工具调用本质上就是一个隐式 DAG：

```
[snapshot] ──→ [act:click] ──→ [snapshot] ──→ [act:type]
                                     ↑
              [navigate] ────────────┘
```

当 Claude 决定并行调用多个工具时，实际上是在执行 DAG 的一个波次：

```json
// Claude 同时调用多个无依赖的工具
[
  {"tool": "web_search", "query": "..."},
  {"tool": "read", "path": "file.md"},
  {"tool": "exec", "command": "git log"}
]
```

这就是 OpenClaw 在 `tool_use` 响应中支持数组的原因——并行执行独立工具。

---

## Agent 工作流 DAG 的高级模式

### 1. 条件分支（Fan-out）

```python
async def route(input_type: str) -> str:
    return "code" if "代码" in input_type else "text"

async def process_code(route: str) -> str: ...
async def process_text(route: str) -> str: ...

# 根据 route 结果动态决定下一步
# 可以在 run() 前根据条件动态构建 DAG
```

### 2. 错误隔离（Partial Failure）

```python
async def run_with_fallback(self) -> dict[str, Any]:
    """允许部分节点失败，记录错误但继续执行"""
    results = {}
    for wave in self._topo_waves():
        tasks = []
        for nid in wave:
            node = self.nodes[nid]
            # 跳过依赖失败的节点
            failed_deps = [
                dep for dep in node.deps
                if self.nodes[dep].error is not None
            ]
            if failed_deps:
                node.error = Exception(f"依赖失败: {failed_deps}")
                continue
            tasks.append(run_node(nid))
        await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

### 3. 进度追踪

```python
class DagEngine:
    async def run(self, on_progress=None) -> dict:
        total = len(self.nodes)
        done = 0
        for wave in self._topo_waves():
            await asyncio.gather(*[run_node(n) for n in wave])
            done += len(wave)
            if on_progress:
                await on_progress(done / total, wave)
```

---

## 什么时候用 DAG？

| 场景 | 推荐方案 |
|------|---------|
| 线性流程（A→B→C） | 简单链式调用 |
| 有并行机会的工作流 | **DAG 引擎** ✅ |
| 复杂研究任务（搜索+分析+总结） | **DAG 引擎** ✅ |
| 多数据源聚合报告 | **DAG 引擎** ✅ |
| 动态生成任务图 | DAG + 动态构建 |

---

## 关键要点

1. **DAG 消除不必要的串行等待**：无依赖的节点并行执行
2. **拓扑排序是核心算法**：保证依赖的正确执行顺序
3. **依赖结果自动注入**：子节点通过参数接收父节点输出
4. **错误传播可控**：可以选择快速失败或容忍部分失败
5. **OpenClaw 已内置**：Claude 并行工具调用就是 DAG 的实践

---

## 参考代码

- `learn-claude-code/tools/parallel.py` - Python 并行工具执行
- `pi-mono/src/agent/workflow.ts` - TypeScript 工作流管理
- OpenClaw 工具层：`tool_use` 数组响应 → 并行执行波次
