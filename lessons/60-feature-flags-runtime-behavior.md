# 60 - Feature Flags & 动态行为开关

> **核心问题：** 你的 Agent 上线了，突然发现某个工具有 bug，或者需要给 VIP 用户开启高级功能——你能在不重启服务的情况下搞定吗？

---

## 为什么 Agent 需要 Feature Flags？

传统应用用 Feature Flags 来灰度发布新功能。Agent 的场景更复杂：

| 场景 | 传统应用 | Agent |
|------|---------|-------|
| 紧急关闭某功能 | 隐藏 UI 按钮 | 动态移除工具定义 |
| VIP 差异化体验 | 显示高级页面 | 切换到更强的模型 |
| A/B 测试 | 不同界面 | 不同系统提示词 / 工具集 |
| 安全熔断 | 403 响应 | 拒绝执行特定工具调用 |
| 成本控制 | 限流 | 限制 tool_use 次数 / 降级模型 |

**Agent 的工具调用是动态的**——这意味着你可以在运行时精确控制 Agent 能做什么、怎么做。

---

## 核心架构

```
┌─────────────────────────────────────────────────────┐
│                   Agent Loop                         │
│                                                     │
│  Request → [Flag Resolver] → Filtered Tool Set      │
│                    │                                │
│                    ▼                                │
│            [System Prompt Injector]                 │
│                    │                                │
│                    ▼                                │
│              LLM Call → Tool Execution              │
│                              │                      │
│                    [Tool Gate] ← Flag Check         │
└─────────────────────────────────────────────────────┘
```

Flag 的作用点有两个：
1. **提前过滤**：组装工具列表时就剔除禁用的工具（LLM 根本看不到）
2. **执行拦截**：工具调用时二次检查（防止 prompt injection 绕过）

---

## 实现：从简单到复杂

### Level 1：静态 JSON/YAML 配置

最简单的实现，适合小项目：

```typescript
// flags.json
{
  "tools": {
    "web_search": true,
    "file_write": false,     // 临时关闭
    "send_email": "vip_only" // 条件启用
  },
  "models": {
    "default": "claude-3-5-haiku-20241022",
    "power_users": "claude-sonnet-4-5"
  },
  "safety": {
    "max_tool_calls_per_turn": 10,
    "require_confirmation": ["delete", "send_email"]
  }
}
```

```typescript
// flag-resolver.ts
import { readFileSync } from 'fs';
import { watchFile } from 'fs';

class FlagResolver {
  private flags: Record<string, any>;
  private flagPath: string;

  constructor(flagPath: string) {
    this.flagPath = flagPath;
    this.flags = this.load();

    // 文件变更时自动重载——不用重启服务！
    watchFile(flagPath, () => {
      console.log('[Flags] Reloading feature flags...');
      this.flags = this.load();
    });
  }

  private load() {
    try {
      return JSON.parse(readFileSync(this.flagPath, 'utf8'));
    } catch (e) {
      console.error('[Flags] Failed to load flags:', e);
      return this.flags; // 保留旧配置，不崩溃
    }
  }

  isToolEnabled(toolName: string, userId?: string): boolean {
    const flag = this.flags.tools?.[toolName];
    if (flag === undefined || flag === true) return true;
    if (flag === false) return false;
    if (flag === 'vip_only') return this.isVipUser(userId);
    return false;
  }

  getModel(userId?: string): string {
    if (this.isVipUser(userId)) {
      return this.flags.models?.power_users ?? this.flags.models?.default;
    }
    return this.flags.models?.default ?? 'claude-3-5-haiku-20241022';
  }

  requiresConfirmation(toolName: string): boolean {
    return this.flags.safety?.require_confirmation?.includes(toolName) ?? false;
  }

  private isVipUser(userId?: string): boolean {
    return this.flags.vip_users?.includes(userId) ?? false;
  }
}

export const flags = new FlagResolver('./flags.json');
```

**关键技巧**：`watchFile` 实现热重载，修改 `flags.json` 立即生效，0 停机时间。

---

### Level 2：集成到 Agent Loop

```typescript
// agent.ts - 结合 Flag 的工具过滤
import Anthropic from '@anthropic-ai/sdk';
import { flags } from './flag-resolver';
import { ALL_TOOLS } from './tools';

async function runAgent(
  message: string,
  userId: string,
  sessionId: string
) {
  const client = new Anthropic();

  // 1. 根据 Flag 过滤工具集
  const availableTools = ALL_TOOLS.filter(tool =>
    flags.isToolEnabled(tool.name, userId)
  );

  // 2. 根据用户等级选择模型
  const model = flags.getModel(userId);

  // 3. 注入 Flag 相关的系统提示词
  const systemPrompt = buildSystemPrompt(userId);

  const response = await client.messages.create({
    model,
    max_tokens: 4096,
    system: systemPrompt,
    tools: availableTools,  // LLM 只看到被允许的工具
    messages: [{ role: 'user', content: message }]
  });

  // 4. 处理工具调用时再次检查（防止绕过）
  if (response.stop_reason === 'tool_use') {
    return handleToolCalls(response, userId, sessionId);
  }

  return response;
}

async function handleToolCalls(
  response: any,
  userId: string,
  sessionId: string
) {
  const toolUseBlocks = response.content.filter(
    (b: any) => b.type === 'tool_use'
  );

  const results = await Promise.all(
    toolUseBlocks.map(async (toolCall: any) => {
      // 二次 Flag 检查（防 prompt injection 绕过）
      if (!flags.isToolEnabled(toolCall.name, userId)) {
        return {
          type: 'tool_result',
          tool_use_id: toolCall.id,
          content: `Tool "${toolCall.name}" is currently disabled.`,
          is_error: true
        };
      }

      // 需要确认的工具
      if (flags.requiresConfirmation(toolCall.name)) {
        const confirmed = await requestUserConfirmation(
          toolCall, userId, sessionId
        );
        if (!confirmed) {
          return {
            type: 'tool_result',
            tool_use_id: toolCall.id,
            content: 'User cancelled the operation.',
            is_error: true
          };
        }
      }

      // 正常执行
      return executeTool(toolCall);
    })
  );

  return results;
}
```

---

### Level 3：OpenClaw 的实战方案

OpenClaw 用 `config.patch` 实现运行时行为控制，这本质上就是 Feature Flags 系统：

```typescript
// OpenClaw 的工具策略管道就是 Feature Flags 的一种实现
// 来自 OpenClaw 源码思路：

interface ToolPolicy {
  name: string;
  mode: 'allow' | 'deny' | 'ask' | 'allowlist';
  allowlist?: string[];  // allowlist 模式下的白名单
}

// config.patch 更新策略（不重启！）
// POST /config/patch
{
  "toolPolicies": [
    { "name": "exec", "mode": "ask" },         // 执行命令需确认
    { "name": "browser", "mode": "allow" },    // 浏览器自由使用
    { "name": "message", "mode": "allowlist",  // 只能发消息给特定频道
      "allowlist": ["-1001234567890"] }
  ]
}
```

OpenClaw 的 `gateway config.patch` 就是在做这件事——动态调整 Agent 的工具权限：

```bash
# 紧急关闭 exec 工具（不重启服务）
openclaw config patch --path toolPolicies.exec.mode --value deny

# 给特定 session 降级模型
openclaw session set-model claude-3-5-haiku-20241022 --session <id>
```

---

### Level 4：分布式 Flag 服务（生产级）

当你有多个 Agent 实例时，需要集中管理 Flags：

```typescript
// flag-service.ts - 基于 Redis 的分布式 Flag
import Redis from 'ioredis';

class DistributedFlagService {
  private redis: Redis;
  private localCache: Map<string, any> = new Map();
  private cacheExpiry: Map<string, number> = new Map();
  private TTL = 5000; // 5秒本地缓存，减少 Redis 调用

  constructor(redisUrl: string) {
    this.redis = new Redis(redisUrl);

    // 订阅 Flag 变更事件——实时推送更新
    const subscriber = new Redis(redisUrl);
    subscriber.subscribe('flag:updates');
    subscriber.on('message', (_, message) => {
      const { key, value } = JSON.parse(message);
      this.localCache.set(key, value);
      console.log(`[Flags] Updated: ${key} = ${JSON.stringify(value)}`);
    });
  }

  async getFlag(key: string, defaultValue: any = null): Promise<any> {
    // 先查本地缓存
    const now = Date.now();
    if (
      this.localCache.has(key) &&
      (this.cacheExpiry.get(key) ?? 0) > now
    ) {
      return this.localCache.get(key);
    }

    // 缓存过期，查 Redis
    const value = await this.redis.get(`flag:${key}`);
    const parsed = value ? JSON.parse(value) : defaultValue;

    this.localCache.set(key, parsed);
    this.cacheExpiry.set(key, now + this.TTL);

    return parsed;
  }

  async setFlag(key: string, value: any): Promise<void> {
    await this.redis.set(`flag:${key}`, JSON.stringify(value));

    // 广播变更给所有实例
    await this.redis.publish('flag:updates', JSON.stringify({ key, value }));
  }

  // 原子性 toggle（避免并发问题）
  async toggleTool(toolName: string, enabled: boolean): Promise<void> {
    await this.setFlag(`tool:${toolName}`, enabled);
  }

  // 灰度发布：按百分比开放
  async isFeatureEnabled(
    featureName: string,
    userId: string
  ): Promise<boolean> {
    const config = await this.getFlag(`feature:${featureName}`, {
      enabled: false,
      rolloutPercent: 0
    });

    if (!config.enabled) return false;
    if (config.rolloutPercent >= 100) return true;

    // 用 userId hash 保证同一用户始终得到相同结果
    const hash = this.hashUser(userId, featureName);
    return hash < config.rolloutPercent;
  }

  private hashUser(userId: string, feature: string): number {
    // 简单的确定性哈希（生产可用 murmurhash）
    const str = `${userId}:${feature}`;
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash) % 100;
  }
}
```

---

## pi-mono 的实现思路

pi-mono 中通过 Session Config 实现类似效果：

```typescript
// pi-mono 风格：Session 级别的行为控制
interface SessionConfig {
  model: string;
  tools: ToolConfig[];
  systemPrompt: string;
  flags: {
    maxToolCalls: number;
    allowFileWrite: boolean;
    requireHumanApproval: string[];
    debugMode: boolean;
  };
}

class PiAgent {
  private config: SessionConfig;

  async step(messages: Message[]) {
    // Flag 影响每一次 step
    const tools = this.config.tools.filter(t =>
      this.isFlagEnabled(`tool.${t.name}`)
    );

    const response = await this.llm.complete({
      model: this.config.model,  // 可动态切换
      tools,
      messages
    });

    return this.applyFlagConstraints(response);
  }

  private applyFlagConstraints(response: LLMResponse) {
    if (this.config.flags.maxToolCalls > 0) {
      // 截断超出限制的工具调用
      const toolCalls = response.toolCalls.slice(
        0, this.config.flags.maxToolCalls
      );
      return { ...response, toolCalls };
    }
    return response;
  }
}
```

---

## 实战：紧急响应 SOP

```bash
# 场景：某工具出现严重 bug，需要立即关闭

# 方案 A：修改配置文件（热重载，秒级生效）
echo '{"tools": {"broken_tool": false}}' > flags.json

# 方案 B：Redis 命令（多实例同时生效）
redis-cli SET "flag:tool:broken_tool" "false"
redis-cli PUBLISH "flag:updates" '{"key":"tool:broken_tool","value":false}'

# 方案 C：OpenClaw 配置（适用于 OpenClaw 部署）
# 在 Telegram 发送：
openclaw config patch toolPolicies broken_tool deny

# 验证：
redis-cli GET "flag:tool:broken_tool"
# 预期: "false"
```

---

## 设计原则

### 1. Fail-Safe Default（安全默认值）
```typescript
// 读取失败时默认关闭，而非开启
const isEnabled = await flags.getFlag('dangerous_feature', false);
//                                                          ^^^^^ false 更安全
```

### 2. 两点检查（Defense in Depth）
```
工具列表过滤（LLM 不知道有这个工具）
         +
执行前检查（即使 LLM 知道了，也无法执行）
```
不要只做其中一个，因为 prompt injection 可能让 LLM "想起"被过滤的工具。

### 3. Flag 的粒度
```
粗粒度: 开/关整个工具
中粒度: 按用户组/角色
细粒度: 按操作类型 (tool:exec:write vs tool:exec:read)
```

### 4. 审计日志
```typescript
// 每次 Flag 影响行为时记录
logger.info({
  event: 'flag_gate',
  toolName: 'send_email',
  userId,
  flagValue: false,
  action: 'blocked',
  timestamp: new Date().toISOString()
});
```

---

## 小结

| 层次 | 方案 | 生效速度 | 复杂度 |
|------|------|---------|-------|
| 本地文件 | JSON + watchFile | ~1s | ⭐ |
| 环境变量 | process.env | 需重启 | ⭐ |
| Redis | Pub/Sub | 毫秒级 | ⭐⭐⭐ |
| 配置服务 | LaunchDarkly 等 | 毫秒级 | ⭐⭐⭐⭐ |
| OpenClaw config | gateway config.patch | 秒级 | ⭐⭐ |

**Feature Flags 是 Agent 从"玩具"到"生产系统"的关键一步。** 它给你紧急制动的能力——在问题出现时，能在 30 秒内关闭问题功能，而不是手忙脚乱地重新部署。

---

*下一讲预告：Agent 分布式追踪——当你的 Agent 调用链跨越多个服务，如何像侦探一样还原现场？*
