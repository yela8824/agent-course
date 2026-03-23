# 118 - Agent 成本归因与用量配额（Cost Attribution & Usage Quotas）

> **核心问题**：API 账单来了，你知道哪个用户、哪个功能、哪个 Agent 花了多少钱吗？

---

## 为什么需要成本归因？

多租户系统里，LLM API 费用可能是最大的变量成本。  
没有归因 = 烧钱不知道烧在哪 = 无法定价、无法限制滥用、无法优化。

三个核心目标：

| 目标 | 说明 |
|------|------|
| **归因（Attribution）** | 每一次 token 消耗，都知道是谁花的 |
| **配额（Quota）** | 每个用户 / 租户 / 功能，有上限 |
| **告警（Alert）** | 快到上限时提前通知，超出时拒绝或降级 |

---

## 核心数据结构

```typescript
interface TokenUsage {
  promptTokens: number;
  completionTokens: number;
  totalTokens: number;
}

interface CostRecord {
  sessionId: string;
  userId: string;        // 租户/用户 ID
  agentId: string;       // 哪个 Agent
  feature: string;       // 功能标签，如 "chat" / "code-review" / "summary"
  model: string;         // "claude-sonnet-4" / "gpt-4o"
  usage: TokenUsage;
  costUsd: number;       // 计算后的美元成本
  timestamp: number;
}

interface UsageQuota {
  userId: string;
  dailyLimitUsd: number;
  monthlyLimitUsd: number;
  dailyUsedUsd: number;
  monthlyUsedUsd: number;
  lastResetDay: string;   // "2026-03-23"
  lastResetMonth: string; // "2026-03"
}
```

---

## 模型定价表

```typescript
const MODEL_PRICING: Record<string, { input: number; output: number }> = {
  // 单位：$ per 1M tokens
  "claude-opus-4":      { input: 15.0,  output: 75.0  },
  "claude-sonnet-4-6":  { input: 3.0,   output: 15.0  },
  "claude-haiku-3-5":   { input: 0.8,   output: 4.0   },
  "gpt-4o":             { input: 2.5,   output: 10.0  },
  "gpt-4o-mini":        { input: 0.15,  output: 0.6   },
};

function calculateCost(model: string, usage: TokenUsage): number {
  const pricing = MODEL_PRICING[model];
  if (!pricing) return 0;
  return (
    (usage.promptTokens / 1_000_000) * pricing.input +
    (usage.completionTokens / 1_000_000) * pricing.output
  );
}
```

---

## 成本追踪中间件

### LLM 调用拦截层

```typescript
// cost-tracker.ts
import { redis } from "./redis";

class CostTracker {
  async track(record: CostRecord): Promise<void> {
    const pipeline = redis.pipeline();
    const dayKey   = new Date().toISOString().slice(0, 10);   // "2026-03-23"
    const monthKey = new Date().toISOString().slice(0, 7);    // "2026-03"

    // 1. 写入明细日志（供审计和分析）
    const logKey = `cost:log:${record.userId}:${dayKey}`;
    pipeline.rpush(logKey, JSON.stringify(record));
    pipeline.expire(logKey, 90 * 86400); // 保留 90 天

    // 2. 按用户累计每日 / 每月成本
    pipeline.incrbyfloat(`cost:daily:${record.userId}:${dayKey}`,    record.costUsd);
    pipeline.incrbyfloat(`cost:monthly:${record.userId}:${monthKey}`, record.costUsd);

    // 3. 按 feature 维度归因
    pipeline.incrbyfloat(`cost:feature:${record.feature}:${dayKey}`, record.costUsd);

    // 4. 全局统计
    pipeline.incrbyfloat(`cost:total:${dayKey}`, record.costUsd);

    await pipeline.exec();
  }

  async getDailyUsage(userId: string, day?: string): Promise<number> {
    const d = day ?? new Date().toISOString().slice(0, 10);
    const val = await redis.get(`cost:daily:${userId}:${d}`);
    return parseFloat(val ?? "0");
  }

  async getMonthlyUsage(userId: string, month?: string): Promise<number> {
    const m = month ?? new Date().toISOString().slice(0, 7);
    const val = await redis.get(`cost:monthly:${userId}:${m}`);
    return parseFloat(val ?? "0");
  }
}

export const costTracker = new CostTracker();
```

### 包装 Anthropic 调用

```typescript
// llm-with-tracking.ts
import Anthropic from "@anthropic-ai/sdk";
import { costTracker, calculateCost } from "./cost-tracker";

const client = new Anthropic();

export async function trackedCompletion(
  params: Anthropic.MessageCreateParams,
  context: { userId: string; agentId: string; feature: string }
): Promise<Anthropic.Message> {
  const message = await client.messages.create(params);

  // 从响应中提取 usage
  const usage: TokenUsage = {
    promptTokens:     message.usage.input_tokens,
    completionTokens: message.usage.output_tokens,
    totalTokens:      message.usage.input_tokens + message.usage.output_tokens,
  };

  const costUsd = calculateCost(params.model as string, usage);

  await costTracker.track({
    sessionId: crypto.randomUUID(),
    userId:    context.userId,
    agentId:   context.agentId,
    feature:   context.feature,
    model:     params.model as string,
    usage,
    costUsd,
    timestamp: Date.now(),
  });

  return message;
}
```

---

## 配额检查与执行

```typescript
// quota-guard.ts

interface QuotaConfig {
  dailyLimitUsd: number;
  monthlyLimitUsd: number;
  warningThreshold: number; // 0.8 = 80% 时发告警
}

const DEFAULT_QUOTAS: Record<string, QuotaConfig> = {
  free:    { dailyLimitUsd: 0.10,  monthlyLimitUsd: 1.00,   warningThreshold: 0.8 },
  starter: { dailyLimitUsd: 1.00,  monthlyLimitUsd: 20.00,  warningThreshold: 0.8 },
  pro:     { dailyLimitUsd: 10.00, monthlyLimitUsd: 200.00, warningThreshold: 0.9 },
};

export class QuotaGuard {
  async check(userId: string, tier: string): Promise<QuotaCheckResult> {
    const config = DEFAULT_QUOTAS[tier] ?? DEFAULT_QUOTAS.free;
    const [dailyUsed, monthlyUsed] = await Promise.all([
      costTracker.getDailyUsage(userId),
      costTracker.getMonthlyUsage(userId),
    ]);

    // 超限 → 拒绝
    if (dailyUsed >= config.dailyLimitUsd) {
      return { allowed: false, reason: "daily_limit_exceeded", dailyUsed, monthlyUsed };
    }
    if (monthlyUsed >= config.monthlyLimitUsd) {
      return { allowed: false, reason: "monthly_limit_exceeded", dailyUsed, monthlyUsed };
    }

    // 接近上限 → 告警（非阻塞）
    const dailyRatio   = dailyUsed / config.dailyLimitUsd;
    const monthlyRatio = monthlyUsed / config.monthlyLimitUsd;
    const warning = dailyRatio >= config.warningThreshold || monthlyRatio >= config.warningThreshold;
    if (warning) {
      // 异步发告警，不阻塞主流程
      this.sendWarning(userId, { dailyUsed, monthlyUsed, config }).catch(() => {});
    }

    return { allowed: true, warning, dailyUsed, monthlyUsed };
  }

  private async sendWarning(userId: string, ctx: object): Promise<void> {
    console.warn(`[QuotaGuard] User ${userId} approaching limit`, ctx);
    // 接入通知渠道：Telegram、Email、Webhook 等
  }
}

interface QuotaCheckResult {
  allowed: boolean;
  reason?: string;
  warning?: boolean;
  dailyUsed: number;
  monthlyUsed: number;
}
```

### 接入 Agent Loop

```typescript
// agent-loop.ts（改造版）
const quotaGuard = new QuotaGuard();

async function runAgent(userId: string, tier: string, prompt: string) {
  // ① 先检查配额
  const quota = await quotaGuard.check(userId, tier);
  if (!quota.allowed) {
    return {
      error: quota.reason === "daily_limit_exceeded"
        ? "今日额度已用完，明天再来哦 🙏"
        : "本月额度已用完，请升级套餐",
    };
  }

  // ② 正常执行（带成本追踪）
  const response = await trackedCompletion(
    { model: "claude-sonnet-4-6", max_tokens: 1024, messages: [{ role: "user", content: prompt }] },
    { userId, agentId: "main", feature: "chat" }
  );

  return { text: response.content[0].type === "text" ? response.content[0].text : "" };
}
```

---

## 报表与分析

```typescript
// cost-report.ts

export async function generateDailyReport(day: string): Promise<DailyReport> {
  // 获取所有用户的当日用量（生产环境用 SCAN 代替 KEYS）
  const keys = await redis.keys(`cost:daily:*:${day}`);
  
  const entries = await Promise.all(
    keys.map(async (key) => {
      const userId = key.split(":")[2];
      const cost   = parseFloat((await redis.get(key)) ?? "0");
      return { userId, cost };
    })
  );

  entries.sort((a, b) => b.cost - a.cost);

  const totalCost = entries.reduce((s, e) => s + e.cost, 0);
  const topUsers  = entries.slice(0, 10);

  // 按 feature 统计
  const featureKeys = await redis.keys(`cost:feature:*:${day}`);
  const featureBreakdown: Record<string, number> = {};
  for (const fk of featureKeys) {
    const feature = fk.split(":")[2];
    featureBreakdown[feature] = parseFloat((await redis.get(fk)) ?? "0");
  }

  return { day, totalCost, topUsers, featureBreakdown };
}

interface DailyReport {
  day: string;
  totalCost: number;
  topUsers: Array<{ userId: string; cost: number }>;
  featureBreakdown: Record<string, number>;
}
```

---

## OpenClaw 实战：Cron 每日报表

```typescript
// 每天早上 9 点发送成本报表
// cron.add({
//   name: "daily-cost-report",
//   schedule: { kind: "cron", expr: "0 9 * * *", tz: "Australia/Sydney" },
//   payload: { kind: "agentTurn", message: "生成昨日成本报表并发送到运营群" },
//   sessionTarget: "isolated"
// })

// Agent 收到指令后：
const yesterday = new Date(Date.now() - 86400_000).toISOString().slice(0, 10);
const report = await generateDailyReport(yesterday);

const lines = [
  `📊 **${yesterday} 成本报表**`,
  `💰 总成本：$${report.totalCost.toFixed(4)}`,
  ``,
  `**Top 用户：**`,
  ...report.topUsers.map((u, i) => `${i + 1}. ${u.userId}: $${u.cost.toFixed(4)}`),
  ``,
  `**功能分布：**`,
  ...Object.entries(report.featureBreakdown)
    .sort((a, b) => b[1] - a[1])
    .map(([f, c]) => `• ${f}: $${c.toFixed(4)}`),
];

await sendToChannel(lines.join("\n"));
```

---

## pi-mono 的实践参考

pi-mono 里，每个 `AgentSession` 都携带 `tenantId`，LLM 调用经过统一的 `ModelRouter`。  
成本追踪就挂在 Router 的 `afterCall` hook 上，对业务层完全透明：

```typescript
// pi-mono: src/model/router.ts（简化）
class ModelRouter {
  async call(params, context) {
    const result = await this.provider.complete(params);
    
    // hook: 成本追踪（不影响返回值）
    setImmediate(() => {
      costTracker.track({
        userId:    context.tenantId,
        agentId:   context.agentId,
        feature:   context.feature ?? "unknown",
        model:     params.model,
        usage:     result.usage,
        costUsd:   calculateCost(params.model, result.usage),
        sessionId: context.sessionId,
        timestamp: Date.now(),
      });
    });

    return result;
  }
}
```

`setImmediate` 确保追踪异步执行，不增加 P50 延迟。

---

## 常见陷阱

| 陷阱 | 解决方案 |
|------|----------|
| 配额检查和扣费不原子 | 用 Lua 脚本或 Redis 事务保证先读后写原子性 |
| 每次调用都实时查 Redis | 本地缓存 5 秒，高频场景减少 RTT |
| 忘记计算 cache read tokens | Anthropic 有 `cache_read_input_tokens`，单独计价 |
| 免费用户被 1 个人滥用 | 结合 IP / 设备指纹做多维度限制 |
| 月初重置逻辑 bug | 用 `YYYY-MM` 作 key，自然过期，不需要主动重置 |

---

## 小结

```
成本归因 = 每次 LLM 调用都打标签（用户、功能、模型）
用量配额 = Redis 累计 + 每次调用前 check
告警 = 阈值检测 + 异步通知，不阻塞主流程
报表 = Cron 定时聚合，运营可见
```

核心原则：**追踪要轻（异步、低延迟），执行要严（超限必拒）**。

---

*下一讲预告：Agent 多模态输入路由（Multi-Modal Input Routing）——图片、音频、文档混合输入时，如何智能路由到最合适的模型和处理管道。*
