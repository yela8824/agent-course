# 57. Agent Behavior Trees（行为树）

> 比有限状态机更灵活、更可复用的 Agent 行为建模方式

---

## 为什么需要行为树？

上一节（[40-agent-fsm](40-agent-fsm.md)）讲了有限状态机（FSM）。FSM 很直观，但当状态变多时会变成"状态爆炸"——每对状态之间都可能需要转换规则，维护成本指数级上升。

**行为树（Behavior Tree / BT）** 来自游戏 AI 领域（Halo、《全境封锁》等大作都用它控制 NPC），它把 Agent 的行为组织成一棵**可复用、可组合、易调试**的树结构。

```
         Root
          │
       Selector  ← 有 Task 就处理，否则 Idle
      ┌──┴──┐
   Sequence  Idle
  ┌──┬──┐
验证 执行 汇报
```

---

## 核心节点类型

```
┌─────────────┬──────────────────────────────────────────────┐
│ 节点类型    │ 语义                                          │
├─────────────┼──────────────────────────────────────────────┤
│ Sequence    │ 所有子节点成功才算成功（AND 逻辑）            │
│ Selector    │ 任一子节点成功即成功（OR 逻辑）               │
│ Parallel    │ 并发执行所有子节点                            │
│ Decorator   │ 包装单个子节点，修改其行为（重试/取反/限速）  │
│ Leaf/Action │ 实际执行的动作（工具调用、LLM 调用）         │
│ Condition   │ 检查条件，返回 Success/Failure               │
└─────────────┴──────────────────────────────────────────────┘
```

每个节点只返回三种状态：
- ✅ **Success** — 完成了
- ❌ **Failure** — 失败了
- ⏳ **Running** — 还在执行中

---

## Python 实现：最小行为树引擎

```python
# behavior_tree.py
from abc import ABC, abstractmethod
from enum import Enum
from typing import Any, Dict, List, Optional
import asyncio

class Status(Enum):
    SUCCESS = "success"
    FAILURE = "failure"
    RUNNING = "running"

class BlackBoard:
    """共享黑板 - Agent 的工作内存"""
    def __init__(self):
        self._data: Dict[str, Any] = {}
    
    def get(self, key: str, default=None):
        return self._data.get(key, default)
    
    def set(self, key: str, value: Any):
        self._data[key] = value
    
    def has(self, key: str) -> bool:
        return key in self._data

class Node(ABC):
    """行为树节点基类"""
    def __init__(self, name: str):
        self.name = name
    
    @abstractmethod
    async def tick(self, bb: BlackBoard) -> Status:
        """执行一次节点逻辑"""
        pass

# ── 组合节点 ──

class Sequence(Node):
    """顺序节点：所有子节点都成功才成功"""
    def __init__(self, name: str, children: List[Node]):
        super().__init__(name)
        self.children = children
    
    async def tick(self, bb: BlackBoard) -> Status:
        for child in self.children:
            status = await child.tick(bb)
            if status != Status.SUCCESS:
                return status  # FAILURE 或 RUNNING 立即返回
        return Status.SUCCESS

class Selector(Node):
    """选择节点：任一子节点成功即成功"""
    def __init__(self, name: str, children: List[Node]):
        super().__init__(name)
        self.children = children
    
    async def tick(self, bb: BlackBoard) -> Status:
        for child in self.children:
            status = await child.tick(bb)
            if status != Status.FAILURE:
                return status  # SUCCESS 或 RUNNING 立即返回
        return Status.FAILURE

class Parallel(Node):
    """并行节点：同时执行所有子节点"""
    def __init__(self, name: str, children: List[Node], 
                 success_threshold: int = None):
        super().__init__(name)
        self.children = children
        # 需要多少个子节点成功才算整体成功
        self.success_threshold = success_threshold or len(children)
    
    async def tick(self, bb: BlackBoard) -> Status:
        results = await asyncio.gather(
            *[child.tick(bb) for child in self.children],
            return_exceptions=True
        )
        success_count = sum(1 for r in results if r == Status.SUCCESS)
        if success_count >= self.success_threshold:
            return Status.SUCCESS
        if any(r == Status.RUNNING for r in results):
            return Status.RUNNING
        return Status.FAILURE

# ── 装饰器节点 ──

class Retry(Node):
    """重试装饰器：子节点失败时重试"""
    def __init__(self, name: str, child: Node, max_retries: int = 3):
        super().__init__(name)
        self.child = child
        self.max_retries = max_retries
    
    async def tick(self, bb: BlackBoard) -> Status:
        for attempt in range(self.max_retries):
            status = await self.child.tick(bb)
            if status == Status.SUCCESS:
                return Status.SUCCESS
            if status == Status.RUNNING:
                return Status.RUNNING
            # FAILURE: 继续重试
            print(f"[{self.name}] 第 {attempt+1} 次失败，重试...")
        return Status.FAILURE

class Invert(Node):
    """取反装饰器：SUCCESS↔FAILURE"""
    def __init__(self, child: Node):
        super().__init__(f"NOT({child.name})")
        self.child = child
    
    async def tick(self, bb: BlackBoard) -> Status:
        status = await self.child.tick(bb)
        if status == Status.SUCCESS:
            return Status.FAILURE
        if status == Status.FAILURE:
            return Status.SUCCESS
        return status  # RUNNING 不变

# ── 叶节点 ──

class Condition(Node):
    """条件节点：检查 BlackBoard 中的值"""
    def __init__(self, name: str, key: str, expected=None):
        super().__init__(name)
        self.key = key
        self.expected = expected
    
    async def tick(self, bb: BlackBoard) -> Status:
        value = bb.get(self.key)
        if self.expected is not None:
            return Status.SUCCESS if value == self.expected else Status.FAILURE
        return Status.SUCCESS if value else Status.FAILURE

class Action(Node):
    """动作节点：执行具体操作"""
    def __init__(self, name: str, func):
        super().__init__(name)
        self.func = func
    
    async def tick(self, bb: BlackBoard) -> Status:
        try:
            result = await self.func(bb)
            return Status.SUCCESS if result is not False else Status.FAILURE
        except Exception as e:
            print(f"[{self.name}] 执行失败: {e}")
            return Status.FAILURE
```

---

## 实战：用行为树构建 Agent 任务处理器

```python
# agent_bt.py
import anthropic
import json

client = anthropic.Anthropic()

# ── 定义 Agent 的工具 ──

tools = [
    {
        "name": "web_search",
        "description": "搜索互联网",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"]
        }
    },
    {
        "name": "write_file",
        "description": "写文件",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["path", "content"]
        }
    }
]

# ── 定义 Action 节点 ──

async def validate_task(bb: BlackBoard) -> bool:
    task = bb.get("task")
    if not task or len(task) < 5:
        print("任务太短，拒绝执行")
        return False
    bb.set("validated", True)
    return True

async def call_llm(bb: BlackBoard) -> bool:
    messages = bb.get("messages", [])
    task = bb.get("task")
    
    if not messages:
        messages = [{"role": "user", "content": task}]
    
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        tools=tools,
        messages=messages
    )
    
    bb.set("response", response)
    bb.set("messages", messages + [{"role": "assistant", "content": response.content}])
    
    # 判断是否需要工具调用
    bb.set("needs_tool", response.stop_reason == "tool_use")
    return True

async def execute_tools(bb: BlackBoard) -> bool:
    response = bb.get("response")
    tool_results = []
    
    for block in response.content:
        if block.type == "tool_use":
            result = await dispatch_tool(block.name, block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": json.dumps(result)
            })
    
    messages = bb.get("messages", [])
    messages.append({"role": "user", "content": tool_results})
    bb.set("messages", messages)
    bb.set("tool_executed", True)
    return True

async def extract_result(bb: BlackBoard) -> bool:
    response = bb.get("response")
    for block in response.content:
        if hasattr(block, "text"):
            bb.set("final_answer", block.text)
            return True
    return False

async def dispatch_tool(name: str, input_data: dict) -> dict:
    """实际工具分发（简化版）"""
    if name == "web_search":
        return {"results": f"搜索 '{input_data['query']}' 的结果..."}
    if name == "write_file":
        return {"success": True, "path": input_data["path"]}
    return {"error": f"未知工具: {name}"}

# ── 构建行为树 ──

def build_agent_tree() -> Node:
    """
    Agent 行为树结构：
    
    Root (Selector)
    ├── 完整任务流程 (Sequence)
    │   ├── 验证任务 (Action)
    │   ├── 重试LLM调用 (Retry)
    │   │   └── 调用LLM (Action)  
    │   ├── 工具执行循环 (Retry/Sequence)
    │   │   ├── 需要工具? (Condition)
    │   │   ├── 执行工具 (Action)
    │   │   └── 再次调用LLM (Action)
    │   └── 提取结果 (Action)
    └── 降级处理 (Action)  ← 整个流程失败时的兜底
    """
    
    # 工具执行子树
    tool_loop = Sequence("工具执行循环", [
        Condition("需要工具?", "needs_tool", expected=True),
        Action("执行工具", execute_tools),
        Action("调用LLM(续)", call_llm),
    ])
    
    # 主流程
    main_flow = Sequence("主流程", [
        Action("验证任务", validate_task),
        Retry("重试LLM调用", Action("首次LLM调用", call_llm), max_retries=3),
        Selector("工具或完成", [
            Invert(Condition("需要工具?", "needs_tool", expected=True)),  # 不需要工具直接成功
            Retry("工具执行", tool_loop, max_retries=5),  # 最多5轮工具调用
        ]),
        Action("提取结果", extract_result),
    ])
    
    # 降级处理
    async def fallback(bb: BlackBoard):
        bb.set("final_answer", "抱歉，任务执行失败，请重试。")
        return True
    
    return Selector("Agent根节点", [
        main_flow,
        Action("降级处理", fallback),
    ])

# ── 运行 ──

async def run_agent(task: str) -> str:
    bb = BlackBoard()
    bb.set("task", task)
    
    tree = build_agent_tree()
    status = await tree.tick(bb)
    
    print(f"行为树执行状态: {status.value}")
    return bb.get("final_answer", "无结果")

# 使用示例
# result = asyncio.run(run_agent("搜索 Python asyncio 最佳实践并整理成文档"))
```

---

## OpenClaw 中的行为树模式

OpenClaw 的技能加载机制本质上就是一棵隐式的行为树：

```
处理消息 (Selector)
├── 匹配技能 (Sequence)
│   ├── 扫描 available_skills (Condition)
│   ├── 选择最匹配的技能 (Action)
│   └── 加载并执行 SKILL.md (Action)
└── 直接LLM回答 (Action) ← 兜底
```

你可以把 `AGENTS.md` 里的「Know When to Speak」逻辑也建模成行为树：

```python
speak_decision_tree = Selector("是否回复", [
    Sequence("直接被@", [
        Condition("被@提及?", "is_mentioned"),
        Action("生成回复", generate_reply),
    ]),
    Sequence("有价值内容", [
        Condition("群聊中?", "is_group_chat"),
        Invert(Condition("已有人回答?", "already_answered")),
        Action("评估价值", evaluate_value),
        Action("生成回复", generate_reply),
    ]),
    Action("保持沉默", lambda bb: bb.set("reply", None)),
])
```

---

## FSM vs 行为树：怎么选？

| 维度         | FSM                     | 行为树                        |
|--------------|-------------------------|-------------------------------|
| **状态数量** | 少（<10）时清晰         | 多状态时不爆炸                |
| **复用性**   | 差（状态强耦合）        | 高（子树可复用）              |
| **并发行为** | 需要并行 FSM，复杂      | Parallel 节点原生支持         |
| **调试**     | 需要追踪当前状态        | 树形结构，每个节点独立可视化  |
| **适用场景** | 简单流程、协议状态机    | 复杂 Agent 行为、游戏 NPC     |

**经验法则：**
- 3-5 个状态？用 FSM
- 10+ 个状态，有复用需求？用行为树
- 需要优雅降级 + 重试？行为树的 Retry + Selector 天生支持

---

## 进阶：行为树 + LLM 动态生成

更高级的用法——让 LLM 在运行时动态构建行为树节点：

```python
async def llm_dynamic_node(bb: BlackBoard) -> bool:
    """LLM 决定下一步执行哪个子树"""
    context = bb.get("context")
    
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"""当前上下文: {context}
            
可用动作: [search, write, analyze, summarize]
选择下一个最合适的动作（只返回动作名）:"""
        }]
    )
    
    action_name = response.content[0].text.strip()
    bb.set("next_action", action_name)
    return True

# 然后用 Condition 节点路由到对应子树
dynamic_router = Selector("动态路由", [
    Sequence("搜索路径", [
        Condition("选了搜索?", "next_action", "search"),
        Action("执行搜索", do_search),
    ]),
    Sequence("写作路径", [
        Condition("选了写作?", "next_action", "write"),
        Action("执行写作", do_write),
    ]),
    # ...
])
```

这就是现代 Agent 框架（LangGraph、AutoGen）的核心思路：**LLM 作为行为树的决策节点**。

---

## 总结

```
行为树三大优势：
1. 📦 模块化 — 子树可以独立开发、测试、复用
2. 🔄 弹性 — Retry + Selector 天然支持降级和重试
3. 🔍 可观测 — 每个节点都有明确的 Success/Failure/Running 状态
```

**下节预告：** Agent 中的背压与流量控制（Backpressure & Flow Control）——当工具调用堆积时如何优雅处理。
