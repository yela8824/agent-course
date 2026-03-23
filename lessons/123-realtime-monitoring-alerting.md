# 123 - Agent 实时监控与告警（Real-time Monitoring & Alerting）

> 追踪告诉你发生了什么；**监控告诉你现在怎么样、需不需要叫醒你**。

---

## 为什么需要专门的监控层？

| 手段 | 粒度 | 延迟 | 适合场景 |
|------|------|------|---------|
| OpenTelemetry Trace | 单次请求 | 低 | 排查具体 bug |
| 结构化日志 | 单次事件 | 低 | 事后分析 |
| **指标 Metrics** | 聚合时序 | 秒级 | **实时状态 + 趋势** |
| 告警 Alert | 阈值触发 | 秒~分 | **自动叫人** |

生产 Agent 的典型 SLA：
- P99 工具响应 < 3s
- 错误率 < 1%
- Token 成本每日不超预算
- 任务队列积压 < 50

这些都需要 **指标 + 告警** 来保障，而不是靠看日志。

---

## 核心概念：四大黄金指标（Four Golden Signals）

```
1. Latency   —— 请求耗时（区分成功与失败）
2. Traffic   —— 请求量 / TPS
3. Errors    —— 错误率
4. Saturation —— 资源饱和度（队列深度、内存、Token 用量）
```

Agent 额外关注：
- **Tool call success rate** —— 每个工具的成功率
- **LLM token burn rate** —— Token 消耗速率
- **Queue depth** —— 等待执行的任务数
- **Agent turn duration** —— 完整一轮 Agent 执行时间

---

## 实现：轻量指标收集器

### 1. 指标注册表（TypeScript）

```typescript
// metrics/registry.ts
type MetricType = 'counter' | 'gauge' | 'histogram';

interface Metric {
  type: MetricType;
  name: string;
  help: string;
  labels: string[];
  values: Map<string, number>;
  buckets?: number[]; // histogram only
  sum?: Map<string, number>;
  count?: Map<string, number>;
}

class MetricsRegistry {
  private metrics = new Map<string, Metric>();

  counter(name: string, help: string, labels: string[] = []) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, {
        type: 'counter', name, help, labels,
        values: new Map()
      });
    }
    return {
      inc: (labelValues: Record<string, string> = {}, by = 1) => {
        const key = this.labelKey(labelValues);
        const m = this.metrics.get(name)!;
        m.values.set(key, (m.values.get(key) ?? 0) + by);
      }
    };
  }

  gauge(name: string, help: string, labels: string[] = []) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, {
        type: 'gauge', name, help, labels,
        values: new Map()
      });
    }
    return {
      set: (value: number, labelValues: Record<string, string> = {}) => {
        const key = this.labelKey(labelValues);
        this.metrics.get(name)!.values.set(key, value);
      },
      inc: (labelValues: Record<string, string> = {}, by = 1) => {
        const key = this.labelKey(labelValues);
        const m = this.metrics.get(name)!;
        m.values.set(key, (m.values.get(key) ?? 0) + by);
      },
      dec: (labelValues: Record<string, string> = {}, by = 1) => {
        const key = this.labelKey(labelValues);
        const m = this.metrics.get(name)!;
        m.values.set(key, (m.values.get(key) ?? 0) - by);
      }
    };
  }

  histogram(name: string, help: string,
            buckets = [0.1, 0.5, 1, 2, 5, 10, 30],
            labels: string[] = []) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, {
        type: 'histogram', name, help, labels,
        values: new Map(), buckets,
        sum: new Map(), count: new Map()
      });
    }
    return {
      observe: (value: number, labelValues: Record<string, string> = {}) => {
        const m = this.metrics.get(name)!;
        const key = this.labelKey(labelValues);
        // 更新 sum & count
        m.sum!.set(key, (m.sum!.get(key) ?? 0) + value);
        m.count!.set(key, (m.count!.get(key) ?? 0) + 1);
        // 更新 buckets
        for (const b of m.buckets!) {
          const bKey = `${key}|le=${b}`;
          if (value <= b) {
            m.values.set(bKey, (m.values.get(bKey) ?? 0) + 1);
          }
        }
      }
    };
  }

  // 导出 Prometheus 格式
  toPrometheus(): string {
    const lines: string[] = [];
    for (const m of this.metrics.values()) {
      lines.push(`# HELP ${m.name} ${m.help}`);
      lines.push(`# TYPE ${m.name} ${m.type}`);
      if (m.type === 'histogram') {
        for (const [bKey, count] of m.values) {
          const [labelPart, lePart] = bKey.split('|le=');
          lines.push(`${m.name}_bucket{${labelPart ? labelPart + ',' : ''}le="${lePart}"} ${count}`);
        }
        for (const [key, sum] of m.sum!) {
          lines.push(`${m.name}_sum{${key}} ${sum}`);
          lines.push(`${m.name}_count{${key}} ${m.count!.get(key) ?? 0}`);
        }
      } else {
        for (const [key, value] of m.values) {
          lines.push(`${m.name}{${key}} ${value}`);
        }
      }
    }
    return lines.join('\n');
  }

  private labelKey(labelValues: Record<string, string>) {
    return Object.entries(labelValues)
      .map(([k, v]) => `${k}="${v}"`)
      .join(',');
  }
}

export const registry = new MetricsRegistry();
```

### 2. Agent 核心指标定义

```typescript
// metrics/agent-metrics.ts
import { registry } from './registry';

export const metrics = {
  // 工具调用次数
  toolCalls: registry.counter(
    'agent_tool_calls_total',
    'Total tool invocations',
    ['tool', 'status'] // status: success | error | timeout
  ),

  // 工具调用耗时（秒）
  toolDuration: registry.histogram(
    'agent_tool_duration_seconds',
    'Tool execution duration',
    [0.05, 0.1, 0.5, 1, 2, 5, 10, 30],
    ['tool']
  ),

  // LLM Token 消耗
  llmTokens: registry.counter(
    'agent_llm_tokens_total',
    'LLM tokens consumed',
    ['model', 'type'] // type: input | output | cache_read
  ),

  // LLM 请求耗时
  llmDuration: registry.histogram(
    'agent_llm_request_duration_seconds',
    'LLM API request duration',
    [0.5, 1, 2, 5, 10, 30, 60],
    ['model']
  ),

  // Agent turn 总耗时
  turnDuration: registry.histogram(
    'agent_turn_duration_seconds',
    'Full agent turn duration',
    [1, 2, 5, 10, 30, 60, 120],
    ['session_type']
  ),

  // 活跃任务数（Gauge）
  activeTasks: registry.gauge(
    'agent_active_tasks',
    'Currently executing tasks',
    ['priority']
  ),

  // 任务队列深度
  queueDepth: registry.gauge(
    'agent_queue_depth',
    'Tasks waiting in queue',
    ['priority']
  ),

  // 错误计数
  errors: registry.counter(
    'agent_errors_total',
    'Agent errors by category',
    ['category'] // tool_error | llm_error | timeout | validation
  ),
};
```

### 3. 工具中间件：自动埋点

```typescript
// middleware/metrics-middleware.ts
import { metrics } from '../metrics/agent-metrics';

export function withMetrics<T extends (...args: any[]) => Promise<any>>(
  toolName: string,
  fn: T
): T {
  return (async (...args: any[]) => {
    const start = Date.now();
    try {
      const result = await fn(...args);
      const duration = (Date.now() - start) / 1000;
      metrics.toolCalls.inc({ tool: toolName, status: 'success' });
      metrics.toolDuration.observe(duration, { tool: toolName });
      return result;
    } catch (err) {
      const duration = (Date.now() - start) / 1000;
      metrics.toolCalls.inc({ tool: toolName, status: 'error' });
      metrics.toolDuration.observe(duration, { tool: toolName });
      metrics.errors.inc({ category: 'tool_error' });
      throw err;
    }
  }) as T;
}

// 使用方式
const webSearch = withMetrics('web_search', async (query: string) => {
  // 实际搜索逻辑
  return await callSearchAPI(query);
});
```

---

## 告警引擎

### 告警规则定义

```typescript
// alerting/alert-rules.ts
interface AlertRule {
  name: string;
  severity: 'info' | 'warning' | 'critical';
  condition: () => Promise<boolean>;
  message: () => string;
  cooldownMs: number; // 防止告警风暴
}

class AlertEngine {
  private rules: AlertRule[] = [];
  private lastFired = new Map<string, number>();
  private channels: AlertChannel[] = [];

  addRule(rule: AlertRule) {
    this.rules.push(rule);
  }

  addChannel(channel: AlertChannel) {
    this.channels.push(channel);
  }

  async evaluate() {
    const now = Date.now();
    for (const rule of this.rules) {
      const lastTime = this.lastFired.get(rule.name) ?? 0;
      // 冷却期内跳过
      if (now - lastTime < rule.cooldownMs) continue;

      try {
        const triggered = await rule.condition();
        if (triggered) {
          this.lastFired.set(rule.name, now);
          await this.fire(rule);
        }
      } catch (err) {
        console.error(`Alert rule ${rule.name} evaluation failed:`, err);
      }
    }
  }

  private async fire(rule: AlertRule) {
    const alert = {
      name: rule.name,
      severity: rule.severity,
      message: rule.message(),
      firedAt: new Date().toISOString(),
    };
    await Promise.allSettled(
      this.channels.map(ch => ch.send(alert))
    );
  }
}
```

### 具体告警规则

```typescript
// alerting/agent-alert-rules.ts
import { metricsStore } from './metrics-store'; // 时序存储

export function registerAgentAlerts(engine: AlertEngine) {

  // 1. 错误率告警
  engine.addRule({
    name: 'high_error_rate',
    severity: 'critical',
    cooldownMs: 5 * 60 * 1000, // 5分钟冷却
    condition: async () => {
      const { errors, total } = await metricsStore.getRate('agent_errors_total', '5m');
      return total > 10 && errors / total > 0.05; // >5% 错误率
    },
    message: () => '🔴 Agent 错误率超过 5%（5分钟窗口）'
  });

  // 2. P99 延迟告警
  engine.addRule({
    name: 'high_latency_p99',
    severity: 'warning',
    cooldownMs: 10 * 60 * 1000,
    condition: async () => {
      const p99 = await metricsStore.getPercentile('agent_turn_duration_seconds', 99, '5m');
      return p99 > 30; // P99 > 30秒
    },
    message: () => '⚠️ Agent turn P99 延迟超过 30s'
  });

  // 3. 队列积压告警
  engine.addRule({
    name: 'queue_depth_high',
    severity: 'warning',
    cooldownMs: 2 * 60 * 1000,
    condition: async () => {
      const depth = await metricsStore.getGauge('agent_queue_depth');
      return depth > 50;
    },
    message: () => `⚠️ 任务队列积压超过 50 个`
  });

  // 4. Token 日消耗预算告警
  engine.addRule({
    name: 'daily_token_budget',
    severity: 'warning',
    cooldownMs: 60 * 60 * 1000, // 1小时冷却
    condition: async () => {
      const todayTokens = await metricsStore.getCounter('agent_llm_tokens_total', '24h');
      const DAILY_BUDGET = 5_000_000; // 500万 tokens/天
      return todayTokens > DAILY_BUDGET * 0.8; // 超过 80% 就警告
    },
    message: () => '💰 今日 Token 消耗已超 80% 日预算'
  });

  // 5. 工具连续失败
  engine.addRule({
    name: 'tool_consecutive_failures',
    severity: 'critical',
    cooldownMs: 5 * 60 * 1000,
    condition: async () => {
      // 检查过去 1 分钟某个工具是否全部失败
      const stats = await metricsStore.getToolStats('1m');
      return Object.values(stats).some(
        ({ success, total }) => total >= 5 && success === 0
      );
    },
    message: () => '🔴 某工具近 1 分钟连续全部失败'
  });
}
```

### 告警渠道：Telegram 推送

```typescript
// alerting/telegram-channel.ts
interface AlertPayload {
  name: string;
  severity: string;
  message: string;
  firedAt: string;
}

interface AlertChannel {
  send(alert: AlertPayload): Promise<void>;
}

class TelegramAlertChannel implements AlertChannel {
  constructor(
    private botToken: string,
    private chatId: string
  ) {}

  async send(alert: AlertPayload) {
    const emoji = {
      info: 'ℹ️',
      warning: '⚠️',
      critical: '🚨'
    }[alert.severity] ?? '📢';

    const text = [
      `${emoji} *[${alert.severity.toUpperCase()}] ${alert.name}*`,
      ``,
      alert.message,
      ``,
      `🕐 ${new Date(alert.firedAt).toLocaleString('zh-CN', { timeZone: 'Asia/Shanghai' })}`
    ].join('\n');

    await fetch(`https://api.telegram.org/bot${this.botToken}/sendMessage`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        chat_id: this.chatId,
        text,
        parse_mode: 'Markdown'
      })
    });
  }
}
```

---

## Prometheus + Grafana：生产级方案

### /metrics HTTP 端点（Express）

```typescript
// server/metrics-endpoint.ts
import express from 'express';
import { registry } from '../metrics/registry';

const app = express();

// 供 Prometheus scrape
app.get('/metrics', (req, res) => {
  res.set('Content-Type', 'text/plain; version=0.0.4');
  res.send(registry.toPrometheus());
});

// 内部健康检查
app.get('/health', async (req, res) => {
  const checks = {
    queue: await checkQueueHealth(),
    llm: await checkLLMHealth(),
    db: await checkDBHealth(),
  };
  const healthy = Object.values(checks).every(c => c.ok);
  res.status(healthy ? 200 : 503).json({ healthy, checks });
});

app.listen(9090, () => console.log('Metrics server on :9090'));
```

### prometheus.yml 配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'agent'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/metrics'

# 告警规则文件
rule_files:
  - 'agent_alerts.yml'

# Alertmanager（负责去重、分组、路由）
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### agent_alerts.yml（Prometheus 告警规则）

```yaml
groups:
  - name: agent_sla
    interval: 30s
    rules:
      - alert: AgentHighErrorRate
        expr: |
          rate(agent_errors_total[5m]) /
          rate(agent_tool_calls_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Agent 错误率 > 5%"
          description: "过去 5m 错误率: {{ $value | humanizePercentage }}"

      - alert: AgentHighP99Latency
        expr: |
          histogram_quantile(0.99,
            rate(agent_turn_duration_seconds_bucket[5m])
          ) > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent turn P99 > 30s"

      - alert: AgentQueueBacklog
        expr: agent_queue_depth > 50
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "任务队列积压 > 50"
```

---

## OpenClaw 实践：Cron 定时巡检

```typescript
// OpenClaw cron job 每分钟跑告警规则
// 在 cron payload 里：
{
  "schedule": { "kind": "cron", "expr": "* * * * *", "tz": "UTC" },
  "payload": {
    "kind": "agentTurn",
    "message": "Run alerting engine evaluation. Check agent metrics and fire alerts if thresholds exceeded."
  },
  "sessionTarget": "isolated"
}
```

或者直接在主 Agent 循环里挂一个定时检查：

```typescript
// agent/monitoring-loop.ts
export async function startMonitoringLoop(engine: AlertEngine, intervalMs = 30_000) {
  setInterval(async () => {
    try {
      await engine.evaluate();
    } catch (err) {
      console.error('Monitoring loop error:', err);
    }
  }, intervalMs);

  console.log(`Monitoring loop started, checking every ${intervalMs / 1000}s`);
}
```

---

## SLA Dashboard 关键面板

```
┌─────────────────────────────────────────────────────┐
│  Agent Health Dashboard                              │
├─────────────────┬─────────────────┬─────────────────┤
│  Error Rate     │  P99 Latency    │  Queue Depth    │
│  0.2%  ✅      │  2.3s  ✅       │  12  ✅         │
├─────────────────┼─────────────────┼─────────────────┤
│  Token/hr       │  Active Tasks   │  Tool Success   │
│  48.2k          │  7              │  99.1%  ✅      │
├─────────────────────────────────────────────────────┤
│  [工具调用耗时分布 - Heatmap]                        │
│  [Token 消耗趋势 - 折线图]                           │
│  [错误类型分布 - 饼图]                               │
└─────────────────────────────────────────────────────┘
```

---

## 与 OpenTelemetry 的分工

| | OpenTelemetry Trace | Metrics + Alerting |
|--|--|--|
| **问题** | "这次请求为什么慢？" | "现在系统健康吗？" |
| **数据** | 请求级 Span | 聚合时序数据 |
| **存储** | Jaeger/Tempo | Prometheus/InfluxDB |
| **响应** | 开发者手动排查 | **自动告警、叫人** |
| **成本** | 较高（全量采样）| 低（聚合后很小）|

**最佳实践：两者结合**
- Metrics 发现问题（触发告警）
- Trace 定位根因（排查具体请求）

---

## 小结

1. **四大黄金信号** —— 延迟、流量、错误率、饱和度，缺一不可
2. **工具中间件自动埋点** —— 不改业务代码，一行 `withMetrics()` 搞定
3. **冷却期防告警风暴** —— 同一告警 N 分钟内只触发一次
4. **分级告警** —— info/warning/critical，对应不同响应流程
5. **Prometheus 格式** —— 开放标准，Grafana 直接接入
6. **Cron 巡检** —— OpenClaw 原生支持，无需额外基础设施

> 没有监控的 Agent 上生产，就像在黑暗中开飞机。
