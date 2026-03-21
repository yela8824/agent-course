# 108 - Agent 健康检查与自动重启（Health Check & Auto-Restart）

> **核心问题：** Agent 进程挂了谁来救它？—— 它得自己救自己，或者让守卫来救。

---

## 为什么 Agent 需要健康检查？

LLM 调用、工具执行、网络请求……Agent 任何一环都可能：

- **卡死（Hang）**：LLM 不返回、工具无限等待
- **内存泄漏**：长跑进程越来越慢，直到 OOM
- **状态腐败**：内部状态不一致，回复乱七八糟
- **外部依赖断联**：DB 断连、Redis 失联、API 限流

传统服务挂了重启就行。但 Agent 有**状态**（对话上下文、任务进度），重启前要先保存，重启后要能恢复。

这就是健康检查与自动重启要解决的问题。

---

## 三种探针：Liveness / Readiness / Startup

借鉴 Kubernetes 的探针概念，Agent 也需要三种自检：

```typescript
interface AgentProbe {
  // 存活探针：Agent 进程还活着吗？（不活则重启）
  liveness(): Promise<ProbeResult>;
  
  // 就绪探针：Agent 准备好接受新任务了吗？（不就绪则暂停接单）
  readiness(): Promise<ProbeResult>;
  
  // 启动探针：Agent 初始化完成了吗？（未完成则不暴露）
  startup(): Promise<ProbeResult>;
}

interface ProbeResult {
  healthy: boolean;
  reason?: string;
  details?: Record<string, unknown>;
}
```

### 实现一个 Agent 健康检查器

```typescript
// health/AgentHealthChecker.ts

export class AgentHealthChecker {
  private lastHeartbeat = Date.now();
  private taskCount = 0;
  private errorCount = 0;
  private readonly maxErrorRate = 0.3; // 30% 错误率触发告警
  
  // 存活检查：心跳超时 = 进程卡死
  async checkLiveness(): Promise<ProbeResult> {
    const elapsed = Date.now() - this.lastHeartbeat;
    const timeout = 60_000; // 60 秒无心跳 = 卡死
    
    if (elapsed > timeout) {
      return {
        healthy: false,
        reason: `Heartbeat timeout: ${elapsed}ms since last beat`,
      };
    }
    
    // 检查事件循环是否被阻塞
    const loopLag = await this.measureEventLoopLag();
    if (loopLag > 5000) {
      return {
        healthy: false,
        reason: `Event loop lag: ${loopLag}ms (CPU block?)`,
      };
    }
    
    return { healthy: true };
  }
  
  // 就绪检查：依赖项可用吗？
  async checkReadiness(): Promise<ProbeResult> {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkLLMEndpoint(),
      this.checkMemoryUsage(),
    ]);
    
    const failures = checks
      .filter(r => r.status === 'rejected' || (r.status === 'fulfilled' && !r.value.healthy))
      .map((r, i) => ['database', 'llm', 'memory'][i]);
    
    if (failures.length > 0) {
      return {
        healthy: false,
        reason: `Dependencies unavailable: ${failures.join(', ')}`,
      };
    }
    
    // 检查错误率
    const errorRate = this.taskCount > 0 ? this.errorCount / this.taskCount : 0;
    if (errorRate > this.maxErrorRate) {
      return {
        healthy: false,
        reason: `Error rate too high: ${(errorRate * 100).toFixed(1)}%`,
      };
    }
    
    return { healthy: true };
  }
  
  // 心跳：Agent 主循环每次迭代调用
  heartbeat() {
    this.lastHeartbeat = Date.now();
  }
  
  recordTaskResult(success: boolean) {
    this.taskCount++;
    if (!success) this.errorCount++;
  }
  
  private async measureEventLoopLag(): Promise<number> {
    return new Promise(resolve => {
      const start = Date.now();
      setImmediate(() => resolve(Date.now() - start));
    });
  }
  
  private async checkDatabase(): Promise<ProbeResult> {
    // 执行轻量 ping 查询
    try {
      await db.query('SELECT 1');
      return { healthy: true };
    } catch (e) {
      return { healthy: false, reason: String(e) };
    }
  }
  
  private async checkLLMEndpoint(): Promise<ProbeResult> {
    // 用超时短的请求测 LLM API 可达性
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 3000);
    try {
      await fetch('https://api.anthropic.com/health', {
        signal: controller.signal,
      });
      return { healthy: true };
    } catch {
      return { healthy: false, reason: 'LLM endpoint unreachable' };
    } finally {
      clearTimeout(timer);
    }
  }
  
  private async checkMemoryUsage(): Promise<ProbeResult> {
    const used = process.memoryUsage();
    const heapUsedMB = used.heapUsed / 1024 / 1024;
    const limit = 1024; // 1GB
    
    if (heapUsedMB > limit) {
      return {
        healthy: false,
        reason: `Heap usage ${heapUsedMB.toFixed(0)}MB exceeds ${limit}MB`,
      };
    }
    return { healthy: true, details: { heapUsedMB } };
  }
}
```

---

## 看门狗：进程级自动重启

健康检查发现问题后，谁来重启？两层机制：

### 第一层：进程内 Watchdog（自救）

```typescript
// health/Watchdog.ts

export class AgentWatchdog {
  private checker: AgentHealthChecker;
  private checkInterval: NodeJS.Timeout | null = null;
  private consecutiveFailures = 0;
  private readonly maxFailures = 3;
  
  constructor(private agent: Agent, checker: AgentHealthChecker) {
    this.checker = checker;
  }
  
  start() {
    this.checkInterval = setInterval(async () => {
      await this.runCheck();
    }, 15_000); // 每 15 秒检查一次
    
    console.log('[Watchdog] Started, checking every 15s');
  }
  
  stop() {
    if (this.checkInterval) {
      clearInterval(this.checkInterval);
      this.checkInterval = null;
    }
  }
  
  private async runCheck() {
    const [liveness, readiness] = await Promise.all([
      this.checker.checkLiveness(),
      this.checker.checkReadiness(),
    ]);
    
    if (!liveness.healthy) {
      console.error('[Watchdog] LIVENESS FAILED:', liveness.reason);
      this.consecutiveFailures++;
      
      if (this.consecutiveFailures >= this.maxFailures) {
        console.error('[Watchdog] Max failures reached, triggering restart');
        await this.triggerRestart('liveness');
      }
      return;
    }
    
    if (!readiness.healthy) {
      console.warn('[Watchdog] READINESS DEGRADED:', readiness.reason);
      // 就绪失败：暂停接单但不重启
      await this.agent.pauseIncomingTasks('degraded');
    } else {
      // 恢复
      this.consecutiveFailures = 0;
      await this.agent.resumeIncomingTasks();
    }
  }
  
  private async triggerRestart(reason: string) {
    try {
      // 1. 保存当前状态（关键！）
      await this.agent.checkpoint(`watchdog-restart:${reason}`);
      
      // 2. 通知监控系统
      await alerting.send({
        severity: 'critical',
        message: `Agent restarting due to ${reason} failure`,
      });
      
      // 3. 优雅退出（让进程管理器重启）
      process.exit(1);
    } catch (e) {
      // 连 checkpoint 都失败了，直接退
      console.error('[Watchdog] Failed to checkpoint, forcing exit:', e);
      process.exit(1);
    }
  }
}
```

### 第二层：进程管理器（外部守卫）

```typescript
// OpenClaw 的 gateway restart 机制本质上就是外部守卫
// 类似地，可以用 PM2 / systemd / Docker restart policy

// pm2.config.js
module.exports = {
  apps: [{
    name: 'my-agent',
    script: './dist/main.js',
    
    // 重启策略
    max_restarts: 10,          // 最多重启 10 次
    min_uptime: '30s',         // 运行不足 30 秒算崩溃
    restart_delay: 5000,       // 重启前等 5 秒
    exp_backoff_restart_delay: 100, // 指数退避：第 N 次失败等 100*2^N ms
    
    // 内存限制
    max_memory_restart: '1G',
    
    // 环境变量
    env: {
      NODE_ENV: 'production',
      CHECKPOINT_DIR: '/var/agent/checkpoints',
    },
  }],
};
```

---

## OpenClaw 中的实际用法

OpenClaw 本身就有 gateway restart 机制，结合 HEARTBEAT.md 可以实现定期自检：

```markdown
<!-- HEARTBEAT.md -->
## 健康自检清单（每 30 分钟）

- [ ] 检查内存使用：process.memoryUsage().heapUsed < 512MB
- [ ] 检查 LLM API 可达性（最近 5 分钟有成功调用？）
- [ ] 检查 Redis 连接状态
- [ ] 汇报 metrics 到 Grafana
- 若发现异常 → 通知老板 + 触发 `gateway restart`
```

实际触发重启：

```typescript
// 在 Agent 代码里调用 OpenClaw 的 gateway restart
import { exec } from 'child_process';

async function selfRestart(reason: string) {
  console.log(`Self-restart triggered: ${reason}`);
  
  // 保存状态
  await saveCheckpoint();
  
  // 通过 OpenClaw gateway 工具重启（会自动恢复会话）
  // openclaw gateway restart
  exec('openclaw gateway restart', (err) => {
    if (err) process.exit(1); // fallback
  });
}
```

---

## 状态保存与恢复：重启不等于从头来

重启最怕的是**丢失进行中的任务**。正确做法：

```typescript
// checkpoint/AgentCheckpoint.ts

interface AgentState {
  activeTasks: Task[];
  conversationHistory: Message[];
  pendingToolCalls: ToolCall[];
  version: number;
  timestamp: number;
}

export class CheckpointManager {
  private readonly dir: string;
  
  constructor(dir = '/var/agent/checkpoints') {
    this.dir = dir;
    fs.mkdirSync(dir, { recursive: true });
  }
  
  // 保存检查点（原子写入，防止半写）
  async save(state: AgentState): Promise<void> {
    const path = join(this.dir, 'state.json');
    const tmp = `${path}.tmp`;
    
    await fs.promises.writeFile(tmp, JSON.stringify(state, null, 2));
    await fs.promises.rename(tmp, path); // 原子替换
    
    console.log(`[Checkpoint] Saved v${state.version} at ${new Date(state.timestamp).toISOString()}`);
  }
  
  // 启动时恢复
  async restore(): Promise<AgentState | null> {
    const path = join(this.dir, 'state.json');
    
    try {
      const raw = await fs.promises.readFile(path, 'utf-8');
      const state = JSON.parse(raw) as AgentState;
      
      // 版本过期检查（超过 1 小时的状态不恢复）
      const age = Date.now() - state.timestamp;
      if (age > 3_600_000) {
        console.warn('[Checkpoint] State too old, starting fresh');
        return null;
      }
      
      console.log(`[Checkpoint] Restored v${state.version}, age: ${(age / 1000).toFixed(0)}s`);
      return state;
    } catch {
      return null; // 无检查点，全新启动
    }
  }
}

// 启动时使用
const checkpoint = new CheckpointManager();
const savedState = await checkpoint.restore();

const agent = new Agent({
  initialTasks: savedState?.activeTasks ?? [],
  history: savedState?.conversationHistory ?? [],
});
```

---

## 监控大盘：让健康状态可观测

```typescript
// metrics/HealthMetrics.ts — 配合 Prometheus/Grafana

import { Gauge, Counter } from 'prom-client';

const livenessGauge = new Gauge({
  name: 'agent_liveness',
  help: '1 = alive, 0 = dead',
});

const readinessGauge = new Gauge({
  name: 'agent_readiness',
  help: '1 = ready, 0 = degraded',
});

const restartCounter = new Counter({
  name: 'agent_restarts_total',
  help: 'Total number of agent restarts',
  labelNames: ['reason'],
});

// 暴露 /health HTTP 端点
app.get('/health/live', async (req, res) => {
  const result = await healthChecker.checkLiveness();
  livenessGauge.set(result.healthy ? 1 : 0);
  res.status(result.healthy ? 200 : 503).json(result);
});

app.get('/health/ready', async (req, res) => {
  const result = await healthChecker.checkReadiness();
  readinessGauge.set(result.healthy ? 1 : 0);
  res.status(result.healthy ? 200 : 503).json(result);
});
```

---

## 关键原则

| 场景 | 做法 |
|------|------|
| 进程卡死 | Watchdog 检测心跳超时 → 保存状态 → 退出 → 进程管理器重启 |
| 依赖断联 | Readiness 降级 → 暂停接单 → 依赖恢复后自动就绪 |
| 内存泄漏 | 设内存上限 → 超限触发 GC 或优雅退出 |
| 错误率飙升 | 熔断 + 降级 + 告警（结合 Lesson 56 Circuit Breaker） |
| 重启后恢复 | Checkpoint 原子写入 → 启动时恢复未完成任务 |

---

## 小结

- **三种探针**：Liveness（活着吗）/ Readiness（能干活吗）/ Startup（初始化完了吗）
- **两层守卫**：进程内 Watchdog（自救） + 外部进程管理器（他救）
- **原子 Checkpoint**：重启前保存状态，重启后无缝恢复
- **HTTP /health 端点**：让负载均衡/k8s/监控系统能感知 Agent 健康状态
- **OpenClaw 实践**：HEARTBEAT.md 做定期自检，`gateway restart` 做外部重启

> Agent 不死则已，死则快速恢复、状态不丢。这才是生产级 Agent 的韧性标准。
