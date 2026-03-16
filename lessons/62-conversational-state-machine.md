# 62 - 对话状态机与多轮上下文管理

> Conversational State Machine & Multi-Turn Context Management：让 Agent 在复杂多轮对话中不迷路

---

## 问题背景

你的 Agent 能处理 "帮我查一下天气" 这种单轮问答，但能处理这种对话吗？

```
用户: 帮我分析一下我们公司 Q1 的销售数据
Agent: 好的，请先告诉我数据源在哪？
用户: 在 MySQL，表名 sales_2026
Agent: 需要按哪些维度分析？地区、产品线、销售员？
用户: 先按地区，等等——先查一下数据库连不连得上
Agent: 连接成功！现在继续按地区分析吗？
用户: 对，还有，把结果发给我们 CFO
Agent: CFO 的邮件地址是什么？
```

这里涉及：**上下文跨轮保持、任务挂起恢复、多条件分支、并行子任务**。

这就是 **Conversational State Machine（对话状态机）** 要解决的问题。

---

## 核心概念

```
对话状态机 = 状态（State）+ 转换（Transition）+ 上下文（Context）

State: 对话当前处于哪个阶段？
  - IDLE              # 空闲，等待新任务
  - COLLECTING_INFO   # 收集参数中
  - EXECUTING         # 执行任务中
  - AWAITING_CONFIRM  # 等待用户确认
  - PAUSED            # 任务挂起
  - ERROR             # 错误处理中

Transition: 什么事件触发状态切换？
  - user_message      # 用户发消息
  - tool_result       # 工具返回结果
  - timeout           # 超时
  - error             # 发生错误

Context: 贯穿整个对话的记忆
  - task_stack        # 任务调用栈（支持嵌套）
  - collected_params  # 已收集的参数
  - pending_intents   # 待处理的意图
```

---

## 最简实现：Python 对话状态机

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Optional
import json

class ConvState(Enum):
    IDLE = "idle"
    COLLECTING = "collecting"
    EXECUTING = "executing"
    CONFIRMING = "confirming"
    PAUSED = "paused"

@dataclass
class TaskFrame:
    """任务栈帧 - 就像函数调用栈"""
    task_id: str
    intent: str
    required_params: list[str]
    collected_params: dict = field(default_factory=dict)
    result: Any = None
    parent_task_id: Optional[str] = None  # 支持嵌套任务

@dataclass  
class ConversationContext:
    session_id: str
    state: ConvState = ConvState.IDLE
    task_stack: list[TaskFrame] = field(default_factory=list)
    global_vars: dict = field(default_factory=dict)  # 跨任务共享变量
    turn_count: int = 0

class ConversationalAgent:
    def __init__(self):
        self.contexts: dict[str, ConversationContext] = {}
    
    def get_or_create_context(self, session_id: str) -> ConversationContext:
        if session_id not in self.contexts:
            self.contexts[session_id] = ConversationContext(session_id=session_id)
        return self.contexts[session_id]
    
    async def handle_message(self, session_id: str, user_message: str) -> str:
        ctx = self.get_or_create_context(session_id)
        ctx.turn_count += 1
        
        # 根据当前状态决定行为
        match ctx.state:
            case ConvState.IDLE:
                return await self._handle_new_intent(ctx, user_message)
            case ConvState.COLLECTING:
                return await self._handle_param_collection(ctx, user_message)
            case ConvState.CONFIRMING:
                return await self._handle_confirmation(ctx, user_message)
            case ConvState.PAUSED:
                return await self._handle_resume(ctx, user_message)
            case ConvState.EXECUTING:
                # 执行中被打断 - 处理插队意图
                return await self._handle_interrupt(ctx, user_message)
    
    async def _handle_new_intent(self, ctx: ConversationContext, message: str) -> str:
        # 用 LLM 识别意图和需要的参数
        intent_info = await self._detect_intent(message)
        
        frame = TaskFrame(
            task_id=f"task_{ctx.turn_count}",
            intent=intent_info["intent"],
            required_params=intent_info["required_params"],
        )
        
        # 从消息中提取已有参数
        frame.collected_params = intent_info.get("extracted_params", {})
        
        ctx.task_stack.append(frame)
        
        # 检查是否参数齐全
        missing = [p for p in frame.required_params if p not in frame.collected_params]
        
        if not missing:
            ctx.state = ConvState.EXECUTING
            return await self._execute_task(ctx, frame)
        else:
            ctx.state = ConvState.COLLECTING
            return f"需要了解一下：{missing[0]} 是什么？"
    
    async def _handle_interrupt(self, ctx: ConversationContext, message: str) -> str:
        """处理执行中的插队请求"""
        # 判断是否是紧急打断
        is_urgent = await self._is_urgent_interrupt(message)
        
        if is_urgent:
            # 挂起当前任务，处理插队
            ctx.state = ConvState.PAUSED
            # 递归处理新意图（推入任务栈）
            return f"好的，先暂停当前任务。{await self._handle_new_intent(ctx, message)}"
        else:
            return "我正在执行任务，稍等一下，或者你有紧急的事情需要我先处理吗？"
```

---

## OpenClaw 实战：多轮对话的上下文注入

OpenClaw 的每个 session 天然就是一个状态容器。看看它如何管理对话状态：

```typescript
// OpenClaw 的 session 结构（简化）
interface Session {
  sessionKey: string;
  messages: Message[];          // 完整对话历史
  metadata: {
    state?: string;             // 自定义状态标记
    pendingTask?: TaskInfo;     // 挂起的任务
    collectedParams?: Record<string, unknown>;
  };
}

// 在系统提示词中注入当前状态（Context Injection）
function buildSystemPrompt(session: Session): string {
  const base = `你是一个 Agent...`;
  
  if (session.metadata.pendingTask) {
    return `${base}
    
【当前状态】你正在帮用户完成：${session.metadata.pendingTask.description}
已收集参数：${JSON.stringify(session.metadata.collectedParams)}
还需要收集：${session.metadata.pendingTask.missing.join(', ')}

优先完成当前任务，除非用户明确要求切换。`;
  }
  
  return base;
}
```

---

## 任务栈：处理嵌套对话

对话中经常出现 **任务中套任务** 的情况：

```
主任务: 分析 Q1 销售
  ├── 子任务: 连接数据库（临时插入）
  └── 子任务: 发邮件给 CFO

用户流: 
  "分析销售" → 推入主任务
  "先测试数据库" → 推入子任务（保存主任务上下文）
  "好了，继续分析" → 弹出子任务，恢复主任务
```

```python
class TaskStackManager:
    def push_task(self, ctx: ConversationContext, new_task: TaskFrame):
        """推入新任务（当前任务自动挂起）"""
        if ctx.task_stack:
            # 记录父任务 ID
            new_task.parent_task_id = ctx.task_stack[-1].task_id
        ctx.task_stack.append(new_task)
    
    def pop_task(self, ctx: ConversationContext) -> Optional[TaskFrame]:
        """弹出完成的任务，恢复父任务"""
        if not ctx.task_stack:
            return None
        completed = ctx.task_stack.pop()
        
        # 如果有父任务，把结果注入进去
        if ctx.task_stack and completed.parent_task_id:
            parent = ctx.task_stack[-1]
            parent.collected_params[f"subtask_{completed.intent}_result"] = completed.result
        
        return completed
    
    def current_task(self, ctx: ConversationContext) -> Optional[TaskFrame]:
        return ctx.task_stack[-1] if ctx.task_stack else None
```

---

## pi-mono 的实现思路

pi-mono 用一个更简洁的方式处理多轮状态：

```typescript
// pi-mono/src/session/conversation-tracker.ts（示意）
interface ConversationTracker {
  // 当前意图链（从外到内）
  intentStack: IntentFrame[];
  
  // 对话槽位（slot filling）
  slots: Map<string, SlotValue>;
  
  // 挂起点（checkpoint）
  checkpoint?: {
    frameIndex: number;
    reason: string;
    resumeHint: string;
  };
}

// 槽位填充：收集参数的核心机制
interface SlotValue {
  value: unknown;
  confidence: number;      // 提取置信度
  source: 'user' | 'inferred' | 'default';
  turnCollected: number;   // 第几轮收集的
}

// 对话系统用 slot filling 避免重复询问
function fillSlots(
  tracker: ConversationTracker, 
  extraction: ParamExtraction
): string[] {
  const stillMissing: string[] = [];
  
  for (const [key, value] of Object.entries(extraction.params)) {
    if (value.confidence > 0.8) {
      tracker.slots.set(key, value);
    }
  }
  
  const currentFrame = tracker.intentStack.at(-1);
  if (currentFrame) {
    for (const required of currentFrame.requiredSlots) {
      if (!tracker.slots.has(required)) {
        stillMissing.push(required);
      }
    }
  }
  
  return stillMissing;
}
```

---

## 关键设计：上下文窗口里放什么

多轮对话的 Context 不能无限增长，要精心设计注入什么：

```
【低效做法】塞入全部历史对话 → Context 爆炸，成本飙升

【高效做法】结构化状态摘要 + 最近 N 轮原文

系统提示词中注入:
┌─────────────────────────────────────┐
│ 当前任务: 分析 Q1 销售数据          │
│ 已知条件: DB=MySQL, 表=sales_2026   │
│ 当前步骤: 等待用户确认维度          │
│ 待处理: 发邮件给 CFO               │
└─────────────────────────────────────┘

+ 最近 3 轮对话原文（保留原话语气）
+ 工具调用结果摘要（不是全文）
```

```python
def build_context_prompt(ctx: ConversationContext, recent_turns: int = 3) -> str:
    current = ctx.task_stack[-1] if ctx.task_stack else None
    
    parts = ["## 当前对话状态\n"]
    
    if current:
        parts.append(f"**当前任务**: {current.intent}")
        parts.append(f"**已收集参数**: {json.dumps(current.collected_params, ensure_ascii=False)}")
        
        missing = [p for p in current.required_params if p not in current.collected_params]
        if missing:
            parts.append(f"**还需要**: {', '.join(missing)}")
    
    if len(ctx.task_stack) > 1:
        parts.append(f"\n**挂起中的任务**: {[t.intent for t in ctx.task_stack[:-1]]}")
    
    parts.append(f"\n**对话轮次**: 第 {ctx.turn_count} 轮")
    
    return "\n".join(parts)
```

---

## 超时与遗忘机制

用户有时候会半途而废，需要处理 **对话超时**：

```python
import time

@dataclass
class ConversationContext:
    # ...之前的字段...
    last_active: float = field(default_factory=time.time)
    timeout_seconds: int = 300  # 5分钟无响应 → 重置

class ConversationalAgent:
    def handle_message(self, session_id: str, message: str):
        ctx = self.get_or_create_context(session_id)
        
        # 检查超时
        if time.time() - ctx.last_active > ctx.timeout_seconds:
            if ctx.state != ConvState.IDLE:
                # 友好地告知用户之前的任务已超时
                old_task = ctx.task_stack[-1].intent if ctx.task_stack else "未知"
                self._reset_context(ctx)
                return f"由于长时间未响应，之前的「{old_task}」任务已取消。有什么新的需求吗？"
        
        ctx.last_active = time.time()
        # 正常处理...
    
    def _reset_context(self, ctx: ConversationContext):
        ctx.state = ConvState.IDLE
        ctx.task_stack.clear()
        ctx.global_vars.clear()
```

---

## 实战完整示例：数据分析助手

```python
# 完整流程演示
agent = ConversationalAgent()
session = "user_123"

# 第1轮：建立主任务
r1 = await agent.handle_message(session, "帮我分析 Q1 销售数据")
# → "需要了解一下：数据源在哪？"

# 第2轮：收集参数
r2 = await agent.handle_message(session, "MySQL，表叫 sales_2026")  
# → "需要按哪个维度分析？地区、产品线还是销售员？"

# 第3轮：插入子任务（打断！）
r3 = await agent.handle_message(session, "等等，先帮我测试一下数据库能不能连")
# 状态: EXECUTING → PAUSED，推入子任务
# → "好，先测试连接。[执行连接测试]... 连接成功！继续之前的分析吗？"

# 第4轮：恢复主任务
r4 = await agent.handle_message(session, "对，按地区分析，完了发给 CFO")
# 弹出子任务，恢复主任务，推入"发邮件"子任务
# → "按地区分析完成：[结果]。CFO 的邮箱是？"

# 第5轮：完成
r5 = await agent.handle_message(session, "cfo@company.com")
# → "已发送给 cfo@company.com，任务完成！"
```

---

## 总结

| 概念 | 作用 |
|------|------|
| 状态机 (State Machine) | 明确对话处于哪个阶段，避免「不知道该做什么」 |
| 任务栈 (Task Stack) | 支持嵌套/插队任务，用完弹出自动恢复 |
| 槽位填充 (Slot Filling) | 结构化收集参数，不重复询问 |
| 超时机制 | 优雅处理用户半途而废 |
| 上下文摘要注入 | 把状态压缩后注入 prompt，比塞原文高效 10x |

**一句话记住**：对话状态机 = 让 Agent 记住「我们聊到哪里了，还差什么」。

---

*下节课预告：Agent 知识图谱——用图结构组织 Agent 的世界观*
