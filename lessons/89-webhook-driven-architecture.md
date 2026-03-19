# 89 - Agent Webhook 驱动架构（Webhook-Driven Architecture）

> **核心思想**：别让 Agent 傻乎乎地轮询，让事件来找 Agent。

---

## 为什么需要 Webhook 驱动？

大多数初学者写的 Agent 是**主动轮询**模式：

```python
while True:
    emails = check_email()
    if emails:
        process(emails)
    time.sleep(60)  # 每分钟查一次
```

问题很明显：
- **浪费资源**：99% 的轮询结果是"没有新消息"
- **延迟高**：最坏情况要等一整个轮询周期
- **难扩展**：100 个监控源 = 100 个轮询循环

**Webhook 驱动**翻转了这个模式：

```
事件发生 → 外部系统 HTTP POST → Agent 接收 → 立即处理
```

---

## 三种 Agent 事件驱动架构

### 模式 1：直接 Webhook（同步处理）

```
GitHub Push → POST /webhook/github → Agent 立即处理 → 200 OK
```

适合：处理时间 < 5秒的轻量任务

### 模式 2：Webhook + 队列（异步处理）

```
Stripe 支付 → POST /webhook/stripe → 入队 → 返回 202
                                         ↓
                                    Worker Agent 消费
```

适合：处理耗时长、需要重试保障的任务

### 模式 3：Webhook → Cron 触发（OpenClaw 风格）

```
外部事件 → POST /webhook → 写入文件/DB → 触发 Cron Job → 隔离 Agent 处理
```

OpenClaw 本身就是这个模式的完整实现！

---

## 实战：用 OpenClaw 实现 GitHub PR Webhook

### 第一步：注册 Webhook 端点

OpenClaw 通过 `gateway.config` 可以暴露 HTTP 端点：

```yaml
# openclaw gateway config
webhooks:
  github_pr:
    path: /hooks/github-pr
    secret: ${GITHUB_WEBHOOK_SECRET}
    handler: inject_system_event  # 转成 systemEvent 注入主会话
```

### 第二步：验签 + 解析 Payload

生产环境必须验证签名，防止伪造请求：

```python
import hmac
import hashlib

def verify_github_signature(payload: bytes, signature: str, secret: str) -> bool:
    """
    GitHub 用 HMAC-SHA256 签名每个 webhook 请求
    Header: X-Hub-Signature-256: sha256=<hex>
    """
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    received = signature.removeprefix("sha256=")
    
    # 用 compare_digest 防时序攻击
    return hmac.compare_digest(expected, received)


def handle_github_pr(request):
    payload = request.body
    sig = request.headers.get("X-Hub-Signature-256", "")
    
    if not verify_github_signature(payload, sig, GITHUB_SECRET):
        return Response(status=401)  # 直接拒绝伪造请求
    
    event = request.headers.get("X-GitHub-Event")
    data = json.loads(payload)
    
    if event == "pull_request" and data["action"] in ["opened", "synchronize"]:
        # 触发 Agent 审查
        enqueue_pr_review(data["pull_request"])
    
    return Response(status=202)  # 快速返回，异步处理
```

### 第三步：pi-mono 的 Webhook 处理器

来自 [pi-mono](https://github.com/badlogic/pi-mono) 的真实模式：

```typescript
// packages/api/src/webhooks/github.ts

interface WebhookHandler {
  verify(req: Request): Promise<boolean>;
  handle(event: string, payload: unknown): Promise<void>;
}

class GitHubWebhookHandler implements WebhookHandler {
  constructor(
    private readonly secret: string,
    private readonly agentQueue: AgentQueue
  ) {}

  async verify(req: Request): Promise<boolean> {
    const sig = req.headers.get("x-hub-signature-256");
    const body = await req.arrayBuffer();
    
    const key = await crypto.subtle.importKey(
      "raw",
      new TextEncoder().encode(this.secret),
      { name: "HMAC", hash: "SHA-256" },
      false,
      ["sign"]
    );
    
    const expected = await crypto.subtle.sign("HMAC", key, body);
    const expectedHex = "sha256=" + bufToHex(expected);
    
    return timingSafeEqual(sig ?? "", expectedHex);
  }

  async handle(event: string, payload: unknown): Promise<void> {
    const pr = payload as GitHubPRPayload;
    
    if (event !== "pull_request") return;
    if (!["opened", "synchronize"].includes(pr.action)) return;
    
    // 推入 Agent 任务队列
    await this.agentQueue.push({
      type: "pr_review",
      priority: "high",
      idempotencyKey: `pr-${pr.pull_request.id}-${pr.pull_request.head.sha}`,
      payload: {
        repo: pr.repository.full_name,
        prNumber: pr.pull_request.number,
        diff: pr.pull_request.diff_url,
        author: pr.pull_request.user.login,
      },
    });
  }
}
```

---

## 幂等性：Webhook 必须考虑的问题

外部系统**会重复投递** webhook（网络超时重试、系统故障等）。

Agent 必须实现幂等处理：

```typescript
class IdempotentWebhookProcessor {
  constructor(
    private readonly db: Database,
    private readonly agent: Agent
  ) {}

  async process(webhookId: string, handler: () => Promise<void>): Promise<void> {
    // 检查是否已处理
    const existing = await this.db.query(
      "SELECT status FROM webhook_log WHERE id = ?",
      [webhookId]
    );
    
    if (existing?.status === "processed") {
      console.log(`[Webhook] ${webhookId} 已处理，跳过`);
      return;  // 幂等：重复投递直接忽略
    }
    
    // 记录处理中
    await this.db.execute(
      "INSERT INTO webhook_log (id, status, received_at) VALUES (?, 'processing', NOW())",
      [webhookId]
    );
    
    try {
      await handler();
      
      // 标记完成
      await this.db.execute(
        "UPDATE webhook_log SET status = 'processed', processed_at = NOW() WHERE id = ?",
        [webhookId]
      );
    } catch (err) {
      // 标记失败，允许重试
      await this.db.execute(
        "UPDATE webhook_log SET status = 'failed', error = ? WHERE id = ?",
        [String(err), webhookId]
      );
      throw err;
    }
  }
}

// 使用
await processor.process(
  req.headers.get("X-GitHub-Delivery"),  // GitHub 提供唯一投递 ID
  () => handler.handle(event, payload)
);
```

---

## OpenClaw 实战：Cron + Webhook 混合调度

OpenClaw 的 cron 系统可以被 webhook 触发（one-shot at 模式）：

```typescript
// 当 webhook 到达时，动态创建一个立即执行的 cron job

async function onWebhookReceived(payload: PRPayload) {
  await cron.add({
    name: `pr-review-${payload.prNumber}`,
    schedule: {
      kind: "at",
      at: new Date().toISOString(),  // 立即执行
    },
    payload: {
      kind: "agentTurn",
      message: `
        请审查这个 PR：
        - 仓库：${payload.repo}
        - PR #${payload.prNumber}
        - 作者：${payload.author}
        - Diff：${payload.diffUrl}
        
        检查代码质量、潜在 bug、安全问题，给出审查意见。
      `,
      timeoutSeconds: 300,
    },
    sessionTarget: "isolated",
    delivery: {
      mode: "announce",
      channel: "pr-reviews",
    },
  });
}
```

---

## 关键设计原则

### 1. 快速返回 202
Webhook 处理器必须在 **< 3 秒**内响应，否则发送方会认为失败并重试：

```python
@app.post("/webhook/github")
async def github_webhook(request: Request):
    # ✅ 验签后立即入队，返回 202
    payload = await request.body()
    verify_signature(payload, request.headers)
    
    asyncio.create_task(process_async(payload))  # 异步处理
    return Response(status_code=202)  # 立即返回

# ❌ 错误做法：同步等待 Agent 处理完
@app.post("/webhook/github")  
async def github_webhook_bad(request: Request):
    result = await agent.process(payload)  # 可能超时！
    return {"result": result}
```

### 2. 签名验证是红线

没有签名验证 = 任何人都能伪造事件触发你的 Agent。

### 3. 事件去重

用幂等键（webhook delivery ID）去重，别让同一个事件触发两次 Agent。

### 4. 死信队列

处理失败的事件要进死信队列，支持人工介入重试：

```
失败 → 重试 3 次 → 进 DLQ → 告警 → 人工确认 → 手动重放
```

---

## 常见 Webhook 集成场景

| 场景 | 触发源 | Agent 行为 |
|------|--------|-----------|
| PR 自动审查 | GitHub PR opened | 拉取 diff → 分析 → 评论 |
| 支付成功处理 | Stripe payment.succeeded | 发送确认邮件 → 开通权限 |
| 监控告警响应 | Grafana alert | 分析指标 → 给出诊断 → 通知 |
| IoT 异常处理 | 传感器超阈值 | 识别异常模式 → 触发响应动作 |
| 用户注册欢迎 | 数据库 trigger | 个性化欢迎消息 → 引导流程 |

---

## 总结

```
轮询模式：Agent 主动找事件（低效、高延迟）
Webhook 模式：事件主动找 Agent（高效、低延迟）

Webhook 三件套：
  ✅ 验签（防伪造）
  ✅ 幂等（防重复）
  ✅ 快速返回（防超时重试）

OpenClaw 实现：
  Webhook → 入队 → Cron at 模式 → 隔离 Sub-agent 处理
```

事件驱动让 Agent 从"主动轮询"变成"被动响应"，是生产级 Agent 的必备架构能力。

---

## 延伸阅读

- [GitHub Webhooks 文档](https://docs.github.com/en/webhooks)
- [Stripe Webhook 最佳实践](https://stripe.com/docs/webhooks/best-practices)
- OpenClaw cron 系统源码（`packages/gateway/src/cron/`）
