# 87 - Agent 协作记忆黑板模式（Collaborative Blackboard Pattern）

> 多个 Agent 围着一块"黑板"写写擦擦，协同完成单个 Agent 搞不定的复杂任务。

---

## 🧠 核心思想

黑板模式（Blackboard Pattern）来自 AI 经典架构，原意是：多个"专家"（Knowledge Sources）围着一块共享黑板协作——谁能回答就上去写，其他人看到黑板变化后继续贡献。

放到多 Agent 系统里，黑板就是一块**共享的结构化状态空间**：

```
┌─────────────────────────────────────────┐
│            BLACKBOARD (共享黑板)          │
│                                         │
│  task: "分析Q1财报并生成摘要"              │
│  status: in_progress                    │
│  slots:                                 │
│    raw_data: [已填 by DataAgent]         │
│    analysis: [已填 by AnalystAgent]      │
│    summary:  [待填]                      │
│    review:   [待填]                      │
└─────────────────────────────────────────┘
        ↑           ↑           ↑
   DataAgent   AnalystAgent  WriterAgent
```

每个 Agent 只关注自己能填的 slots，填完自己那份，下一个 Agent 自然接力。

---

## 🆚 黑板 vs 其他协作模式

| 模式 | 耦合度 | 适用场景 |
|------|--------|----------|
| Pipeline（流水线） | 顺序强耦合 | 固定步骤，顺序明确 |
| Fan-out/Reduce | 并发无状态 | 独立子任务聚合 |
| **Blackboard（黑板）** | **松耦合** | **步骤不固定、Agent 可选参与** |
| Negotiation（协商） | 对等通信 | Agent 需要达成共识 |

黑板的核心优势：**任何 Agent 随时加入、任何 Agent 随时贡献**，不需要预先定义完整的执行图。

---

## 💻 实现：Python 版（learn-claude-code 风格）

### 1. 黑板数据结构

```python
# blackboard.py
import json
import time
import threading
from pathlib import Path
from typing import Any, Optional, Callable
from dataclasses import dataclass, field, asdict

@dataclass
class BlackboardEntry:
    value: Any
    written_by: str          # 写入者的 agent_id
    timestamp: float = field(default_factory=time.time)
    version: int = 1         # 乐观锁版本号

@dataclass
class Blackboard:
    task_id: str
    task_description: str
    status: str = "pending"  # pending / in_progress / completed / failed
    slots: dict[str, BlackboardEntry] = field(default_factory=dict)
    watchers: list[str] = field(default_factory=list)  # 订阅变更的 agent_id
    errors: list[str] = field(default_factory=list)

class BlackboardStore:
    """线程安全的黑板存储，支持文件持久化"""
    
    def __init__(self, persist_path: Optional[Path] = None):
        self._boards: dict[str, Blackboard] = {}
        self._lock = threading.RLock()
        self._subscribers: dict[str, list[Callable]] = {}  # slot → callbacks
        self._persist_path = persist_path
        
        if persist_path and persist_path.exists():
            self._load()
    
    def create(self, task_id: str, description: str) -> Blackboard:
        with self._lock:
            board = Blackboard(task_id=task_id, task_description=description)
            self._boards[task_id] = board
            self._save()
            return board
    
    def read_slot(self, task_id: str, slot: str) -> Optional[BlackboardEntry]:
        with self._lock:
            board = self._boards.get(task_id)
            return board.slots.get(slot) if board else None
    
    def write_slot(self, task_id: str, slot: str, value: Any, 
                   agent_id: str, expected_version: int = 0) -> bool:
        """乐观锁写入：expected_version=0 表示新建，否则必须匹配当前版本"""
        with self._lock:
            board = self._boards.get(task_id)
            if not board:
                return False
            
            existing = board.slots.get(slot)
            
            # 乐观锁检查
            if existing and existing.version != expected_version:
                return False  # 版本冲突，写入失败
            
            new_version = (existing.version + 1) if existing else 1
            board.slots[slot] = BlackboardEntry(
                value=value,
                written_by=agent_id,
                version=new_version
            )
            self._save()
            
            # 通知订阅者
            self._notify(task_id, slot, value)
            return True
    
    def subscribe(self, task_id: str, slot: str, callback: Callable):
        key = f"{task_id}:{slot}"
        self._subscribers.setdefault(key, []).append(callback)
    
    def _notify(self, task_id: str, slot: str, value: Any):
        key = f"{task_id}:{slot}"
        for cb in self._subscribers.get(key, []):
            threading.Thread(target=cb, args=(value,), daemon=True).start()
    
    def _save(self):
        if self._persist_path:
            data = {
                tid: {
                    "task_id": b.task_id,
                    "task_description": b.task_description,
                    "status": b.status,
                    "slots": {
                        k: {"value": e.value, "written_by": e.written_by,
                            "timestamp": e.timestamp, "version": e.version}
                        for k, e in b.slots.items()
                    }
                }
                for tid, b in self._boards.items()
            }
            self._persist_path.write_text(json.dumps(data, indent=2))
    
    def _load(self):
        data = json.loads(self._persist_path.read_text())
        for tid, bd in data.items():
            board = Blackboard(
                task_id=bd["task_id"],
                task_description=bd["task_description"],
                status=bd["status"]
            )
            for k, e in bd["slots"].items():
                board.slots[k] = BlackboardEntry(**e)
            self._boards[tid] = board
```

### 2. 支持黑板的 Agent 基类

```python
# blackboard_agent.py
import anthropic
from typing import Optional

class BlackboardAgent:
    """能读写黑板的 Agent 基类"""
    
    def __init__(self, agent_id: str, board_store: BlackboardStore):
        self.agent_id = agent_id
        self.store = board_store
        self.client = anthropic.Anthropic()
    
    def can_contribute(self, task_id: str) -> bool:
        """判断我能否为这个任务贡献（子类覆盖）"""
        raise NotImplementedError
    
    def contribute(self, task_id: str):
        """往黑板上写我的贡献（子类覆盖）"""
        raise NotImplementedError
    
    def _read(self, task_id: str, slot: str) -> Optional[Any]:
        entry = self.store.read_slot(task_id, slot)
        return entry.value if entry else None
    
    def _write(self, task_id: str, slot: str, value: Any) -> bool:
        return self.store.write_slot(task_id, slot, value, self.agent_id)
    
    def _call_llm(self, system: str, user: str) -> str:
        response = self.client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2048,
            system=system,
            messages=[{"role": "user", "content": user}]
        )
        return response.content[0].text


class DataAgent(BlackboardAgent):
    """负责获取原始数据"""
    
    def can_contribute(self, task_id: str) -> bool:
        # 只要 raw_data 还没有，我就能贡献
        return self._read(task_id, "raw_data") is None
    
    def contribute(self, task_id: str):
        board = self.store._boards.get(task_id)
        # 模拟获取数据
        data = {"revenue": 1_200_000, "cost": 800_000, "yoy_growth": 0.15}
        self._write(task_id, "raw_data", data)
        print(f"[DataAgent] 写入 raw_data: {data}")


class AnalystAgent(BlackboardAgent):
    """负责分析数据"""
    
    def can_contribute(self, task_id: str) -> bool:
        # 需要 raw_data 存在，且 analysis 还没写
        has_data = self._read(task_id, "raw_data") is not None
        no_analysis = self._read(task_id, "analysis") is None
        return has_data and no_analysis
    
    def contribute(self, task_id: str):
        raw = self._read(task_id, "raw_data")
        result = self._call_llm(
            system="你是财务分析师，用简洁语言解读数据。",
            user=f"分析以下财务数据：{raw}"
        )
        self._write(task_id, "analysis", result)
        print(f"[AnalystAgent] 写入 analysis")


class WriterAgent(BlackboardAgent):
    """负责生成最终摘要"""
    
    def can_contribute(self, task_id: str) -> bool:
        return (self._read(task_id, "analysis") is not None and
                self._read(task_id, "summary") is None)
    
    def contribute(self, task_id: str):
        analysis = self._read(task_id, "analysis")
        summary = self._call_llm(
            system="你是商业写手，把分析转化为高管摘要，100字以内。",
            user=analysis
        )
        self._write(task_id, "summary", summary)
        print(f"[WriterAgent] 写入 summary")
```

### 3. 黑板控制器（Orchestrator）

```python
# blackboard_orchestrator.py
import time

def run_blackboard(task_id: str, description: str, 
                   agents: list[BlackboardAgent], max_rounds: int = 10):
    """
    控制器：轮询所有 Agent，让能贡献的去贡献
    直到没有 Agent 能贡献为止（收敛）
    """
    store = agents[0].store
    store.create(task_id, description)
    
    print(f"🚀 任务开始: {description}")
    
    for round_num in range(max_rounds):
        contributed = False
        
        for agent in agents:
            if agent.can_contribute(task_id):
                print(f"  Round {round_num+1}: {agent.agent_id} 开始贡献")
                agent.contribute(task_id)
                contributed = True
        
        if not contributed:
            print(f"✅ 黑板收敛，任务完成（共 {round_num+1} 轮）")
            break
        
        time.sleep(0.1)  # 避免空转
    
    # 输出最终黑板内容
    board = store._boards[task_id]
    print("\n📋 最终黑板:")
    for slot, entry in board.slots.items():
        print(f"  [{slot}] by {entry.written_by}: {str(entry.value)[:80]}")
    
    return board

# 使用示例
if __name__ == "__main__":
    from pathlib import Path
    
    store = BlackboardStore(persist_path=Path("/tmp/blackboard.json"))
    agents = [
        DataAgent("DataAgent", store),
        AnalystAgent("AnalystAgent", store),
        WriterAgent("WriterAgent", store),
    ]
    
    run_blackboard(
        task_id="q1-2026",
        description="分析2026年Q1财报并生成高管摘要",
        agents=agents
    )
```

---

## 💻 TypeScript 版（pi-mono 风格）

```typescript
// blackboard.ts
import { Anthropic } from "@anthropic-ai/sdk";

interface BlackboardSlot<T = unknown> {
  value: T;
  writtenBy: string;
  timestamp: number;
  version: number;
}

interface Blackboard {
  taskId: string;
  description: string;
  status: "pending" | "in_progress" | "completed" | "failed";
  slots: Record<string, BlackboardSlot>;
}

class BlackboardStore {
  private boards = new Map<string, Blackboard>();
  private listeners = new Map<string, Array<(v: unknown) => void>>();

  create(taskId: string, description: string): Blackboard {
    const board: Blackboard = {
      taskId,
      description,
      status: "pending",
      slots: {},
    };
    this.boards.set(taskId, board);
    return board;
  }

  read<T>(taskId: string, slot: string): T | null {
    const board = this.boards.get(taskId);
    return (board?.slots[slot]?.value as T) ?? null;
  }

  write(taskId: string, slot: string, value: unknown, agentId: string): boolean {
    const board = this.boards.get(taskId);
    if (!board) return false;

    const existing = board.slots[slot];
    const version = existing ? existing.version + 1 : 1;

    board.slots[slot] = {
      value,
      writtenBy: agentId,
      timestamp: Date.now(),
      version,
    };

    // 通知订阅者
    const key = `${taskId}:${slot}`;
    this.listeners.get(key)?.forEach((cb) => cb(value));
    return true;
  }

  // 等待某个 slot 被填入（响应式）
  waitForSlot<T>(taskId: string, slot: string, timeoutMs = 30_000): Promise<T> {
    return new Promise((resolve, reject) => {
      const existing = this.read<T>(taskId, slot);
      if (existing !== null) return resolve(existing);

      const key = `${taskId}:${slot}`;
      const timer = setTimeout(() => {
        this.listeners.get(key)?.filter((cb) => cb !== handler);
        reject(new Error(`Timeout waiting for slot ${slot}`));
      }, timeoutMs);

      const handler = (value: unknown) => {
        clearTimeout(timer);
        resolve(value as T);
      };

      if (!this.listeners.has(key)) this.listeners.set(key, []);
      this.listeners.get(key)!.push(handler);
    });
  }
}

// Agent 基类
abstract class BlackboardAgent {
  constructor(
    readonly agentId: string,
    protected store: BlackboardStore,
    protected client = new Anthropic()
  ) {}

  abstract canContribute(taskId: string): boolean;
  abstract contribute(taskId: string): Promise<void>;

  protected read<T>(taskId: string, slot: string) {
    return this.store.read<T>(taskId, slot);
  }

  protected write(taskId: string, slot: string, value: unknown) {
    return this.store.write(taskId, slot, value, this.agentId);
  }

  protected async llm(system: string, user: string): Promise<string> {
    const res = await this.client.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 1024,
      system,
      messages: [{ role: "user", content: user }],
    });
    return (res.content[0] as { text: string }).text;
  }
}

// 具体 Agent 实现
class ResearchAgent extends BlackboardAgent {
  canContribute(taskId: string) {
    return this.read(taskId, "research") === null;
  }

  async contribute(taskId: string) {
    // 模拟爬取竞品数据
    const data = { competitors: ["A", "B", "C"], avgPrice: 299 };
    this.write(taskId, "research", data);
    console.log("[ResearchAgent] ✍️ 写入 research");
  }
}

class StrategyAgent extends BlackboardAgent {
  canContribute(taskId: string) {
    return (
      this.read(taskId, "research") !== null &&
      this.read(taskId, "strategy") === null
    );
  }

  async contribute(taskId: string) {
    const research = this.read<object>(taskId, "research");
    const strategy = await this.llm(
      "你是产品策略师",
      `基于以下竞品调研制定定价策略：${JSON.stringify(research)}`
    );
    this.write(taskId, "strategy", strategy);
    console.log("[StrategyAgent] ✍️ 写入 strategy");
  }
}

// 异步版控制器（支持并行）
async function runBlackboard(
  taskId: string,
  description: string,
  agents: BlackboardAgent[],
  maxRounds = 10
) {
  const store = (agents[0] as any).store as BlackboardStore;
  store.create(taskId, description);

  for (let round = 0; round < maxRounds; round++) {
    // 找出本轮所有能贡献的 Agent，并行执行
    const contributors = agents.filter((a) => a.canContribute(taskId));
    if (contributors.length === 0) {
      console.log(`✅ 黑板收敛，共 ${round} 轮`);
      break;
    }

    await Promise.all(contributors.map((a) => a.contribute(taskId)));
  }
}
```

---

## 🔌 OpenClaw 中的黑板实现

OpenClaw 的多 Agent 协作天然具备黑板语义 —— **workspace 文件系统就是黑板**：

```
/workspace/
├── blackboard/
│   └── task-q1-analysis/
│       ├── manifest.json     ← 任务描述 + 状态
│       ├── raw_data.json     ← DataAgent 写入
│       ├── analysis.md       ← AnalystAgent 写入
│       └── summary.md        ← WriterAgent 写入（最终输出）
```

```typescript
// openclaw 风格：用文件作为黑板 slot
// Agent 写入：
fs.writeFileSync(`blackboard/${taskId}/raw_data.json`, JSON.stringify(data));

// Agent 读取：
const hasData = fs.existsSync(`blackboard/${taskId}/raw_data.json`);

// 控制器通过 fs.watch 感知变化（响应式黑板！）
fs.watch(`blackboard/${taskId}/`, (event, filename) => {
  if (filename) checkAndDispatch(taskId, agents);
});
```

**OpenClaw sessions_spawn 作为黑板 Agent：**

```python
# 主 Agent 作为控制器，spawn 子 Agent 写黑板
async def orchestrate(task_id: str):
    # 生成任务 manifest
    write_manifest(task_id, status="in_progress")
    
    # 并行 spawn 多个专家 Agent
    data_task = sessions_spawn(
        task=f"读取 blackboard/{task_id}/manifest.json，获取数据并写入 raw_data.json",
        runtime="subagent"
    )
    
    # 等 raw_data 就绪后自动 spawn 分析 Agent
    await wait_for_file(f"blackboard/{task_id}/raw_data.json")
    analyst_task = sessions_spawn(
        task=f"读取 raw_data.json，生成分析报告写入 analysis.md",
        runtime="subagent"
    )
```

---

## ⚡ 黑板的三个关键设计决策

### 1. Pull vs Push（轮询 vs 响应式）

```python
# Pull：控制器轮询（简单但耗 CPU）
while not done:
    for agent in agents:
        if agent.can_contribute(task_id):
            agent.contribute(task_id)
    time.sleep(0.1)

# Push：Agent 订阅感兴趣的 slot（高效但复杂）
store.subscribe(task_id, "raw_data", lambda v: analyst.contribute(task_id))
```

### 2. Slot 粒度（细粒度 vs 粗粒度）

```python
# 太粗：整个输出一个 slot → 并行度低
slots = {"output": final_result}

# 太细：每个字段一个 slot → 管理复杂
slots = {"revenue": 1_200_000, "cost": 800_000, "growth": 0.15, ...}

# 适中：按逻辑阶段划分
slots = {
    "raw_data": {...},      # 阶段1
    "analysis": "...",      # 阶段2  
    "summary": "...",       # 阶段3（最终输出）
}
```

### 3. 冲突解决（乐观锁 vs 悲观锁）

```python
# 乐观锁（适合读多写少）：
success = store.write_slot(task_id, "analysis", result, 
                           agent_id, expected_version=1)
if not success:
    # 版本冲突，重新读取再重试
    current = store.read_slot(task_id, "analysis")
    # merge or abort

# 悲观锁（适合高竞争）：
with store.lock(task_id, "analysis"):
    current = store.read_slot(task_id, "analysis")
    store.write_slot(task_id, "analysis", merge(current, result), agent_id)
```

---

## 🎯 适用场景 vs 不适用

**✅ 用黑板当：**
- 多步骤任务，步骤间有数据依赖
- Agent 池动态扩展（随时加入新专家）
- 任务路径不固定（哪个 Agent 能做就谁做）
- 需要可审计性（黑板天然记录谁写了什么）

**❌ 别用黑板当：**
- 简单线性 Pipeline（用 chain 更清晰）
- 需要强事务保证（黑板是最终一致）
- 超高频写入（文件/DB 锁会成瓶颈）

---

## 📝 总结

黑板模式的本质是**共享状态驱动的松耦合协作**：

1. **黑板** = 共享状态空间（文件、Redis、内存）
2. **Agent** = 知识源，各司其职
3. **控制器** = 轮询或事件驱动，决定让谁上
4. **收敛** = 没有 Agent 能再贡献时，任务完成

相比 Pipeline 的刚性，黑板的弹性让你的多 Agent 系统能**优雅处理不确定性** —— 哪个 Agent 挂了，换一个；新来一个专家 Agent，无缝插入；任务路径自然涌现，不需要提前写死。
