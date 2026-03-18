# 78 - Cron 驱动的自主 Agent 工作流

> **核心问题：** Agent 如何从"被动响应"进化为"主动执行"？

---

## 为什么需要 Cron 驱动？

绝大多数 Agent 教程都在讲"用户问 → Agent 答"这种**被动模式**。但真实的生产 Agent 往往需要：

- 每天早上 9 点自动汇总昨日数据
- 每 30 分钟检查一次监控指标
- 每周五生成周报并发送给团队
- 某个时间点提醒用户做某件事

这就是 **Cron 驱动的自主 Agent**：不等人问，到点自己干。

---

## 核心架构

```
┌──────────────────────────────────────────────────┐
│              Cron Scheduler                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ at(一次) │  │every(循环)│  │ cron(表达式) │   │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       └─────────────┴────────────────┘            │
│                      │                            │
│              触发 Payload                          │
│         ┌────────────┴──────────────┐             │
│         │ systemEvent (注入主会话)  │             │
│         │ agentTurn   (独立子会话)  │             │
│         └───────────────────────────┘             │
└──────────────────────────────────────────────────┘
```

### 两种 Payload 模式

| 模式 | 适用场景 | 隔离性 |
|------|---------|-------|
| `systemEvent` | 需要主会话上下文，轻量检查 | 低（共享主会话） |
| `agentTurn` | 独立任务，有完整 Agent 能力 | 高（独立子会话） |

---

## OpenClaw 实战：Cron Job 完整示例

### 1. 每日数据报告（一次性 + 循环）

```javascript
// 一次性任务：明天早上 9 点执行
await cron.add({
  name: "daily-report",
  schedule: {
    kind: "at",
    at: "2026-03-19T09:00:00+11:00"  // 带时区的 ISO 时间
  },
  payload: {
    kind: "agentTurn",
    message: `
      请执行今日数据报告：
      1. 查询过去 24 小时的关键指标
      2. 对比昨日数据，标注异常
      3. 生成 Markdown 报告发送到 Telegram
    `,
    timeoutSeconds: 120
  },
  sessionTarget: "isolated",
  delivery: {
    mode: "announce",
    channel: "telegram"
  }
})

// 循环任务：每 3 小时执行一次
await cron.add({
  name: "monitor-check",
  schedule: {
    kind: "every",
    everyMs: 3 * 60 * 60 * 1000,  // 3小时
    anchorMs: Date.now()            // 从现在开始计算
  },
  payload: {
    kind: "agentTurn",
    message: "检查监控指标，如有异常立即报警"
  },
  sessionTarget: "isolated"
})
```

### 2. Cron 表达式（精确时间控制）

```javascript
// 每周一早上 9 点（带时区！）
await cron.add({
  name: "weekly-summary",
  schedule: {
    kind: "cron",
    expr: "0 9 * * 1",          // 标准 cron 表达式
    tz: "Australia/Sydney"       // 关键：指定时区
  },
  payload: {
    kind: "agentTurn",
    message: "生成本周工作总结并发送到团队"
  },
  sessionTarget: "isolated",
  delivery: {
    mode: "announce"
  }
})
```

> ⚠️ **时区陷阱**：不指定 `tz` 的 ISO 时间戳默认 UTC。
> 悉尼是 UTC+11（夏令时）/ UTC+10（标准时间），搞错了任务会在凌晨触发！

---

## 实战模式：Heartbeat vs Cron

这是生产中最常见的设计决策：

```
用 Heartbeat（心跳）：
  ✅ 多个检查可以批量合并（邮件 + 日历 + 通知 一次搞定）
  ✅ 需要主会话上下文
  ✅ 时间偏差 ±5分钟 可以接受
  ✅ 减少 API 调用

用 Cron：
  ✅ 精确到分钟的时间控制
  ✅ 任务需要完全隔离（不受主会话干扰）
  ✅ 不同任务用不同模型（省成本）
  ✅ 输出直接推送到 Webhook 而非主会话
```

**OpenClaw 的 HEARTBEAT.md 模式：**

```markdown
# HEARTBEAT.md

## 每次心跳检查（按优先级）
- [ ] 有无新邮件（>2h 未查则检查）
- [ ] 日历：未来 2h 内有无事件
- [ ] MysteryBox 监控：有无异常告警

## 状态追踪
lastEmailCheck: {timestamp}
lastCalendarCheck: {timestamp}
```

---

## pi-mono 实现参考：任务调度器

```typescript
// pi-mono 风格的 Cron Job 管理
interface CronJob {
  id: string
  name: string
  schedule: Schedule
  payload: Payload
  enabled: boolean
  lastRun?: Date
  nextRun?: Date
}

class AgentCronScheduler {
  private jobs = new Map<string, CronJob>()
  private timers = new Map<string, NodeJS.Timeout>()

  add(job: CronJob): void {
    this.jobs.set(job.id, job)
    this.scheduleNext(job)
  }

  private scheduleNext(job: CronJob): void {
    const delay = this.calculateDelay(job.schedule)
    
    const timer = setTimeout(async () => {
      await this.execute(job)
      
      // 循环任务：执行后重新调度
      if (job.schedule.kind === 'every' || job.schedule.kind === 'cron') {
        this.scheduleNext(job)
      }
    }, delay)
    
    this.timers.set(job.id, timer)
  }

  private async execute(job: CronJob): Promise<void> {
    job.lastRun = new Date()
    
    if (job.payload.kind === 'agentTurn') {
      // 启动隔离子会话执行任务
      const session = await spawnIsolatedSession({
        task: job.payload.message,
        model: job.payload.model,
        timeoutSeconds: job.payload.timeoutSeconds ?? 60
      })
      
      const result = await session.wait()
      await this.deliver(job, result)
    } else {
      // systemEvent：注入主会话
      await mainSession.injectEvent(job.payload.text)
    }
  }
}
```

---

## 关键设计：隔离执行防止污染

```
❌ 错误做法：所有 Cron 任务共享主会话
   - 任务 A 的上下文污染任务 B
   - 长任务阻塞主会话响应用户
   - 任务失败导致主会话状态混乱

✅ 正确做法：每个 Cron 任务独立子会话
   - 完整的 Agent 能力（工具调用、推理等）
   - 失败不影响主会话
   - 可以设置独立超时
   - 可以使用更便宜的模型
```

**OpenClaw 实现：**

```javascript
// sessionTarget: "isolated" = 每次触发都是全新子会话
{
  sessionTarget: "isolated",  // ← 这一行是关键
  payload: {
    kind: "agentTurn",
    message: "...",
    model: "anthropic/claude-haiku-3-5",  // 便宜模型跑定时任务
    timeoutSeconds: 60
  }
}
```

---

## 交付模式：结果去哪里？

```javascript
// 模式1：Announce（推送到聊天频道）
delivery: {
  mode: "announce",
  channel: "telegram",
  to: "-1001234567890"  // 指定群组
}

// 模式2：Webhook（回调你的服务）
delivery: {
  mode: "webhook",
  to: "https://your-service.com/agent-results"
  // 任务完成后 POST finished-run 事件
}

// 模式3：无交付（主会话自己处理）
delivery: { mode: "none" }
```

---

## 防重复执行：幂等性保障

```typescript
// 问题：网络抖动导致同一任务触发两次怎么办？
class IdempotentCronJob {
  private executed = new Set<string>()
  
  async execute(job: CronJob, scheduledTime: Date): Promise<void> {
    // 用任务ID + 计划时间生成幂等键
    const idempotencyKey = `${job.id}:${scheduledTime.toISOString()}`
    
    if (this.executed.has(idempotencyKey)) {
      console.log(`[SKIP] 任务已执行: ${idempotencyKey}`)
      return
    }
    
    this.executed.add(idempotencyKey)
    
    try {
      await this.doExecute(job)
    } catch (err) {
      // 失败时移除，允许重试
      this.executed.delete(idempotencyKey)
      throw err
    }
  }
}
```

---

## 真实案例：OpenClaw 的课程更新 Cron

这门课程本身就是一个 Cron Agent 的真实案例：

```javascript
// 这门课的触发 Job（每3小时）
{
  name: "agent-course-update",
  schedule: {
    kind: "every",
    everyMs: 3 * 60 * 60 * 1000
  },
  payload: {
    kind: "systemEvent",
    text: `Agent 开发课程时间！
    1. 给 Rust学习小组分享新知识点
    2. 写入 lessons/XX-topic.md
    3. 更新 README.md 目录
    4. git commit && git push`
  },
  sessionTarget: "main"  // 注入主会话，需要访问工具和工作区
}
```

整个课程内容的生产、发布、归档都是 **Cron Agent 自主完成**，无需人工干预。

---

## 小结

| 概念 | 要点 |
|------|------|
| 触发模式 | `at`（一次）/ `every`（循环）/ `cron`（表达式） |
| Payload | `systemEvent`（轻量）/ `agentTurn`（完整 Agent） |
| 隔离性 | `isolated` 子会话 防止主会话污染 |
| 时区 | ISO 时间戳默认 UTC，cron 表达式必须指定 `tz` |
| 幂等性 | 任务ID + 计划时间 生成幂等键防重复执行 |
| 交付 | announce（聊天）/ webhook（回调）/ none（自处理） |

**一句话总结：Cron Agent = 给 Agent 装一个闹钟，让它按时自己干活。**

---

## 延伸阅读

- [37-long-running-tasks-checkpointing](./37-long-running-tasks-checkpointing.md) - 长任务断点续传
- [61-task-queue-priority-scheduling](./61-task-queue-priority-scheduling.md) - 任务队列与优先级
- [13-state-persistence](./13-state-persistence.md) - 状态持久化
