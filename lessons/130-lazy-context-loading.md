# 130 - Agent 懒加载上下文与按需数据注入（Lazy Context Loading & On-Demand Data Injection）

## 为什么需要懒加载？

大多数 Agent 实现都在 system prompt 里塞一堆上下文：
- 用户资料
- 工具文档
- 业务规则
- 历史记录
- 配置项

结果呢？每次请求都消耗几千个 token，但 LLM 实际上只用到其中 10%。

**懒加载的核心思想**：先给 Agent 一个"目录"，让它知道有什么数据可以用；只有当 Agent 真正需要某段数据时，才动态注入。

```
传统方式：
System Prompt = [用户信息 500 tokens] + [规则手册 2000 tokens] + [工具文档 1000 tokens]
每次消耗: 3500 tokens

懒加载方式：
System Prompt = [数据目录 100 tokens] + 按需注入
平均消耗: 100 + 实际需要的 ~300 tokens = 400 tokens
节省: 89%
```

---

## 核心架构

```typescript
// 上下文注册表 - 注册可懒加载的数据源
interface ContextSource {
  key: string;           // 唯一标识
  description: string;  // 给 LLM 看的描述（出现在目录中）
  loader: () => Promise<string>;  // 实际加载函数
  maxTokens?: number;   // 截断限制
  ttl?: number;         // 缓存秒数
}

class LazyContextRegistry {
  private sources = new Map<string, ContextSource>();
  private cache = new Map<string, { data: string; expiresAt: number }>();

  register(source: ContextSource) {
    this.sources.set(source.key, source);
  }

  // 生成系统提示词中的"目录"部分
  buildDirectory(): string {
    const entries = [...this.sources.values()].map(
      s => `- ${s.key}: ${s.description}`
    );
    return [
      "## 可用上下文数据（按需获取）",
      "如需以下数据，请调用 fetch_context 工具：",
      ...entries,
    ].join("\n");
  }

  async load(key: string): Promise<string> {
    const source = this.sources.get(key);
    if (!source) throw new Error(`Unknown context key: ${key}`);

    // 检查缓存
    const cached = this.cache.get(key);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.data;
    }

    // 懒加载
    const data = await source.loader();
    const truncated = source.maxTokens
      ? truncateToTokens(data, source.maxTokens)
      : data;

    // 写入缓存
    if (source.ttl) {
      this.cache.set(key, {
        data: truncated,
        expiresAt: Date.now() + source.ttl * 1000,
      });
    }

    return truncated;
  }
}
```

---

## 注册数据源

```typescript
const registry = new LazyContextRegistry();

// 注册用户档案
registry.register({
  key: "user_profile",
  description: "当前用户的详细资料：偏好、历史、VIP等级",
  loader: async () => {
    const user = await db.users.findById(session.userId);
    return JSON.stringify(user, null, 2);
  },
  maxTokens: 500,
  ttl: 300, // 5分钟缓存
});

// 注册业务规则手册
registry.register({
  key: "business_rules",
  description: "退款政策、促销规则、权限限制等业务规则",
  loader: async () => {
    return fs.readFile("./rules/business.md", "utf-8");
  },
  maxTokens: 2000,
  ttl: 3600, // 1小时缓存（规则变化慢）
});

// 注册实时库存数据
registry.register({
  key: "inventory",
  description: "当前商品库存状态（实时数据）",
  loader: async () => {
    const items = await redis.hgetall("inventory:current");
    return Object.entries(items)
      .map(([k, v]) => `${k}: ${v}`)
      .join("\n");
  },
  maxTokens: 1000,
  // 无 ttl = 不缓存（每次都拉最新）
});
```

---

## fetch_context 工具

这是 Agent 主动拉取上下文的入口：

```typescript
const fetchContextTool = {
  name: "fetch_context",
  description: "按需获取指定的上下文数据。只在真正需要该数据时调用，避免浪费 token。",
  input_schema: {
    type: "object",
    properties: {
      key: {
        type: "string",
        description: "上下文数据的 key（来自目录列表）",
      },
      reason: {
        type: "string",
        description: "为什么需要这段数据（便于审计和调试）",
      },
    },
    required: ["key"],
  },
};

// Tool 执行器
async function handleFetchContext(
  params: { key: string; reason?: string },
  registry: LazyContextRegistry
): Promise<string> {
  console.log(`[LazyContext] Loading "${params.key}", reason: ${params.reason}`);
  
  try {
    const data = await registry.load(params.key);
    return `## Context: ${params.key}\n\n${data}`;
  } catch (err) {
    return `Error loading context "${params.key}": ${err.message}`;
  }
}
```

---

## Agent Loop 集成

```typescript
async function runAgentWithLazyContext(userMessage: string) {
  const registry = new LazyContextRegistry();
  // ... 注册数据源 ...

  const systemPrompt = `
你是一个智能助手。

${registry.buildDirectory()}

重要：只在真正需要某类数据时才调用 fetch_context，不要预防性地加载所有数据。
`;

  const messages: Message[] = [
    { role: "user", content: userMessage }
  ];

  // Agent Loop
  while (true) {
    const response = await claude.messages.create({
      model: "claude-opus-4-5",
      system: systemPrompt,
      messages,
      tools: [fetchContextTool, ...otherTools],
    });

    if (response.stop_reason === "end_turn") {
      return extractText(response.content);
    }

    // 处理工具调用
    const toolResults: ToolResult[] = [];
    for (const block of response.content) {
      if (block.type !== "tool_use") continue;

      let result: string;
      if (block.name === "fetch_context") {
        // 懒加载上下文
        result = await handleFetchContext(block.input, registry);
      } else {
        result = await dispatchTool(block.name, block.input);
      }

      toolResults.push({ tool_use_id: block.id, content: result });
    }

    messages.push({ role: "assistant", content: response.content });
    messages.push({ role: "user", content: toolResults });
  }
}
```

---

## OpenClaw 中的实际应用

OpenClaw 里 TOOLS.md / USER.md / MEMORY.md 就是一种懒加载设计：

```
# Skills 系统就是懒加载的体现
# 不是一次性加载所有 SKILL.md，而是：
# 1. 系统提示词只包含 <available_skills> 目录（key + description）
# 2. Agent 判断需要哪个 skill
# 3. 用 read 工具按需加载对应 SKILL.md
```

类似模式在 pi-mono 里：

```typescript
// pi-mono/src/context/lazy.ts（概念示例）
export class LazyContextProvider {
  private loaded = new Set<string>();

  async inject(key: string, messages: Message[]): Promise<Message[]> {
    if (this.loaded.has(key)) return messages; // 已注入，跳过
    
    const data = await this.fetch(key);
    this.loaded.add(key);
    
    // 作为 user message 注入（不修改 system prompt）
    return [
      ...messages,
      {
        role: "user",
        content: `[Context Injected: ${key}]\n${data}`,
      },
      {
        role: "assistant", 
        content: "收到，我已了解相关背景信息。",
      },
    ];
  }
}
```

---

## 高级：预测性预加载

结合意图识别，提前预判 Agent 可能需要哪些上下文：

```typescript
class PredictiveContextLoader {
  // 转移矩阵：意图 → 可能需要的上下文
  private transitions: Record<string, string[]> = {
    "check_order": ["user_profile", "order_history"],
    "apply_refund": ["user_profile", "business_rules", "order_history"],
    "check_inventory": ["inventory"],
    "vip_discount": ["user_profile", "business_rules"],
  };

  async preload(detectedIntent: string, registry: LazyContextRegistry) {
    const keysToPreload = this.transitions[detectedIntent] ?? [];
    
    // 并行预加载（后台，不阻塞主流程）
    await Promise.all(
      keysToPreload.map(key => 
        registry.load(key).catch(err => 
          console.warn(`Preload failed for ${key}:`, err)
        )
      )
    );
  }
}
```

---

## 效果对比

| 场景 | 传统方式 | 懒加载 | 节省 |
|------|---------|--------|------|
| 简单问候 | 3500 tokens | 150 tokens | 96% |
| 查订单 | 3500 tokens | 850 tokens | 76% |
| 复杂退款 | 3500 tokens | 2200 tokens | 37% |
| 平均 | 3500 tokens | 1067 tokens | **70%** |

---

## 核心要点

1. **目录 vs 数据**：System prompt 只放目录（key + 描述），不放原始数据
2. **按需加载**：Agent 通过工具调用主动拉取，而不是被动接收
3. **缓存分层**：静态数据长 TTL（业务规则），动态数据短 TTL 或不缓存（库存）
4. **注入方式**：可以作为 tool result 注入，也可以作为额外的 user/assistant 消息对注入
5. **与 Prompt Caching 结合**：频繁加载的上下文命中缓存后几乎零成本

> OpenClaw Skills 系统就是最好的懒加载示例：
> 目录在系统提示词里，Agent 判断后 read 加载对应文件。
> 这是一种工程直觉，现在你知道它的名字了。
