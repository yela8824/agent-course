# 51 - Agent Versioning & A/B Testing：行为版本管理与灰度测试

> 你改了 system prompt，上线后用户全崩了。怎么避免？答案是：像部署代码一样部署 Agent 行为。

---

## 为什么需要 Agent 版本管理？

代码有版本控制，但 Agent 的"行为"——system prompt、工具集、模型参数——往往随意改动，没有回滚机制。

**典型翻车场景：**
- 把 system prompt 里的一句话改了，Agent 开始拒绝执行本该执行的任务
- 换了个模型版本，structured output 格式变了，下游解析全崩
- 新工具上线后 Agent 开始乱调，导致数据污染
- 想知道新 prompt 是否比旧 prompt 好，但没有对比数据

**解决方案：把 Agent 行为当成可版本化、可灰度、可回滚的资产。**

---

## 核心概念

### Agent 配置的组成

```
AgentConfig {
  version: "v2.3.1"
  system_prompt: "..."
  tools: ["search", "code_exec", "file_read"]
  model: "claude-sonnet-4-5"
  parameters: { temperature: 0.7, max_tokens: 4096 }
  skill_set: ["coding-agent", "github"]
}
```

这些加在一起定义了 Agent 的**行为版本（Behavioral Version）**。

### 版本 vs 配置 vs 实例

```
AgentVersion (不可变快照)
    └── AgentConfig (当前活跃配置)
            └── AgentInstance (运行时实例)
```

---

## 实现方案

### 1. 配置快照 + Hash 指纹

最简单的版本管理：给配置算一个 hash，变了就是新版本。

```typescript
// pi-mono 风格的配置版本化
import { createHash } from 'crypto'

interface AgentBehavior {
  systemPrompt: string
  tools: string[]
  model: string
  temperature: number
}

function computeBehaviorHash(behavior: AgentBehavior): string {
  const canonical = JSON.stringify({
    systemPrompt: behavior.systemPrompt,
    tools: [...behavior.tools].sort(),  // 排序保证一致性
    model: behavior.model,
    temperature: behavior.temperature,
  })
  return createHash('sha256').update(canonical).digest('hex').slice(0, 12)
}

// 使用
const v1: AgentBehavior = {
  systemPrompt: "You are a helpful assistant",
  tools: ["search", "code_exec"],
  model: "claude-sonnet-4-5",
  temperature: 0.7
}

const hash = computeBehaviorHash(v1)
// → "a3f9c2b1d4e7"  这就是这个行为的"版本指纹"
```

### 2. 行为注册表（Behavior Registry）

```typescript
// 存储所有历史版本
class BehaviorRegistry {
  private versions = new Map<string, AgentBehavior>()
  private history: Array<{ hash: string; timestamp: Date; label?: string }> = []

  register(behavior: AgentBehavior, label?: string): string {
    const hash = computeBehaviorHash(behavior)
    
    if (!this.versions.has(hash)) {
      this.versions.set(hash, behavior)
      this.history.push({ hash, timestamp: new Date(), label })
      console.log(`[Registry] New behavior registered: ${hash} (${label ?? 'unlabeled'})`)
    }
    
    return hash
  }

  get(hash: string): AgentBehavior | undefined {
    return this.versions.get(hash)
  }

  rollback(steps = 1): string {
    // 回退 N 个版本
    const target = this.history[this.history.length - 1 - steps]
    if (!target) throw new Error('No previous version to rollback to')
    return target.hash
  }

  list(): Array<{ hash: string; timestamp: Date; label?: string }> {
    return [...this.history].reverse()  // 最新在前
  }
}

const registry = new BehaviorRegistry()

const v1Hash = registry.register({
  systemPrompt: "You are a helpful assistant",
  tools: ["search"],
  model: "claude-haiku-3-5",
  temperature: 0.7
}, "initial-release")

const v2Hash = registry.register({
  systemPrompt: "You are an expert assistant. Be concise.",
  tools: ["search", "code_exec"],
  model: "claude-sonnet-4-5",
  temperature: 0.5
}, "upgraded-model")
```

### 3. A/B 测试路由器

把不同用户路由到不同版本的 Agent 行为：

```typescript
interface ABExperiment {
  id: string
  variants: Array<{
    behaviorHash: string
    weight: number  // 流量权重，如 0.5 = 50%
    label: string
  }>
  metrics: string[]  // 要追踪的指标
}

class ABTestRouter {
  private experiments = new Map<string, ABExperiment>()

  createExperiment(experiment: ABExperiment) {
    // 验证权重之和 = 1
    const totalWeight = experiment.variants.reduce((sum, v) => sum + v.weight, 0)
    if (Math.abs(totalWeight - 1.0) > 0.001) {
      throw new Error(`Variant weights must sum to 1.0, got ${totalWeight}`)
    }
    this.experiments.set(experiment.id, experiment)
  }

  // 根据用户 ID 确定性地分组（同一用户永远在同一组）
  assignVariant(experimentId: string, userId: string): string {
    const experiment = this.experiments.get(experimentId)
    if (!experiment) throw new Error(`Experiment ${experimentId} not found`)

    // 用 hash 保证同一 userId 每次分到同一组
    const seed = createHash('md5')
      .update(`${experimentId}:${userId}`)
      .digest('hex')
    const normalized = parseInt(seed.slice(0, 8), 16) / 0xFFFFFFFF  // 0~1

    let cumulative = 0
    for (const variant of experiment.variants) {
      cumulative += variant.weight
      if (normalized < cumulative) {
        return variant.behaviorHash
      }
    }

    return experiment.variants[experiment.variants.length - 1].behaviorHash
  }
}

// 使用示例
const router = new ABTestRouter()

router.createExperiment({
  id: "prompt-upgrade-test",
  variants: [
    { behaviorHash: v1Hash, weight: 0.5, label: "control" },
    { behaviorHash: v2Hash, weight: 0.5, label: "treatment" }
  ],
  metrics: ["task_success_rate", "user_satisfaction", "avg_turns"]
})

// 请求来了，先查用户在哪个组
const userId = "user_12345"
const behaviorHash = router.assignVariant("prompt-upgrade-test", userId)
const behavior = registry.get(behaviorHash)!

// 用这个行为版本创建 Agent 实例
```

---

## OpenClaw 里的实战做法

OpenClaw 用 `config.patch` 动态更新配置，配合版本追踪：

```bash
# 查看当前 Agent 配置
openclaw config.get

# 灰度更新：先改一个实例测试
openclaw config.patch --path "agent.systemPrompt" --value "新的 prompt 内容"

# 如果出问题，查配置历史
cat ~/.openclaw/config-history.jsonl | tail -5

# 回滚到上一个版本
openclaw config.rollback --steps 1
```

背后的实现（简化版）：

```typescript
// OpenClaw 的配置变更日志
interface ConfigChange {
  timestamp: string
  path: string
  oldValue: unknown
  newValue: unknown
  behaviorHash: string
  operator: string
}

class ConfigHistoryManager {
  private historyFile = path.join(homedir(), '.openclaw/config-history.jsonl')

  async record(change: ConfigChange) {
    const line = JSON.stringify(change) + '\n'
    await fs.appendFile(this.historyFile, line)
  }

  async getRecent(n = 10): Promise<ConfigChange[]> {
    const content = await fs.readFile(this.historyFile, 'utf-8')
    return content
      .trim()
      .split('\n')
      .slice(-n)
      .map(line => JSON.parse(line))
  }

  async rollback(steps = 1): Promise<ConfigChange> {
    const history = await this.getRecent(steps + 1)
    return history[0]  // 最旧的那条就是目标状态
  }
}
```

---

## 指标追踪：知道哪个版本更好

A/B 测试的核心是数据，不是感觉。

```typescript
interface TurnMetrics {
  experimentId: string
  variantLabel: string
  userId: string
  sessionId: string
  
  // 效果指标
  taskCompleted: boolean
  turnsToComplete: number
  toolCallsCount: number
  errorCount: number
  
  // 质量指标（需要另一个 LLM 打分，或人工标注）
  userSatisfactionScore?: number  // 1-5
  responseRelevance?: number      // 0-1
  
  // 效率指标
  totalTokens: number
  latencyMs: number
  costUsd: number
}

class ExperimentTracker {
  private metrics: TurnMetrics[] = []

  record(m: TurnMetrics) {
    this.metrics.push(m)
  }

  // 统计两个变体的对比
  compare(experimentId: string): Record<string, {
    sampleSize: number
    taskSuccessRate: number
    avgTurns: number
    avgCost: number
    avgLatency: number
  }> {
    const grouped = new Map<string, TurnMetrics[]>()
    
    for (const m of this.metrics.filter(m => m.experimentId === experimentId)) {
      const group = grouped.get(m.variantLabel) ?? []
      group.push(m)
      grouped.set(m.variantLabel, group)
    }

    const result: Record<string, any> = {}
    for (const [label, samples] of grouped) {
      result[label] = {
        sampleSize: samples.length,
        taskSuccessRate: samples.filter(s => s.taskCompleted).length / samples.length,
        avgTurns: samples.reduce((sum, s) => sum + s.turnsToComplete, 0) / samples.length,
        avgCost: samples.reduce((sum, s) => sum + s.costUsd, 0) / samples.length,
        avgLatency: samples.reduce((sum, s) => sum + s.latencyMs, 0) / samples.length,
      }
    }
    return result
  }
}

// 输出示例
// {
//   "control": { sampleSize: 500, taskSuccessRate: 0.72, avgTurns: 4.2, avgCost: 0.031 },
//   "treatment": { sampleSize: 498, taskSuccessRate: 0.81, avgTurns: 3.1, avgCost: 0.028 }
// }
// → treatment 胜出，全量推送
```

---

## 渐进式发布（Progressive Rollout）

不要一刀切换，分阶段扩大流量：

```typescript
class ProgressiveRollout {
  private stage = 0
  private stages = [0.01, 0.05, 0.10, 0.25, 0.50, 1.0]  // 1% → 5% → ... → 100%
  
  private experiment: ABExperiment

  constructor(controlHash: string, treatmentHash: string) {
    this.experiment = {
      id: `rollout-${Date.now()}`,
      variants: [
        { behaviorHash: controlHash, weight: 1 - this.stages[0], label: "control" },
        { behaviorHash: treatmentHash, weight: this.stages[0], label: "treatment" }
      ],
      metrics: ["task_success_rate", "error_rate"]
    }
  }

  advance(errorRate: number, successRate: number) {
    // 如果新版本没问题，推进到下一阶段
    if (errorRate < 0.02 && successRate > 0.75) {
      this.stage = Math.min(this.stage + 1, this.stages.length - 1)
      const treatmentWeight = this.stages[this.stage]
      this.experiment.variants[0].weight = 1 - treatmentWeight
      this.experiment.variants[1].weight = treatmentWeight
      console.log(`[Rollout] Advanced to stage ${this.stage}: ${treatmentWeight * 100}% traffic`)
    } else {
      console.log(`[Rollout] ⚠️ Metrics degraded, holding at stage ${this.stage}`)
    }
  }

  abort() {
    // 紧急回滚：把所有流量切回 control
    this.experiment.variants[0].weight = 1.0
    this.experiment.variants[1].weight = 0.0
    console.log(`[Rollout] 🚨 Emergency rollback! All traffic to control.`)
  }
}
```

---

## 一个完整的使用流程

```
1. 注册新版本配置
   registry.register(newBehavior, "v2-concise-prompt")

2. 创建 A/B 实验（10% 先跑）
   router.createExperiment({ variants: [{control, 0.9}, {treatment, 0.1}] })

3. 上线，追踪指标 72 小时
   tracker.record(metrics) // 每次对话后记录

4. 查看对比结果
   const result = tracker.compare("prompt-upgrade-test")

5a. 如果 treatment 更好 → 渐进推全量
    rollout.advance(0.01, 0.82)  // 错误率1%，成功率82%，推进

5b. 如果 treatment 更差 → 回滚
    rollout.abort()
```

---

## 关键原则

1. **行为变更 = 代码变更**：改 prompt 要走 review 流程，不要随手改
2. **先数据后直觉**：感觉新 prompt 更好没用，要有指标说话
3. **小步快跑**：1% → 10% → 50% → 100%，不要直接全量
4. **保留回滚能力**：任何时候都能一键切回上一版本
5. **同一用户一致性**：A/B 分组要稳定，不要同一用户今天 A 明天 B

---

## 小结

| 工具 | 作用 |
|------|------|
| 行为 Hash | 唯一标识一个 Agent 配置版本 |
| 注册表 | 存储所有历史版本，支持回滚 |
| A/B 路由器 | 按用户 ID 稳定分组 |
| 指标追踪器 | 量化两个版本的好坏 |
| 渐进发布器 | 安全地把新版本推全量 |

把这套机制建起来，你改 Agent 行为就跟发布代码一样——有记录、可测试、能回滚。

---

*下一课预告：Agent 知识库更新策略——当外部知识变化时，如何高效更新 RAG 索引而不影响线上服务*
