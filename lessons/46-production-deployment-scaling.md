# 46 - Agent 生产部署与水平扩容

> 代码能跑不等于能上线。这一课讲 Agent 从本地走向生产的完整路径。

---

## 为什么 Agent 部署比普通服务难？

普通 Web 服务是无状态的，水平扩容很简单。Agent 有几个特殊挑战：

1. **会话状态**：对话历史、工具调用状态存在哪？
2. **长连接**：Streaming 需要保持 HTTP 连接，不能随便杀
3. **工具副作用**：工具调用是有副作用的，不能随意重试
4. **LLM 费用**：单次请求可能很贵，需要精细控制并发
5. **不可预测延迟**：LLM 响应时间 1s~60s 不等

---

## 架构原则：无状态 Agent Worker

**核心思路**：把 Agent 的状态外置，Worker 本身无状态，可以随意扩缩。

```
                    ┌─────────────────────────────────┐
                    │           Load Balancer          │
                    └────────────┬────────────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                       │
   ┌──────▼──────┐        ┌──────▼──────┐        ┌──────▼──────┐
   │  Agent Pod  │        │  Agent Pod  │        │  Agent Pod  │
   │  (Worker)   │        │  (Worker)   │        │  (Worker)   │
   └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
          │                      │                       │
          └──────────────────────┼──────────────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                       │
   ┌──────▼──────┐        ┌──────▼──────┐        ┌──────▼──────┐
   │  Redis      │        │  PostgreSQL │        │  S3/R2      │
   │  (Sessions) │        │  (History)  │        │  (Files)    │
   └─────────────┘        └─────────────┘        └─────────────┘
```

---

## OpenClaw 的部署架构

OpenClaw 本身就是这种设计的例子，看它的核心结构：

```typescript
// openclaw/src/gateway/server.ts (简化)
export class GatewayServer {
  private sessionStore: RedisSessionStore;  // 状态外置
  private workerPool: AgentWorkerPool;       // 无状态 Worker 池
  
  async handleRequest(req: Request) {
    // 1. 从 Redis 恢复 session
    const session = await this.sessionStore.get(req.sessionId);
    
    // 2. 选一个空闲 Worker 处理
    const worker = await this.workerPool.acquire();
    
    try {
      // 3. Worker 执行 agent loop
      const result = await worker.run(session, req.message);
      
      // 4. 结果写回 Redis
      await this.sessionStore.save(session);
      
      return result;
    } finally {
      // 5. 归还 Worker（无论成功失败）
      this.workerPool.release(worker);
    }
  }
}
```

**关键点**：Worker 是无状态的工具执行器，状态完全在 sessionStore 里。

---

## 容器化：Dockerfile 最佳实践

```dockerfile
# 多阶段构建，减小镜像体积
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# 单独安装 devDeps 用于构建
COPY . .
RUN npm ci && npm run build

# 最终镜像只包含生产依赖
FROM node:22-alpine AS runtime
WORKDIR /app

# 非 root 用户运行（安全最佳实践）
RUN addgroup -S agent && adduser -S agent -G agent

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json .

USER agent

# Health check：每30s检查一次
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget -q -O- http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## pi-mono 的 Worker 池设计

`pi-mono` 里有个经典的 Worker Pool 实现，值得学习：

```typescript
// pi-mono/packages/runtime/src/worker-pool.ts
export class AgentWorkerPool {
  private available: AgentWorker[] = [];
  private busy: Set<AgentWorker> = new Set();
  private waitQueue: Array<(w: AgentWorker) => void> = [];
  
  constructor(
    private readonly maxWorkers: number,
    private readonly factory: () => AgentWorker
  ) {
    // 预热：提前创建 Worker
    for (let i = 0; i < Math.min(2, maxWorkers); i++) {
      this.available.push(factory());
    }
  }
  
  async acquire(): Promise<AgentWorker> {
    // 有空闲 Worker 直接用
    if (this.available.length > 0) {
      const worker = this.available.pop()!;
      this.busy.add(worker);
      return worker;
    }
    
    // 还能创建新 Worker
    if (this.busy.size < this.maxWorkers) {
      const worker = this.factory();
      this.busy.add(worker);
      return worker;
    }
    
    // 池满了，排队等待
    return new Promise(resolve => {
      this.waitQueue.push(resolve);
    });
  }
  
  release(worker: AgentWorker): void {
    this.busy.delete(worker);
    
    // 有人在排队，优先给他
    if (this.waitQueue.length > 0) {
      const next = this.waitQueue.shift()!;
      this.busy.add(worker);
      next(worker);
    } else {
      this.available.push(worker);
    }
  }
  
  get stats() {
    return {
      available: this.available.length,
      busy: this.busy.size,
      waiting: this.waitQueue.length,
      utilization: this.busy.size / this.maxWorkers
    };
  }
}
```

---

## Kubernetes 部署配置

```yaml
# agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-worker
spec:
  replicas: 3  # 起步3个实例
  selector:
    matchLabels:
      app: agent-worker
  template:
    spec:
      containers:
      - name: agent
        image: myregistry/agent:v1.2.0
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"     # Agent 内存用量波动大
            cpu: "2000m"      # LLM 调用期间 CPU 也高
        env:
        - name: WORKER_POOL_SIZE
          value: "5"          # 每个 Pod 5个并发 Worker
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: redis-url
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health/ready   # 检查 Worker 池是否就绪
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
---
# 自动扩容：根据 CPU 和自定义指标（等待队列长度）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: agent_queue_depth    # 自定义指标：等待队列深度
      target:
        type: AverageValue
        averageValue: "10"         # 平均每 Pod 排队超过10个就扩容
```

---

## 健康检查：三层设计

```typescript
// src/health.ts
export function setupHealthRoutes(app: Express, pool: AgentWorkerPool) {
  
  // 1. Liveness: 进程活着吗？
  // K8s 检测失败会重启 Pod
  app.get('/health/live', (req, res) => {
    res.json({ status: 'alive', pid: process.pid });
  });
  
  // 2. Readiness: 能处理请求吗？
  // K8s 检测失败会从 Service 摘掉，不重启
  app.get('/health/ready', async (req, res) => {
    const stats = pool.stats;
    
    // Worker 池利用率超过95%，拒绝新请求
    if (stats.utilization > 0.95) {
      return res.status(503).json({
        status: 'busy',
        ...stats
      });
    }
    
    // 检查依赖服务
    const redisOk = await checkRedis();
    if (!redisOk) {
      return res.status(503).json({ status: 'deps_down', redis: false });
    }
    
    res.json({ status: 'ready', ...stats });
  });
  
  // 3. Startup: 启动完成了吗？
  // 给 Agent 预热时间，避免过早收到流量
  let isStarted = false;
  warmup().then(() => { isStarted = true; });
  
  app.get('/health/startup', (req, res) => {
    if (!isStarted) return res.status(503).json({ status: 'starting' });
    res.json({ status: 'started' });
  });
}
```

---

## 优雅关闭（Graceful Shutdown）

这是 Agent 部署最容易忽略的细节。滚动更新时，正在执行的工具调用不能被强制中断：

```typescript
// src/shutdown.ts
export function setupGracefulShutdown(server: Server, pool: AgentWorkerPool) {
  let isShuttingDown = false;
  
  async function shutdown(signal: string) {
    if (isShuttingDown) return;
    isShuttingDown = true;
    
    console.log(`[${signal}] Starting graceful shutdown...`);
    
    // 1. 停止接受新连接
    server.close();
    
    // 2. 等待现有任务完成（最多等60s）
    const deadline = Date.now() + 60_000;
    while (pool.stats.busy > 0 && Date.now() < deadline) {
      console.log(`Waiting for ${pool.stats.busy} active workers...`);
      await sleep(1000);
    }
    
    if (pool.stats.busy > 0) {
      console.warn(`Force closing ${pool.stats.busy} workers after timeout`);
    }
    
    // 3. 清理资源
    await pool.destroy();
    process.exit(0);
  }
  
  process.on('SIGTERM', () => shutdown('SIGTERM'));  // K8s 发的
  process.on('SIGINT', () => shutdown('SIGINT'));    // Ctrl+C
}
```

K8s 的 `terminationGracePeriodSeconds` 要配合这个 deadline：

```yaml
spec:
  terminationGracePeriodSeconds: 90  # 给 Agent 90s 收尾
```

---

## 关键监控指标

部署上线后，这些指标要持续关注：

```
# Worker 池状态
agent_worker_available_total    # 空闲 Worker 数
agent_worker_busy_total         # 繁忙 Worker 数
agent_worker_queue_depth        # 排队等待数 ← 最重要

# 性能指标
agent_request_duration_p50      # 中位延迟
agent_request_duration_p99      # 尾部延迟（能看出 LLM 超时问题）
agent_llm_tokens_total          # Token 消耗（直接影响成本）

# 错误指标
agent_tool_errors_total         # 工具调用失败数
agent_llm_errors_total          # LLM API 错误数
agent_session_timeout_total     # 会话超时数
```

---

## 实战检查清单

部署前问自己：

- [ ] Worker 是无状态的吗？状态都外置了？
- [ ] 有 Liveness / Readiness 健康检查吗？
- [ ] 优雅关闭逻辑写了吗？terminationGracePeriodSeconds 够长？
- [ ] 内存限制设合理了吗？（Agent 内存很容易爆）
- [ ] Worker 池大小和 Pod 副本数匹配吗？
- [ ] HPA 的扩容指标是业务指标（队列深度）而不只是 CPU？
- [ ] 滚动更新策略设了吗？`maxUnavailable: 0` 保证不中断服务？

---

## 小结

Agent 部署的核心是**把状态从 Worker 剥离**。状态外置之后，Worker 就变成了普通无状态服务，水平扩容就跟扩容 Web API 一样简单。其他的健康检查、优雅关闭、HPA，都是这个思路的延伸。

下节课可能讲：**Agent 版本灰度发布（A/B Testing & Canary Deployment）**
