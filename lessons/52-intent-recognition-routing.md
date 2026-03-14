# 52 - Intent Recognition & Routing：意图识别与路由

> 让 Agent 听懂"你想干什么"，然后精准分发

---

## 为什么需要意图识别？

真实世界的 Agent 面对的输入千变万化：

```
"帮我查一下今天的天气"           → 调天气工具
"用代码帮我排序这个数组"         → 调编程工具
"昨天我让你记的那个事情是什么？" → 查记忆
"哈哈哈这个笑话真好笑"           → 不需要工具，直接聊天
```

如果每条消息都走完整的工具调用流程，既慢又贵。
**意图识别**就是在 LLM 完整推理之前，先判断"这是什么类型的请求"，然后**路由**到最合适的处理器。

---

## 三种路由策略

### 策略 1：规则路由（最快，0 token）

```python
# learn-claude-code 风格：基于关键词/正则的快速路由
import re

INTENT_PATTERNS = {
    "weather": [r"天气", r"weather", r"几度", r"下雨"],
    "code":    [r"代码", r"code", r"函数", r"bug", r"报错"],
    "memory":  [r"记得吗", r"之前说的", r"我让你记", r"remember"],
    "chat":    [],  # fallback
}

def classify_by_rules(message: str) -> str:
    msg = message.lower()
    for intent, patterns in INTENT_PATTERNS.items():
        if any(re.search(p, msg) for p in patterns):
            return intent
    return "chat"

# 优点：纳秒级，零成本
# 缺点：覆盖不全，维护成本高
```

### 策略 2：LLM 路由（最准，耗 token）

```typescript
// pi-mono 风格：用小模型做意图分类
interface Intent {
  type: "tool_call" | "memory_query" | "chat" | "task";
  confidence: number;
  params?: Record<string, unknown>;
}

async function classifyIntent(message: string): Promise<Intent> {
  const response = await anthropic.messages.create({
    model: "claude-haiku-3-5",  // 用最小最快的模型
    max_tokens: 100,
    system: `你是一个意图分类器。将用户消息分类为以下之一：
- tool_call: 需要调用外部工具（天气/搜索/代码执行等）
- memory_query: 查询历史对话或记忆
- task: 需要多步骤执行的复杂任务
- chat: 普通对话，不需要工具

只返回 JSON，格式：{"type": "...", "confidence": 0.0-1.0}`,
    messages: [{ role: "user", content: message }],
  });

  return JSON.parse(response.content[0].text);
}

// 优点：理解复杂语义
// 缺点：每次分类都要 API 调用（约 50-100ms，$0.0001）
```

### 策略 3：混合路由（推荐！）

```typescript
// OpenClaw 实际使用的模式：先规则，规则不确定再 LLM
async function hybridRoute(message: string): Promise<string> {
  // 第一层：极速规则匹配（高置信度场景）
  const ruleResult = classifyByRules(message);
  if (ruleResult.confidence > 0.9) {
    return ruleResult.intent;  // 直接返回，不走 LLM
  }

  // 第二层：LLM 分类（低置信度场景）
  const llmResult = await classifyIntent(message);
  return llmResult.type;
}

// 结果：80% 的请求走规则（纳秒），20% 走 LLM（百毫秒）
// 成本降低 80%，准确率接近纯 LLM
```

---

## 实战：OpenClaw 的 Skill 路由

OpenClaw 用的正是混合路由的变体。每个 Skill 有一个 `description`，由 LLM 在系统提示中决定是否加载：

```typescript
// 来自 OpenClaw skills 系统
// 每个 skill 的 SKILL.md 开头都有 description：
// "Get current weather and forecasts. Use when: user asks about weather..."

// Agent 在处理消息前，先扫描所有 skill descriptions
// 如果 description 匹配，加载 SKILL.md 注入上下文
// 等于是：LLM 自己做了意图识别 + 工具加载决策

const AVAILABLE_SKILLS = [
  { name: "weather", description: "Get current weather..." },
  { name: "github",  description: "GitHub operations via gh CLI..." },
  { name: "gog",     description: "Google Workspace CLI for Gmail..." },
];

// 这段逻辑在系统提示里：
const systemPrompt = `
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md
- If multiple could apply: choose the most specific one
`;
// LLM 读到这个 prompt，自动做了意图→skill 的路由
```

**关键洞见：** OpenClaw 把"意图识别"的逻辑嵌入到了系统提示里，让 LLM 自己路由——这是最优雅的实现方式，不需要额外的分类模型。

---

## 意图路由器的完整实现

```typescript
// 生产级意图路由器
type Handler = (message: string, context: Context) => Promise<Response>;

class IntentRouter {
  private routes = new Map<string, Handler>();
  private middlewares: Array<(intent: Intent) => Intent> = [];

  // 注册路由
  on(intent: string, handler: Handler): this {
    this.routes.set(intent, handler);
    return this;
  }

  // 注册中间件（用于日志、权限检查等）
  use(middleware: (intent: Intent) => Intent): this {
    this.middlewares.push(middleware);
    return this;
  }

  async dispatch(message: string, context: Context): Promise<Response> {
    // 1. 识别意图
    let intent = await this.classify(message);

    // 2. 过中间件管道
    for (const mw of this.middlewares) {
      intent = mw(intent);
    }

    // 3. 找到对应 handler
    const handler = this.routes.get(intent.type) 
                 ?? this.routes.get("fallback");
    
    if (!handler) {
      throw new Error(`No handler for intent: ${intent.type}`);
    }

    // 4. 执行 handler
    return handler(message, context);
  }

  private async classify(message: string): Promise<Intent> {
    // 混合路由逻辑（见上）
    return hybridRoute(message);
  }
}

// 使用方式
const router = new IntentRouter()
  .use(loggingMiddleware)      // 记录所有意图
  .use(authMiddleware)         // 权限检查
  .on("tool_call",   handleToolCall)
  .on("memory_query", handleMemoryQuery)
  .on("task",        handleTask)
  .on("chat",        handleChat)
  .on("fallback",    handleFallback);

// 每条消息进来都走这一个入口
const response = await router.dispatch(userMessage, context);
```

---

## 多维度路由

真实场景往往需要同时识别多个维度：

```python
# 二维路由：意图 + 紧急程度
@dataclass
class MultiDimIntent:
    type: str         # 意图类型
    urgency: str      # "critical" | "normal" | "low"  
    language: str     # "zh" | "en"
    requires_auth: bool

async def classify_multi_dim(message: str) -> MultiDimIntent:
    # 用结构化输出一次性提取所有维度
    response = await client.messages.create(
        model="claude-haiku-3-5",
        max_tokens=200,
        tools=[{
            "name": "classify",
            "description": "分类消息意图",
            "input_schema": {
                "type": "object",
                "properties": {
                    "type": {"enum": ["tool_call","memory","task","chat"]},
                    "urgency": {"enum": ["critical","normal","low"]},
                    "language": {"enum": ["zh","en","other"]},
                    "requires_auth": {"type": "boolean"}
                }
            }
        }],
        messages=[{"role":"user","content": message}]
    )
    return MultiDimIntent(**response.content[0].input)

# 路由决策
async def smart_route(intent: MultiDimIntent, message: str):
    if intent.urgency == "critical":
        # 紧急任务：用更强的模型
        return await handle_with_model(message, model="claude-opus-4-5")
    elif intent.requires_auth and not user.is_authenticated:
        return "请先登录才能执行此操作"
    else:
        return await default_handler(message, intent.type)
```

---

## 常见坑 & 解决方案

| 问题 | 原因 | 解决 |
|------|------|------|
| 意图误分类 | 规则太简单 / LLM 理解偏差 | 加置信度阈值，低置信度转人工 |
| 路由延迟高 | 每次都走 LLM 分类 | 缓存相似问题的分类结果 |
| 新意图漏处理 | 没有 fallback | 必须注册 fallback handler |
| 多意图并存 | 用户一句话包含多个请求 | 先拆分，再分别路由 |

---

## 小结

```
用户消息
    ↓
[规则路由] → 高置信度 → 直接执行（0 token）
    ↓ 低置信度
[LLM 路由] → 意图分类（小模型，快而便宜）
    ↓
[中间件管道] → 日志/鉴权/限流
    ↓
[Handler 分发] → 执行具体逻辑
```

**核心原则：**
1. 先规则后 LLM，降低 80% 路由成本
2. 用最小的模型做分类，不要大材小用
3. 中间件处理横切关注点（鉴权、日志、限流）
4. 永远有 fallback，别让请求消失在黑洞里
5. OpenClaw 的"skill description 路由"是最优雅的实践：让 LLM 自己路由

---

*下节预告：Agent 知识图谱 —— 让 Agent 从关系网络中推理*
