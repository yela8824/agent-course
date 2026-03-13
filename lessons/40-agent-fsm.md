# 第40课：Agent FSM — 用有限状态机让 Agent 行为可预测

> "不可预测的 Agent 是危险的 Agent。状态机是给混沌套上缰绳的工具。"

---

## 🤔 问题：Agent 为什么会"乱跑"？

你有没有见过这种情况：

- Agent 明明已经完成了任务，却还在继续调用工具
- Agent 在"等待用户输入"时突然自己做了个决定
- Agent 的行为因为 prompt 措辞不同而完全不同
- 同一个 Agent 在不同情况下走完全不同的路径，难以调试

这些问题的本质是：**Agent 的状态是隐式的，隐藏在 LLM 的"脑子"里，外部无法观察和控制。**

**有限状态机（FSM）** 就是解决这个问题的经典方案。

---

## 📐 什么是有限状态机？

FSM 由三个要素组成：

```
States（状态）+ Events（事件）+ Transitions（转换规则）
```

举个例子，一个订单处理 Agent 的状态：

```
IDLE → COLLECTING_INFO → CONFIRMING → PROCESSING → DONE
                    ↘              ↗
                    CANCELLED
```

**关键特性：**
- Agent 在任何时刻只能处于**一个状态**
- 状态转换必须通过**明确的事件**触发
- 每个状态有**明确的允许操作**
- 状态是**外部可见的**，不藏在 LLM 脑子里

---

## 🏗️ 实战：用 TypeScript 实现 Agent FSM

### 基础类型定义

```typescript
// types.ts
type StateId = string;
type EventId = string;

interface State<Context> {
  id: StateId;
  description: string;
  // 进入状态时执行
  onEnter?: (ctx: Context) => Promise<void>;
  // 离开状态时执行
  onExit?: (ctx: Context) => Promise<void>;
  // 该状态允许调用哪些工具
  allowedTools?: string[];
}

interface Transition<Context> {
  from: StateId;
  event: EventId;
  to: StateId;
  // 转换条件（可选）
  guard?: (ctx: Context) => boolean;
  // 转换时执行的动作
  action?: (ctx: Context) => Promise<void>;
}

interface FSM<Context> {
  states: Map<StateId, State<Context>>;
  transitions: Transition<Context>[];
  initial: StateId;
  current: StateId;
  context: Context;
}
```

### 实现 FSM 引擎

```typescript
// fsm-engine.ts
class AgentFSM<Context> {
  private states: Map<StateId, State<Context>>;
  private transitions: Transition<Context>[];
  public current: StateId;
  public context: Context;

  constructor(config: {
    states: State<Context>[];
    transitions: Transition<Context>[];
    initial: StateId;
    context: Context;
  }) {
    this.states = new Map(config.states.map(s => [s.id, s]));
    this.transitions = config.transitions;
    this.current = config.initial;
    this.context = config.context;
  }

  // 触发事件，驱动状态转换
  async dispatch(event: EventId): Promise<boolean> {
    const transition = this.transitions.find(
      t => t.from === this.current && 
           t.event === event &&
           (!t.guard || t.guard(this.context))
    );

    if (!transition) {
      console.warn(`No transition: ${this.current} + ${event}`);
      return false;
    }

    const fromState = this.states.get(this.current)!;
    const toState = this.states.get(transition.to)!;

    // 执行生命周期钩子
    await fromState.onExit?.(this.context);
    await transition.action?.(this.context);
    
    this.current = transition.to;
    
    await toState.onEnter?.(this.context);

    console.log(`FSM: ${transition.from} --[${event}]--> ${transition.to}`);
    return true;
  }

  // 检查当前状态是否允许某个工具
  isToolAllowed(toolName: string): boolean {
    const state = this.states.get(this.current);
    if (!state?.allowedTools) return true; // 未限制则全部允许
    return state.allowedTools.includes(toolName);
  }

  // 获取当前可触发的事件列表
  availableEvents(): EventId[] {
    return this.transitions
      .filter(t => t.from === this.current && (!t.guard || t.guard(this.context)))
      .map(t => t.event);
  }
}
```

---

## 🛒 真实案例：订单处理 Agent

```typescript
// order-agent.ts
interface OrderContext {
  userId: string;
  items: string[];
  total: number;
  confirmed: boolean;
  paymentId?: string;
  error?: string;
}

const orderFSM = new AgentFSM<OrderContext>({
  initial: 'IDLE',
  context: {
    userId: '',
    items: [],
    total: 0,
    confirmed: false
  },
  
  states: [
    {
      id: 'IDLE',
      description: '等待用户开始',
      allowedTools: ['search_products', 'get_recommendations']
    },
    {
      id: 'COLLECTING',
      description: '收集订单信息',
      allowedTools: ['search_products', 'add_to_cart', 'remove_from_cart'],
      onEnter: async (ctx) => {
        console.log(`开始为用户 ${ctx.userId} 收集订单`);
      }
    },
    {
      id: 'CONFIRMING',
      description: '等待用户确认',
      // 确认阶段禁止修改购物车！
      allowedTools: ['show_cart_summary'],
      onEnter: async (ctx) => {
        // 展示订单摘要给用户
        await showOrderSummary(ctx);
      }
    },
    {
      id: 'PROCESSING',
      description: '处理支付',
      allowedTools: ['process_payment', 'check_payment_status'],
      onEnter: async (ctx) => {
        // 锁定库存
        await lockInventory(ctx.items);
      }
    },
    {
      id: 'DONE',
      description: '订单完成',
      allowedTools: [], // 完成后不允许任何工具调用
      onEnter: async (ctx) => {
        await sendConfirmationEmail(ctx);
      }
    },
    {
      id: 'CANCELLED',
      description: '订单已取消',
      allowedTools: ['search_products'], // 允许重新开始
      onEnter: async (ctx) => {
        await releaseInventory(ctx.items);
      }
    },
    {
      id: 'ERROR',
      description: '处理出错',
      allowedTools: ['notify_support'],
      onEnter: async (ctx) => {
        console.error('订单错误:', ctx.error);
      }
    }
  ],

  transitions: [
    { from: 'IDLE',       event: 'START',     to: 'COLLECTING' },
    { from: 'COLLECTING', event: 'CHECKOUT',  to: 'CONFIRMING',
      guard: (ctx) => ctx.items.length > 0 }, // 购物车非空才能结账
    { from: 'CONFIRMING', event: 'CONFIRM',   to: 'PROCESSING',
      action: async (ctx) => { ctx.confirmed = true; } },
    { from: 'CONFIRMING', event: 'EDIT',      to: 'COLLECTING' },
    { from: 'PROCESSING', event: 'SUCCESS',   to: 'DONE' },
    { from: 'PROCESSING', event: 'FAILURE',   to: 'ERROR' },
    // 任何状态都可以取消
    { from: 'COLLECTING', event: 'CANCEL',    to: 'CANCELLED' },
    { from: 'CONFIRMING', event: 'CANCEL',    to: 'CANCELLED' },
    { from: 'CANCELLED',  event: 'START',     to: 'COLLECTING' },
    { from: 'ERROR',      event: 'RETRY',     to: 'PROCESSING' },
    { from: 'ERROR',      event: 'CANCEL',    to: 'CANCELLED' }
  ]
});
```

---

## 🔗 与 LLM 集成：FSM 控制工具调用

FSM 最关键的作用是**过滤工具调用**：

```typescript
// agent-loop.ts
async function agentLoop(userMessage: string, fsm: AgentFSM<OrderContext>) {
  // 构建 system prompt，注入当前状态
  const systemPrompt = `
你是一个订单处理助手。
当前状态：${fsm.current}
状态说明：${fsm.states.get(fsm.current)?.description}
可触发的事件：${fsm.availableEvents().join(', ')}

重要：只能调用当前状态允许的工具。
`;

  // 过滤工具：只提供当前状态允许的工具
  const availableTools = ALL_TOOLS.filter(
    tool => fsm.isToolAllowed(tool.name)
  );

  const response = await anthropic.messages.create({
    model: 'claude-opus-4-5',
    system: systemPrompt,
    messages: [{ role: 'user', content: userMessage }],
    tools: availableTools  // ← FSM 控制可用工具！
  });

  // 处理 LLM 返回的工具调用
  for (const toolUse of response.tool_use_blocks) {
    // 双重检查（防止 LLM 越权）
    if (!fsm.isToolAllowed(toolUse.name)) {
      console.error(`FSM 拦截：状态 ${fsm.current} 不允许调用 ${toolUse.name}`);
      continue;
    }

    const result = await executeTool(toolUse);
    
    // 根据工具结果驱动状态转换
    if (toolUse.name === 'process_payment') {
      if (result.success) {
        await fsm.dispatch('SUCCESS');
      } else {
        fsm.context.error = result.error;
        await fsm.dispatch('FAILURE');
      }
    }
  }
}
```

---

## 🌟 OpenClaw 中的 FSM 实践

OpenClaw 的 Session 本质上就是一个 FSM：

```
INIT → IDLE → PROCESSING → RESPONDING → IDLE
                   ↓
              TOOL_EXECUTION → PROCESSING
```

OpenClaw 的 Heartbeat 机制依赖状态判断：

```typescript
// OpenClaw 心跳状态机（简化）
const sessionFSM = {
  IDLE: {
    allowedActions: ['receive_message', 'heartbeat_check'],
    onHeartbeat: async () => {
      // 检查邮件、日历等
      return 'HEARTBEAT_OK';
    }
  },
  PROCESSING: {
    allowedActions: ['tool_call', 'llm_inference'],
    // 处理中不接受新消息（队列等待）
    onHeartbeat: async () => 'SKIP'
  },
  TOOL_EXECUTION: {
    allowedActions: ['execute_tool', 'timeout_check'],
    onTimeout: async () => {
      await fsm.dispatch('TOOL_TIMEOUT');
    }
  }
};
```

---

## 🐍 Python 实现：learn-claude-code 风格

```python
# fsm_agent.py
from dataclasses import dataclass, field
from typing import Optional, Callable, Any
from enum import Enum

class OrderState(Enum):
    IDLE = "idle"
    COLLECTING = "collecting" 
    CONFIRMING = "confirming"
    PROCESSING = "processing"
    DONE = "done"
    ERROR = "error"

@dataclass
class Transition:
    from_state: OrderState
    event: str
    to_state: OrderState
    guard: Optional[Callable] = None
    action: Optional[Callable] = None

class OrderFSM:
    def __init__(self):
        self.state = OrderState.IDLE
        self.context = {}
        self.transitions = [
            Transition(OrderState.IDLE,       "start",   OrderState.COLLECTING),
            Transition(OrderState.COLLECTING, "checkout", OrderState.CONFIRMING,
                      guard=lambda ctx: len(ctx.get('items', [])) > 0),
            Transition(OrderState.CONFIRMING, "confirm", OrderState.PROCESSING),
            Transition(OrderState.CONFIRMING, "edit",    OrderState.COLLECTING),
            Transition(OrderState.PROCESSING, "success", OrderState.DONE),
            Transition(OrderState.PROCESSING, "failure", OrderState.ERROR),
        ]
    
    def dispatch(self, event: str) -> bool:
        transition = next(
            (t for t in self.transitions 
             if t.from_state == self.state 
             and t.event == event
             and (t.guard is None or t.guard(self.context))),
            None
        )
        
        if not transition:
            print(f"无效转换: {self.state.value} + {event}")
            return False
        
        if transition.action:
            transition.action(self.context)
            
        old_state = self.state.value
        self.state = transition.to_state
        print(f"FSM: {old_state} --[{event}]--> {self.state.value}")
        return True

    def get_system_prompt(self) -> str:
        """动态生成包含状态信息的 system prompt"""
        state_prompts = {
            OrderState.IDLE:       "等待用户开始购物",
            OrderState.COLLECTING: "帮助用户选择商品，可以搜索和添加到购物车",
            OrderState.CONFIRMING: "展示订单摘要，等待用户确认或修改",
            OrderState.PROCESSING: "正在处理支付，不要接受新的商品请求",
            OrderState.DONE:       "订单完成，感谢用户",
            OrderState.ERROR:      "处理出错，向用户道歉并提供帮助",
        }
        return f"""当前状态: {self.state.value}
行为指导: {state_prompts[self.state]}
"""

# 使用示例
fsm = OrderFSM()
fsm.dispatch("start")       # IDLE → COLLECTING
fsm.context['items'] = ['商品A', '商品B']
fsm.dispatch("checkout")    # COLLECTING → CONFIRMING
fsm.dispatch("confirm")     # CONFIRMING → PROCESSING
fsm.dispatch("success")     # PROCESSING → DONE
```

---

## ⚡ 进阶：层次状态机（HFSM）

当状态复杂时，可以嵌套状态机：

```typescript
// 层次状态机：PROCESSING 状态内部有子状态
const processingSubFSM = new AgentFSM({
  initial: 'VALIDATING',
  states: [
    { id: 'VALIDATING',   allowedTools: ['validate_order'] },
    { id: 'CHARGING',     allowedTools: ['charge_card'] },
    { id: 'FULFILLING',   allowedTools: ['create_shipment'] },
  ],
  transitions: [
    { from: 'VALIDATING', event: 'VALID',   to: 'CHARGING' },
    { from: 'CHARGING',   event: 'CHARGED', to: 'FULFILLING' },
  ]
});

// 父 FSM 进入 PROCESSING 时，启动子 FSM
{
  id: 'PROCESSING',
  onEnter: async (ctx) => {
    // 运行子状态机直到完成
    while (processingSubFSM.current !== 'FULFILLING') {
      await runSubFSMStep(processingSubFSM, ctx);
    }
  }
}
```

---

## 📊 FSM 带来的好处总结

| 问题 | 没有 FSM | 有 FSM |
|------|----------|--------|
| 调试困难 | "Agent 怎么走到这一步的？" | 完整的状态转换日志 |
| 越权操作 | LLM 可能调用任意工具 | 只有当前状态允许的工具 |
| 无限循环 | Agent 停不下来 | 终止状态明确定义 |
| 并发冲突 | 两个请求同时修改状态 | 状态是原子的，转换是串行的 |
| 测试困难 | 需要跑完整的 LLM | 可以直接测试状态转换逻辑 |

---

## 🎯 什么时候用 FSM？

**适合 FSM 的场景：**
- 业务流程明确（订单、审批、预订）
- 需要强制执行操作顺序
- 需要防止越权工具调用
- 多人协作的 Agent（状态需要外部可见）

**不适合 FSM 的场景：**
- 开放式对话（状态太多且难以预定义）
- 简单问答 Agent
- 状态转换完全由 LLM 自主决定的创意任务

---

## 💡 实践建议

1. **从简单开始**：先画出状态图，再写代码
2. **状态要有意义**：每个状态应对应一个清晰的业务阶段
3. **事件要明确**：用动词命名，如 `CONFIRM`、`CANCEL`、`RETRY`
4. **记录转换日志**：状态机的最大价值之一是可追溯性
5. **测试每个路径**：FSM 让你可以穷举所有合法路径并测试

---

## 📚 扩展阅读

- [XState](https://xstate.js.org/) - 最成熟的 JS 状态机库
- [statecharts.dev](https://statecharts.dev/) - 状态图标准规范
- [Harel Statecharts](https://www.sciencedirect.com/science/article/pii/0167642387900359) - 层次状态机原始论文

---

*第40课完 | 下节课：Agent Knowledge Graph — 用知识图谱增强 Agent 的结构化推理能力*
